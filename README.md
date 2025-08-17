<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Baladeur SD sans écran — Contrôle BLE (Open Hardware)

Lecteur audio **local** (microSD) sans écran, contrôlé via **Bluetooth Low Energy** (BLE) depuis une appli mobile/PC. Conçu pour être **ouvert**, **modulaire** et **facile à fabriquer** (KiCad + JLCPCB). 

> **Statut** : v1 en développement — BLE = **contrôle uniquement** (pas d’A2DP). Audio local → I²S → DAC → casque / ligne. 

---

## ✨ Fonctionnalités clés
- Lecture **FLAC/OPUS/WAV** (44,1/48 kHz, 16/24 bit) depuis **microSD**
- **BLE GATT** pour Play/Pause/Next/Prev, Volume, Browse dossiers/pistes
- **Sortie casque** 3,5 mm (INA1620) et **sortie ligne** (board dédiée)
- **Option ampli fixe** : TPA3116D2 (2×50 W @ 24 V)
- **ESP32-S3** (Wi-Fi + BLE) — BLE pour contrôle / télémétrie uniquement
- **USB-C 5 V** (alimentation), **batterie** 18650/14500 (module séparé)
- **KiCad** (source de vérité), **JLCPCB** pour fab/assemblage

---

## 🗂 Arborescence
```
/hardware/
  main-board_kicad/
  hp-board_kicad/
  lineout-board_kicad/
  amp-tpa3116_board_kicad/
/firmware/
  app/
  components/
  docs/ble-gatt.md
/docs/
  README.md
  user-guide.md
  diagrams.md
/ LICENSES/
  CERN-OHL-S-2.0.txt
  CC-BY-SA-4.0.txt
  Apache-2.0.txt
LICENSE
README.md (ce fichier)
```
- **Diagrammes** : voir `docs/diagrams.md` (flux, FSM, séquences, alimentation, GATT). 
- **GATT BLE** : spécification complète dans `firmware/docs/ble-gatt.md`.

---

## 💻 Firmware (ESP-IDF + PlatformIO)

### Pré-requis
- PlatformIO (VSCode ou CLI) 
- Toolchain ESP-IDF (pilotée par PlatformIO)

### Build & flash (exemple générique)
```bash
cd firmware/app
pio run                 # build
pio run -t upload       # flash (USB-Série)
pio device monitor -b 115200
```

### Paramètres par défaut
- **Nom BLE** : `TIRESIAS-PLAYER` (configurable)
- **MTU** cible : 185 (gère MTU=23)
- **Notifs** : `TrackPosition` ~500 ms ; `DeviceState/Battery` sur changement
- **Sécurité** : écriture **chiffrée** (LE Secure Connections, Just Works min.; Passkey recommandé), **bonding** activé

> Détails du **GATT** : voir `firmware/docs/ble-gatt.md` (UUIDs, TLV, séquences, sécurité).
