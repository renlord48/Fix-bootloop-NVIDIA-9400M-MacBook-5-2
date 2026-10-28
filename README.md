# Fix para bootloop/pantalla negra del driver NVIDIA 9400M en Windows 10 (MacBook5,2 / 2009, modo UEFI)

## Problema

Al instalar Windows 10 en una **MacBook5,2 (finales 2008 / principios 2009)** usando un pendrive grabado en modo UEFI/GPT (por ejemplo con Rufus en modo DD), la instalación de Windows funciona perfectamente. El problema aparece al instalar el **driver oficial de la GPU NVIDIA GeForce 9400M**: el sistema entra en bucle de reinicio o pantalla negra. Windows suele detectar el fallo automáticamente y, tras un par de reinicios, elimina el driver por sí solo, volviendo a iniciar pero sin aceleración gráfica.

## Causa raíz

El firmware EFI de Apple de esa generación **no inicializa el espacio de compatibilidad VGA** en los registros PCI de la GPU cuando Windows arranca en modo UEFI nativo. El driver de NVIDIA, al cargar, no encuentra ese espacio habilitado y cuelga el sistema.

Este setup (Windows en EFI nativo en una Mac pre-2012) nunca fue soportado oficialmente por Apple vía Boot Camp — Boot Camp para estos modelos asume arranque BIOS/CSM (legacy), donde este problema no ocurre. Si tu Mac permite instalar en modo Legacy/MBR, esa es la vía más simple y estable. Esta guía es para quienes, como en mi caso, **solo logran instalar en modo UEFI** (el pendrive en MBR no es reconocido como arrancable por el firmware).

## Solución

Interceptar el arranque con **rEFInd**, lanzar un **EFI Shell completo** y correr un script (`startup.nsh`) que modifica manualmente los registros PCI correctos para habilitar el espacio VGA antes de que cargue `bootmgfw.efi`.

⚠️ **Importante:** las direcciones PCI exactas (bus/dispositivo/función) **varían según la revisión exacta de placa**, incluso dentro del mismo modelo de Mac. No asumas que las direcciones de este README van a funcionar igual en tu unidad — la sección de diagnóstico más abajo te muestra cómo encontrar las tuyas.

---

## 0. Requisitos previos

- Windows 10 ya instalado en modo UEFI (GPT) en la MacBook, arrancando normalmente sin el driver de GPU.
- Acceso a CMD como Administrador.
- Un pendrive adicional o conexión a internet desde la propia Mac para descargar las herramientas.

---

## 1. Descargar las herramientas necesarias

| Herramienta | Para qué sirve | Descarga |
|---|---|---|
| **rEFInd** (binario, última versión estable) | Gestor de arranque que permite interceptar el proceso antes de Windows | https://www.rodsbooks.com/refind/getting.html (sección "Binary Zip File") |
| **EFI Shell completo (Shell_Full.efi)** | El shell EFI básico no trae el comando `pci`; este sí | https://github.com/tianocore/edk2-archive/blob/master/EdkShellBinPkg/FullShell/X64/Shell_Full.efi (botón "Download raw file") |
| **Driver NVIDIA legacy** | Rama estable recomendada para la 9400M en Windows 10 | Busca manualmente la versión **341.74 / 341.81 WHQL** en https://www.nvidia.com/Download/Find.aspx (filtrando por GeForce 9400M). Los drivers más nuevos no soportan esta GPU. |

---

## 2. Instalar rEFInd desde Windows

Abre **CMD como Administrador**:

```cmd
:: 1. Montar la partición EFI con la letra R:
mountvol R: /S

:: 2. Copiar los archivos de rEFInd (descomprimidos) a la partición EFI
xcopy /E refind R:\EFI\refind\

:: 3. Activar el archivo de configuración base
R:
cd EFI\refind
rename refind.conf-sample refind.conf

:: 4. Registrar rEFInd como gestor de arranque por defecto en el BCD de Windows
bcdedit /set "{bootmgr}" path \EFI\refind\refind_x64.efi
bcdedit /set "{bootmgr}" description "rEFInd Boot Manager"
```

## 3. Agregar el EFI Shell completo

```cmd
:: Crear carpeta de herramientas si no existe
mkdir R:\EFI\tools

:: Copiar el Shell_Full.efi descargado (ajusta la ruta de origen)
copy "%USERPROFILE%\Downloads\Shell_Full.efi" R:\EFI\tools\shell.efi
```

rEFInd detecta automáticamente los `.efi` dentro de `EFI\tools\` y los agrega como opción en su menú.

## 4. Preparar el arranque en modo "manual" para diagnosticar

Antes de automatizar nada, necesitas entrar al shell con un prompt interactivo (`Shell>`), no que se auto-ejecute nada.

Edita `R:\EFI\refind\refind.conf` con notepad y asegúrate de que **no** tenga:
```
timeout -1
default_selection shell
```
Déjalo en algo como:
```
timeout 20
```
(sin `default_selection shell`, o comentado con `#` adelante).

Reinicia:
```cmd
mountvol R: /D
shutdown /r /t 0
```

Al reiniciar verás el menú gráfico de rEFInd. Elige la opción **Shell**.

## 5. Diagnosticar tu hardware específico

Ya en el prompt `Shell>`:

```
pci -b
```

(la salida es larga, se pagina — presiona Enter para avanzar página por página).

Busca dos cosas:

1. La línea **"Display Controller - VGA/8514 controller"** → es tu GPU. Anota su `Seg Bus Dev Func` (ej: `00 02 00 00`).
2. Todas las líneas **"Bridge Device - PCI/PCI bridge"** en Bus `00` → puede haber varias (en mi caso había 4: Dev 09, 10, 15 y 16). Anótalas todas.

### 5.1 Verificar el registro de la GPU

```
pci -i -b <bus> <dev> <func>
```
(usa los valores de tu VGA, ej: `pci -i -b 02 00 00`)

Fíjate en **Command (4)**. Si **"I/O space access enabled"** está en `0`, ese es el problema.

### 5.2 Habilitar el registro de la GPU (en vivo, para probar)

```
mm -PCI <dirección en formato bbddffrr>
```
Ejemplo para bus 02, dev 00, func 00, registro 04:
```
mm -PCI 0002000004
```
Te muestra el valor actual (normalmente `0x02`). Escribe el nuevo valor sumando el bit de I/O (si Memory ya está en 1, el nuevo valor suele ser `3`), presiona Enter, y en el siguiente prompt escribe `.` + Enter para salir (si se traba, apaga forzado con el botón de encendido — el cambio es volátil, no rompe nada).

### 5.3 Identificar y habilitar el puente PCI-PCI correcto

No asumas cuál de los puentes detectados es el correcto — pruébalos con:
```
pci -i -b 00 <dev> 00
```
Busca el registro **Bridge Control (3E)** y el bit **"VGA Enable"**. Cuando encuentres el puente cuyo "VGA Enable" está en 0 y corresponde a la ruta hacia tu GPU (no hacia red/otros buses), habilítalo:
```
mm -PCI 00<dev>003E
```
Escribe `8` (activa el bit VGA Enable) y confirma.

### 5.4 Probar el arranque manual

Con ambos registros modificados, sin reiniciar:
```
map
```
(identifica qué `fsX:` corresponde a tu partición de Windows — busca la que tenga `\EFI\Microsoft\Boot\`)

```
fs0:\EFI\Microsoft\Boot\bootmgfw.efi
```
(ajusta el número de `fs` según tu caso)

Si Windows arranca bien manualmente con estos valores, están confirmados. Si no, repite el diagnóstico con otro puente candidato.

## 6. Automatizar con `startup.nsh`

Una vez confirmadas tus direcciones, desde Windows (CMD como Administrador):

```cmd
mountvol R: /S
notepad R:\startup.nsh
```

Contenido (reemplaza con **tus** direcciones confirmadas):
```
mm <dirección del puente, ej 0015003E> 1 ;PCI :8
mm <dirección de la GPU, ej 02000004> 1 ;PCI :3
fs0:\EFI\Microsoft\Boot\bootmgfw.efi
```

Guarda. Luego restaura el arranque automático en `R:\EFI\refind\refind.conf`:
```
timeout -1
default_selection shell
```

Desmonta:
```cmd
mountvol R: /D
```

## 7. Instalar el driver NVIDIA

Instala la versión **341.74 / 341.81 WHQL** descargada en el paso 1. Cuando pida reiniciar, déjalo — rEFInd lanzará el Shell automáticamente, aplicará el script, y arrancará Windows con la GPU ya habilitada.

## 8. Verificar

- Administrador de dispositivos → la GPU no debe mostrar triángulo amarillo ni código de error.
- Reinicia una vez más (normal) para confirmar que el flujo se repite consistentemente, no que fue casualidad.

## 9. Opcional — punto de restauración

Antes y después del fix, desde PowerShell como Administrador:
```powershell
Enable-ComputerRestore -Drive "C:\"
Checkpoint-Computer -Description "GPU NVIDIA funcionando OK post-fix PCI" -RestorePointType MODIFY_SETTINGS
```

---

## Notas

- Esto es un workaround, no una solución "oficial": depende de que rEFInd y el script corran en cada arranque. Si algún día actualizas o reemplazas rEFInd, revisa que `startup.nsh` y la configuración sigan intactos.
- Si tu Mac sí logra bootear un instalador de Windows en modo Legacy/MBR, esa vía evita todo este proceso porque el VBIOS legacy se inicializa normalmente.
- Las direcciones PCI de este README corresponden a mi unidad específica; úsalas solo como referencia, siempre verifica con `pci -b` en la tuya.

## Créditos

Basado en el hilo de Super User: *"Graphics card not working on Windows 8.1 on Mac"* y respuestas de la comunidad sobre MacBook5,2 / MacBookPro5,5 / MacBook6,1.
