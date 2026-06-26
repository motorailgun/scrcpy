# camstream - Android カメラ H.264 ストリーミングツール

scrcpy のサーバー JAR を再利用し、Android 端末のカメラから H.264 ストリームを取得して stdout に出力するツール。

## アーキテクチャ

```
Android 端末                                 PC (ホスト)
┌──────────────────────────┐              ┌─────────────────────┐
│  scrcpy-server.jar       │              │  camstream (Python)  │
│                          │  adb forward │                     │
│  Camera2 API             │──────────────│  プロトコル解析      │
│    ↓  (4K 解像度)        │ LocalSocket  │    ↓                │
│  MediaCodec (HW H.264)  │  → TCP       │  raw H.264 Annex B  │
│    ↓                     │              │    ↓                │
│  Streamer (scrcpy独自    │              │  stdout             │
│    フレーミング)          │              │    ↓                │
│                          │              │  ffplay / VLC / file │
└──────────────────────────┘              └─────────────────────┘
```

## 動作フロー

1. `adb push` で `scrcpy-server.jar` を `/data/local/tmp/` に転送
2. `adb forward tcp:0 localabstract:scrcpy_XXXXXXXX` でソケットトンネルを確立
3. `adb shell CLASSPATH=/data/local/tmp/scrcpy-server.jar app_process / com.genymobile.scrcpy.Server` でサーバーを起動
   - 起動パラメータ: `video_source=camera`, `camera_size=3840x2160`, `audio=false`, `control=false`
4. PC 側から TCP で接続し、scrcpy プロトコルを解析:
   - 64 バイト: デバイス名
   - 4 バイト: コーデック ID (H.264 = `0x68323634`)
   - 12 バイト: セッションヘッダ (幅・高さ)
5. 各フレームの 12 バイトヘッダ (PTS + flags + size) を除去し、raw H.264 Annex B データを stdout に出力

## scrcpy プロトコル詳細

### 接続シーケンス

```
[デバイス名: 64 bytes]  →  UTF-8 文字列 (null-padded)
[コーデック ID: 4 bytes] →  Big-endian uint32 ("h264" = 0x68323634)
```

### パケットフォーマット

各パケットは 12 バイトのヘッダ + ペイロードで構成される。

**セッションパケット** (MSB = 1):

```
byte 0-3:   1_______ ________ ________ _______R   (R = client resized flag)
byte 4-7:   width  (big-endian uint32)
byte 8-11:  height (big-endian uint32)
```

**メディアパケット** (MSB = 0):

```
byte 0-7:   0CK_____ ________ ... ________   PTS (62bit) + flags
              C = config packet (SPS/PPS)
              K = key frame (IDR)
byte 8-11:  packet size (big-endian uint32)
byte 12+:   raw H.264 Annex B データ (MediaCodec 出力そのまま)
```

## 使い方

### 前提条件

- `adb` が PATH に存在すること
- `scrcpy-server.jar` がビルド済みであること
- Android 12 以上 (Camera2 API によるカメラキャプチャの要件)

### サーバー JAR のビルド

```bash
cd server
./gradlew assembleDebug
```

ビルド成果物は `server/build/outputs/apk/debug/scrcpy-server-debug.apk` に生成される。
環境変数 `SCRCPY_SERVER_PATH` で任意のパスを指定することも可能。

### 実行例

```bash
# 4K カメラストリーム → ffplay でリアルタイム再生
./camstream | ffplay -f h264 -

# 1080p で H.264 ファイルとして保存
./camstream --size 1920x1080 > capture.h264

# 背面カメラ、高ビットレート、30fps
./camstream --camera-facing back --bitrate 20000000 --fps 30 | ffplay -f h264 -

# VLC で再生
./camstream | vlc --demux h264 -

# カメラ一覧を表示
./camstream --list-cameras
```

### コマンドラインオプション

| オプション | デフォルト | 説明 |
|---|---|---|
| `--serial`, `-s` | (自動検出) | ADB デバイスシリアル |
| `--size` | `3840x2160` | カメラキャプチャ解像度 |
| `--max-size` | `0` (無制限) | 最大辺の長さ (超過時ダウンスケール) |
| `--bitrate`, `-b` | `8000000` | 映像ビットレート (bps) |
| `--fps` | `0` (デバイス既定) | カメラ FPS |
| `--camera-id` | (先頭カメラ) | カメラ ID |
| `--camera-facing` | (指定なし) | `front` / `back` / `external` |
| `--video-codec` | `h264` | `h264` / `h265` / `av1` |
| `--encoder` | (既定エンコーダ) | デバイス固有のエンコーダ名 |
| `--server-path` | (自動検出) | scrcpy-server JAR のパス |
| `--list-cameras` | - | カメラ一覧を表示して終了 |

## scrcpy コードベースとの関係

camstream は scrcpy の以下のコンポーネントをそのまま再利用している:

| scrcpy コンポーネント | ファイル | 役割 |
|---|---|---|
| `CameraCapture` | `server/.../video/CameraCapture.java` | Camera2 API でカメラを開き Surface に描画 |
| `SurfaceEncoder` | `server/.../video/SurfaceEncoder.java` | MediaCodec HW エンコーダの設定・エンコードループ |
| `Streamer` | `server/.../device/Streamer.java` | エンコード済みパケットにメタデータを付与してソケットに書き出し |
| `DesktopConnection` | `server/.../device/DesktopConnection.java` | LocalSocket によるクライアント接続の確立 |
| `Server` | `server/.../Server.java` | エントリポイント、オプション解析、各コンポーネントの初期化 |

camstream の Python クライアントは、scrcpy の C クライアント (`app/src/demuxer.c`) が行っている処理と同等のプロトコル解析を行い、フレーミングを除去して raw H.264 を出力する。

## 制限事項

- Android 12 未満ではカメラキャプチャ非対応 (scrcpy と同じ制限)
- 4K 解像度のサポートはデバイスのカメラとエンコーダの能力に依存
- stdout がターミナルの場合はバイナリ破損を防ぐためエラー終了する
- 音声ストリームには非対応 (映像のみ)
