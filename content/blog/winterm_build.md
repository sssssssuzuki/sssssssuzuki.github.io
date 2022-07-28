+++
title = "Windows Terminalを時分でビルドしてインストールする方法"
date = 2022-07-28
[taxonomies]
categories = ["blog"]
tags = ["Windows Terminal"]
[extra]
toc = true
+++

Google日本語入力を使用している場合, Windows Terminalを起動した際に入力モードが半角英数になってしまう問題が発生するようになった.

半角英数モードだと, 英数字を入力する際に日本語入力と同様にまず変換状態になる.
そのため, 確定するために余分にEnterキーを押す必要があり煩わしい.

こうなった原因を調べてみる.
Windows Terminalのアップデートが原因だろうと思い, [Release Notes](https://github.com/microsoft/terminal/releases)を調べていくと, それっぽいものが見つかった.

> The IME input mode now defaults to English when interacting with Windows Terminal (#13028) (thanks @YanceyChiew!)

余計なお世話である.

Pull Requestを見てみると, TSFInputControl inputScopeというものが`Text`から`AlphanumericHalfWidth`に変更になったらしい.
ソースを調べてみると, ハードコードされていて設定で修正はできないようなので, これを戻したものを自分でビルドすることにした.

ビルドに必要なものと手順は[README](https://github.com/microsoft/terminal)に書いてあるが, ここにもメモとして残しておく.

## 必要なもの

* You must be running Windows 10 2004 (build >= 10.0.19041.0) or later to run Windows Terminal
* You must enable Developer Mode in the Windows Settings app to locally install and run Windows Terminal
* You must have PowerShell 7 or later installed
* You must have the Windows 11 (10.0.22000.0) SDK installed
* You must have at least VS 2019 installed
* You must install the following Workloads via the VS Installer. Note: Opening the solution in VS 2019 will prompt you to install missing components automatically:
    * Desktop Development with C++
    * Universal Windows Platform Development
    * The following Individual Components
        * C++ (v142) Universal Windows Platform Tools
* You must install the .NET Framework Targeting Pack to build test projects

VS 2019と書いてあるが2022でも大丈夫だった.

## ビルド手順

まず, GithubからWindows TerminalのRepositoryをクローンしてくる.

```
git clone https://github.com/microsoft/terminal.git --recursive
```
    
recursiveをつけ忘れた場合は, 以下のコマンドでサブモジュールをアップデートする.

```
git submodule update --init --recursive
```

そして, ビルドする前に当該コードを修正する.

`src\cascadia\TerminalControl\TSFInputControl.cpp`の以下の部分を修正する.

```diff
-        _editContext.InputScope(CoreTextInputScope::AlphanumericHalfWidth);
+        _editContext.InputScope(CoreTextInputScope::Text);
```

そして, `OpenConsole.sln`を開き, ビルドタイプを`Release/x64`に設定し, ソリューションエクスプローラーから`Terminal/CascadiaPackage`を右クリックして"配置/Deploy"をクリックすれば良い.

ビルドは結構時間がかかる.

インストールされた実行ファイルは`C:\Users\<User>\AppData\Local\Microsoft\WindowsApps\wtd.exe`にある.
普通にインストールしたものは`wt.exe`で, 自分でビルドしたものは`wtd.exe`になるようだ.
