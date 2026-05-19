---
title: 'デュアルブートLinuxでBluetoothマウスを使えるようにする'
emoji: "🖱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["デュアルブート", "linux", "windows", "endeavouros", "bluetooth"]
published: true
---

# 概要

Windows 11 / EndeavourOS のデュアルブート環境で、Bluetooth マウスを両方の OS から使えるように設定したときの備忘録です。

通常、デュアルブート環境で Bluetooth マウスを片方の OS にペアリングすると、OS ごとに異なるペアリングキーが生成され、もう片方の OS では接続できなくなることがあります。

この記事では、Windows 側のペアリング情報を取得し、Linux 側の BlueZ の設定に反映することで両方のOSからマウスを使えるようにするための手順をまとめます。

設定にあたっては、ArchWiki の Bluetooth ページを参考にしました。

# 環境

今回使用した環境は以下の通りです。

## PC

- Host: Inspiron 14 5430
- OS1: Windows 11 Home
  - Version: 25H2
- OS2: EndeavourOS Linux x86_64
  - Kernel: 7.0.8-arch1-1
  - WM: i3
  - Session: X11

## マウス

ELECOM SHELLPHA M-SH30BBSKBK
[商品ページ](https://shop.elecom.co.jp/category/SHELLPHA/4549550273107.html)

# やること

- Linux 側でマウスをペアリングする
- Windows 側でマウスをペアリングする
- Windows 側のレジストリからペアリング情報を取得する
- Linux 側のペアリング情報を Windows 側の情報に書き換える

# 最初に注意

マウスによっては、ペアリングのたびに MAC アドレスが変わるものがあります。
この場合、Linux 側から見た MAC アドレスと Windows 側から見た MAC アドレスが異なるため、両方の MAC アドレスを確認しておく必要があります。

# Linux側のペアリング

ここでは、Linux 側でマウスをペアリングする方法を説明します。

## パッケージの用意

`bluez` と `bluez-utils` が入っているか確認します。入っていなければインストールします。

```bash
sudo pacman -S bluez bluez-utils
```

## bluetooth.service の起動

bluetooth.service が起動しているか確認します。

```bash
sudo systemctl status bluetooth.service
```

`enabled` になっていなければ、以下のコマンドで有効化して起動します。

```bash
sudo systemctl enable --now bluetooth.service
```

## bluetoothctl を使ってペアリングする

まず、bluetoothctl を起動します。

```bash
bluetoothctl
```

Bluetooth デバイスのスキャンを開始します。

```text
scan on
```

検出されたデバイスを確認します。

```text
devices
```

以下のように表示されるので、対象デバイスの MAC アドレスを控えておきます。

```text
Device XX:XX:XX:XX:XX:XX デバイス名
```

その MAC アドレスを指定してペアリングします。

```text
pair XX:XX:XX:XX:XX:XX
```

後で `/var/lib/bluetooth/` 以下にある `info` ファイルのディレクトリ名を編集するため、ここで確認した MAC アドレスを控えておきます。

# Windows 側のペアリング

ここでは、Windows 側でマウスをペアリングする方法を説明します。

## 接続

Windows の設定から `Bluetooth とデバイス` > `デバイス` > `デバイスの追加` を開き、マウスをペアリングします。

## デバイスの MAC アドレスの確認

デバイスマネージャーを開き、`Bluetooth` の項目から対象のマウスを選択してプロパティを開きます。

`詳細` タブで `ハードウェア ID` を選択すると、MAC アドレスを確認できます。

今回は以下のような形式で表示されていました。

```text
BTHLE\Dev_xxxxxxxxxxxx
```

# レジストリ情報の取得

Windows 側の Bluetooth ペアリング情報は、通常の管理者権限では閲覧できない場所に保存されています。そのため、`regedit.exe` を SYSTEM アカウントとして起動する必要があります。

ここでは、Microsoft Sysinternals の [PsExec](https://docs.microsoft.com/en-us/sysinternals/downloads/psexec) を使います。

まず、[PsTools](https://download.sysinternals.com/files/PSTools.zip) をダウンロードし、適当な場所に解凍します。
その後、解凍したディレクトリで管理者権限のコマンドプロンプトを開き、以下のコマンドを実行します。

```cmd
.\PsExec64.exe -s -i regedit.exe
```

:::message alert
この状態でレジストリエディタを操作すると、通常のユーザーアカウントではなく SYSTEM アカウントの権限でレジストリを操作することになります。

誤ってレジストリキーを削除・変更すると、システムの安定性やセキュリティに重大な影響を与える可能性があります。

基本的には、必要なキーを確認してエクスポートするだけにしてください。
:::

`regedit.exe` が起動したら、以下のパスを開きます。

```text
HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\BTHPORT\Parameters\Keys
```

自分の PC の Bluetooth アダプタの MAC アドレスに対応するキーを開きます。このキー名では、MAC アドレスの `:` は省略されています。

その中に、ペアリングしたマウスの MAC アドレスに対応するキーを右クリックし、`.reg` ファイルとしてエクスポートします。

エクスポートした `.reg` ファイルは、Linux 側からも参照できる場所に保存しておきます。

# Linux 側のペアリング情報の書き換え

ここでは、Windows 側のペアリング情報を Linux 側の BlueZ の設定に反映する方法を説明します。

## Bluetooth 形式の確認

`.reg` ファイルの中身によって、Linux 側の `info` ファイルへ書き込む内容が異なります。

`.reg` ファイル内にペアリングキーが 1 つだけある場合は、従来の `[LinkKey]` を使う形式です。
一方、`LTK` や `IRK` など複数のキーが含まれている場合は、Bluetooth LE 系のキーを BlueZ の形式に変換して書き込む必要があります。

### 5.1 以降の形式の変換方法

Windows 側の `.reg` に `LTK`, `IRK`, `ERand`, `EDIV` などが含まれている場合は、BlueZ の `info` に書き込む形式へ変換します。

デバイスによって、Windows 側の `.reg` から BlueZ の `info` へ書き込む値の変換方法が異なります。

代表的な変換方法は以下の通りです。

なお、以下のデバイスごとの変換方法は ArchWiki の Bluetooth ページを参考にしています。
実際の値はデバイスや環境によって異なる可能性があるため、`.reg` の内容と BlueZ の `info` の形式を確認しながら作業してください。

#### Logitech MX Master 3 / Xbox One S Wireless Controller

| 項目名 | 変換方法 |
|---|---|
| `IRK` | コピーし、16 進数の桁間のスペースやカンマを除去する。 |
| `LTK` | コピーし、16 進数の桁間のスペースやカンマを除去する。 |
| `ERand` | `0` にする。 |
| `EDIV` | `0` にする。 |

#### ThinkPad TrackPoint Keyboard II / Pebble M350 マウス / Logitech G604 Lightspeed マウス

| 項目名 | 変換方法 |
|---|---|
| `IRK` | コピーし、バイト順を逆にする。 |
| `LTK` | コピーし、16 進数の桁間のスペースやカンマを除去する。 |
| `ERand` | コピーし、バイト順を逆にしてから 10 進数に変換する。 |
| `EDIV` | コピーし、バイト順を逆にしてから 10 進数に変換する。 |

#### その他のデバイス

| 項目名 | 変換方法 |
|---|---|
| `LTK` | コピーし、16 進数の桁間のスペースやカンマを除去する。 |
| `ERand` | コピーし、バイト順を逆にしてから 10 進数に変換する。 |
| `EDIV` | 16 進数を 10 進数に変換する。 |

#### Xbox ワイヤレスコントローラー

| 項目名 | 変換方法 |
|---|---|
| `LTK` | コピーし、16 進数の桁間のスペースやカンマを除去する。 |

たとえば、Windows 側の `.reg` でその他のデバイスに当てはまる以下の場合を考えます。

```reg
"LTK"=hex:e1,4a,e7,53,15,ca,c6,89,bf,f9,24,13,18,fd,14,58
"KeyLength"=dword:00000010
"ERand"=hex(b):77,ff,f8,84,8e,2b,c2,ea
"EDIV"=dword:00004601
"AuthReq"=dword:0000002d
````

この場合、Linux 側の `info` には以下のように書きます。

```ini
[LongTermKey]
Key=E14AE75315CAC689BFF9241318FD1458
Authenticated=1
EncSize=16
EDiv=17921
Rand=16916131041473265527
```

変換内容は以下の通りです。

| Windows側の項目 | Windows側の値                                        | Linux側の項目       | Linuxに書く値                          |
| ----------- | ------------------------------------------------- | --------------- | ---------------------------------- |
| `LTK`       | `e1,4a,e7,53,15,ca,c6,89,bf,f9,24,13,18,fd,14,58` | `Key`           | `E14AE75315CAC689BFF9241318FD1458` |
| `KeyLength` | `00000010`                                        | `EncSize`       | `16`                               |
| `ERand`     | `77,ff,f8,84,8e,2b,c2,ea`                         | `Rand`          | `16916131041473265527`             |
| `EDIV`      | `00004601`                                        | `EDiv`          | `17921`                            |
| `AuthReq`   | `0000002d`                                        | `Authenticated` | `1`                                |

`ERand` は `hex(b)` で保存されているため、バイト順を逆にしてから10進数に変換します。

```text
77 ff f8 84 8e 2b c2 ea
↓
ea c2 2b 8e 84 f8 ff 77
↓
0xeac22b8e84f8ff77
↓
16916131041473265527
```

`EDIV` は `dword` なので、16進数として10進数に変換します。

```text
0x4601 = 17921
```

## info ファイルの書き換え

まず root になります。

```bash
su
```

次に、bluetooth アダプタのディレクトリへ移動します。

```bash
cd /var/lib/bluetooth/[bluetoothアダプタのMACアドレス]
```

ここで、マウスの MAC アドレスが windows 側と linux 側で異なる場合は、linux 側で作成されたマウスのディレクトリ名を windows 側の MAC アドレスに変更します。

```bash
mv [linux側のマウスのMACアドレス] [windows側のマウスのMACアドレス]
```

次に、対象デバイスの `info` ファイルを開きます。

```bash
vim [マウスのMACアドレス]/info
```

`.reg` ファイルにペアリングキーのみが含まれている場合は、`[linkkey]` 以下の `key` を windows 側の値に変更します。

```ini
[linkkey]
key=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

`ltk` や `irk` などが含まれている場合は、`[longtermkey]` 以下を windows 側の値から変換した内容に変更します。

```ini
[longtermkey]
key=[変換後のltk]
authenticated=[変換後または既存の値]
encsize=[変換後または既存の値]
ediv=[変換後のediv]
rand=[変換後のerand]
```

編集後、bluetooth サービスを再起動します。

```bash
systemctl restart bluetooth.service
```

これで、windows 側で最後にペアリングした bluetooth マウスを linux 側からも接続できるようになります。

# 参考

- [ArchWiki - Bluetooth](https://wiki.archlinux.jp/index.php/Bluetooth)
- [ArchWiki - Bluetooth / Dual boot pairing](https://wiki.archlinux.org/title/Bluetooth#Dual_boot_pairing)
- [bluetoothctl のコマンド一覧と使い方をまとめてみた](https://qiita.com/noraworld/items/55c0cb1eb52cf8dccc12#devices)
