# I/Oおさらい 〜システムコールからnginxまで〜

POSIX準拠システムにおけるI/Oについてのまとめ。

## 本書の範囲

本書で説明する内容はユーザランドの範囲に留める。システムコールの**中身**については扱わない。低レイヤに興味を持った読者は、[はじめてのOSコードリーディング ~UNIX V6で学ぶカーネルのしくみ (Software Design plus) | 青柳 隆宏 |本 | 通販 | Amazon](https://www.amazon.co.jp/dp/4774154644)　などを参照されたい。

## 低レイヤ技術を学ぶ意義

- 仕組みを学ぶ楽しさ
- システムのメンタルモデル構築の助け
- ミドルウェアの実装を読む・自分で作る

## 想定読者

- 初めて学んだプログラミング言語がLL言語だという人
- OSが提供している標準機能に興味がある人
- ミドルウェアの実装に興味がある人

## Disclaimer

用意したサンプルプログラムは以下の環境でテストしているが、実際のプロダクトで用いてはいけない。移植性には気を使っていないし、エラーを無視している箇所もある。あくまでもシステムの概念の把握や実験を目的としたコードである。

## Man page of XXX

| セクション | 番号 | 内容 | 例 |
| :--- | :--- | :--- | :--- |
| ユーザコマンド | 1 | コマンドの使い方 | [Man page of LS](https://linuxjm.osdn.jp/html/gnumaniak/man1/ls.1.html), [Man page of CHOWN](https://linuxjm.osdn.jp/html/GNU_fileutils/man1/chown.1.html) |
| システムコール | 2 | システムコール | [Man page of FORK](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/fork.2.html) |
| 標準Cライブラリ | 3 | 標準Cライブラリ関数やマクロの使い方 | [Man page of PRINTF](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/printf.3.html) |
| 概念、規約、その他 | 7 | 概念、コンセプトの説明 | [Man page of PTHREADS](https://linuxjm.osdn.jp/html/LDP_man-pages/man7/pthreads.7.html) |



## 進捗 / TODO

- サンプルプログラムが全体的につまらない(ほならね)
- ソースコードへのリンク
- CI

```
          0%      100%
          [----------]
- intro   ####
- file    ######
- stream  ####
- process ###
- thread  ##
- pipe    ##
- fifo    #
- network ##
- unix    #
- event   #
- libev   #
- memc    #
```
