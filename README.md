🎯 Objetivo

**KOCHIN-OPTILINUX** busca llevar el nivel de optimización que normalmente se logra a mano en distros como **EndeavourOS** **cachyOS** hacia **Linux Mint XFCE**, mediante un script auto-adaptativo que detecta el hardware del equipo y ajusta automáticamente:

- Kernel (XanMod / Liquorix según la CPU)
- GRUB (mitigations, pstate, blk_mq)
- ZRAM y swap (sin hibernación)
- Parámetros de sysctl (memoria, red, CPU)
- Governor de CPU (performance)
- Tuning de red (Ethernet/WiFi según el chip detectado)
- Paquetes de gaming (Steam, Lutris, Mesa)
- Disco / EXT4 (fstrim, fstab)

Todo esto pensado especialmente para **equipos de bajos recursos** como: (Intel Core i3-4025U, Celeron N4020, similares hasta las cpu actuales...), sin que el usuario tenga que tocar configuraciones manualmente.

El script de optimizacion tiene dos diseño uno al estilo identico a platinum-optimizer. O el basico que es más rapido de aplicar las opciones aplicadas....

> # KOCHIN-OPTILINUX — Documentación de Parametros aplicados:
## 1. Kernel

| Comando / paquete | Qué hace |
|---|---|
| `linux-xanmod-x64v2/v3/v4` | Kernel XanMod compilado para un nivel específico de instrucciones de CPU (microarquitectura). A mayor nivel (v2→v4), más instrucciones modernas usa el compilador para optimizar código — pero si tu CPU no las soporta, el kernel ni arranca. |
| `linux-xanmod-lts` | Versión genérica de XanMod (nivel v1, compatible con cualquier CPU x86-64). Más segura para hardware viejo, menos optimizada que las variantes vX. |
| Liquorix (`install-liquorix.sh`) | Kernel alternativo enfocado en baja latencia de escritorio (mejor "responsividad" percibida, útil para multitarea con audio/video/juegos). No depende de microarquitectura, sirve para cualquier CPU. |
| `wget -qO - ... \| gpg --dearmor -o ...` | Descarga la clave de firma del repositorio (GPG) y la convierte a formato binario que `apt` puede verificar. Sin esto, `apt` rechaza el repo por no poder confirmar que los paquetes son auténticos. |
| `apt update && apt install` | Actualiza la lista de paquetes disponibles e instala el kernel elegido. |

**Por qué importa el nivel de microarquitectura (v1-v4):** cada nivel exige que la CPU tenga ciertas instrucciones (SSE4, AVX, AVX2, AVX512, etc.). El script las detecta leyendo `/proc/cpuinfo` para no ofrecerte un kernel que tu CPU no puede correr.

---

## 2. GRUB (parámetros de arranque del kernel)

| Parámetro | Qué hace | Trade-off |
|---|---|---|
| `quiet splash` | Oculta el log de arranque y muestra un splash gráfico. | Puramente estético, sin impacto en rendimiento. |
| `mitigations=off` | Desactiva los parches de mitigación contra vulnerabilidades de CPU (Spectre, Meltdown, etc.). | Recupera rendimiento (más notorio en CPUs viejas), pero baja la seguridad frente a exploits que exploten esas fallas de hardware. |
| `nowatchdog` | Desactiva el watchdog por software del kernel (detector de cuelgues). | Libera un pequeño overhead de interrupciones periódicas; en la práctica, casi nadie depende de este watchdog en un equipo de escritorio. |
| `pcie_aspm=off` | Desactiva el ahorro de energía de dispositivos PCIe (Active State Power Management). | Puede resolver drops/lag de red o USB causados por dispositivos que "duermen" el bus PCIe agresivamente; a cambio, algo más de consumo energético. |
| `amd_pstate=active` | (Solo CPUs AMD Ryzen) Activa el driver moderno de gestión de frecuencia, que usa CPPC para responder más rápido a cambios de carga. | Mejor solo en kernels 6.x+; en CPUs AMD viejas puede no aplicar. |
| `intel_pstate=passive` | (Solo CPUs Intel) Pone el driver de frecuencia en modo "pasivo", cediendo el control al governor de cpufreq en vez de manejarlo el propio hardware (HWP). | Necesario para que el governor `performance` (ver sección 6) tenga efecto real — en modo activo, Intel HWP suele ignorar el governor manual. |
| `scsi_mod.use_blk_mq=1` | Fuerza el uso de colas múltiples de E/S a nivel de bloque. | En kernels recientes ya viene activado por defecto; forzarlo no hace daño pero tampoco suele notarse. |
| `sudo update-grub` | Regenera el archivo de configuración final de GRUB (`grub.cfg`) a partir de `/etc/default/grub`. | Obligatorio después de editar `/etc/default/grub` — si no lo corrés, los cambios no toman efecto. |

---

## 3. ZRAM (compresión de RAM como swap)

| Comando/parámetro | Qué hace |
|---|---|
| `systemd-zram-generator` | Paquete que crea automáticamente un dispositivo de swap comprimido en RAM al arrancar. |
| `zram-size = ram / N` | Define qué fracción de tu RAM total se reserva para zram. El script ajusta N según cuánta RAM tenés: con poca RAM (≤4GB) usa el 100% como zram, porque el "colchón" extra pesa más que el costo de CPU de comprimir/descomprimir. |
| `compression-algorithm = zstd` | Algoritmo de compresión. zstd da mejor relación velocidad/compresión que lz4 o lzo en hardware moderno, sin ser muy pesado en CPU. |
| `swap-priority = 100` | Prioridad de este swap frente a otros (como el swapfile en disco). Mayor número = se usa primero. zram con 100 se usa antes que el swapfile en disco (prioridad 10), porque comprimir en RAM es muchísimo más rápido que escribir a disco. |

---

## 4. Swap en disco (respaldo detrás de zram)

| Comando | Qué hace |
|---|---|
| `fallocate -l <tamaño> /swapfile` | Reserva el espacio en disco para el archivo de swap, de forma instantánea (sin escribir ceros byte a byte). |
| `chmod 600 /swapfile` | Restringe permisos para que solo root pueda leer/escribir el archivo — un swapfile con permisos abiertos es un riesgo de seguridad (puede contener datos sensibles de la RAM volcados). |
| `mkswap /swapfile` | Formatea el archivo con la estructura que el kernel espera para usarlo como swap. |
| `swapon /swapfile` | Activa el swap inmediatamente, sin esperar a reiniciar. |
| `/swapfile none swap sw,pri=10 0 0` (en `/etc/fstab`) | Línea que hace persistente el swap entre reinicios, con prioridad 10 (menor que zram, ver sección 3). |
| `systemctl mask hibernate.target ...` | Bloquea a nivel de systemd que cualquier proceso pueda disparar hibernación — evita bugs raros de sistemas que hibernan sin tener partición de swap dimensionada para eso. |
| `AllowHibernation=no` (en `/etc/systemd/sleep.conf`) | Refuerza a nivel de configuración que el sistema nunca debe intentar hibernar. |

---

## 5. sysctl (parámetros del kernel en caliente)

| Parámetro | Qué hace |
|---|---|
| `vm.swappiness` | Qué tan agresivo es el kernel para mandar páginas de memoria a swap en vez de mantenerlas en RAM. 0-200 en kernels modernos; valores bajos (5-10) priorizan mantener todo en RAM, valores altos (20+) usan swap más seguido. El script lo ajusta según tu RAM total. |
| `vm.vfs_cache_pressure` | Qué tan agresivo es el kernel para liberar el caché de metadatos de archivos (inodos/dentries) bajo presión de memoria. Valores bajos retienen más caché (mejor con RAM de sobra), valores altos (100+) lo liberan más rápido (mejor con poca RAM). |
| `vm.page-cluster` | Cuántas páginas se leen/escriben juntas al hacer swap. En 0, cada página se maneja individual — mejor para swap en RAM (zram) o SSD rápido, donde no hay ventaja en agrupar operaciones como sí la hay en un HDD mecánico. |
| `vm.dirty_bytes` / `vm.dirty_background_bytes` | Cuánta memoria "sucia" (escrita pero no volcada a disco) se permite acumular antes de forzar la escritura. Valores más bajos = escrituras más frecuentes y pequeñas (menos riesgo de picos largos de I/O que congelen la interfaz). |
| `vm.dirty_writeback_centisecs` | Cada cuántos centisegundos el kernel revisa si hay que volcar memoria sucia a disco. |
| `kernel.nmi_watchdog` | Watchdog por hardware (Non-Maskable Interrupt) que detecta cuelgues del kernel. Desactivarlo (0) libera una interrupción periódica por núcleo. |
| `kernel.split_lock_mitigate` | Mitigación para instrucciones "split lock" (acceso a memoria no alineado) que en ciertos juegos/emuladores generan mucho overhead. Desactivarla (0) recupera rendimiento en esos casos específicos, a costa de la protección que brinda contra ciertos ataques de denegación de servicio local. |
| `kernel.pid_max` | Máximo número de PIDs (identificadores de proceso) simultáneos. Subirlo evita quedarte sin PIDs disponibles en sistemas con muchos procesos/hilos (común en compilaciones grandes o muchos contenedores). |
| `net.core.default_qdisc = fq_codel` | Algoritmo de gestión de colas de red por defecto. `fq_codel` reduce el "bufferbloat" (latencia acumulada por buffers de red sobrecargados), mejorando el ping bajo carga (por ejemplo, descargando algo mientras jugás online). |
| `net.core.netdev_max_backlog` | Cuántos paquetes de red pueden esperar en cola antes de ser procesados por el kernel. Subirlo ayuda en conexiones rápidas con ráfagas de tráfico. |

---

## 6. Governor de CPU

| Comando | Qué hace |
|---|---|
| `cpufrequtils` | Paquete que permite fijar manualmente el governor (política de frecuencia) de la CPU. |
| `GOVERNOR="performance"` | Fuerza a la CPU a correr siempre a su frecuencia máxima permitida, sin escalar hacia abajo en reposo. Más rendimiento constante, pero más consumo/calor — recomendable con el equipo enchufado, no tanto a batería. |
| `ondemand` / `schedutil` (alternativas, no forzadas por el script) | Escalan la frecuencia según la carga detectada — mejor autonomía de batería, algo de latencia extra al pasar de reposo a carga. |

---

## 7. Red (Ethernet / WiFi)

| Comando | Qué hace |
|---|---|
| `ethtool -s <iface> speed 1000 duplex full autoneg on` | Fija manualmente la velocidad y modo dúplex de la placa de red. **Riesgo:** si tu switch/router no negocia bien ese modo forzado, la conexión puede quedar caída — por eso el script explica cómo revertir con `ethtool -s <iface> autoneg on`. |
| `options rtw88_core disable_lps_deep=y` (y variantes rtw89) | Desactiva el modo de ahorro de energía profundo (LPS deep) de chips WiFi Realtek modernos — reduce micro-cortes de conexión a cambio de algo más de consumo. |
| `options ath9k nohwcrypt=1` | Para chips WiFi Qualcomm Atheros (driver `ath9k`, como el AR9565/QCA9565): pasa el cifrado WPA de hardware a software. Corrige cortes/cuelgues conocidos de ese chip, a costa de un poco más de uso de CPU. |
| `wifi.powersave = 2` (en NetworkManager) | Desactiva el ahorro de energía de la interfaz WiFi a nivel de NetworkManager, sin importar el chip — reduce latencia y drops intermitentes. |

---

## 8. Paquetes de gaming

| Comando | Qué hace |
|---|---|
| `dpkg --add-architecture i386` | Habilita la instalación de paquetes de 32 bits en un sistema de 64 bits — necesario porque Steam y muchos juegos/dependencias (Wine, Proton) todavía usan librerías de 32 bits. |
| `mesa-vulkan-drivers:i386`, `libgl1-mesa-dri:i386` | Drivers gráficos Mesa (OpenGL/Vulkan de código abierto) en su versión de 32 bits, para que los juegos de 32 bits puedan renderizar. |
| `steam`, `lutris` | Clientes de distribución/gestión de juegos (Steam: tienda + Proton; Lutris: gestor multi-plataforma para juegos de otras tiendas/emuladores). |
| `ppa:kisak/kisak-mesa` | Repositorio externo con versiones más nuevas de Mesa que las que trae Mint por defecto — útil porque el rendimiento gráfico en Linux mejora seguido con cada versión de Mesa, y las distros estables suelen tardar en actualizarla. |

---

## 9. Disco / EXT4

| Comando/opción | Qué hace |
|---|---|
| `fstrim.timer` | Servicio que ejecuta TRIM periódicamente en discos SSD/eMMC — le avisa al disco qué bloques están libres, ayudando a mantener la velocidad de escritura a largo plazo. **No aplica a HDD** (por eso el script lo activa solo si detecta disco no rotacional). |
| `noatime` (opción de montaje en `/etc/fstab`) | Evita que el sistema escriba la fecha de último acceso cada vez que se lee un archivo — reduce escrituras innecesarias, especialmente notorio en SSD/eMMC de baja durabilidad. |
| `commit=60` (opción de montaje) | Agrupa las escrituras del sistema de archivos cada 60 segundos en vez de los 5 segundos por defecto — menos desgaste del disco y algo de rendimiento, a cambio de perder más datos en caso de un corte de luz repentino (poco probable en uso normal con UPS o notebook). |

---

## Notas generales

- **Todo lo que edita archivos del sistema hace backup primero** (`/root/mint-optimizer-auto-backups/<fecha>/`), así que cualquier cambio es reversible manualmente restaurando el archivo original.
- **`mitigations=off`, `kernel.split_lock_mitigate=0` y `intel_pstate=passive`/`amd_pstate=active`** son los ajustes con mayor impacto en rendimiento, pero también los que más vale la pena entender antes de aplicar — no son gratis, cambian el balance seguridad/latencia del sistema.
- El script nunca aplica un ajuste sin preguntar primero, y detecta tu hardware real antes de sugerir valores — no copia parámetros de una guía genérica sin verificar que tu CPU/RAM/disco los soporten.
