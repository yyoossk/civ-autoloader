# civ-autoloader

## About this fork / このフォークについて

This fork of [SeanHeuc/civ-autoloader](https://github.com/SeanHeuc/civ-autoloader) changes one thing: the `third_party/CivSwitchFS` submodule points to [yyoossk/CivSwitchFS](https://github.com/yyoossk/CivSwitchFS).
That payload carries seven improvements on top of upstream CivSwitchFS, added in three releases.
Everything else is upstream civ-autoloader.

### improved-20261117

1. **Timing learning**
When the FS canary campaign succeeds, the winning timing schedule is recorded inside the save itself.
The next boot starts from the recorded schedule instead of the stock table.
Invalid records are ignored, and reinstalling the save resets the record.
On screen: `TIMING: recorded schedule loaded (win N); starting from it.`

2. **Freeze fix**
IPC sends were bare `svcSendSyncRequest` calls with no timeout, so an unresponsive file-system service froze the console at `Checking the canary window...`.
Sends now run on a sacrificial thread with a 10-second bounded wait.
If the service stops replying, the run ends safely with a message instead of hanging.

Verified on real hardware (Switch 2, firmware 23.0.1, Japanese console and the Japanese version of Civilization VII).
With timing learning, kaiHEN booted at a high rate from the second boot onward.

### improved-20261121 — retry reliability

The design of these three changes was ported from the slop branch of [s8n/CivSwitchFS-kaiHEN](https://github.com/s8n/CivSwitchFS-kaiHEN).

3. **Retirement precheck**
Before running the cache-retirement waves, the payload reads back the candidate cache entry and classifies it.
If the alias is already gone, the retirement waves are skipped entirely and the state counts as verified.
Fewer runs end in a false `Cleanup failed; another attempt is unsafe.` stop, so more failures become safe retries within the same boot.

4. **Cleanup cache invalidate**
Before deleting the scratch tree and committing, cleanup invalidates the cached dirty scratch file via IFile::OperateRange.
The delete and commit therefore start from a clean cache state.

5. **Retune threshold**
The canary campaign now retunes its delays unless at least 80% of windows qualify (previously 50%).
In-run timing adjustments engage more readily across the attempt budget.

### improved-20261124

6. **OperateRange packet size corrected**
The cache-invalidate commands declared 0x0C raw words for a 24-byte input.
The correct count is 0x0E: 0x10 bytes of padding + the 0x10-byte CMIF header + the 0x18-byte input, divided by 4.
Both call sites now declare 0x0E, matching the libnx reference formula.
The old value caused no observed failure.

7. **Dirty file size reduced**
DIRTY_FILE_SIZE goes from 0x800000 (8 MiB) to 0x700000 (7 MiB).
This leaves 1 MiB of headroom under the BC save-data cap for metadata and other outstanding data.

Verification status: improvements 1 and 2 are verified on real hardware.
Improvements 3 to 7 are verified at the source and payload level only.
See [yyoossk/CivSwitchFS](https://github.com/yyoossk/CivSwitchFS) for the full details of each change.

### このフォークの改良点

このフォークは [SeanHeuc/civ-autoloader](https://github.com/SeanHeuc/civ-autoloader) の `third_party/CivSwitchFS` サブモジュールの向き先を、改良版の [yyoossk/CivSwitchFS](https://github.com/yyoossk/CivSwitchFS) に変更したものです。
そのペイロードには、upstream の CivSwitchFS に対して3回のリリースで7つの改良が入っています。
それ以外は upstream の civ-autoloader のままです。

improved-20261117:

1. **タイミング学習**
FS の canary が成功したときのタイミングを、セーブ内に記録します。
次回起動時は初期テーブルではなく、記録したタイミングから開始します。
記録が不正なら無視され、セーブを入れ直せばリセットされます。
画面表示: `TIMING: recorded schedule loaded (win N); starting from it.`

2. **フリーズ修正**
IPC 送信はタイムアウトのない `svcSendSyncRequest` で、FS が無応答になると「Checking the canary window...」で本体が凍結していました。
送信を使い捨てスレッドで実行し、最大10秒で打ち切るようにしました。
無応答時は固まらずにメッセージを表示して安全に終了します。

実機確認済みです (Switch 2 / FW 23.0.1 / 国内版本体・日本版シヴィライゼーション7)。
タイミング学習により、2回目以降に高い確率で kaiHEN が起動しました。

improved-20261121 (リトライの信頼性。設計は [s8n/CivSwitchFS-kaiHEN](https://github.com/s8n/CivSwitchFS-kaiHEN) の slop ブランチから移植):

3. **リタイア前チェック**
キャッシュのリタイア用ウェーブを走らせる前に、候補エントリを読み戻して状態を判定します。
エイリアスが既に消えていればウェーブ自体を省略し、安全とみなします。
「Cleanup failed; another attempt is unsafe.」で止まってしまうケースが減り、同一ブート内で安全に再試行できる幅が広がります。

4. **クリーンアップ時のキャッシュ無効化**
スクラッチを削除してコミットする前に、汚れたスクラッチファイルのキャッシュを IFile::OperateRange で無効化します。
削除とコミットがクリーンなキャッシュ状態から始まります。

5. **再調整のしきい値**
タイミング再調整が働く条件を「合格率50%未満」から「80%未満」に変更しました。
アテンプトの中で調整が効きやすくなります。

improved-20261124:

6. **OperateRange のパケットサイズを修正**
キャッシュ無効化の命令が、24バイトの入力に対して 0x0C 語と宣言していました。
正しいのは 0x0E 語です。
パディング 0x10 バイト + CMIF ヘッダ 0x10 バイト + 入力 0x18 バイトを4で割った値です。
両方の呼び出しを 0x0E に揃え、libnx の参照式と一致させました。
古い値でも失敗した形跡はありません。

7. **ダーティファイルのサイズを縮小**
DIRTY_FILE_SIZE を 0x800000 (8 MiB) から 0x700000 (7 MiB) にしました。
BC セーブデータの上限に対して、メタデータなどの分として 1 MiB の余裕を残します。

検証の状態: 改良1と2は実機確認済みです。
改良3から7は、ソースとペイロードのレベルでの検証のみです。
各変更の詳しい内容は [yyoossk/CivSwitchFS](https://github.com/yyoossk/CivSwitchFS) を見てください。

---

Autoloader for **Civilization VII** on Switch 2.

A host joins the game's local-wireless lobby, delivers the CivSwitch userland
loader, and installs the **autoloader save**. On the next `Load Game` the save
runs CivSwitchFS, prepares kaiHEN, and boots it.

## Status

| Entry | Host | Status |
|---|---|---|
| ESP32-S3 | fully offline board | **tested** (Switch 2, 23.0.1) |
| Switch 1 NRO | second console | **tested** (Switch 2, 23.0.1) |
| Linux PC | Linux + Wi-Fi radio | **not tested yet** |

The ESP32 and Switch 1 NRO paths have both completed an install on hardware
and reached `kaiHEN installation complete`. The Linux entry builds and shares
the same payload and save, but nobody has completed an install with it yet.
See [`packages/transport/README.md`](packages/transport/README.md) for each
entry's requirements and caveats.

## What you need

| Item | Notes |
|---|---|
| Console | Switch 2, firmware **23.0.1** |
| Game | Civilization VII **1.0.0 physical cartridge** |
| Host | an ESP32-S3 (16 MB flash), **or** Linux with a Wi-Fi radio, **or** a Switch 1 with homebrew |
| Keys | your own `prod.keys` (not needed for the Switch 1 NRO) |
| SD card | **microSD Express**, formatted **exFAT** (recommended on an updated Switch 2) or **FAT32** for smaller cards |

The SD card is required for kaiHEN (`hbmenu.nro` and `kaihen/`). Without it,
the FS stage still runs, but kaiHEN preparation stops with an on-screen
message.

## Layout

```text
packages/
  firmware/esp32/    ESP-IDF firmware (tested entry)
  transport/
    linux/           Linux PC entry (untested)
    nro/             Switch 1 homebrew entry (untested)
    build-payload.js shared chat payload builder
    compress.js      shared chat compressor
    list-engines.js  reads every engine.json
  sdcard/            SD card staging and pinned prebuilt binaries
  userland/          notes for the CivSwitch userland loader
  fs/                notes for the CivSwitchFS stage
  kaihen/            notes for the kaiHEN installer stage
  orchestrator/      notes for the FS -> kaiHEN stage pipeline
third_party/
  CivSwitch/         fork: game-side userland loader
  CivSwitchFS/       fork: FS elevation + autoloader overlay
  kaihen/            fork: HEN / RO / bootstrap
docs/                architecture, build, flashing, compatibility
```

Each entry declares itself in an `engine.json`:

```sh
node packages/transport/list-engines.js
```

```text
id         name                                   platforms              port  status   tested
esp32-ldn  ESP32-S3 board over local wireless     esp32s3                9027  present  yes
linux-ldn  Linux PC over local wireless           linux-x64,linux-arm64  9027  present  no
nro-ldn    Switch 1 homebrew over local wireless  switch1-homebrew       9027  present  no
```

## 1. Prepare the SD card

```sh
python3 packages/sdcard/stage.py
```

Copy the **contents** of `packages/sdcard/contents/` to the root of the SD
card. The card must contain:

```text
/hbmenu.nro
/kaihen/bootstrap.bin
/kaihen/bin/game/main
/kaihen/bin/hen/main
/kaihen/bin/ro/main
/kaihen/hen-options.txt
/licenses/...
```

`kaihen/plugins/ftpd.nso` is optional (FTP server on port 5000).

## 2. Build the save blobs

All three entries deliver the same two blobs. Build them once:

```sh
git submodule update --init --recursive

# Build dependencies (once)
(cd third_party/CivSwitchFS && npm ci)
(cd third_party/CivSwitch/PicoSwitch && npm install)
(cd third_party/CivSwitch/installer-creator && npm install)

# Put your own prod.keys here (gitignored)
cp /path/to/prod.keys packages/firmware/esp32/prod.keys

cd packages/firmware/esp32
source /path/to/esp-idf/export.sh          # ESP32 entry only
python3 tools/build_and_flash.py --keys prod.keys
```

Outputs:

```text
blobs/blob_tpl.bin    save template
blobs/blob_apply.bin  save installer + compressed CFS stage
```

The storage partition is 4 MB.

kaiHEN only arms for **Switch 2 games**: titles whose program ID has the `0x04`
family prefix (`0x04xxxxxxxxxxxxxx`). A Switch 1 game (`0x0100...`) runs from its
own process and is not a supported target for this chain.

## 3a. Install with the ESP32 (tested)

```sh
cd packages/firmware/esp32
python3 tools/build_and_flash.py --keys prod.keys \
  --port /dev/cu.usbmodemXXXX --flash
```

1. Insert the SD card into the console.
2. Start Civ VII.
3. `MULTIPLAYER -> WIRELESS -> CREATE GAME -> HOST LOBBY`.
4. Power the board (USB power is enough). It comes up as the **N-Hole**
   hotspot.
5. Hold **BOOT** for 1.5 s: the board leaves the hotspot, scans and installs.
6. Wait for `PERSIST INSTALL DONE - restart the game, then Load Game`.
7. Close Civ VII completely, reopen it, choose **Load Game**, and load
   `XSTRIG.Civ7Save`.

The install ends back in the **N-Hole** hotspot, which serves a Nintendo-aware
DNS sinkhole and a mini HTTP responder (the nanodns policy). While it is
scanning and no lobby is visible, another BOOT hold returns to the hotspot; once
the board has joined a lobby the button is ignored until the pass ends. N-Hole
has no uplink, so it answers Nintendo names only — see
[`packages/firmware/esp32/README.md`](packages/firmware/esp32/README.md#n-hole).

## 3b. Install with a Linux PC (not tested yet)

Requirements: real Linux (not WSL), one Wi-Fi radio with nl80211,
NetworkManager, Python 3.12+, `libcap2-bin`.

```sh
cd packages/transport/linux
npm install                 # creates ldn/.venv, asks sudo for setcap
cp /path/to/prod.keys prod.keys
node index.js --verbose
```

Keep the Civ VII wireless lobby open. The script temporarily releases the
Wi-Fi radio, joins the lobby over LDN, sends the payload, streams the save over
UDP 9027, and restores networking when it finishes. Then close Civ VII,
reopen it, and `Load Game`.

See [`packages/transport/linux/README.md`](packages/transport/linux/README.md).

## 3c. Install with a Switch 1 NRO

**Tested on hardware** with a Switch 1 running the NRO and a Switch 2 on
23.0.1: the save installs over LDN and `Load Game` reaches kaiHEN.

Requirements: Switch 1 homebrew, `ldn_mitm` removed and the console rebooted,
both consoles close together (local wireless only), Civ VII 1.0.0 on the
Switch 2.

```sh
cd packages/transport/nro
export DEVKITPRO=/opt/devkitpro
make
```

Copy `civ-autoloader-nx.nro` to `/sdcard/switch/` and launch it from the
Album. The app shows two install options:

```text
civ-autoloader NX by sean v0.10
press ZR: install CivSwitch
press R: install autoloader (kaiHEN)
```

| Button | Save | What `Load Game` does next |
|---|---|---|
| **ZR** | standard CivSwitch save | CivSwitch loader is back on TCP `:9026` |
| **R** | autoloader save | CivSwitchFS -> prepare kaiHEN -> boot hbmenu |

With no lobby joined, pressing either button scans immediately and, when only
one Civ VII room is visible, joins it and installs without further input. With
several rooms visible, pick one with Up/Down and press **A**.

After `Save installer: Persistent trigger save installed`, close Civ VII on the
Switch 2, reopen it, and choose `Load Game`.

**Warning:** this replaces every Civilization VII single-player save.

See [`packages/transport/nro/README.md`](packages/transport/nro/README.md).

## 4. After the install

The save runs:

```text
CivSwitchFS -> prepare kaiHEN -> boot kaiHEN -> hbmenu
```

When it finishes, the console shows:

```text
AUTOLOADER: kaiHEN installation complete. You can close Civ7 now.
```

The save is offline: it does not wait for a network after the autoloader.
The same-session CFS stage is intentionally not used because running CFS
immediately after the save install can freeze the console. The save is the
hand-off point.

## Fallback

If the persistent autoloader save causes FS canary failures, the standard
CivSwitch save is a fallback: use `make_install_blobs.py` and the `save`
stage. The default build keeps the persistent save because it is the
one-click path.

## Dev / manual mode

For manual testing, set `devMode` to `true` in
`third_party/CivSwitchFS/src/core/autoloader.mjs`, rebuild, and reflash:

```js
const CONFIG = {
    modules: ["fs", "kaihen"],
    kaihenAuto: true,
    devMode: true
};
```

The autoloader then only prepares kaiHEN and keeps the FS bridge alive; it
prints `dev mode; kaiHEN boot skipped`. Use the bridge's manual commands to
inspect or boot HEN.

The kaiHEN SD log is **off by default**. To enable it, uncomment `log-sd` in
`/kaihen/hen-options.txt`.

## Troubleshooting

- The console shows the autoloader progress on screen.
- The ESP32 serial log is available with `tools/wait-log.py`.
- No SD log is written before kaiHEN starts.
- If the SD card is missing, the FS stage still completes and the console
  reports which kaiHEN files are missing.

## Standing on the shoulders of giants

This project would not exist without the work below. Thank you.

### The chain

| Project | What we used | License |
|---|---|---|
| [CivSwitch](https://github.com/Gezine/CivSwitch) | userland exploit, Remote JS Loader, persistent save installer, chat compression | MIT |
| [CivSwitchFS](https://github.com/ItsNerisma/CivSwitchFS) | FS elevation, persistent save codec/installer, `onElevated` hook | MIT |
| [kaiHEN](https://github.com/ouncebrew/kaihen) | HEN, RO, bootstrap, plugin runtime, hbmenu/hbloader packaging | GPL-2.0-only; `libkaihen` ISC |

### LDN transport

| Project | What we used | License |
|---|---|---|
| [CivSwitchLDN](https://github.com/ItsNerisma/CivSwitchLDN) | LDN client, Civ VII lobby protocol, UDP installer design; the Linux entry is a fork of it | GPL-3.0-only |
| [Kinnay LDN](https://github.com/kinnay/LDN) | key derivation, advertisement/auth format, nl80211 interface handling | GPL-3.0 |
| [pokeldn](https://github.com/Decryptu/pokeldn) | ESP32 `wpa_funcs` patch for handshake-less association | AGPL-3.0 |
| [IZCIVSwitch](https://github.com/byizmysh/IZCIVSwitch) | interface-level radio release (`nmcli device set managed no`), netlink teardown, shield around the association cleanup, `engine.json` idea | GPL-2.0-or-later (its own code) |
| [xiaolizi23/civswitch-ldn-nx](https://github.com/xiaolizi23/civswitch-ldn-nx) | the Switch 1 NRO: LDN join, `ldn_mitm` guard, launch-mode guard | GPL-3.0-only |

### Homebrew runtime

| Project | What we used | License |
|---|---|---|
| [ounce-hbloader](https://github.com/ouncebrew/ounce-hbloader) | NS2 `game/main` | ISC |
| [ounce-hbmenu](https://github.com/ouncebrew/ounce-hbmenu) | NS2 `hbmenu.nro` | ISC |
| [nx-hbloader](https://github.com/switchbrew/nx-hbloader) | NS1 loader reference (not shipped) | ISC |
| [nx-hbmenu](https://github.com/switchbrew/nx-hbmenu) | NS1 menu reference (not shipped) | ISC |
| [OunceShell](https://github.com/kurumuz/ounceshell) | `ftpd` plugin source, optional file manager | GPL-3.0 |
| [ftpd](https://github.com/kurumuz/ounceshell) | FTP server (`switch2/third_party/ftpd`) | Unlicense |
| [libnx](https://github.com/switchbrew/libnx) | Switch runtime | ISC |
| [libounce](https://github.com/ouncebrew/ounce-devkit) | Switch 2 runtime, plugin API | ISC |
| [ESP-IDF](https://github.com/espressif/esp-idf) | ESP32-S3 firmware framework | Apache-2.0 |
| [devkitPro](https://devkitpro.org/) | toolchain and portlibs | various; see `packages/sdcard/licenses/` |

### Research references

No source tree from these is shipped here; they informed the implementation.

| Project | What it informed |
|---|---|
| [ns2jb](https://gitlab.com/kurumuz/ns2jb) | FS path-normalizer overflow, reserved-gap allocator race |
| [civ7-fs-kaihen](https://github.com/s8n/civ7-fs-kaihen) | arm flow, raw thread scheduling, FS memory primitives |
| CivSwitch ns2jb port (Slapah) | FS acquisition idea and reference shape checks |
| [civ7-fs](https://github.com/HerXayah/civ7-fs) | Civ7 FS port lineage |
| [civswitch-kit](https://gitee.com/mza1010/civswitch-kit) | reference / control implementation |
| [Atmosphere](https://github.com/Atmosphere-NX/Atmosphere) | FS source comparison, session-limit analysis |
| [SwitchBrew](https://switchbrew.org/) | system documentation, title IDs, service tables |

The full license texts for the shipped components are in
`packages/sdcard/licenses/` and `THIRD_PARTY_NOTICES.md`.

## License

The original JavaScript payloads and tooling are MIT. The ESP32 firmware's LDN
transport is ported from GPL-3.0-only / AGPL-3.0 projects, so
`packages/firmware/esp32` is distributed under GPL-3.0 (and the pokeldn-derived
part under AGPL-3.0). Third-party components keep their own licenses; see
`THIRD_PARTY_NOTICES.md` and `packages/sdcard/licenses/`.