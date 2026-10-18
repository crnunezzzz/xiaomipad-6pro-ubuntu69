# Volver a Android y recuperar la tablet

Guía práctica para la Xiaomi Pad 6 Pro (liuqin) instalada en modo dual. Cada
sección empieza por la situación en la que te encuentras. Los comandos de
`fastboot` se ejecutan en un PC con Android platform-tools y la tablet en modo
fastboot (Vol− + encendido).

## Antes de instalar: lo que debes guardar en el PC

Sin estos archivos, varias recuperaciones no son posibles:

| Archivo | Para qué sirve | De dónde sale |
|---|---|---|
| La ROM fastboot china completa (`liuqin_images_OS2.0.6.0.VMYCNXM_…`) | Reinstalar Android de fábrica | Descarga oficial de Xiaomi |
| El `boot.img` de Android que vas a usar (el de la ROM o `boot-ksu.img`) | Arrancar Android si Ubuntu no arranca | La ROM o `tools/patch-android-boot-ksu.py` |
| El `boot.img` de Ubuntu del paquete de instalación | Arrancar Ubuntu si Android no puede volver | El paquete de la versión instalada |
| El directorio de `--backup` completo | GPT de fábrica, `persist`, `boot_a`/`boot_b` y las particiones de Android sustituidas | Lo crea el instalador |

La partición de arranque mide 192 MiB y `liuqin-switch` identifica su
contenido por la partición completa. Por eso cualquier imagen se graba
**rellenada con ceros** hasta ese tamaño:

```sh
cp boot.img boot-padded.img
truncate -s 201326592 boot-padded.img
```

## 1. Cambiar de sistema en el uso diario

- **De Ubuntu a Android:** abre *Reboot to Android* en el menú de aplicaciones,
  o ejecuta `sudo liuqin-switch to-android`.
- **De Android a Ubuntu:** usa el botón *Reboot to Ubuntu* del módulo KernelSU
  `liuqin_boot_ubuntu`, o ejecuta
  `su -c 'sh /data/adb/modules/liuqin_boot_ubuntu/system/bin/boot-ubuntu'`.
  Sin KernelSU no existe esta opción y necesitas el método de la sección 3.

Antes de cambiar, `sudo liuqin-switch status` muestra qué hay en `boot_a` y
qué hay en el almacén. `sudo liuqin-switch verify` termina con error si algo
no cuadra. Si el cambio falla al escribir, el conmutador restaura lo que había
y **no reinicia**.

## 2. Ubuntu no arranca y quiero Android

Graba la imagen de Android en `boot_a`:

```sh
cp boot-ksu.img boot-padded.img        # o el boot.img de la ROM
truncate -s 201326592 boot-padded.img
fastboot flash boot_a boot-padded.img
fastboot --set-active=a
fastboot reboot
```

Más adelante, desde Ubuntu, `sudo liuqin-switch verify` debe aceptar el estado.
Si te dice que la imagen de Android no está en el almacén, impórtala con
`sudo liuqin-switch import-android boot-ksu.img`.

## 3. Android no puede volver a Ubuntu (sin KernelSU o el botón falla)

```sh
cp ubuntu-boot.img boot-padded.img      # boot.img del paquete de instalación
truncate -s 201326592 boot-padded.img
fastboot flash boot_a boot-padded.img
fastboot --set-active=a
fastboot reboot
```

## 4. La tablet arranca Ubuntu desde el slot B

Si `liuqin-switch` dice *"this boot came from slot B"*, el bootloader no pudo
cargar `boot_a` y usó la copia de respaldo. El conmutador se niega a escribir
porque eso no cambiaría lo que arranca. Repara `boot_a` con los comandos de la
sección 3 (Ubuntu) o de la sección 2 (Android). El `fastboot --set-active=a`
final es imprescindible.

## 5. Una actualización de Android (OTA) se instaló

La OTA escribe el slot B y lo activa, así que se pierde la copia de Ubuntu en
`boot_b`. **No** uses `fastboot --set-active` para volver al slot A sin más,
porque el slot A tiene la cadena de arranque anterior. Hay dos opciones:

- Volver a Android de fábrica (sección 7) y reinstalar después.
- Pedir ayuda en un issue con la salida de `fastboot getvar all`, quitando el
  número de serie.

Para evitarlo: desactiva las actualizaciones automáticas en Ajustes y usa
Android con el módulo KernelSU, que deshabilita el actualizador en cada arranque.

## 6. Un kernel nuevo de Ubuntu no funciona

Para instalar una imagen nueva sin reinstalar el sistema:

```sh
sudo liuqin-switch import-ubuntu /ruta/al/boot-nuevo.img
sudo liuqin-switch to-ubuntu
```

La imagen anterior queda en `/var/lib/liuqin/switch/ubuntu.prev/`. Para volver
a ella desde Ubuntu:

```sh
sudo liuqin-switch import-ubuntu /var/lib/liuqin/switch/ubuntu.prev/boot.img
sudo liuqin-switch to-ubuntu
```

Si Ubuntu ya no arranca, graba la imagen anterior desde el PC (sección 3).
`boot_b` sigue teniendo la imagen que puso el instalador.

## 7. Quitar Ubuntu y dejar Android de fábrica

1. **Restaura la tabla de particiones de fábrica.** El instalador solo acepta
   la copia tomada de esta misma tablet:
   ```sh
   python3 install.py --bundle . --serial NUMERO_DE_SERIE \
     --restore-partition-table /ruta/a/tu-backup
   ```
2. **Flashea la ROM completa** desde su directorio con `flash_all.sh`.
   **Nunca** uses `flash_all_lock.sh`. Comprueba antes que la versión
   anti-rollback de la ROM sea mayor o igual que la de la tablet
   (`fastboot getvar anti`).
3. El primer arranque de Android tarda varios minutos, porque formatea
   `userdata`.
4. **No vuelvas a bloquear el bootloader** hasta confirmar que Android arranca
   con la ROM oficial completa. Bloquearlo con imágenes no oficiales deja la
   tablet sin arrancar.

`persist` (calibración, MAC) nunca lo escribe el instalador. No restaures el
`persist` de otra tablet.

## 8. Restaurar una partición concreta desde el backup

El instalador guarda, antes de sustituirlas, todas las particiones de Android
que reemplaza: `modem_a`, `bluetooth_a`, `dsp_a`, `vendor_boot_a`, `dtbo_a`,
`vbmeta_a` y `vbmeta_system_a`. La excepción es `super`, que se recupera con la
ROM. Primero verifica las sumas y luego restaura:

```sh
cd /ruta/a/tu-backup && sha256sum -c SHA256SUMS
fastboot flash vendor_boot_a vendor_boot_a.img   # o modem_a, dtbo_a, …
```

## Qué está probado

La lógica del conmutador y del instalador está cubierta por pruebas en el PC
(`tests/`). Los cambios de sistema y las recuperaciones de esta guía todavía no
se han ejecutado en una tablet real. Haz la primera prueba con el PC preparado
y los archivos de la tabla inicial a mano.
