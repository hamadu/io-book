# ストリームI/O

普通のファイルに対して読み書きを行うのに、前章で説明したファイルI/Oシステムコールを直接ラップした関数を使うことはあまりない。代わりに、本章で説明するバッファリング機能を備えてラップした、ストリーム操作関数群を使うことが多い。バッファリング機能とは、入出力のデータをある程度**貯めておき**、一定量に達したら実際の入出力を行う仕組みである。結果として、前章のシステムコール `read()` および `write()` を呼び出す回数が減り、高速化が期待できる。

## ストリームとファイルの操作

これらの関数群は、ファイルディスクリプタの代わりに、引数として **ストリーム / stream** を与えたり受け取ったりする。ストリームには、読み書き用のバッファとファイルディスクリプタが含まれ、FILE構造体で表される。


前章で見たファイルI/Oシステムコールと同様に、ストリームを通じてファイルを操作するには最初にオープンし、その後読み／書きの命令を実行、操作が終わったらクローズという流れになる。そのため、前章で紹介した関数群のストリーム版というべき関数がある。以降、順番に紹介していこう。

## ストリーム操作関数群

### fopen

ストリームを開くには、`fopen` 関数を使う。

```c
FILE* fopen(const char *filename, const char *opentype)
```
> [Man page of FOPEN](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/fopen.3.html)

`filename` にはファイルへのパス、`opentype` にはモードを指定する。ファイルのオープンに成功すれば、ストリームへのポインタが返ってくる。ストリームオブジェクトはFILE構造体で表される。この構造体は標準ライブラリ内部で扱われるため、その内容を意識する必要はない。

### fgetc, fgets

指定したストリームから文字や文字列を読み込むには、`fgetc` 関数、`fgets` 関数を使う。

> ```c
> #include <stdio.h>
>
> int fgetc(FILE *stream);
>
> char *fgets(char *s, int size, FILE *stream);
> ```
> [Man page of FGETC](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/getc.3.html)

`fgetc` 関数は指定したストリームから1文字 `unsigned char` で読んで、`int` にキャストして返す。わざわざキャストする理由はエラーハンドリングのためである。


`fgets` は指定したストリーム `stream` から改行に出会うか、`size-1` 文字に達するまで読み込み、`s` に格納する。`size` より 1文字少ない理由は、最後にヌル文字が挿入されるためである。

### fputc, fputs

指定したストリームに文字や文字列を書き出すには、`fputc` 関数、`fputs` 関数を使う。

> ```c
> #include <stdio.h>
>
> int fputc(int c, FILE *stream);
>
> int fputs(const char *s, FILE *stream);
> ```
> [Man page of PUTS](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/puts.3.html)

`fputc` 関数は指定したストリーム `stream` に文字 `c` を書き出す。

`fputs` 関数は指定したストリーム `stream` に文字列 `ｓ` を書き出す。終端のヌル文字は取り除かれる。

### fclose

指定したストリームを閉じるには、`fclose` 関数を使う。

> ```c
> #include <stdio.h>
>
> int fclose(FILE *stream);
> ```
> [Man page of FCLOSE](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/fclose.3.html)

`fclose` 関数は、内部でバッファリングされているデータを全て書き出した後、ストリーム `stream` をクローズする。

## バッファリング

### バッファリングモード

バッファに貯めたデータを実際に書き出すことを **フラッシュ / flush** と呼ぶ。フラッシュはライブラリが適切なタイミングで行うが、大まかな挙動を各ストリームごとにバッファリングモードで指定できる。

| 種類 | 説明 |
| :--- | :--- |
| アンバッファド / unbuffered | バッファリングを行わない。操作直後に、常にフラッシュを行う。 |
| 行バッファリング / line buffered | 改行ごとにフラッシュを行う。 |
| 完全バッファリング / fully buffered | バッファが貯まるまでフラッシュを行わない。 |

開いた直後のファイルに対して、このモードを変更する関数が用意されている。[→setbuf(3)](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/setbuf.3.html)

## 標準入力、標準出力、標準エラー出力

プロセスの開始と同時に、自動で開かれるストリームが3つある。標準入力、標準出力、標準エラー出力だ。

これらはデフォルトでそのプロセスに紐付いた端末 / terminal からの入出力を受け付けるようになっている。対話型のプログラムにおいて、標準入力はユーザからの入力を受けつけ、標準出力はユーザへのメッセージを出力、標準エラー出力はログ等を吐く想定になっている。

標準入出力のファイルディスクリプタは、慣例上それぞれ0\(標準入力\)、1\(標準出力\)、2\(標準エラー出力\)と決まっている。

### printf

前回の例でしれっと使った `printf` 関数を紹介しよう。これは標準出力ストリームに文字列を指定フォーマットで出力する関数である。出力先が決まっているので、ストリームの引数はない。

> ```
> #include <stdio.h>
> int printf(const char *format, ...);
> ```
>　[Man page of PRINTF](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/printf.3.html)

`printf` 関数はフォーマットされた文字列をストリームに書き込む関数群の仲間で、他にも引数としてストリームを受け取るものなどがある。




詳しくはマニュアルを。[→printf(3)](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/printf.3.html)

### 入出力のリダイレクト

システムプログラミングとは少し離れ、シェルの話になってしまうがついでなので入出力のリダイレクトについて触れておく。
プログラムをシェルから実行するとき、末尾に `> (ファイル名)` を付けることで、標準出力を指定ファイルへの出力に置き換える。

```sh
$ ls -al > list.txt
```

同様に、`< (ファイル名)` を付けると、標準入力を指定ファイル名からの読み込みに置き換える。

```sh
$ command < in.txt
```

これらを組み合わせて使うこともできる。

```sh
$ command < in.txt > out.txt
```

bashにおいて、標準出力と標準エラーを同じファイルに出力する方法として、最後に `2>&1` を付ける方法が知られている。

```sh
$ command > out.log 2>&1
```

ここに出現する 2 や 1 の数値はファイルディスクリプタのことである、と理解すると覚えやすい。詳しくはマニュアルを参照。[→bash(1)](https://linuxjm.osdn.jp/html/GNU_bash/man1/bash.1.html)

## 例: ファイルのコピー(ストリーム版)

最後に、前章で作ったファイルをコピーするプログラムを、ストリーム操作関数を用いたものに改造しよう。

```c
#include <sys/types.h>
#include <fcntl.h>
#include <errno.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void print_error_and_exit() {
  printf("error(%d): %s\n", errno, strerror(errno));
  exit(1);
}

int main(int argc, char* argv[]) {
  FILE *src = fopen("./src.txt", "r");
  if (src == NULL) {
    print_error_and_exit();
  }

  FILE *dst = fopen("./dst.txt", "w");
  if (dst == NULL) {
    print_error_and_exit();
  }

  while (1) {
    int ch = fgetc(src);
    if (ch == EOF) {
      break;
    }
    fputc(ch, dst);
  }
  fclose(src);
  fclose(dst);
  return 0;
}
```

### 性能評価

大きめのファイル(50MB程度)を用意して、前章のプログラムと実行速度を比べてみよう。速度を正確に測るには、[time(1)](https://linuxjm.osdn.jp/html/LDP_man-pages/man1/time.1.html) が便利だ。使い方は簡単で、`time` の次にスペースを入れて計測対象コマンドを入れると、実行開始から終了までの経過時間を報告してくれる。

```sh
$ time (プログラム名)
```

筆者の環境()では以下の通りとなった。ストリーム関数を用いたほうが合計実行時間が短い。前章で用いたプログラム(copy.c)のバッファ長は 16 と短く、`write` システムコールが発行される回数が多くなってしまうのでシステムCPU時間が大きくなっている。バッファをより長くすれば、システムCPU時間が減り合計実行時間の差が縮まるだろう。一方、本章で作ったプログラム(stream_copy.c)は文字を1文字ずつコピーする形式なので、ユーザ側での命令実行数が多いためユーザCPU時間が長く出ているが、`write` システムコールを発行する回数は少ないためシステムCPU時間は短い。

| 種類 | ユーザCPU時間 | システムCPU時間 | 合計実行時間 |
| :--- | :--- | :--- | :--- |
| copy.c(16byte)   | 0.76s | 7.40s | 8.635s |
| copy.c(256byte)  | 0.76s | 7.40s | 8.635s |
| copy.c(1024byte) | 0.76s | 7.40s | 8.635s |
| copy.c(4096byte) | 0.76s | 7.40s | 8.635s |
| stream_copy.c    | 4.79s | 0.28s | 5.401s |
