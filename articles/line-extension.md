---
title: "ChromeのLINE拡張機能をアプリみたいに使おう"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["chrome", "line", "linux"]
published: true
---

# はじめに

Linux で Chrome 版 LINE を使うとき、毎回 Chrome を起動してから LINE 拡張機能を開くのは少し面倒ではないでしょうか。

そこで今回は、Chrome 拡張機能の LINE をコマンドから直接起動し、`.desktop` ファイルを作成することで、通常のアプリのようにアプリケーションランチャーから起動できるようにします。

# 仕組み

Chrome や Chromium には、指定した URL をアプリ風のウィンドウで開く `--app` オプションがあります。

```bash
chromium --app="[開きたいURL]"
```

このオプションを使うと、URL バーやタブバーのない独立したウィンドウとして開けます。

また、Chrome 拡張機能のページは次のような URL で表されます。

```txt
chrome-extension://[拡張機能ID]/[拡張機能内のHTMLファイル]
```

つまり、LINE 拡張機能の URL が分かれば、次のようにして直接起動することができます。

```bash
chromium --app="chrome-extension://[拡張機能ID]/[拡張機能内のHTMLファイル]"
```

# やること

今回は次の2つを行います。

* LINE 拡張機能の URL を調べる
* `.desktop` ファイルを作成する

# 拡張機能の URL を調べる

まず、Chrome または Chromium で次のページを開きます。

```txt
chrome://extensions/
```

LINE 拡張機能の詳細を開くと、拡張機能 ID を確認できます。

この記事を書いている時点では、LINE 拡張機能の ID は次のようになっていました。

```txt
ophjlpahpchlmihnnnihgmmeilfjmjjc
```

次に、デベロッパーモードを有効にします。

LINE 拡張機能の欄にある `Inspect views` を確認すると、環境によっては次のように表示されます。

```txt
Inspect views
- service worker (Inactive)
- index.html
```

ここから、LINE 拡張機能の画面として `index.html` が使われていることが分かります。

そのため、LINE 拡張機能の URL は次のようになります。

```txt
chrome-extension://ophjlpahpchlmihnnnihgmmeilfjmjjc/index.html#/
```

実際にこの URL を Chrome または Chromium に打ち込んでみると、LINE をページとして開くことができます。

# `.desktop` ファイルを作成する

Linux では、`.desktop` ファイルを作成することで、アプリケーションランチャーから任意のコマンドを起動できるようになります。

今回は次の場所に `line.desktop` を作成します。

```txt
~/.local/share/applications/line.desktop
```

内容は次のようにします。

```ini
[Desktop Entry]
Type=Application
Name=LINE
Comment=LINE for Chrome
Exec=chromium --app="chrome-extension://ophjlpahpchlmihnnnihgmmeilfjmjjc/index.html#/"
Icon=~/.local/share/icons/line.png
Terminal=false
Categories=Network;InstantMessaging;
StartupNotify=true
```

`Exec` には、LINE 拡張機能をアプリ風に起動するコマンドを書いています。

```ini
Exec=chromium --app="chrome-extension://ophjlpahpchlmihnnnihgmmeilfjmjjc/index.html#/"
```

`Icon` には、アプリケーションランチャーに表示したいアイコン画像のパスを指定します。

```ini
Icon=~/.local/share/icons/line.png
```

ここでは、適当な LINE のアイコン画像を用意して、次の場所に置いています。
このパスは自分の環境に合わせて変更してください。

Google Chrome を使っている場合は、`Exec` の `chromium` を `google-chrome-stable` などに変更します。

```ini
Exec=google-chrome-stable --app="chrome-extension://ophjlpahpchlmihnnnihgmmeilfjmjjc/index.html#/"
```

ここまで設定すると、rofi などのアプリケーションランチャーから `LINE` として起動できるようになります。

# おわりに

これで、Linux でも Chrome 版 LINE を通常のアプリのように起動できるようになりました。

それでは、よい LINE ライフを！
