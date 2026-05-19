---
title: 'LinuxでBluetoothマウスをペアリングする'
emoji: "🖱"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["linux", "endeavouros", "bluetooth"]
published: false
---

# 概要

EndeavouOS環境でノートパソコンをマウスをbluetooth接続する方法のメモ書き。

# 環境

## PC

- OS: EndeavourOS Linux x86_64
- Host: Inspiron 14 5430
- Kernel: 7.0.8-arch1-1
- WM: i3
- Session: X11

## マウス

ELECOM SHELLPHA M-SH30BBSKBK

<https://shop.elecom.co.jp/category/SHELLPHA/4549550273107.html>

# ペアリングの方法

## パッケージの用意

パッケージ `bluez`, `bluez-utils` が入っているか確認する。
入っていなければ入れる。

`sudo pacman -S bluez bluez-utils`

## bluetooth.serviceの起動

`sudo systemctl status bluetooth.service`

enableになっていなければ

`sudo systemctl enable --now bluetooth.service`

## bluetoothctlを使う

`bluetoothctl`

`scan on`

`devices`

からDevice XX:XX:XX:XX:XX:XX [デバイス名]
のXX:XX:XX:XX:XX:XXをコピーして、

`pair XX:XX:XX:XX:XX:XX`

# 参考

arch wiki <https://wiki.archlinux.jp/index.php/Bluetooth>

bluetoothctl のコマンド一覧と使い方をまとめてみた <https://qiita.com/noraworld/items/55c0cb1eb52cf8dccc12#devices>
