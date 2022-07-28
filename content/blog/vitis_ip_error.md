+++
title = "Vitis 2022.1で自作IPを使うとビルド時にエラーが出る"
date = 2022-07-14
[taxonomies]
categories = ["blog"]
tags = ["vivado", "vitis", "zynq", "fpga"]
+++

Arty Z7で開発時に, 自作IPを含んだPlatform Projectのビルド時にエラーが発生する現象に遭遇した.

いまいち原因は分からないが, とりあえず対処療法的に治ったのでメモとして残しておく.

エラーを見るに
`\<platform project\>/ps7_cortexa9_0/bsp/ps7_cortexa9_0/libsrc/\<IP name\>/src`
及び,
`\<platform project\>/zynq_fsbl/zynq_fsbl_bsp/ps7_cortexa9_0/libsrc/\<IP name\>/src`
の`Makefile`が間違っているっぽい.

デフォルトで用意されているIP (例えば, AXI_GPIOとか) はビルドに成功するので, その`Makefile`で上記の自作IPの`Makefile`を上書きすればエラーは出なくなった.

ただし, ハードウェアやBSP settingsを更新するたびに同じ作業が必要になる.
どうにかならないものか.

