<!-- SPDX-License-Identifier: CC-BY-SA-4.0 -->

# Diagrammes — Baladeur SD sans écran (contrôle BLE)

Ce fichier regroupe les schémas **Mermaid** de référence à inclure dans le cahier des charges / la documentation. Copier/coller tel quel dans `docs/diagrams.md` du dépôt.

---


```mermaid
flowchart LR
  %% ======== LÉGENDE ========
  classDef audio stroke-width:2px;
  classDef control stroke-dasharray: 5 5;

  %% ======== BLOCS ========
  Phone["Appli mobile\n(smartphone)"]
  subgraph BLE["BLE GATT (ACS/BAS/DIS)"]
    Direction["→ commandes\n← notifications"]
  end

  Btn["Bouton\n(mécanique)"]
  LED["LED\n(indication)"]

  SD[("microSD\n(FAT32/exFAT)")]
  SoC["SoC ESP32‑S3\nFS + Index + Décodage + Buffer"]
  I2S["Bus I²S\n(PCM)"]
  DAC["DAC ES9020Q"]
  AMP["Ampli casque INA1620\n/ Line-Out"]
  HP[("Casque / Entrée ligne")]

  %% ======== LIENS AUDIO (pleins) ========
  SD -->|"fichiers FLAC/OPUS/WAV"| SoC:::audio
  SoC -->|"PCM"| I2S:::audio
  I2S --> DAC:::audio
  DAC --> AMP:::audio
  AMP --> HP:::audio

  %% ======== LIENS CONTRÔLE (pointillés) ========
  Phone -.->|"BLE GATT\n(Playback/Volume/Lib)"| BLE:::control
  BLE -.-> SoC:::control
  SoC -.->|"Notify: état, position, volume"| Phone:::control

  Btn -.->|GPIO| SoC:::control
  SoC -.->|"pattern LED"| LED:::control
```


## 1) Flux logiques (données + contrôle) — v2

```mermaid
flowchart LR
  App[(Appli mobile/PC BLE)]
  subgraph BLE["BLE GATT (ACS + BAS + DIS)"]
    PlayCtrl[PlaybackControl]
    Vol[Volume]
    Pos[TrackPosition/Length]
    LibReq[LibraryRequest]
    LibRes[LibraryResult]
    Dev[DeviceState]
  end
  UI[UI locale Bouton + LED + JackDetect]
  SD[(microSD)]
  IDX[Indexation]
  DEC[Décodeur/Buffer FLAC/OPUS/PCM]
  I2S[I²S]
  DAC[DAC ES9020Q]
  OUT[Carte casque / Line-Out / Ampli fixe option]
  Batt[(Batterie 18650/14500)]
  USB[USB-C 5V]

  App <---> BLE
  UI --> BLE
  SD --> IDX --> DEC --> I2S --> DAC --> OUT
  BLE --> DEC
  DEC --> BLE
  Batt --> OUT
  USB --> OUT
```

---

## 2) Modules firmware (niveau 1) — v2

```mermaid
flowchart TB
  subgraph Firmware
    CTRL[State Machine play/pause/seek]
    BLE[(BLE ACS/BAS/DIS)]
    FS[FS Abstraction FAT32/exFAT]
    IDX[Indexation & Cache]
    META[Parse métadonnées]
    DEC[Pipeline audio FLAC/OPUS/WAV]
    I2S[I²S driver]
    UI[GPIO ISR Bouton/LED/Jack]
    PWR[Power Mgmt Sleep/Wake/Timers]
    ERR[ErrLog/Watchdog]
  end
  BLE <--> CTRL
  UI --> CTRL
  CTRL <--> FS
  CTRL --> IDX
  FS --> IDX
  IDX --> META
  CTRL --> DEC
  META --> DEC
  DEC --> I2S
  CTRL <--> PWR
  CTRL --> ERR
```

---

## 3) Séquence : connexion & lecture

```mermaid
sequenceDiagram
  autonumber
  participant C as Client BLE (App)
  participant P as Player (ESP32-S3)
  C->>P: CONNECT + MTU Request (ex. 185)
  C->>P: Enable Notify (Pos, Dev, LibRes, Battery)
  C->>P: Read ProtoVersion / DeviceState
  C->>P: Write PlaybackControl=PLAY
  P-->>C: Notify DeviceState(PLAYING)
  loop lecture
    P-->>C: Notify TrackPosition (~500 ms)
  end
  C->>P: Write Volume=60%
  Note over C,P: Pause/Next/Prev idem via PlaybackControl
```

---

## 4) Séquence : parcours bibliothèque

```mermaid
sequenceDiagram
  autonumber
  participant C as Client BLE
  participant P as Player
  C->>P: LibraryRequest = LIST_TRACKS{folder_id=123, page=0, size=20}
  P-->>C: LibraryResult (FRAG seq=0..n, last=1)
  C->>C: Réassemblage & affichage liste
  C->>P: (option) GET_TRACKINFO{track_id}
  P-->>C: TrackInfo TLV (Read/Notify)
  C->>P: PlaybackControl=PLAY (piste choisie)
```

---

## 5) États principaux du lecteur (FSM)

```mermaid
stateDiagram-v2
  [*] --> OFF
  OFF --> BOOT : bouton long / jack-detect
  BOOT --> IDLE : init OK
  IDLE --> ADVERTISING : pas de lien BLE
  ADVERTISING --> CONNECTED : lien BLE établi
  CONNECTED --> PLAYING : Play
  PLAYING --> PAUSED : Pause
  PAUSED --> PLAYING : Play
  PLAYING --> SEEKING : Seek
  SEEKING --> PLAYING : Seek done
  IDLE --> SCANNING_SD : SD insérée / rescan
  SCANNING_SD --> IDLE : index prêt
  any --> ERROR : erreur critique
  any --> DEEP_SLEEP : timeout / bouton long
  DEEP_SLEEP --> BOOT : bouton / jack-detect / charge
```

---

## 6) Domaines d’alimentation & sources de réveil

```mermaid
flowchart LR
  Batt[(Batterie 3.7V)] --> SW[Inter général]
  USB[USB-C 5V] --> OR[Diode-OR / Power Path]
  SW --> OR
  OR --> LDO_A[Reg 3.3V analog<br/>(LDO low-noise)]
  OR --> LDO_D[Reg 3.3V digital]
  LDO_D --> SoC[ESP32-S3]
  LDO_A --> DAC[ES9020Q] --> AMP[INA1620/Line-Out]
  BTN[Bouton] -->|Wake| SoC
  JACK[Jack Detect] -->|Wake| SoC
  RTC[Timer/RTC] -->|Wake| SoC
  SoC -. "BLE adv" .-> Antenna((BLE))
```

---

## 7) Carte GATT — vue d’ensemble

```mermaid
mindmap
  root((BLE GATT))
    ACS(Audio Control Service 128b)
      PlaybackControl (RW)
      Volume (RW,N)
      TrackPosition (R,N)
      TrackLength (R)
      TrackInfo (R)
      LibraryRequest (W)
      LibraryResult (R,N)
      DeviceState (R,N)
      ProtoVersion (R)
    BAS(Battery Service 0x180F)
      Battery Level (R,N)
    DIS(Device Info 0x180A)
      Manufacturer/Model/FW (R)
```

---

## 8) (Option) FSM d’indication LED

> Voir la table détaillée dans `firmware/docs/ble-gatt.md` §17. Ce mini schéma illustre le **chemin nominal**.

```mermaid
stateDiagram-v2
  [*] --> LED_IDLE
  LED_IDLE --> LED_PAIRING : advertising/pairing
  LED_IDLE --> LED_PLAY : playing
  LED_IDLE --> LED_PAUSE : paused
  LED_* --> LED_LOW_BATT : batt<15%
  LED_* --> LED_CRIT_BATT : batt<7%
  LED_* --> LED_ERROR : ERROR_CODE!=0
  LED_* --> LED_OVERTEMP : over_temp
  LED_* --> LED_SLEEP : deep sleep
```

---

## Notes d’utilisation

- Ces diagrammes **n’exposent pas** d’API réseau/audio : le BLE sert **au contrôle** uniquement ; l’audio reste **local** (microSD → I²S → DAC).
- Conserver ce fichier sous **CC-BY-SA-4.0** ; les fichiers matériels sont sous **CERN-OHL-S-2.0** (voir `LICENSES/`).

