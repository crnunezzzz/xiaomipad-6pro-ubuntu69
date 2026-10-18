# Auditoría técnica: Ubuntu en Xiaomi Pad 6 Pro (liuqin)

Fecha de revisión: 2026-10-17. Base auditada: `main` en `f5a535f` (v0.6.0).
Alcance: instalador, motor de particiones, initramfs, conmutador de arranque
dual, módulo KernelSU, overlay de Ubuntu, configuración del kernel, CI y
documentación. El repositorio del kernel (`linux-sm8450-liuqin`) no estaba
disponible en esta sesión: lo que se dice del kernel sale de `kernel/source.json`,
de los fragmentos de `device/configs/` y de los argumentos de arranque.

Contexto del equipo del usuario: 256 GB, ROM china (HyperOS), estado del
bootloader sin verificar.

## Resumen ejecutivo

El proyecto está bien diseñado para un port experimental. Verifica sumas antes
de tocar el dispositivo, pide confirmación interactiva, guarda copias de la
GPT, de `boot_a`, `boot_b` y `persist`, y protege el disco con banderas de solo
lectura. Además, el conmutador `liuqin-switch` comprueba cada escritura
leyéndola de vuelta y tiene 210 pruebas en el host. Los riesgos que quedan no
vienen de errores de programación, sino de **supuestos sobre el estado del
dispositivo** que el instalador no comprobaba y de **funciones anunciadas que
nadie ha probado en hardware**.

| # | Hallazgo principal | Gravedad | Estado |
|---|---|---|---|
| S1 | No se comprobaba que el slot A estuviera activo, aunque la documentación lo exige | Alta | **Corregido** en esta rama |
| S2 | `anti_version` de la ROM estaba fijado pero nunca se comparaba con el dispositivo | Alta | **Corregido** (cuando el ABL informa `anti`) |
| S3 | El modo dual instala imágenes de **una única ROM china exacta** (OS2.0.6.0.VMYCNXM) junto a una cadena de arranque que puede ser más nueva | Alta | Pendiente: requiere un procedimiento y hardware |
| D1 | Sin KernelSU no hay vuelta a Ubuntu desde Android ni bloqueo de OTA; HyperOS CN puede actualizarse sola y destruir la copia de respaldo en `boot_b` | Alta | El instalador ahora **avisa**; guía en `RECUPERACION.es.md`; falta validación en hardware |
| U1 | No existe una vía para actualizar el kernel sin reinstalar. Reinstalar con `--keep-home` borra las cuentas y las huellas | Alta (mantenimiento) | Propuesta de diseño |
| S4 | Las comprobaciones de batería, RAM y `persist` se hacían **después** de reparticionar | Media | **Corregido** (verbo `preflight`) |
| K1 | El kernel se basa en 6.17, que no es LTS y ya no recibe parches de seguridad | Media-Alta | Recomendación: rebase a 6.18 LTS |
| K2 | Los argumentos de arranque de producción son de depuración (`clk_ignore_unused`, `pd_ignore_unused`, `initcall_debug`, `panic=0`) | Media | Recomendación |

### Corregido en esta rama

- **S1, S2:** el instalador se niega a continuar si el slot activo no es A o si
  el anti-rollback de la tablet es mayor que el de la ROM fijada.
- **S4:** un verbo `preflight` del script del dispositivo comprueba la batería,
  la RAM y los datos de fábrica de `persist` antes de cualquier copia o
  escritura.
- **S5:** el mensaje de error ya no dice que restaura la tabla.
- **S6:** cada partición que recibe una imagen de la ROM (excepto `super`) se
  respalda y verifica antes de sobrescribirla.
- **D1:** aviso explícito si el modo dual se instala sin imagen KernelSU.
- **D3:** `liuqin-switch import-ubuntu` permite instalar una imagen nueva de
  Ubuntu sin reinstalar, conservando la anterior en `store/ubuntu.prev`.
- **Reversión exacta:** si una escritura de `boot_a` falla, se restaura byte a
  byte lo que había, no la imagen del otro sistema.
- **S10:** la CI ejecuta las pruebas del conmutador, del almacén y de los slots.
- **Guía de recuperación:** [`RECUPERACION.es.md`](RECUPERACION.es.md).

Sigue pendiente lo que necesita hardware: S3, D2, D4, K1 y K2.

## 0. Antes de tocar tu tablet: lista de verificación

Haz esto antes de ejecutar el instalador. Nada de lo siguiente borra datos.

1. **Versión de Android.** En *Ajustes → Acerca de la tablet*, anota la versión
   exacta de HyperOS. El modo dual solo admite los archivos de
   `OS2.0.6.0.VMYCNXM` (China, Android 15), según `tools/lib/liuqin-rom-images.json`.
2. **Desactiva las actualizaciones automáticas** de HyperOS ahora mismo, porque
   una OTA cambia el slot activo (ver D1).
3. **Estado del bootloader.** En modo fastboot ejecuta:
   ```sh
   fastboot getvar product        # debe responder liuqin
   fastboot getvar unlocked       # yes / no
   fastboot getvar current-slot   # debe ser a
   fastboot getvar anti           # índice anti-rollback; para el modo dual debe ser ≤ 1
   ```
   En ROM china con HyperOS, Xiaomi pone requisitos de cuenta para desbloquear
   (Xiaomi Community, plazos de espera y cupos). Revisa las condiciones vigentes
   antes de contar con el desbloqueo. Desbloquear **borra todos los datos**.
4. **Si `current-slot` es `b`**, no instales. Flashea la ROM oficial completa con
   `flash_all.sh`, nunca con `flash_all_lock.sh`, y solo si su anti-rollback es
   mayor o igual al de la tablet. Así ambos slots quedan coherentes y el slot A
   vuelve a estar activo. Después de este cambio, el instalador también se niega
   a continuar en este caso.
5. **Si tu versión de HyperOS es más nueva que OS2.0.6.0** (lo normal en
   octubre de 2026), lee S3 antes de elegir el modo dual.
6. Guarda copias de tus archivos fuera de la tablet y descarga la ROM fastboot
   china correspondiente a tu equipo.
7. Guarda el directorio de `--backup` en dos sitios, idealmente cifrado. Contiene
   `persist`, con la calibración, las MAC y otros datos únicos de **tu** tablet
   que no se pueden recuperar de otra.

## 1. Seguridad del instalador

Leyenda de validación: **Auto** significa que se puede probar en el host o en
la CI sin tablet. **HW** significa que requiere la tablet real.

| ID | Problema | Gravedad | Dificultad | Riesgo del cambio | Validación |
|---|---|---|---|---|---|
| S1 | No se comprobaba `current-slot` | Alta | Baja | Bajo | Auto ✅ |
| S2 | No se comprobaba `anti` | Alta | Baja | Bajo | Auto ✅ |
| S3 | La ROM fijada se mezcla con la cadena de arranque instalada | Alta | Media | Medio | HW |
| S4 | Las comprobaciones previas se hacían tras reparticionar | Media | Baja | Bajo | Auto |
| S5 | Mensaje falso de "restaurando la tabla" | Media | Baja | Nulo | Auto ✅ |
| S6 | Sin copia de `vendor_boot_a`, `dtbo_a`, `vbmeta_a` ni `vbmeta_system_a` | Media | Baja | Bajo | Auto + HW |
| S7 | Copias de seguridad sensibles guardadas sin cifrar | Media | Baja | Nulo | Documentación |
| S8 | Shell de rescate sin autenticación en la fase 1 | Baja | Media | Medio | Auto + HW |
| S9 | `/dev/sda` (disco completo) queda en lectura y escritura dentro de Ubuntu | Baja | Alta | Medio | HW |
| S10 | La CI no ejecutaba las pruebas del conmutador ni de los slots | Media | Baja | Nulo | Auto ✅ |
| S11 | El modo Linux-only nunca se ha instalado en hardware | Media | — | — | HW |

**S1. Slot activo (corregido).** `docs/FLASHING.md` exige que el slot A esté
activo, pero `tools/install-liuqin.py` nunca lo consultaba. Al final ejecuta
`fastboot --set-active=a` (modo dual) o `=b` (Linux-only). Si una OTA dejó la
tablet en el slot B, el slot A contiene el xbl, abl y TrustZone de la versión
**anterior**. Activarlo combina esa cadena antigua con el firmware más nuevo que
escribe el instalador. En el peor caso, arranca un xbl más viejo que el fusible
anti-rollback, y eso suele requerir EDL con cuenta autorizada para recuperarse.
Ahora `check_bootloader_state()` se niega a continuar si `current-slot` no es `a`.

**S2. Anti-rollback (corregido).** `liuqin-rom-images.json` declara
`"anti_version": 1`, pero ningún código lo leía. Ahora, en modo dual, el
instalador rechaza la tablet si `fastboot getvar anti` informa un índice mayor.
Si el ABL no informa la variable, solo muestra una advertencia.

**S3. ROM fijada frente a firmware instalado (pendiente).** El modo dual
escribe `super`, `vendor_boot_a`, `dtbo_a`, `vbmeta*_a`, `modem_a`,
`bluetooth_a` y `dsp_a` de OS2.0.6.0.VMYCNXM. **No** toca xbl, abl, tz, hyp,
devcfg, keymaster ni las demás particiones de firmware. Si la tablet trae
HyperOS más nuevo, Android arrancará con su sistema y su módem de 2.0.6.0 sobre
una TrustZone y un keymaster más recientes. Eso puede producir bucles de
arranque o fallos de Keymint o FBE en Android. Ubuntu no se ve afectado, porque
lleva su propio firmware en `/usr/lib/firmware`. Propuestas:

- Documentar como paso obligatorio flashear antes la ROM completa OS2.0.6.0
  (con `flash_all.sh`, si el anti-rollback lo permite). Así todo el firmware
  queda coherente.
- Alternativa más flexible: admitir varias ROM en `liuqin-rom-images.json`,
  una entrada por versión validada.
- No se puede leer la versión de HyperOS desde fastboot. Se puede pedir al
  usuario que la escriba (`--expect-rom`) y compararla con la tabla.

**S4. Orden de las comprobaciones (parcialmente corregido).** La secuencia de
`main()` era: copias de seguridad → borrado de la cabecera de `userdata` y
`metadata` → nueva GPT → **y después** `install-root.sh`, que comprueba la
batería (≥ 30 %), la RAM libre y la presencia de `wlan_mac.bin`, `.bt_nv.bin`,
`crus_calr.bin` y del registro de sensores en `persist`. Si cualquiera fallaba,
la tablet quedaba reparticionada y sin sistema. Ahora la batería se comprueba
en la RAM antes de cualquier escritura (`check_battery`). Falta mover la
comprobación de `persist` y la de RAM a un verbo `preflight` de
`install-root.sh` que se ejecute antes de `apply_layout()`.

**S5. Mensaje engañoso (corregido).** `apply_layout()` imprimía *"restoring the
saved table"*, pero no restauraba nada: solo lanzaba el error. El mensaje ahora
dice la verdad. Una restauración automática opcional, reutilizando
`restore_partition_table`, es una mejora razonable con pruebas en el host.

**S6. Copias incompletas en modo dual.** Antes de sobrescribirlas se respaldan
`modem_a`, `bluetooth_a` y `dsp_a`, pero no `vendor_boot_a` (96 MiB),
`dtbo_a` (24 MiB) ni `vbmeta_a` y `vbmeta_system_a` (12 KiB). Son pequeñas y
respaldarlas es barato. `super` (7 GiB) puede seguir sin copia.

**S7. Copias sensibles.** El directorio de `--backup` tiene permisos 0700 y
archivos 0600, lo cual es correcto, pero los datos no van cifrados.
Recomendación: documentar el cifrado (por ejemplo `age` o un volumen LUKS) y
una segunda copia, y verificar `SHA256SUMS` al restaurar.

**S8. Shell de rescate.** El telnet de rescate en Ubuntu es opcional y está
desactivado por defecto, lo cual es correcto. En cambio, la shell RAM de la
fase 1, que se abre si falla el contrato de la raíz, da root sin
autenticación a cualquiera conectado por USB. Esto exige acceso físico, pero
una clave SSH (dropbear) generada en la instalación y guardada en el host lo
cerraría.

**S9. Disco completo escribible.** `liuqin-gnome-storage-guard` deja
`/dev/sda` en lectura y escritura porque la raíz vive en esa LUN. La bandera de
solo lectura por partición no protege contra escrituras al disco completo, así
que un `dd` erróneo como root puede borrar la GPT o el `userdata` de Android.
El kernel no permite otra cosa sin mover la raíz. Basta con documentarlo.

**S10. CI (corregido).** `.github/workflows/checks.yml` no ejecutaba
`tests/switch-selftest.sh` (210 casos), `tests/install-switch-store.sh` ni
`tests/slot-tool.py`, aunque todos pasan en local. Ya están añadidos.

## 2. Arranque dual Android/Ubuntu

Mecanismo actual: ambos sistemas arrancan desde el slot A. `liuqin-switch`
escribe la imagen del otro sistema en `boot_a`, la verifica leyéndola de
vuelta y reinicia. `boot_b` guarda una copia de Ubuntu que el ABL usa solo si
`boot_a` no carga. Desde Android, el regreso depende de un módulo KernelSU.

| ID | Problema u oportunidad | Gravedad | Dificultad | Riesgo | Validación |
|---|---|---|---|---|---|
| D1 | Sin KernelSU no hay regreso a Ubuntu ni bloqueo de OTA | Alta | Baja (proceso) | Bajo | HW |
| D2 | Sin salida de emergencia sin PC si Ubuntu no arranca | Alta | Media | Medio | Auto + HW |
| D3 | El almacén de Ubuntu queda congelado: no existe `import-ubuntu` | Media | Media | Bajo | Auto (`--dev-dir`) |
| D4 | El slot se marca como *successful* antes de que arranque el espacio de usuario | Media | Media | Medio | HW |
| D5 | Ninguna dirección del cambio está validada en hardware | Media | — | — | HW |
| D6 | Experiencia de uso: no hay selector al arrancar | Baja | Media | Bajo | HW |
| D7 | Investigación: U-Boot en `boot_a` del lado Ubuntu | — | Alta | Alto | HW |

**D1.** Si instalas en modo dual sin `--android-boot`, Android arranca sin
root: no hay botón "Reboot to Ubuntu" ni `service.sh` que congele las OTA.
HyperOS china descarga e instala actualizaciones de forma agresiva. Una OTA
escribe el slot B, que contiene la copia de Ubuntu, y lo activa. Según la
propia documentación, Android no arranca desde B. El resultado es que
necesitas un PC con fastboot para recuperar la tablet. Recomendaciones:

1. Usar siempre `--android-boot` con la imagen KernelSU, generada con
   `tools/patch-android-boot-ksu.py`.
2. Desactivar las actualizaciones en Ajustes, además del módulo.
3. Validar en hardware que KernelSU se activa en un arranque normal. Hoy figura
   como no verificado.

**D2. Salida de emergencia sin PC (propuesta).** El initramfs de Ubuntu ya
lee GPIO y las teclas, y tiene acceso a `linux_root`. Se propone:

- Si se mantiene pulsado **Vol−** durante el arranque, o si el contrato de la
  raíz falla, mostrar un mensaje en pantalla (hoy solo existe la shell USB).
- Ofrecer "volver a Android", que reutilice exactamente las funciones de
  `liuqin-switch` (`check_set`, `install_set` y `rollback`) para escribir
  `store/android` en `boot_a` y reiniciar.

Esto elimina la dependencia del PC en el caso más común: un Ubuntu que no
arranca. La lógica se puede probar con `--dev-dir`. La lectura de teclas y la
pantalla requieren hardware.

**D3. Almacén de Ubuntu congelado.** `store/ubuntu/boot.img` se escribe solo
al instalar. Con un kernel nuevo, `to-ubuntu` reescribiría la imagen
**antigua**, `verify` fallaría y `boot_b` quedaría desfasado. Se propone
`liuqin-switch update-ubuntu IMG`, con este flujo:

1. Comprobar la cabecera y la suma (reglas existentes).
2. Mover la imagen actual a `store/ubuntu.prev`.
3. Escribir `boot_a` con lectura de vuelta.
4. **No** tocar `boot_b` hasta que el nuevo arranque se confirme (ver D4).
5. Promover entonces la imagen a `boot_b`.

**D4. Marcar el slot como successful demasiado pronto.** `initramfs/init`
marca el slot antes del `switch_root`. Por eso el contador de reintentos del
ABL nunca puede deshacer un kernel que carga pero cuyo espacio de usuario
falla. Propuesta:

- Marcar el slot desde una unidad systemd ordenada tras `graphical.target` o
  `boot-complete.target`.
- Con D3, esto da un esquema A/B real: `boot_a` es la imagen candidata y
  `boot_b` la última que arrancó bien.

Riesgo: si la marca no llega nunca, el ABL cae a `boot_b`. Ese es justo el
comportamiento deseado, pero hay que probar en hardware cuántos reintentos
concede el ABL.

**D5.** La documentación lo reconoce: ni `to-android` ni `to-ubuntu` se han
ejecutado en la tablet. La primera vez, hazlo con un PC preparado para
`fastboot flash boot_a` (ver *Switching Between the Systems* en
`INSTALL-TESTING.md`).

**D7. Investigación a medio plazo.** Poner U-Boot (Qualcomm genérico) como
imagen del lado Ubuntu en `boot_a`, en lugar del kernel. U-Boot cargaría
`extlinux.conf` o systemd-boot desde `/boot`. Ventajas: kernels empaquetados de
forma estándar, varias entradas con una de respaldo y menú en pantalla.
Android seguiría usando el mecanismo de intercambio actual. Hay que confirmar
el soporte para SM8475 y la compatibilidad con el ABL, que espera una imagen
v2 con DTB. Es un cambio de arquitectura y solo conviene después de D2 a D4.

## 3. Soporte de hardware: limitaciones reales

La tabla de compatibilidad del README es honesta: distingue lo funcional de lo
no verificado. Lo siguiente cruza esa tabla con el código.

| Área | Estado real | Limitación o riesgo | Prioridad | Validación |
|---|---|---|---|---|
| Batería | `qcom_battmgr`: porcentaje y estado de carga | Sin límite de carga (útil para una tablet siempre enchufada). Carga con la tablet apagada incompleta | Media | HW |
| Carga rápida | PPS sin verificar **activado por defecto** (`10-allow-unverified-adapter.conf`) | Si el kernel no expone el e-marker del cable, se confía en el adaptador. La ruta de alto voltaje no se ha validado térmicamente | **Media-Alta (seguridad física)** | HW |
| Suspensión | s2idle básico | `clk_ignore_unused` y `pd_ignore_unused` impiden apagar relojes y dominios, lo que gasta batería en reposo. La reanudación del teclado no está cubierta | Alta (energía) | HW |
| GPU | Adreno 730 / Freedreno funciona | Soluciones globales `GDK_DISABLE=dmabuf` y `WEBKIT_DISABLE_DMABUF_RENDERER=1` por un problema de coherencia de caché en el kernel. Cuestan rendimiento en todas las apps GTK/WebKit | Media | HW (+ benchmarks) |
| Pantalla y táctil | 120 Hz, NT36532 con 2 variantes de panel | Lápiz no verificado. Sin brillo automático | Baja | HW |
| Cámara | Trasera con ISP por software | Sin autoenfoque ni calibración, vídeo entrecortado, sin cámara frontal ni codificación por hardware. Probada solo con `fastboot boot`, no con la imagen completa | Baja (esfuerzo alto) | HW |
| Audio | 4 altavoces con DSP de protección calibrado, 1 de 4 micrófonos | Sin cancelación de ruido ni audio por DP. Audio USB-C sin verificar | Media | HW |
| Wi-Fi y BT | Cliente 2.4/5 GHz y BT funcionan | Punto de acceso sin verificar | Baja | HW |
| USB | USB 2 en modo dispositivo, OTG con ratón | SuperSpeed y docks sin verificar | Baja | HW |
| Sensores | Acelerómetro vía SLPI | **En algunas unidades, arrancar SLPI reinicia la tablet** (problema conocido). Giroscopio, magnetómetro y luz sin verificar | Media | HW |
| Vídeo | Decodificación H.264 en espacio de usuario | HEVC, VP9, navegador y codificación sin verificar | Baja | HW |
| RTC | Sin verificar | La hora se pierde sin red | Baja | HW |

Recomendación sobre la carga rápida: hasta que haya una validación térmica,
desactivar `--allow-unverified-adapter` por defecto y dejarlo como opción. La
pérdida es pequeña: se carga a 9 V / 12 W en lugar de PPS de alto voltaje.

## 4. Kernel y controladores Qualcomm

| ID | Hallazgo | Gravedad | Dificultad | Validación |
|---|---|---|---|---|
| K1 | La rama `liuqin-6.17` deriva de sm8450-mainline. 6.17 no es LTS y no recibe correcciones de seguridad | Media-Alta | Alta | Auto (compilación) + HW |
| K2 | Argumentos de arranque de depuración en producción | Media | Baja | HW |
| K3 | Interfaces fuera del árbol oficial (grupo sysfs `xiaomi` en pmic_glink, autenticación MiPPS, ajustes de CAMSS para sm8475) | Media | Alta | HW |
| K4 | Firmware de la GPU en `updates/`, fuera del contrato (TODO en `build-root-contract.py`) | Baja | Baja | Auto |

**K1.** Recomendación: hacer rebase sobre **6.18 LTS** como base de producto y
seguir sm8450-mainline para las novedades. Luego enviar a upstream lo que esté
listo: el DTS de liuqin, el táctil NT36532 SPI, el teclado Nanosic, el sensor
s5kjn1 y el soporte de CAMSS para sm8475. Cada parche aceptado es código que
el proyecto deja de mantener. La CI ya compila el kernel desde
`kernel/source.json`, así que el rebase se puede validar automáticamente al
compilar. El comportamiento exige hardware.

**K2.** `device/native-bootargs.txt` contiene:

- `clk_ignore_unused` y `pd_ignore_unused`: mantienen encendido todo lo que el
  bootloader dejó activo. Esto daña el consumo y oculta relojes que faltan en
  el DT.
- `initcall_debug`, `ignore_loglevel`, `keep_bootcon` y `earlycon`: arranque
  más lento y registro saturado.
- `panic=0` junto con `hung_task_panic=1` y `softlockup_panic=1`: ante un
  bloqueo, la tablet se queda congelada para siempre en lugar de reiniciarse.

Propuesta: separar perfiles *release* y *debug*. En *release*, retirar los
`*_ignore_unused` (la doc de la cámara ya marca esto como pendiente de
verificar) y usar, por ejemplo, `panic=10` para reiniciar sobre `boot_b`.
Medir el consumo en reposo antes y después.

## 5. Estabilidad, rendimiento y consumo

| Propuesta | Beneficio | Riesgo | Validación |
|---|---|---|---|
| Quitar los argumentos de depuración (K2) | Reposo más eficiente, arranque más rápido | Puede revelar relojes que faltan en el DT, lo que causaría cuelgues | HW |
| zram (no aparece en los fragmentos de `device/configs` ni en el overlay; comprobar con `zramctl`) | Menos presión de memoria con GNOME y Waydroid | Bajo | Auto (config) + HW |
| Revisar zonas térmicas y LMh (thermal y cooling maps del DT) | Evita sobrecalentamiento y estrangulamiento brusco | Medio | HW |
| Arreglar la coherencia de dma-buf y retirar `GDK_DISABLE=dmabuf` | Menos copias en la GPU, mejor fluidez | Medio | HW |
| `power-profiles-daemon` con el gobernador `schedutil` y perfiles de ahorro | Más autonomía | Bajo | HW |
| Límite de carga al 80 % si el firmware de battmgr lo admite | Mayor vida de la batería | Bajo | HW |
| Pruebas de suspensión en bucle (`rtcwake`, teclado, Wi-Fi, USB) | Detecta regresiones al reanudar | Bajo | HW (automatizable en la tablet) |

## 6. Actualizaciones de Ubuntu sin romper el arranque dual (U1)

**Estado actual:**

- Las unidades `apt-daily*`, `unattended-upgrades` y
  `update-notifier-download` están enmascaradas (enlazadas a `/dev/null`). No
  hay actualizaciones automáticas, **ni siquiera de seguridad**.
- Los paquetes `liuqin-*` no salen de ningún repositorio APT, así que no hay
  forma de actualizarlos con `apt`.
- `liuqin-kernel` solo copia archivos ("the installer owns boot partition
  updates"). Actualizar el kernel exige reinstalar.
- El initramfs verifica un **contrato** antes de montar la raíz: hashes de
  archivos propios del proyecto, más permisos de `systemd`, `gdm3` y
  `gnome-shell`, y destinos de enlaces como `/usr/sbin/init` y
  `display-manager.service`. Si una actualización de Ubuntu cambia alguno de
  esos destinos, **Ubuntu deja de arrancar** y cae a la shell USB.
- `gnome-control-center` e `iio-sensor-proxy` se sustituyen con `dpkg-divert`
  por binarios del proyecto. Si Ubuntu actualiza esos paquetes, el binario
  parcheado se queda con datos y bibliotecas nuevos y puede fallar
  (Configuración en particular).

**Respuesta:** sí, se puede automatizar, con este plan escalonado.

1. **Seguridad primero.** Activar `unattended-upgrades` solo para
   `${distro_codename}-security`. Añadir un `DPkg::Post-Invoke` que ejecute un
   `liuqin-contract-check`, la misma verificación del initramfs sobre la raíz
   viva. Si falla, se avisa y se bloquean los reinicios automáticos.
   Validación: Auto, en un chroot con el árbol de la raíz y paquetes
   simulados.
2. **Repositorio APT propio**, firmado y publicado en GitHub Pages (reprepro o
   aptly), para los `liuqin-*`, con `Pin-Priority` que proteja
   `gnome-control-center` frente a la versión de Ubuntu. Como alternativa, que
   `liuqin-device-support` declare
   `Breaks: gnome-control-center (>> versión-probada)`.
3. **Kernel por paquete.** `liuqin-kernel` incluye un `boot.img` precompilado y
   su `postinst` llama a `liuqin-switch update-ubuntu`, con la semántica A/B
   de D3 y D4. Así un kernel defectuoso vuelve solo a `boot_b`, que es la
   última imagen buena. Validación: Auto (`--dev-dir`) y HW.
4. **Instantánea antes de actualizar.** ext4 no tiene snapshots. A corto plazo:
   copia de seguridad del sistema con rsync en `linux_home` antes de cada
   actualización mayor. A largo plazo: raíz en btrfs con snapper (requiere
   cambiar el initramfs y el contrato).
5. **Cambio de versión de Ubuntu** (por ejemplo, de 26.04 a 26.10): mantenerlo
   manual hasta tener una CI que construya la raíz nueva, la pase por
   `build-liuqin-native-root.sh` (que ya aplica las comprobaciones de la fase 1)
   y pruebe el arranque en una tablet de referencia.

En cualquier caso, `/var/lib/liuqin/switch` queda fuera del contrato, así que
actualizar Ubuntu nunca rompe el arranque de Android.

## 7. Validación: qué se puede automatizar y qué no

**Automatizable (host o CI, sin tablet):**

- Instalador: argumentos, slot, anti-rollback, batería, orden de los pasos
  destructivos y planes de partición (`tests/installer.py`,
  `tests/installer-layout.py`).
- Scripts del dispositivo con BusyBox simulado: `install-layout.sh`,
  `install-root.sh`, `install-switch-store.sh`.
- `liuqin-switch` completo con `--dev-dir`: escrituras, inyección de fallos y
  reversión (210 casos).
- Política de la fase 1 y del protector de almacenamiento (`persistent-root.sh`,
  `storage-guard.sh`).
- Propuestos: `update-ubuntu` (D3), el verbo `preflight` (S4),
  `liuqin-contract-check` (U1) y la compilación del kernel tras el rebase (K1).

**Requiere hardware real:**

- Arranque desde el slot A tras cada cambio y vuelta desde Android con
  KernelSU.
- Comportamiento del ABL ante fallos de carga y número de reintentos (D4).
- Teclas o pantalla en el initramfs (D2).
- Consumo en reposo, suspensión, temperaturas y carga PPS (sección 3 y K2).
- Cámara, audio, sensores, DP y USB 3.
- Una OTA de HyperOS con el módulo activo (D1).

## 8. Hoja de ruta propuesta

| Fase | Trabajo | Tipo |
|---|---|---|
| 0 (hecho en esta rama) | Comprobaciones de slot A, anti-rollback y batería antes de escribir; mensaje honesto; pruebas del conmutador en la CI | Auto |
| 1 (corto plazo) | Verbo `preflight` (S4); copias de `vendor_boot`, `dtbo` y `vbmeta` (S6); PPS sin verificar desactivado por defecto; documentar D1 y S3 | Auto + doc |
| 2 | `update-ubuntu` + marca de éxito tardía (D3, D4); salida de emergencia Vol− (D2); argumentos de arranque *release* (K2) | Auto + HW |
| 3 | Repositorio APT, `unattended-upgrades` de seguridad con comprobación del contrato (U1); rebase a 6.18 LTS (K1) | Auto + HW |
| 4 (investigación) | U-Boot del lado Ubuntu (D7); raíz en btrfs con snapshots | HW |
