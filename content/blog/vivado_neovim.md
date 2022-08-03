+++
title = "WindowsのVivadoのテキストエディタとしてNeovimを使用する方法"
date = 2022-07-13
[taxonomies]
categories = ["blog"]
tags = ["vivado", "fpga"]
+++

Linux用の設定例は書かれているが, Windows用がなかったのでメモとして残しておく.

Windowsの場合は, Windows Teminalを使用する.

「Tools→Settings→Text Editor」にてCustom Editorを選択し, 右側の…をクリックし, 以下のように入力すれば良い.

```
wt nvim [file name] +[line number]
```

上記はnvimへのPATHが通っている前提.
