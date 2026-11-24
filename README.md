# rpitx v1 en bare metal pour Raspberry Pi 1, avec RTL-SDR

Portage de **rpitx v1** (F5OEO, tag `v1`, 2018) sur Raspberry Pi 1 **sans Linux**, au-dessus du framework bare metal [Circle](https://github.com/rsta2/circle), avec prise en charge des clés **RTL-SDR** (librtlsdr d'Osmocom) et des modes de rpitx v2 (chirp, peinture spectrale, POCSAG, Opera, RTTY, Morse, FreeDV, FM large bande). Le projet fournit également les menus `easytest` et `rtlmenu`, ainsi qu'une interface plein écran clavier (`tui`).

Le Pi démarre sur un shell (écran HDMI + clavier USB, ou liaison série). Les commandes rpitx v1 conservent leurs options usuelles :

```sh
rpitx:SD:/> pifm sampleaudio.wav fm.ft
rpitx:SD:/> rpitx -m RF -i fm.ft -f 433900 -l
rpitx:SD:/> rtl2tx -r 145.5M -f 50500 -q -40
rpitx:SD:/> tui
```

> **Attention :** émettre exige une autorisation pour la fréquence utilisée et un filtre passe-bande. La sortie GPIO est un signal carré riche en harmoniques. La conformité réglementaire n'a pas été testée.

## État et validations

La compilation complète a été vérifiée avec `arm-none-eabi-gcc` 13.2, Circle 51.1.1 et Pi 1 (kernel.img de 445 Ko, sans référence non résolue). Le démarrage, le shell, la carte FAT32, les scripts et Ctrl-C ont été testés sous QEMU (`raspi0`, SoC BCM2835). Plusieurs générateurs, conversions d'images, modes repris de la v2, menus et chaîne DSP RTL ont été comparés sur PC/QEMU à leurs références.

Sur Raspberry Pi 1 B, le démarrage et l'émission FM à 87,8 MHz ont été vérifiés avec un récepteur. Une clé RTL-SDR Blog V4 a été reconnue et a diffusé à environ 2044 kS/s après correction du transfert USB de contrôle. **L'émission radio réelle n'a pas été testée dans son ensemble** ; le RTL-SDR, `rtl2tx`, les modes autres que la FM essayée, la réception multimode, le clavier sous charge et les nouveaux noyaux restent à valider sur matériel. QEMU n'émule pas le PWM, le gestionnaire d'horloges ni le DMA cyclique.

## Matériel

- Raspberry Pi 1 modèle B (cible principale ; A, A+, B+ et Zero devraient fonctionner, non testés).
- Carte SD FAT32.
- Sortie RF : GPIO 18 (broche 12), ou GPIO 4 (broche 7 avec `-c 1`), **via un filtre**.
- Console HDMI + clavier USB ou série 3,3 V GPIO 14/15, 115200 bauds 8N1.
- Optionnel : clé RTL-SDR RTL2832U avec tuner supporté par librtlsdr.

## Carte SD et utilisation

Copier le contenu de `sdcard/` (archive `rpitx-baremetal-sdcard.zip`) à la racine d'une carte FAT32. On y trouve le firmware Pi, `kernel.img`, la configuration et les exemples. Au démarrage, `autoexec.txt` est exécuté puis l'invite `rpitx:SD:/>` apparaît. Ctrl-C (série/clavier) ou Échap arrête l'outil ; l'émission est coupée en arrêtant DMA, PWM et horloge.

### Outils rpitx v1

| Commande | Fonction |
|---|---|
| `rpitx -m IQ\|IQFLOAT\|RF\|RFA\|VFO -i fichier -f kHz [-l] [-p ppm] [-c 1] [-d n] [-w 1] [-r] [-a canal]` | Émetteur |
| `pifm in.wav [out.ft]` | FM bande étroite |
| `piam in.wav [out.rfa]` | AM |
| `pissb in.wav [out.wav]` | BLU (USB) en IQ |
| `pisstv image.rgb out.ft` | SSTV Martin 1 |
| `pifsq "texte" out.ft` | FSQ |
| `pidcf77 out.rfa` | Trame horaire DCF77 |
| `bmp2rgb image.bmp image.rgb` | Conversion BMP 24/32 bits en 320×256 RGB |

Les scripts de la v1 fonctionnent sans modification (les commandes `sudo` et `./` sont ignorées).

### Modes repris de rpitx v2

| Commande | Fonction |
|---|---|
| `pichirp out.ft [largeur Hz] [période s]` | Balayage sinusoïdal |
| `pispectrum image.rgb out.ft [excursion Hz]` | Peinture spectrale |
| `piwfm [-t 50\|75] in.wav [out.ft]` | FM large bande mono |
| `pipocsag [-r 512\|1200\|2400] [-n] [-t n] out.ft "adresse:message"` | Pager POCSAG |
| `piopera INDICATIF mode out.rfa` | Balise Opera |
| `pirtty out.ft décalage_Hz "texte"` | RTTY Baudot |
| `pimorse vitesse "texte" out.rfa` | Morse |
| `pifreedv vco.rf out.ft [taux]` | Relecture FreeDV modulée |

Chaque mode génère un fichier à émettre avec `rpitx` (RF ou RFA selon le mode).

### Interface et menus

`tui` (alias `menu`) ouvre une interface 80×24 sur HDMI ou terminal série VT100. Haut/Bas choisit le mode, Gauche/Droite ajuste la fréquence, `+`/`-` change le pas, `F` saisit la fréquence, `E` règle les paramètres, `S` choisit GPIO 18/4, Entrée lance et Échap/Ctrl-C arrête. `easytest` et `rtlmenu` offrent les choix textuels des scripts v2. Le TUI conserve ses paramètres dans `rpitx.cfg`.

Commandes shell principales : `ls`, `cd`, `pwd`, `cat`, `cp`, `mv`, `rm`, `mkdir`, `df`, `echo`, `sleep`, `clear`, `sh`, `set [plld 500|1000]`, `info`, `usb`, `reboot`, `help`, `timeout s commande`, `txbench [taux] [kHz]`.

## RTL-SDR

- `rtl_test [-t] [-s taux] [-p]` : test de la clé.
- `rtl_sdr -f Hz [-s taux] [-g dB] [-n échantillons] [-S] fichier` : capture IQ brute.
- `rtl2tx -r Hz -f kHz [-m relay|fm2ssb|wide] [options]` : relais ou FM bande étroite vers BLU ; `wide` retransmet une bande FM large.
- `rtl2iq [-f kHz] capture.bin sortie.wav` : conversion d'une capture en WAV IQ.
- `rtl868` et `rtl433` : décodage de capteurs FSK 868 MHz et OOK 433 MHz.
- `rtlfm` : récepteur FM/AM/BLU ; `rtlscan` : scanner ; `rtlwf` : cascade spectrale.

Les outils de réception et certains décodeurs sont validés en simulation, mais la qualité et les performances réelles dépendent du matériel et restent partiellement à tester. Voir `pc/fmdemod.py` pour démoduler une capture FM sur PC.

## Compilation

Prérequis : `gcc-arm-none-eabi` (testé 13.2.1), `make`, `git`. Circle 51.1.1 (commit `7a81e1b`) est nécessaire :

```sh
git clone https://github.com/rsta2/circle
cd circle
git apply ../rpitx-baremetal/patches/circle-usb-devicefactory-hook.patch
git apply ../rpitx-baremetal/patches/circle-usbhid-keep-retrying.patch
./configure -r 1 --keymap FR
./makeall
make -C addon/fatfs && make -C addon/SDCard
cd ../rpitx-baremetal
make CIRCLEHOME=../circle
make CIRCLEHOME=../circle sdcard
```

Pour la variante FIQ, configurer une seconde copie de Circle avec `-d USE_USB_FIQ` et reconstruire le noyau. Les tests PC nécessitent gcc/g++, libsndfile-dev, sox, ImageMagick et Python 3 : `hosttest/run_all.sh <dépôt rpitx v1> <dépôt rpitx master>`.

## Organisation

- `rpitx/` : c��ur v1 porté, couche matérielle et générateurs ; `orig/` contient les originaux.
- `rpitx/v2/` : modes repris de rpitx v2.
- `rtlsdr/` : librtlsdr et pilote USB Circle.
- `tools/` : outils d'émission/réception et DSP.
- `compat/` : mini-libc, fichiers FatFs, WAV et compatibilité des programmes.
- `kernel/` : noyau Circle, shell et menus.
- `boot/`, `hosttest/`, `patches/` : fichiers de démarrage, tests hôte et patches Circle.

## Notes matérielles

Le port utilise PLLD à 500 MHz par défaut (le `dt-blob.bin` de rpitx v1 empêchait le démarrage du Pi 1 B testé). Au-dessus de 125 MHz, le signal passe par une harmonique impaire ; le port corrige le calcul de fréquence harmonique par rapport à la v1. Un bouton d'arrêt facultatif entre GPIO 17 (broche 11, pull-up interne) et masse (broche 9) agit comme Ctrl-C après 30 ms.

La fréquence, les harmoniques et les performances doivent être vérifiées avec une charge fictive et un récepteur adapté. Une clé RTL-SDR peut dépasser l'alimentation disponible sur les ports USB d'origine du Pi 1 B ; un hub alimenté peut être nécessaire.

## Licences

- rpitx v1 et librtlsdr : GPL v2 ou ultérieure ; Circle et rpitx v2 : GPL v3.
- Le noyau lié (`kernel.img`) est distribué sous GPL v3. Le code nouveau de ce portage est GPL v3 ou ultérieure.
- Le codeur POCSAG est MIT (voir `rpitx/v2/LICENSE-pocsag`). Autres mentions et licences : consulter les fichiers de licence des composants.
- Firmware Raspberry Pi redistribué selon la licence Broadcom (`LICENCE.broadcom`).
