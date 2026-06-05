---
title: "ChromeのLINE拡張機能をアプリみたいに使おう"
emoji: "💬"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["chrome", "line", "linux"]
published: true
---

# はじめに

Linux で Chrome 版 LINE を使うとき、LINE を起動するためだけに Chrome を起動するのはかなり面倒です。

そこで今回は、Chrome 拡張機能をコマンドから直接起動し、.desktop ファイルに登録することで、アプリケーションランチャーから起動できるようにします。

# 結論

```txt
~/.local/share/applications/line.desktop
```

を作成し、以下のように記述します。

```ini
[Desktop Entry]
Type=Application
Name=LINE
Comment=LINE for Chrome
Exec=chromium --app="chrome-extension://ophjlpahpchlmihnnnihgmmeilfjmjjc/index.html#/"
Icon=/home/ユーザー名/.local/share/icons/line.png
Terminal=false
Categories=Network;InstantMessaging;
StartupNotify=true
```

`Name`, `Comment`, `Icon`, `Categories`, `StartupNotify` は自由に変更して構いません。

また、Chromium 以外のブラウザを使用している場合は、`Exec` の `chromium` の部分を使用しているブラウザのコマンド名に変更してください。

# 仕組み

Chrome や Chromium では `--app` オプションを使うことで、URL バーやタブバーのない独立したウィンドウとしてページを開くことができます。

```bash
chromium --app="[開きたいURL]"
```

また、Chrome 拡張機能のページには次のような URL が割り当てられています。

```txt
chrome-extension://[拡張機能ID]/[拡張機能内のHTMLファイル]
```

つまり、LINE 拡張機能の URL が分かれば、次のようにして直接起動することができます。

```bash
chromium --app="chrome-extension://[拡張機能ID]/[拡張機能内のHTMLファイル]"
```

## 拡張機能の URL を調べる

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

そのため、LINE 拡張機能の URL は次のように表されます。

```txt
chrome-extension://ophjlpahpchlmihnnnihgmmeilfjmjjc/index.html#/
```

実際にこの URL を Chrome または Chromium に打ち込んでみると、LINE をページとして開くことができます。

## `.desktop` ファイルを作成する

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
Icon=/home/ユーザー名/.local/share/icons/line.png
Terminal=false
Categories=Network;InstantMessaging;
StartupNotify=true
```

`Exec` には、先ほど調べたコマンドを指定しています。

`Icon` には、アプリケーションランチャーに表示したいアイコン画像のパスを指定します。
ここではLINE のアイコン画像を用意して、アイコン置き場に置いています。

Google Chrome を使っている場合は、`Exec` の `chromium` を `google-chrome-stable` などに変更します。

ここまで設定すると、rofi などのアプリケーションランチャーから `LINE` として起動できるようになります。

# おわりに

これで、Linux でも Chrome 版 LINE を通常のアプリのように起動できるようになりました。

それでは、よい LINE ライフを！
