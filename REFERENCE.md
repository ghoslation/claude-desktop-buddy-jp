# Hardware Buddy BLE プロトコル

これは Claude デスクトップアプリが Bluetooth LE 経由で話すワイヤープロトコルです。
実装するために、このリポジトリ内のものは不要です。Nordic UART Service を
アドバタイズし、改行区切り JSON をパースできるデバイスなら動作します。
Arduino、ESP32、nRF52、BLE ドングル付き Raspberry Pi などで実装できます。

## ブリッジの有効化

BLE ブリッジはデフォルトではオフです。macOS 版または Windows 版の Claude で、
次を実行します。

1. **Help → Troubleshooting → Enable Developer Mode** を選びます。メニューバーに
   **Developer** メニューが追加されます。
2. **Developer → Open Hardware Buddy…** を選びます。ペアリングウィンドウが開きます。
3. **Connect** をクリックし、スキャン一覧からデバイスを選択します。初回利用時は
   OS が Bluetooth 権限を求めます。

ペアリング後、ブリッジはバックグラウンドで自動再接続します。このウィンドウが必要なのは、
初回ペアリング、統計パネルの確認、またはフォルダーのドロップ先として使う場合だけです。

## トランスポート

**BLE Nordic UART Service**（事実上の serial-over-BLE 標準）:

|                               | UUID                                   |
| ----------------------------- | -------------------------------------- |
| Service                       | `6e400001-b5a3-f393-e0a9-e50e24dcca9e` |
| RX (desktop → device, write)  | `6e400002-b5a3-f393-e0a9-e50e24dcca9e` |
| TX (device → desktop, notify) | `6e400003-b5a3-f393-e0a9-e50e24dcca9e` |

デバイスピッカーが絞り込めるよう、Nordic UART Service 上で `Claude` から始まる名前を
アドバタイズしてください。BT MAC の数バイトを末尾に付けると、複数デバイスを
ピッカー上で区別しやすくなります。

ワイヤー上のすべてのデータは UTF-8 JSON です。1 行に 1 オブジェクトを置き、
`\n` で終端します。デスクトップ側は複数パケットに分かれた行を再構成します
（通知は MTU 境界で分割されるため、単にバイト列を送れば問題ありません）。
デバイス側も同様に、`\n` を受け取るまでバイトを蓄積し、その後パースしてください。

## ハートビートスナップショット

デスクトップアプリは、何かが変わるたびにハートビートスナップショットを送信します。
加えて、10 秒ごとに keepalive を送信します。

```json
{
  "total": 3,
  "running": 1,
  "waiting": 1,
  "msg": "approve: Bash",
  "entries": ["10:42 git push", "10:41 yarn test", "10:39 reading file..."],
  "tokens": 184502,
  "tokens_today": 31200,
  "prompt": {
    "id": "req_abc123",
    "tool": "Bash",
    "hint": "rm -rf /tmp/foo"
  }
}
```

| Field          | 意味                                                                              |
| -------------- | --------------------------------------------------------------------------------- |
| `total`        | すべてのセッション数                                                              |
| `running`      | 生成中のセッション数                                                              |
| `waiting`      | 権限プロンプトでブロックされているセッション数                                    |
| `msg`          | 小さな画面に適した 1 行の概要                                                     |
| `entries`      | 最近のトランスクリプト行。新しいものが先頭（数件に制限）                          |
| `tokens`       | デスクトップアプリ起動後の累積出力トークン数                                      |
| `tokens_today` | ローカルの午前 0 時以降の出力トークン数（永続化され、再起動後も残る）              |
| `prompt`       | 権限判断が必要なときのみ存在します。`id` はデバイスから返す値です                 |

便利な派生シグナルとして、`running > 0` は少なくとも 1 つのセッションが生成中であること、
`waiting > 0` は権限プロンプトがブロックしていること、`total == 0` は何も開いていないことを
意味します。日次カウンターが必要な場合、`tokens_today` はローカルの午前 0 時にリセットされます。

約 30 秒間スナップショットを受信しない場合は、接続が切れたものとして扱ってください。

## ターンイベント

完了した各ターンでは、raw SDK content array を含む 1 回限りのイベントも送信されます。
内容にはテキストブロック、ツール呼び出し、メッセージ内のその他の content が含まれます。
シリアライズ後に 4KB を超えるイベントは破棄されます（文字数ではなく UTF-8 バイト数で測定）。

```json
{
  "evt": "turn",
  "role": "assistant",
  "content": [{ "type": "text", "text": "..." }]
}
```

## 権限判断

`prompt` が存在する場合、デバイスは応答を返せます。次のいずれかを送信してください。

```json
{"cmd":"permission","id":"req_abc123","decision":"once"}
{"cmd":"permission","id":"req_abc123","decision":"deny"}
```

`id` は `prompt.id` と完全に一致する必要があります。デスクトップはこれを
セッションマネージャーに転送します。`"once"` はツール呼び出しを承認し、
`"deny"` は拒否します。

## 接続時の 1 回限りの送信

時刻同期（epoch 秒 + タイムゾーンオフセット秒）:

```json
{ "time": [1775731234, -25200] }
```

所有者名（アカウント上のユーザーのファーストネーム）:

```json
{ "cmd": "owner", "name": "Felix" }
```

## コマンドと ack

デスクトップが `cmd` フィールド付きで送るコマンドは、一致する ack を期待します。

```json
{ "ack": "<same as cmd>", "ok": true, "n": 0 }
```

実行できなかった場合は `ok:false` を設定し、必要なら `error:"..."` を付けてください。
`n` は汎用カウンターです（chunk ack では書き込み済みバイト数、それ以外では通常 0）。

| Command                          | Payload                    | 返す ack                     |
| -------------------------------- | -------------------------- | ---------------------------- |
| `{"cmd":"status"}`               | —                          | 下記の Status response を参照 |
| `{"cmd":"name","name":"Clawd"}`  | デバイス表示名を設定       | `{"ack":"name","ok":true}`   |
| `{"cmd":"owner","name":"Felix"}` | 所有者名を設定             | `{"ack":"owner","ok":true}`  |
| `{"cmd":"unpair"}`               | 保存済み BLE bond を消去   | `{"ack":"unpair","ok":true}` |

**Status response.** デスクトップは Hardware Buddy ウィンドウの統計パネルを埋めるため、
数秒ごとにこれをポーリングします。

```json
{
  "ack": "status",
  "ok": true,
  "data": {
    "name": "Clawd",
    "sec": true,
    "bat": { "pct": 87, "mV": 4012, "mA": -120, "usb": true },
    "sys": { "up": 8412, "heap": 84200 },
    "stats": { "appr": 42, "deny": 3, "vel": 8, "nap": 12, "lvl": 5 }
  }
}
```

持っていないフィールドは省略できます。`bat.mA` が負の値なら充電中を意味します。

## フォルダープッシュ

Hardware Buddy ウィンドウにはドロップ先があります。そこにフォルダーをドロップすると、
フラットな内容がデバイスにストリーミングされます。このトランスポートは内容に依存しません。
GIF、設定 blob、ファームウェアイメージなど、合計 1.8MB 未満なら任意のものを扱えます。

```
desktop:  {"cmd":"char_begin","name":"bufo","total":184320}
device:   {"ack":"char_begin","ok":true}

desktop:  {"cmd":"file","path":"manifest.json","size":412}
device:   {"ack":"file","ok":true}
desktop:  {"cmd":"chunk","d":"<base64>"}
device:   {"ack":"chunk","ok":true,"n":<bytes_written_so_far>}
          ...repeat chunk until file is done...
desktop:  {"cmd":"file_end"}
device:   {"ack":"file_end","ok":true,"n":<final_size>}

          ...repeat file/chunk/file_end for each file...

desktop:  {"cmd":"char_end"}
device:   {"ack":"char_end","ok":true}
```

デスクトップはフォルダー内の通常ファイルをすべて送信します
（再帰なし、dotfile はスキップ）。各 chunk を base64 エンコードし、次を送る前に
各 ack を待ちます。デバイス側ではデコードして追記します。プロトコルは逐次的なので、
ファイル全体をバッファする必要はありません。

`char_begin.name` は通常フォルダー名です。ただし、フォルダーに `"name"` フィールドを
含む `manifest.json` がある場合は、その値が優先されます。

プッシュされたファイルを受け取りたくない場合は、`char_begin` に ack を返さないでください。
デスクトップは数秒後にタイムアウトし、失敗したことをユーザーに伝えます。

## セキュリティとペアリング

デスクトップアプリは、デバイスがリンク暗号化を要求するかどうかに関係なく接続します。
ただし、トランスクリプトの抜粋やツール呼び出しのヒントがこのリンクを流れるため、
暗号化されていないデバイスは、無線範囲内にいる人が安価な nRF ドングルで傍受できます。
**LE Secure Connections bonding** を必須にすることを推奨します。NUS characteristic
（および TX CCCD）を encrypted-only にし、DisplayOnly IO capability をアドバタイズします。
最初の GATT アクセスで OS ペアリングが開始され、デスクトップはデバイスに表示された
6 桁の passkey をユーザーに求めます。その後、リンクは AES-CCM で暗号化されます。
再接続では保存済み LTK が再利用され、再度プロンプトは出ません。

デスクトップアプリは暗号化デバイスと非暗号化デバイスの両方をサポートします。
ペアリングに関係するプロトコルフックは 2 つです。

- リンクが暗号化されたら、status ack の `data` に `"sec": true` を含めます
  （bond しない場合は `false`、または省略）。
- `{"cmd":"unpair"}` を受け取ったら、保存済み bond を消去します。ユーザーが
  **Forget** をクリックすると、デスクトップがこれを送信します。次回ペアリング時には
  新しい passkey が表示されます。他のコマンドと同じように ack してください。

フォルダープッシュプロトコルを受け入れる場合は、書き込み前に `file.path` を検証してください。
デスクトップはドロップされたフォルダー内のファイル名をそのまま送信するため、
上書きされて困るものがファイルシステム上にあるなら、`..` や絶対パスは拒否してください。

## 利用可能性

BLE API は、デスクトップアプリが開発者モードのときのみ利用できます
（**Help → Troubleshooting → Enable Developer Mode**）。これはメイカーや
開発者向けのものであり、公式にサポートされる製品機能ではありません。
