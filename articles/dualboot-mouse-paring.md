---
title: 'デュアルブート環境（Windows / Linux）でBluetoothデバイスを共有する'
emoji: "🖱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["デュアルブート", "linux", "windows", "endeavouros", "bluetooth"]
published: true
---

# はじめに

Windows と Linux のデュアルブート環境で、同じ Bluetooth デバイスを両方の OS から使いたいことがあります。

しかし、片方の OS でペアリングしたあとに、もう片方の OS でもペアリングし直すと、最初に設定した OS 側で接続できなくなってしまいます。

これは、Bluetooth のペアリング時に「PC 側のアダプタの MAC アドレス」と「デバイス側の MAC アドレス」に紐づく共通のペアリングキーが生成されるためです。デュアルブート環境では両方の OS が同じ Bluetooth アダプタ（同じ MAC アドレス）を使うため、後からペアリングした OS のキーでデバイス側の記憶が上書きされ、前の OS のキーと一致しなくなってしまいます。

この問題は、**Windows 側で生成されたペアリングキーを Linux（BlueZ）の設定ファイルにコピーする**ことで解決できます。

今回は例として Bluetooth マウスを使って手順を説明します。

# 環境

今回作業した環境は以下の通りです（設定には [ArchWiki の Bluetooth ページ](https://wiki.archlinux.org/title/Bluetooth#Dual_boot_pairing)を参考にしました）。

- **PC**: Dell Inspiron 14 5430
  - **OS 1**: Windows 11 Home (25H2)
  - **OS 2**: EndeavourOS Linux x86_64 (Kernel: 7.0.8-arch1-1 / WM: i3 / X11)
- **Bluetooth デバイス:
  - マウス: [ELECOM SHELLPHA M-SH30BBSKBK](https://shop.elecom.co.jp/category/SHELLPHA/4549550273107.html)

# 全体の流れ

以下の順番で作業を進めます。

1. **Linux 側**でデバイスをペアリングする（設定ファイルの雛形を作るため）
2. **Windows 側**でデバイスをペアリングする（デバイス本体に Windows 側のキーを記憶させるため）
3. **Windows 側**のレジストリからペアリング情報をエクスポートする
4. **Linux 側**の設定ファイル（`info`）を Windows 側のペアリング情報に書き換える

:::message
**ペアリングごとに MAC アドレスが変わるデバイスについて**
Bluetooth LE（BLE）対応のマウスやキーボードなどでは、**ペアリングし直すたびにデバイス側の MAC アドレスが変化する**ものがあります。
その場合、Linux でペアリングしたときの MAC アドレスと、Windows でペアリングしたときの MAC アドレスが異なるため、作業中に**両方の MAC アドレスを控えておく**必要があります。
:::

---

# 1. Linux 側でペアリングする

まずは Linux 側でデバイスをペアリングし、`/var/lib/bluetooth/` 配下に設定ファイルを作らせます。
（GUI の Bluetooth 設定ツールからペアリングしても構いません。ここでは `bluetoothctl` を使う手順を記載します）

## パッケージとサービスの確認

`bluez` と `bluez-utils` が入っていなければインストールし、`bluetooth.service` を起動します。

```bash
sudo pacman -S bluez bluez-utils
sudo systemctl enable --now bluetooth.service
```

## `bluetoothctl` でペアリング

`bluetoothctl` を起動し、スキャンを開始します。

```bash
bluetoothctl
```

```text
[bluetooth]# scan on
```

接続したいデバイスをペアリングモードにして、検出された一覧を確認します。

```text
[bluetooth]# devices
Device AA:BB:CC:DD:EE:FF デバイス名
```

対象デバイスの MAC アドレスを指定してペアリングします。

```text
[bluetooth]# pair AA:BB:CC:DD:EE:FF
```

:::message
ここでペアリングした **デバイスの MAC アドレス（Linux 側）** を控えておいてください。
:::

---

# 2. Windows 側でペアリングする

次に Windows を起動し、同じデバイスをペアリングします。

## デバイスの接続

Windows の「設定」から `Bluetooth とデバイス` > `デバイス` > `デバイスの追加` を開き、デバイスをペアリングします。

## デバイスの MAC アドレスの確認

Windows 側で認識されているデバイスの MAC アドレスを確認します。

1. **デバイスマネージャー**を開く
2. `Bluetooth` の項目から対象のデバイス（今回はマウス）を右クリックし、**プロパティ**を開く
3. `詳細` タブを開き、プロパティから **ハードウェア ID** を選択する

以下のように表示されている末尾 12 桁の英数字がデバイスの MAC アドレスです。

```text
BTHLE\Dev_aabbccddeeff
```

:::message
ここで確認した **デバイスの MAC アドレス（Windows 側）** も控えておいてください。
:::

---

# 3. Windows のレジストリからペアリング情報を取得する

Windows の Bluetooth ペアリング情報は、通常の管理者権限では閲覧できない領域に保存されています。そのため、Microsoft 公式ツールである [PsExec](https://learn.microsoft.com/ja-jp/sysinternals/downloads/psexec) を使って、レジストリエディタ（`regedit.exe`）を `SYSTEM` アカウント権限で起動します。

## SYSTEM 権限でレジストリエディタを起動する

1. [PsTools](https://download.sysinternals.com/files/PSTools.zip) をダウンロードし、任意のフォルダに解凍します。
2. 解凍したフォルダで**管理者権限のコマンドプロンプト（または PowerShell）**を開き、以下のコマンドを実行します。

```cmd
.\PsExec64.exe -s -i regedit.exe
```

:::message alert
この状態のレジストリエディタは `SYSTEM` アカウントの権限で動作しています。
誤って関係ないキーを削除・変更すると、システムが起動しなくなるなど重大な問題につながる可能性があります。**必要なキーを確認してエクスポートするだけ**にとどめてください。
:::

## ペアリング情報のエクスポート

起動したレジストリエディタで、以下のパスを開きます。

```text
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\BTHPORT\Parameters\Keys
```

`Keys` の下に、**PC の Bluetooth アダプタの MAC アドレス**（`:` なしの 12 桁）の名前のフォルダ（キー）があります。それを開くと、以下のいずれかの形式でデバイスのペアリング情報が保存されています。

- **Bluetooth LE（BLE）デバイスの場合**:
  アダプタのキーの下に、さらに**デバイスの MAC アドレス**の名前のサブキーがあり、その中に `LTK`、`ERand`、`EDIV`、`IRK` などの複数の値が入っています。
- **従来の Bluetooth デバイスの場合**:
  アダプタのキーの直下に、**デバイスの MAC アドレス**の名前のバイナリ値（1 つだけ）が保存されています。

対象のデバイスのキー（またはアダプタのキー）を右クリックし、**エクスポート**を選択して `.reg` ファイルとして保存します。
保存した `.reg` ファイルは、USB メモリや共有パーティションなど、**Linux 側から参照できる場所**に置いておきます。

---

# 4. Linux 側のペアリング情報を書き換える

再び Linux を起動し、先ほどエクスポートした `.reg` ファイルの中身を Linux（BlueZ）の設定ファイルに反映します。

## 4-1. 設定ファイルの場所へ移動する

`/var/lib/bluetooth/` 配下は root 権限が必要なため、root に切り替えて移動します。

```bash
sudo -i
cd /var/lib/bluetooth/<BluetoothアダプタのMACアドレス>
```

### MAC アドレスが変化していた場合

Linux でペアリングしたときと Windows でペアリングしたときで**デバイスの MAC アドレスが異なっていた場合**は、ディレクトリ名を Windows 側の MAC アドレス（コロン `:` 区切りの大文字）に変更します。

```bash
# 例: Linux側が AA:BB:CC:DD:EE:11、Windows側が AA:BB:CC:DD:EE:22 だった場合
mv AA:BB:CC:DD:EE:11 AA:BB:CC:DD:EE:22
```

続いて、対象デバイスのディレクトリ内にある `info` ファイルをエディタで開きます。

```bash
vim <Windows側のデバイスのMACアドレス>/info
```

---

## 4-2. `.reg` の値を `info` 用の形式に変換して書き込む

エクスポートした `.reg` ファイルの中身に応じて、`info` ファイルを書き換えます。

### パターン A：従来の Bluetooth デバイス

`.reg` ファイルにデバイスの MAC アドレスを名前とした 16 進数の値だけが書かれている場合です。

```reg
"aabbccddeeff"=hex:01,23,45,67,89,ab,cd,ef,01,23,45,67,89,ab,cd,ef
```

この場合は、カンマ `,` を取り除いて大文字にし、`info` ファイルの `[LinkKey]` セクションの `Key` にそのまま貼り付けます。

```ini
[LinkKey]
Key=0123456789ABCDEF0123456789ABCDEF
```

### パターン B：Bluetooth LE デバイス

最近のマウスやキーボード（今回の ELECOM マウスも含む）の多くはこちらの形式です。
たとえば、`.reg` ファイルに以下のような値が出力されているとします。

```reg
"LTK"=hex:e1,4a,e7,53,15,ca,c6,89,bf,f9,24,13,18,fd,14,58
"KeyLength"=dword:00000010
"ERand"=hex(b):77,ff,f8,84,8e,2b,c2,ea
"EDIV"=dword:00004601
"AuthReq"=dword:0000002d
```

これらはそのまま貼り付けるのではなく、以下のように変換して `info` ファイルの `[LongTermKey]` セクションに書き込みます。

| Windows（`.reg`）の項目 | Linux（`info`）の項目 | 変換ルール |
| --- | --- | --- |
| `LTK` | `Key` | カンマを除去して大文字にする |
| `AuthReq` | `Authenticated` | 10 進数に変換する（または既存の `info` の値を維持） |
| `KeyLength` | `EncSize` | 16 進数から 10 進数に変換する（`00000010` → `16`） |
| `EDIV` | `EDiv` | 16 進数から 10 進数に変換する |
| `ERand` | `Rand` | **バイト順を逆にしてから** 10 進数に変換する |

#### `EDIV` と `ERand` の変換例

- **`EDIV`（`dword:00004601`）の変換**:
  そのまま 16 進数 `0x4601` として 10 進数に直します。

  ```text
  0x4601 → 17921
  ```

- **`ERand`（`hex(b):77,ff,f8,84,8e,2b,c2,ea`）の変換**:
  リトルエンディアンで保存されているため、**2 桁（1 バイト）単位で並び順を逆にしてから** 10 進数に直します。

  ```text
  元の値:       77 ff f8 84 8e 2b c2 ea
  逆順にする:   ea c2 2b 8e 84 f8 ff 77
  16進数表記:   0xeac22b8e84f8ff77
  10進数に変換: 16916131041473265527
  ```

結果として、`info` ファイルの `[LongTermKey]` セクションは以下のようになります。

```ini
[LongTermKey]
Key=E14AE75315CAC689BFF9241318FD1458
Authenticated=1
EncSize=16
EDiv=17921
Rand=16916131041473265527
```

:::details デバイスごとの特殊な変換パターン（MX Master 3 / ThinkPad キーボード / Xbox コントローラーなど）
デバイスによっては、`IRK`（Identity Resolving Key）の変換が必要だったり、`ERand` や `EDIV` の扱いが上記と異なったりする場合があります。うまく接続できない場合は、[ArchWiki](https://wiki.archlinux.org/title/Bluetooth#Dual_boot_pairing) に記載されている以下のパターンを試してみてください。

#### Logicool MX Master 3 / Xbox One S Wireless Controller

| 項目名 | 変換方法 |
| --- | --- |
| `IRK`（`[IdentityResolvingKey]` の `Key`） | カンマを除去して大文字にする |
| `LTK`（`[LongTermKey]` の `Key`） | カンマを除去して大文字にする |
| `ERand`（`Rand`） | `0` にする |
| `EDIV`（`EDiv`） | `0` にする |

#### ThinkPad TrackPoint Keyboard II / Logicool Pebble M350 / Logicool G604 Lightspeed

| 項目名 | 変換方法 |
| --- | --- |
| `IRK`（`[IdentityResolvingKey]` の `Key`） | **バイト順を逆にして**大文字にする |
| `LTK`（`[LongTermKey]` の `Key`） | カンマを除去して大文字にする |
| `ERand`（`Rand`） | バイト順を逆にしてから 10 進数に変換する |
| `EDIV`（`EDiv`） | **バイト順を逆にしてから** 10 進数に変換する |

#### Xbox ワイヤレスコントローラー

| 項目名 | 変換方法 |
|---|---|
| `LTK`（`[LongTermKey]` の `Key`） | カンマを除去して大文字にする |
:::

---

## 4-3. Bluetooth サービスを再起動する

`info` ファイルを保存したら、`bluetooth.service` を再起動して設定を反映します。

```bash
systemctl restart bluetooth.service
```

これで、Windows と Linux のどちらを起動しても、再ペアリングなしでそのまま Bluetooth デバイスが使えるようになります。

# 参考

- [ArchWiki - Bluetooth (日本語)](https://wiki.archlinux.jp/index.php/Bluetooth)
- [ArchWiki - Bluetooth / Dual boot pairing](https://wiki.archlinux.org/title/Bluetooth#Dual_boot_pairing)
- [bluetoothctl のコマンド一覧と使い方をまとめてみた](https://qiita.com/noraworld/items/55c0cb1eb52cf8dccc12#devices)
