# Cahier des charges — Baladeur SD sans écran (contrôle BLE)

**Version : v1.1 (intègre retours sécurité/alim et clarifications)**

> **Résumé** : baladeur audio **sans écran**, lecture **locale** depuis **microSD**, contrôle **exclusivement via BLE** (pas d’audio Bluetooth classique). Sorties : **casque 3,5 mm** (v1) et **ligne** (carte fille). Usage fixe : option **ampli classe D**.

---

## 1) Objet & périmètre

- **In** : lecture FLAC/OPUS/WAV (44,1/48 kHz 16/24 bit), navigation dossiers, reprise de lecture, contrôle via appli BLE, bouton/LED, détection jack, veille/ réveil.
- **Out (v1)** : A2DP BT Classic, jack 4,4 mm équilibré, égaliseur HW du DAC, multi‑sorties.

## 2) Licences

- **Matériel** (`/hardware`) : **CERN‑OHL‑S v2.0** (strongly reciprocal).
- **Docs** (`/docs`) : **CC‑BY‑SA‑4.0**.
- **Firmware** (`/firmware`) : **Apache‑2.0** (par défaut ; ajustable).
- En‑têtes **SPDX** obligatoires + textes complets dans `/LICENSES/`.

## 3) Exigences fonctionnelles

- **Formats** : FLAC, OPUS, WAV/PCM ; indexation au boot et à la demande.
- **Commandes** : Play/Pause, Next/Prev, Volume, choix piste/dossier.
- **Stockage** : microSD (FAT32, exFAT en option).
- **Reprise** : dernière piste & position (si activé).
- **UI locale** : 1 bouton mécanique multi‑fonction ; 1 LED (mono ou bi‑couleur) ; jack‑detect.

## 4) Exigences non fonctionnelles

### 4.1 Énergie & modes

- **Deep sleep** : < **1 mA** ; **advertising** lent ; réveil bouton/jack/RTC.
- **Conso lecture** (hors ampli puissance) : objectif **< 200 mW**.
- **Profils LED** : VISUEL/LOW\_POWER (voir `ble-gatt.md` §17).

### 4.2 Alimentation & recharge (variants v1A/v1B)

- **v1A (par défaut)** : **pas de recharge embarquée** ; cellule Li‑ion **protégée (PCM)** obligatoire ; **PTC/eFuse** ; mesure tension → ADC ; inter général.
- **v1B (option)** : **chargeur 1S CC/CV** avec **TS/NTC 10k** (0–45 °C) ; **power‑path / OR‑ing** (load sharing) ; limite USB 500 mA ; télémétrie STAT/PG ; fuel‑gauge I²C optionnel.
- Schémas et FSM dans `docs/diagrams.md` (§6) et `ble-gatt.md` (§9–10).

### 4.3 Conception matérielle

- CAO **KiCad**. Fabrication **JLCPCB** (BOM/CPL/DFM).
- **Modulaire** : carte principale **60×35 mm** ; carte casque **30×20 mm** perpendiculaire ; module batterie séparé/dos.
- **SI/EMI** : masses **analog/digital** dédiées, ferrites DAC, découplage court, I²S court et propre.

## 5) Architecture technique

- **SoC** : **ESP32‑S3** (Wi‑Fi + **BLE** ; pas de BT Classic).
- **DAC** : **ESS ES9020Q** (stéréo) via **I²S**, config I²C ; **horloge TCXO optionnelle**.
- **Ampli casque** : **INA1620** (carte fille).
- **Sortie ligne** : carte dédiée (RCA/jack, gain ≈ 1).
- **Ampli fixe option** : **TPA3116D2** (2×50 W @ 24 V) avec filtres LC adaptés.
- **Alims** : **3v3 digital** et **3v3 analog low‑noise** séparées ; protections.

## 6) Interfaces & connectique

- **USB‑C 5 V** (alim ; debug UART via convertisseur).
- **Headers** : UART0 (EN/IO0/TX/RX), I²S (MCLK/LRCK/BCLK/DATA), GPIO bouton/LED, jack‑detect.
- **Protection** : diode inversion, PTC/eFuse, ESD on‑board.

## 7) Interface BLE (contrôle)

- **GATT** : voir \`\` (ACS + DIS + BAS, UUIDs, TLV, sécurité).
- **Sécurité** : écritures **chiffrées** (LE Secure Connections min.), **bonding** recommandé, **RPA** activé, **whitelist** optionnelle.

## 8) Logiciel & toolchain

- **ESP‑IDF + PlatformIO** (CI facile). Arduino autorisé pour protos, pas base prod.
- **Structure** : `firmware/app` (app), `components/*` (drivers SD, DAC, BLE, UI, power), `docs/` (GATT).
- **Qualité** : logs, watchdog, tests unitaires (drivers purs), formatage (`clang-format`).
- **MàJ** : flash UART/USB ; OTA v2.

## 9) Critères d’acceptation (v1)

- Boot → advertising BLE & LED prêtes en **< 3 s**.
- Lecture **FLAC 16/44,1** et **OPUS 48k/160 kbps** **sans drop**.
- **Play/Pause/Next/Prev/Volume** OK via **BLE** & **bouton**.
- Reprise dernière piste/position (si activée).
- **Deep sleep < 1 mA** ; réveil bouton et jack‑detect.
- **v1B** : charge **0–45 °C**, terminaison **CC→CV→I\_TERM \~0,1·I\_CHG**, limitation USB respectée, signalisation état charge.

## 10) Risques & points ouverts

- Charge CPU décodage OPUS/FLAC (buffer PSRAM).
- Bruit analogique (LDO low‑noise, routage).
- EMI (I²S court, masses, LC classe D).
- Égaliseur interne ES9020Q (report v2).
- Jack 4,4 mm équilibré (v2 pro).
- A2DP non supporté (BLE = contrôle only ; futur LE Audio éventuel).

## 11) Arborescence dépôt (suggestion)

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
  diagrams/
/LICENSES/
  CERN-OHL-S-2.0.txt
  CC-BY-SA-4.0.txt
  Apache-2.0.txt (si choisi)
LICENSE
README.md
```

## 12) Références croisées

- **Spéc BLE détaillée** : `firmware/docs/ble-gatt.md`.
- **Schémas Mermaid** (flux, FSM, power‑path) : `docs/diagrams.md`.
- **Table états LED** : `ble-gatt.md` §17.

## 13) Changelog

- **v1.1** : Ajout variantes alim **v1A/v1B**, sécurité charge Li‑ion, toolchain PlatformIO/ESP‑IDF, BLE sécu (LESC/bonding/RPA), clarifications UI (bouton mécanique).
- **v1.0** : version initiale (périmètre, BOM conceptuelle, archi, critères v1).

