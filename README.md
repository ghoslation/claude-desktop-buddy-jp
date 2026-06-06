# claude-desktop-buddy

macOS 版および Windows 版の Claude は、BLE 経由で Claude Cowork や
Claude Code をメイカー向けデバイスに接続できます。これにより、開発者や
メイカーは、権限確認、最近のメッセージ、その他のインタラクションを表示する
ハードウェアを作れます。Claude の周辺でメイカーコミュニティが生み出してきた
創造性には驚かされてきました。軽量でオプトインの API を提供することは、
Claude と連携する小さく楽しいハードウェアを作りやすくするための取り組みです。

> **自分のデバイスを作る場合** このリポジトリ内のコードは不要です。
> ワイヤープロトコルについては **[REFERENCE.md](REFERENCE.md)** を参照してください。
> Nordic UART Service の UUID、JSON スキーマ、フォルダープッシュの転送方式を
> 記載しています。

例として、ESP32 上で動くデスクペットを作りました。このペットは Claude の
権限承認やインタラクションに反応して過ごします。何も起きていないと眠り、
セッションが始まると起き、承認プロンプトが待機していると目に見えてそわそわし、
デバイス上から承認または拒否できます。

<p align="center">
  <img src="docs/device.jpg" alt="buddy ファームウェアを実行している M5StickC Plus" width="500">
</p>

## ハードウェア

このファームウェアは Arduino フレームワークを使う ESP32 を対象にしています。
現状では、ディスプレイ、IMU、ボタンドライバーとして M5StickCPlus ライブラリに
依存しています。そのため、このボードを使うか、各ドライバーを自分のピン配置に
差し替えたフォークが必要です。

## 書き込み

[PlatformIO Core](https://docs.platformio.org/en/latest/core/installation/)
をインストールしてから、次を実行します。

```bash
pio run -t upload
```

以前に書き込み済みのデバイスから始める場合は、先に消去します。

```bash
pio run -t erase && pio run -t upload
```

起動後は、デバイス本体からすべてを消去することもできます。
**A を長押し → settings → reset → factory reset → 2 回タップ**。

## ペアリング

デバイスを Claude とペアリングするには、まず開発者モードを有効にします
（**Help → Troubleshooting → Enable Developer Mode**）。次に、
**Developer → Open Hardware Buddy…** から Hardware Buddy ウィンドウを開き、
**Connect** をクリックして、一覧からデバイスを選択します。初回接続時には
macOS が Bluetooth 権限を求めるので、許可してください。

<p align="center">
  <img src="docs/menu.png" alt="Developer → Open Hardware Buddy… メニュー項目" width="420">
  <img src="docs/hardware-buddy-window.png"
       alt="Connect ボタンとフォルダードロップ先を持つ Hardware Buddy ウィンドウ"
       width="420">
</p>

ペアリング後は、両側が起きている限りブリッジが自動で再接続します。

検出でスティックが見つからない場合は、次を確認してください。

- スティックが起きていること（いずれかのボタンを押す）
- スティックの設定メニューで bluetooth がオンになっていること

## 操作

|                         | 通常時                     | ペット       | 情報        | 承認待ち    |
| ----------------------- | -------------------------- | ------------ | ----------- | ----------- |
| **A**（前面）           | 次の画面                   | 次の画面     | 次の画面    | **承認**    |
| **B**（右側）           | トランスクリプトをスクロール | 次のページ   | 次のページ  | **拒否**    |
| **A 長押し**            | メニュー                   | メニュー     | メニュー    | メニュー    |
| **Power**（左側、短押し） | 画面オフ切り替え           |              |             |             |
| **Power**（左側、約 6 秒） | 強制電源オフ               |              |             |             |
| **振る**                | めまい状態                 |              |             | —           |
| **伏せる**              | 昼寝（エネルギー回復）     |              |             |             |

操作がない状態が 30 秒続くと画面は自動でオフになります（承認プロンプト表示中は
点灯したままです）。いずれかのボタンを押すと復帰します。

## ASCII ペット

18 種類のペットがあり、それぞれ 7 種類のアニメーション
（sleep、idle、busy、attention、celebrate、dizzy、heart）を持ちます。
Menu → "next pet" でカウンター付きで順に切り替わります。選択は NVS に保存されます。

## GIF ペット

ASCII buddy の代わりにカスタム GIF キャラクターを使いたい場合は、
Hardware Buddy ウィンドウのドロップ先にキャラクターパックのフォルダーを
ドラッグします。アプリが BLE 経由でストリーミングし、スティックはその場で
GIF モードに切り替わります。**Settings → delete char** で ASCII モードに戻せます。

キャラクターパックは、`manifest.json` と幅 96px の GIF を含むフォルダーです。

```json
{
  "name": "bufo",
  "colors": {
    "body": "#6B8E23",
    "bg": "#000000",
    "text": "#FFFFFF",
    "textDim": "#808080",
    "ink": "#000000"
  },
  "states": {
    "sleep": "sleep.gif",
    "idle": ["idle_0.gif", "idle_1.gif", "idle_2.gif"],
    "busy": "busy.gif",
    "attention": "attention.gif",
    "celebrate": "celebrate.gif",
    "dizzy": "dizzy.gif",
    "heart": "heart.gif"
  }
}
```

ステート値には単一のファイル名または配列を指定できます。配列はローテーションします。
各ループの終端で次の GIF に進むため、ホーム画面で 1 つのクリップを延々と
繰り返すのではなく、待機アクションのカルーセルとして使えます。

GIF は幅 96px です。高さは約 140px までなら 135×240 の縦画面に収まります。
キャラクターの周囲はタイトに切り抜いてください。透明な余白は画面を浪費し、
スプライトを小さくしてしまいます。`tools/prep_character.py` はリサイズを処理します。
任意サイズの元 GIF を渡すと、すべてのステートでキャラクターのスケールが揃った
幅 96px のセットを生成します。

フォルダー全体は 1.8MB 未満に収める必要があります。
`gifsicle --lossy=80 -O3 --colors 64` を使うと、通常は 40〜60% 削減できます。

動作する例は `characters/bufo/` を参照してください。

キャラクターを調整中で BLE の往復を省きたい場合は、
`tools/flash_character.py characters/bufo` で `data/` にステージングし、
USB 経由で `pio run -t uploadfs` を直接実行できます。

## 7 つのステート

| State       | トリガー                    | 雰囲気                         |
| ----------- | --------------------------- | ------------------------------ |
| `sleep`     | ブリッジ未接続              | 目を閉じ、ゆっくり呼吸         |
| `idle`      | 接続済み、緊急事項なし      | まばたき、周囲を見る           |
| `busy`      | セッションが実行中          | 汗をかき、作業中               |
| `attention` | 承認待ち                    | 注意喚起、**LED 点滅**         |
| `celebrate` | レベルアップ（50K トークンごと） | 紙吹雪、跳ねる             |
| `dizzy`     | スティックを振った          | 渦巻き目、ふらつき             |
| `heart`     | 5 秒以内に承認              | ハートが浮かぶ                 |

## プロジェクト構成

```
src/
  main.cpp       — ループ、ステートマシン、UI 画面
  buddy.cpp      — ASCII 種別のディスパッチと描画ヘルパー
  buddies/       — 種別ごとに 1 ファイル、各 7 つのアニメーション関数
  ble_bridge.cpp — Nordic UART service、行バッファ付き TX/RX
  character.cpp  — GIF デコードと描画
  data.h         — ワイヤープロトコル、JSON パース
  xfer.h         — フォルダープッシュ受信側
  stats.h        — NVS に保存される統計、設定、所有者、種別選択
characters/      — GIF キャラクターパックの例
tools/           — ジェネレーターとコンバーター
```

## 利用可能性

BLE API は、デスクトップアプリが開発者モードのときのみ利用できます
（**Help → Troubleshooting → Enable Developer Mode**）。これはメイカーや
開発者向けのものであり、公式にサポートされる製品機能ではありません。
