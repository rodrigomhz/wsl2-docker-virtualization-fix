# WSL2 + Docker + Virtualización en ASUS TUF FX505DT (Ryzen 7 3750H)  
**Diagnóstico y resolución de un fallo real con pantallazo negro al activar WSL2**

Este documento recoge todo el proceso que seguí para solucionar un problema bastante serio al intentar usar **WSL2 + Ubuntu + Docker** en un portátil **ASUS TUF FX505DT (Ryzen 7 3750H)** con **Windows 11 (build 26100 / 24H2)**.

El objetivo inicial era simple:  
> Tener un entorno moderno de desarrollo y ciberseguridad con WSL2, Ubuntu y Docker integrado.

Lo que parecía una instalación estándar terminó en:

- Pantallas negras al reiniciar tras activar virtualización.
- WSL1 no soportado en mi build de Windows.
- WSL2 inutilizable por un conflicto entre BIOS antigua y el hypervisor.
- Bastante tiempo de diagnóstico hasta llegar a la causa raíz.

Este README documenta el proceso completo, los comandos utilizados, las dudas que fui resolviendo y lo aprendido por el camino.

---

## 🔧 Entorno

- **Portátil**: ASUS TUF Gaming FX505DT  
- **CPU**: AMD Ryzen 7 3750H (soporta virtualización AMD-V / SVM)  
- **Sistema operativo**: Windows 11 (versión 10.0.26100, 24H2)  
- **Objetivo**:  
  - WSL2 habilitado  
  - Ubuntu como distro principal  
  - Docker Desktop integrado con WSL2  
  - Docker usable desde Ubuntu sin sudo

---

## 1. El problema inicial

Quería instalar WSL2 con:

```powershell
wsl --install -d Ubuntu
```
Sin embargo, al habilitar las características necesarias para WSL2:
```
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
```
Al reiniciar, el portátil se quedaba en pantalla negra, sin llegar al logo de Windows.

La única forma de recuperar el sistema era entrar en Modo Seguro y desactivar las características:

```
dism.exe /online /disable-feature /featurename:VirtualMachinePlatform /norestart
dism.exe /online /disable-feature /featurename:Microsoft-Windows-Subsystem-Linux /norestart

```

## 2. Intento de usar WSL1 (y por qué no funcionó)
Probé:
```
wsl --set-default-version 1
```
Resultado:
```
WSL1 no es compatible con la configuración actual del equipo.
Código de error: Wsl/WSL_E_WSL1_NOT_SUPPORTED
```
En esta build de Windows 11 WSL1 ya no está soportado, así que no era opción.

## 3. Verificaciones de virtualización

### ✔ Virtualización en Windows

  En el Administrador de tareas → Habilitada.

### ✔ Estado de características con DISM
```
DISM /Online /Get-Features /Format:Table
```
Datos relevantes:

  - Microsoft-Windows-Subsystem-Linux → Habilitado

  - VirtualMachinePlatform → Deshabilitado

  - HypervisorPlatform → Deshabilitado

### ✔ BIOS

  - Comprobé lo básico:

    - SVM Mode → Enabled

    - No había opciones avanzadas visibles como Secure Boot o fTPM.

  Todo apuntaba a que Windows sí soportaba virtualización, pero la BIOS podía estar causando el conflicto.

## 4. Diagnóstico técnico

### La evidencia indicaba:

  - La CPU soporta virtualización.

  - Windows también.

  - SVM estaba activado.

El problema aparecía solo al activar VirtualMachinePlatform, lo que dispara el hypervisor de Windows.

La pantalla negra antes del logo indica un fallo previo al arranque de Windows → BIOS / Firmware.

🟡 Conclusión:

La BIOS antigua tenía problemas al cargar el hypervisor moderno de Windows 11.

Había que actualizar la BIOS.

## 5. Actualización de BIOS (ASUS EZ Flash 3)

### 5.1. Descarga

  Desde la página oficial de ASUS (FX505DT), sección BIOS & Firmware:

  Archivo descargado: FX505DT-AS.316.CAP

### 5.2. Preparación del USB

  USB formateado en FAT32.

  El archivo .CAP copiado dentro.

### 5.3. Proceso en BIOS

  1. Entré a BIOS con F2.

  2. F7 para Advanced Mode.

  3. Navegué a:
  ```
  Advanced → ASUS EZ Flash 3 Utility
  ```
  4. Seleccioné el USB (fs0, tamaño coincidente).

  5. Seleccioné el archivo .CAP.

  6. Acepté el flasheo.

### 5.4. Pantalla negra post-flasheo (normal)

  Tras completarse, la pantalla se quedó negra durante ~1 minuto.
  Esto es normal: la BIOS reconfigura NVRAM y microcódigo.
  
  Realicé un apagado forzado (10 segundos) y encendí el portátil.
  Todo arrancó correctamente con la BIOS nueva.

### 5.5. Rehabilitar SVM

  Tras toda actualización de BIOS:

  - SVM Mode vuelve a Disabled

  - Lo reactivé manualmente

  - Guardé con F10

## 6. Reactivar WSL2 sin romper el sistema

Con BIOS nueva y SVM activado:

  ### ✔ Activé VirtualMachinePlatform
  ```
  dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
  ```
Reinicié → ya no hubo pantalla negra.

### ✔ Puse WSL2 como default
```
wsl --set-default-version 2
```
### ✔ Instalé Ubuntu
```
wsl --install -d Ubuntu
```
### ✔ Comprobación
```
wsl -l -v
```
```
Ubuntu      Running      2
```

Todo funcionando.

## 7. Instalación y configuración de Docker

### ✔ Docker Desktop instalado

**Configuración clave:**

  - Settings → Resources → WSL Integration

**Activar:**

  - Enable WSL Integration

  - Ubuntu (On)

### ✔ Comprobación en Ubuntu
````
docker --version
````
### ✔ Error inicial esperado
````
permission denied while trying to connect to the Docker daemon socket
````
### ✔ Solución
````
sudo usermod -aG docker $USER
newgrp docker
````
### ✔ Prueba final
````
docker run hello-world
````
Respuesta (Muestra de un correcto funcionamiento)
````
Hello from Docker!
````
Docker funcionando correctamente dentro de WSL2 sin sudo.

## 8.Lecciones aprendidas

### 1.

  - WSL2 depende directamente de la BIOS.
  - Una BIOS desactualizada puede impedir arrancar el hypervisor.
    
### 2.
  - Pantalla negra tras activar VirtualMachinePlatform = problema de firmware, no de Windows.
  - WSL1 ya no es compatible con builds nuevas.
    
### 3.

Docker en WSL2 requiere:

  - Integración desde Docker Desktop

  - Permisos del grupo docker

Cuando se rompe algo real es cuando más aprendes.

## 10. Estado final del sistema

- BIOS FX505DT versión 316

- SVM Enabled

- VirtualMachinePlatform Enabled

- WSL2 funcionando

- Ubuntu instalado

- Docker Desktop integrado

- Docker funcionando en WSL2

# 💬 Conclusión

**Este proceso demuestra que muchos fallos de virtualización no vienen de Windows, sino del firmware**
**Actualizar la BIOS fue la clave para que WSL2, Docker y Ubuntu funcionaran correctamente.**

# **Guardo este repositorio como referencia completa para cualquiera que se encuentre con un problema similar.**
