# Sistema Linux desde Cero para LicheeRV Nano SG2002

## Resumen del Proyecto

Durante este proceso construimos un sistema Linux completo para la placa LicheeRV Nano, compilando cada componente desde el código fuente. Esto nos permitió entender cómo funciona realmente un sistema embebido desde el nivel más bajo hasta el espacio de usuario.

---

## Arquitectura del Sistema

```
┌─────────────────────────────────────────────────────────────┐
│                    TARJETA SD (16GB)                        │
├─────────────────────────────────────────────────────────────┤
│  Partición 1 (512MB, FAT32)  │  Partición 2 (resto, ext4) │
│  - fip.bin                   │  - Sistema de archivos      │
│  - Image (kernel)            │  - /bin, /lib, /usr, etc    │
│  - .dtb (device tree)        │  - Módulos del kernel       │
│  - boot.scr (opcional)       │  - Aplicaciones de usuario  │
└─────────────────────────────────────────────────────────────┘

              ↓ Se inserta en la placa ↓

┌─────────────────────────────────────────────────────────────┐
│              LicheeRV Nano (SG2002 SoC)                     │
│  ┌───────────────────────────────────────────────────────┐  │
│  │ CPU: RISC-V C906 @ 1GHz                              │  │
│  │ RAM: 256MB DDR3                                       │  │
│  │ WiFi: AIC8800 (integrado)                            │  │
│  │ Ethernet: 10/100 Mbps                                │  │
│  │ USB-C: Poder + Datos                                 │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## Cadena de Arranque

El arranque de la placa es una secuencia de varios pasos donde cada componente inicializa el hardware y pasa el control al siguiente:

```
1. ENCENDIDO
   │
   ↓
2. BootROM (en chip)
   - Código inmutable en el SoC
   - Busca fip.bin en la SD
   │
   ↓
3. fip.bin (Firmware Image Package)
   ├─→ FSBL (First Stage Boot Loader)
   │   - Inicializa memoria DDR
   │   - Configuración básica del SoC
   │
   ├─→ OpenSBI (RISC-V Supervisor Binary Interface)
   │   - Se ejecuta en M-Mode (máximo privilegio)
   │   - Proporciona servicios al kernel
   │
   └─→ U-Boot (Universal Boot Loader)
       - Se ejecuta en S-Mode (supervisor)
       - Carga el kernel desde la SD
       - Presenta terminal interactiva
       │
       ↓
4. Kernel Linux (Image)
   - Inicializa hardware usando el device tree
   - Monta el sistema de archivos raíz
   - Carga drivers y módulos
   │
   ↓
5. Sistema de Usuario (rootfs)
   - Init system arranca servicios
   - Login y shell disponibles
```

---

## Componentes Compilados

### 1. Kernel Linux (Image)

El kernel es el núcleo del sistema operativo. Traduce las operaciones de alto nivel en instrucciones que el hardware entiende.

**Características habilitadas:**
- Sistema de archivos /proc (esencial para herramientas del sistema)
- Soporte USB Gadget (NCM, ECM, RNDIS para comunicación USB)
- Drivers de red Ethernet
- Soporte UART para comunicación serial
- ConfigFS para configuración dinámica de USB
- Crypto básico para conexiones seguras

**Archivo resultante:** `Image` (9MB aprox)

**Comando de compilación:**
```bash
cd /workspace
source build/cvisetup.sh
defconfig sg2002_licheervnano_sd
build_kernel
```

---

### 2. Device Tree Blob (.dtb)

El device tree es un mapa de hardware. Le dice al kernel qué componentes tiene la placa, en qué direcciones de memoria están, y cómo conectarlos.

**Contenido del DTB:**
- Direcciones de periféricos (UART, I2C, SPI, USB)
- Mapeo de pines GPIO
- Configuración de reloj y voltaje
- Especificaciones de pantalla MIPI-DSI
- Configuración de cámara ISP

**Archivo resultante:** `sg2002_licheervnano_sd.dtb` (22KB)

El DTB se compila desde archivos fuente .dts (Device Tree Source) que son archivos de texto legibles.

---

### 3. Rootfs (Sistema de Archivos Raíz)

El rootfs contiene todo el software de espacio de usuario: comandos, bibliotecas, configuración.

**Herramienta usada:** Buildroot

**Componentes incluidos:**
- BusyBox (comandos básicos de Linux: ls, cd, cat, etc)
- Python 3 con pyserial y numpy
- wpa_supplicant (para conexiones WiFi)
- dropbear (servidor SSH)
- dhcpcd (cliente DHCP para obtener IP)
- wireless tools (iwconfig, iwlist)
- kmod (para cargar módulos del kernel)

**Tamaño:** 256MB

---

### 4. Módulos del Kernel

Los módulos son partes del kernel que se cargan dinámicamente según se necesiten, en lugar de estar siempre en memoria.

**Módulos compilados:**

```
Crypto:
- aes_generic, gcm, ccm (encriptación)

USB:
- g_ncm, g_serial (USB Gadget para comunicación)
- usb_f_acm, usb_f_mass_storage

Red:
- mac80211, lib80211 (stack WiFi)
- cfg80211 (configuración inalámbrica)

Display:
- drm, drm_kms_helper (Direct Rendering Manager)

Sistema:
- efivarfs (variables de firmware)
```

**Ubicación:** `/lib/modules/5.10.4-tag-/kernel/`

---

## Entorno de Compilación

Todo se compiló dentro de un contenedor Apptainer (antes Singularity) para evitar conflictos con el sistema anfitrión.

```
┌────────────────────────────────────────────────────┐
│         Linux Mint (Sistema Anfitrión)            │
│  ┌──────────────────────────────────────────────┐ │
│  │  Contenedor Apptainer (Ubuntu 22.04)        │ │
│  │  ┌────────────────────────────────────────┐ │ │
│  │  │  Toolchain RISC-V                     │ │ │
│  │  │  - riscv64-unknown-linux-musl-gcc     │ │ │
│  │  │  - Compilador cruzado                 │ │ │
│  │  └────────────────────────────────────────┘ │ │
│  │                                              │ │
│  │  /workspace → ~/LicheeRV-Nano-Build         │ │
│  └──────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────┘
```

**Ventajas del contenedor:**
- Ambiente reproducible
- No contamina el sistema anfitrión
- Todas las dependencias aisladas

---

## Estructura del Proyecto

```
~/LicheeRV-Nano-Build/
├── linux_5.10/                    # Código fuente del kernel
│   ├── drivers/                   # Drivers de hardware
│   ├── arch/riscv/               # Código específico RISC-V
│   └── build/sg2002_licheervnano_sd/
│       ├── .config               # Configuración del kernel
│       └── arch/riscv/boot/
│           ├── Image            # Kernel compilado
│           └── dts/cvitek/*.dtb # Device trees
│
├── buildroot/                    # Sistema de construcción del rootfs
│   ├── package/                 # Definiciones de paquetes
│   └── output/
│       ├── target/              # Rootfs sin empaquetar
│       │   ├── bin/            # Comandos ejecutables
│       │   ├── lib/            # Bibliotecas compartidas
│       │   └── lib/modules/    # Módulos del kernel
│       └── images/
│           ├── rootfs.tar      # Rootfs empaquetado
│           └── rootfs.ext4     # Rootfs como imagen ext4
│
├── opensbi/                     # Supervisor Binary Interface
├── u-boot-2021.10/             # Boot loader
├── host-tools/                  # Compiladores cruzados
└── build/                       # Scripts de construcción
```

---

## Problemas Encontrados y Soluciones

### Problema 1: Kernel sin /proc
**Síntoma:** Comandos como `lsmod`, `ifconfig` fallaban  
**Causa:** CONFIG_PROC_FS no estaba habilitado  
**Solución:** Habilitar en menuconfig o con sed en .config

### Problema 2: USB Gadget "not attached"
**Síntoma:** USB no detectado por la PC  
**Causa:** Controlador USB en modo incorrecto  
**Estado:** Problema de hardware, no resuelto por software

### Problema 3: Drivers WiFi Realtek
**Síntoma:** Errores de undefined reference a funciones USB  
**Causa:** Drivers built-in pero USB como módulo  
**Solución:** Desactivar drivers Realtek, enfocarse en AIC8800

### Problema 4: WiFi AIC8800 faltante
**Síntoma:** Módulo aic8800_fdrv no se compilaba  
**Causa:** Función `cvi_get_wifi_pwr_on_gpio` faltante  
**Solución:** Habilitar CONFIG_CVI_WIFI_PIN

### Problema 5: Error de solapamiento de memoria
**Síntoma:** "FDT image overlaps OS image"  
**Causa:** DTB cargado muy cerca del kernel  
**Solución:** Cambiar dirección de 0x80e00000 a 0x82000000

---

## Comandos de Arranque Manual

En el prompt de U-Boot:

```
setenv bootargs "console=ttyS0,115200 root=/dev/mmcblk0p2 rootwait rw earlycon"
fatload mmc 0:1 0x80200000 Image
fatload mmc 0:1 0x82000000 sg2002_licheervnano_sd.dtb
booti 0x80200000 - 0x82000000
```

**Explicación:**
- `bootargs`: Parámetros para el kernel
- `console=ttyS0,115200`: Terminal serial
- `root=/dev/mmcblk0p2`: Partición del rootfs
- `fatload`: Carga archivos desde FAT32
- `booti`: Arranca kernel Linux en formato Image

---

## Conceptos Aprendidos

### Compilación Cruzada
Compilar código en una arquitectura (x86_64) para ejecutarse en otra (RISC-V). Requiere un compilador cruzado específico.

### Módulos vs Built-in
- **Built-in (=y)**: Compilado dentro del kernel, siempre en memoria
- **Módulo (=m)**: Archivo .ko separado, se carga dinámicamente

### Device Tree
Separación entre código del kernel (genérico) y descripción del hardware (específico de cada placa). Permite un kernel universal.

### Variables de Entorno
En shell: PATH, HOME, etc. Listas de configuración que afectan el comportamiento de programas.

En U-Boot: bootcmd, bootargs, etc. Controlan el proceso de arranque.

### ConfigFS
Sistema de archivos virtual que permite configurar el kernel en tiempo de ejecución, especialmente útil para USB Gadget.

---

## Flujo de Trabajo Típico

```
1. Modificar configuración
   └→ make menuconfig (interfaz gráfica)
   └→ o editar .config directamente

2. Compilar
   └→ build_kernel (para kernel)
   └→ make (para buildroot)

3. Instalar módulos
   └→ make modules_install INSTALL_MOD_PATH=...

4. Empaquetar rootfs
   └→ tar o mke2fs para crear imagen

5. Copiar a SD
   └→ cp para partición boot
   └→ dd para partición rootfs

6. Probar en placa
   └→ Arrancar y diagnosticar
   └→ Si falla: dmesg, lsmod, ifconfig

7. Iterar
   └→ Volver al paso 1
```

---

## Herramientas Clave

- **gcc/ld**: Compilador y linker
- **make**: Sistema de construcción
- **menuconfig**: Configurador interactivo
- **dd**: Copia bit a bit de datos
- **tar**: Empaquetador de archivos
- **mkimage**: Crea imágenes de boot para U-Boot
- **mke2fs**: Crea sistemas de archivos ext2/3/4
- **lsmod/modprobe**: Gestión de módulos
- **dmesg**: Log del kernel
- **grep/find/sed**: Búsqueda y edición de texto

---

## Estado Final

**Funcional:**
- Kernel arranca correctamente
- Sistema /proc montado
- Python 3 disponible
- Herramientas de red instaladas
- Módulos del kernel cargables
- UART funcional

**Pendiente:**
- WiFi AIC8800 (módulo compilado pero sin probar)
- USB Gadget (problema de hardware)
- Arranque automático (boot.scr creado pero no probado)

---

## Próximos Pasos

1. Verificar módulos WiFi disponibles con `find /lib/modules/ -name "*aic*"`
2. Cargar módulo con `modprobe aic8800_fdrv`
3. Verificar interfaz con `ifconfig -a | grep wlan`
4. Configurar WiFi con `wpa_supplicant`
5. Si WiFi no funciona, usar Ethernet (soldar conector RJ45)
6. Implementar arranque automático verificando que boot.scr se ejecute

---

Este documento resume todo el proceso de construcción de un sistema Linux embebido desde cero, mostrando que cada capa del sistema (bootloader, kernel, rootfs) tiene un propósito específico y debe configurarse correctamente para que el sistema funcione como un todo integrado.
