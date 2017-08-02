# ファイルI/O

プログラム上でファイルから読み込んだり、ファイルに書き出したりする操作をするためには専用のルーチンを呼び出す必要がある。これらの機能は、多くの汎用プログラミング言語で標準ライブラリとして提供されているが、その内部実装で用いられているのは標準Cライブラリの関数達である。本章では、標準CライブラリにおけるファイルI/Oにまつわる諸概念と、ファイルI/Oのシステムコールのラッパー関数群を紹介する。

## ファイルとは何か

ファイルとは、入力/Input と 出力/Output のインタフェースが用意されたもの全般を指す。普通のファイル（テキストファイル、画像ファイル、実行可能ファイル等）の他にも、**パイプ / pipe** や **ソケット / socket**、**仮想端末 / virtual console** などが同じ仕組みで制御できるようになっている。これらの概念および扱い方は、後の章で説明する。

## OSはファイルをどう管理するか

OSはシステム全体で操作中のファイルの集合を管理している。各要素はファイルテーブルエントリ(?)と呼ばれており、ファイルシステム上のノードに紐付けられている。プロセスがファイルを開くと、対応するファイルテーブルエントリが作られる。複数のプロセスで同じファイルを開いた場合、複数のファイルテーブルエントリが、同じファイルを指すこともある。このケースの挙動は「プロセスとスレッド」の章で確認する。

## ファイルI/O関連システムコール群

### open

プログラムがファイルに対して読み書きの操作を行うためには、まずそのファイルをオープンする必要がある。ファイルをオープンするには、open関数を使う。

>
> ```c
> int open(const char *pathname, int flags); [^2]
> ```
>
> [Man page of OPEN](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/open.2.html)



`const char *pathname` にはファイルパス、`flags` にはモードを指定する。
モードには、3種のファイルアクセス方法のいずれかを指定する。

* `O_RDONLY`: 読み込み専用
* `O_WRONLY`: 書き出し専用
* `O_RDWR`: 読み書き用

ファイルフラグ

* `O_APPEND`: 追記モードでのオープン。
* `O_CREAT`: ファイルが存在しなかった場合、作成する。
* `O_NONBLOCK`: ファイルを **ノンブロッキングモード / non-blocking mode** でオープンする。このモードを指定したときの挙動については、後の章で説明する。

その他のモードについては、 [Man page of OPEN](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/open.2.html) を参照されたい。

オープンに成功すると、指定したファイルに紐付いた **ファイルディスクリプタ / file descriptor** が返ってくる。ファイルディスクリプタとは、プロセスがオープン中のファイルを一意に特定するための識別子[^1]である。プロセス固有の非負整数値で、カーネルによって割り振られる。

### read

読み込み専用・あるいは読み書き用のモードでオープンしたファイルからデータを読み込むには、read関数を使う。

> read - ファイルディスクリプターから読み込む
> ```c
> #include <unistd.h>
>
> ssize_t read(int fd, void *buf, size_t count);
> ```
> [Man page of READ](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/read.2.html)

`fd` で指定されたファイルディスクリプタの現在のシーク位置から、**最大** `count` バイトを `buf` に読み込む。成功した場合、実際に読み込まれたバイト数が返る。ファイルのシーク位置は書き出された分だけ増える。

シーク位置とは、次にファイルの読み込み・書き込みを行う位置を先頭からのバイト数で示したものである。この値は読み込み・書き出しの度に、読み・書きに成功した分だけ増える。

### write

書き出し専用・あるいは読み書き用のモードでオープンしたファイルにデータを書き出すには、write関数を使う。

> write - ファイルディスクリプター (file descriptor) に書き込む
> ```c
> #include <unistd.h>
> ssize_t write(int fd, const void *buf, size_t count);
> ```
> [Man page of WRITE](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/write.2.html)

`fd` で指定されたファイルディスクリプタに、`buf` から **最大** `count` バイトを書き出す。成功した場合、実際に書き出されたバイト数が返る。ファイルのオフセットは書き出された分だけ増える。

### lseek

lseek関数を使うと、ファイルの読み書きオフセットを変更できる。使い方の説明は省く。

> lseek - ファイルの読み書きオフセットを変える
>
> ```c
> off_t lseek(int fd, off_t offset, int whence);
> ```
>
> [Man page of LSEEK](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/lseek.2.html)

### close

オープンしたファイルをクローズするには、close()関数を使う。

> close - ファイルディスクリプターをクローズする
> ```c
> #include <unistd.h>
>
> int close(int fd);
```
> [Man page of CLOSE](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/close.2.html)

`fd` で指定されたファイルディスクリプタをクローズする。

## 例 : ファイルのコピー

今までに紹介した関数を使って、特定のファイルをコピーするプログラムを書くことができる。

```c
#include <unistd.h>
#include <fcntl.h>

int main(int argc, char* argv[]) {
int srcFd = open("./src.txt", O_RDONLY);
if (srcFd == -1) {
return 1;
}

int dstFd = open("./dst.txt", O_WRONLY | O_CREAT);
if (dstFd == -1) {
return 1;
}

char buffer[16];
while (1) {
int num = read(srcFd, buffer, 16);
if (num == 0) {
break;
}
write(dstFd, buffer, num);
}
close(srcFd);
close(dstFd);
return 0;
}
```

`./src.txt` および `./dst.txt` をそれぞれ読み取り、書き出しモードで作成する。また、`dst.txt` は `O_CREAT` フラグを付けているので、ファイルが存在しなければ作成する。

`char buffer[16];` は読み込んだデータを一時的に貯めておくための場所。長さはここでは 16 とした。

`read` 関数からは実際に読み込まれたバイト数が返るので、それが 1 以上であれば、`write` 関数に読み込み用のバッファとバイト数を渡してデータを書き出す。

このファイルを適当な名前(例: `copy.c`)で保存し、以下のようにコンパイルする。

```sh
$ gcc -Wall copy.c
```

同じディレクトリにファイル `src.txt` (中身は適当で良い) を用意して、

```sh
$ ./a.out
```

環境によっては、`dst.txt` がパーミッションが無い状態で作成されるかもしれない。その場合、中身を確認するには [chmod(1)](https://linuxjm.osdn.jp/html/GNU_fileutils/man1/chmod.1.html) でファイルのパーミッションを変更する。

```sh
$ chmod 644 dst.txt
```

[diff(1)](https://linuxjm.osdn.jp/html/gnumaniak/man1/diff.1.html) で確かめるとファイルの内容が等しいことが分かる。(内容が等しい場合、何も出力されない。)

```sh
$ diff src.txt dst.txt
```

### エラーの表示

先に示したプログラムを、例えば `src.txt` を削除した状態で走らせると、うんともすんとも言わずに終了する。これではあまりにも不親切なので、open関数実行後、エラーの種別に応じたメッセージを表示して終了するように改造しよう。

```c
#include <unistd.h>
#include <fcntl.h>
#include <errno.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>

void error_and_exit() {
printf("error(%d): %s\n", errno, strerror(errno));
exit(EXIT_FAILURE);
}

int main(int argc, char* argv[]) {
int srcFd = open("./src.txt", O_RDONLY);
if (srcFd == -1) {
error_and_exit();
}

int dstFd = open("./dst.txt", O_WRONLY | O_CREAT);
if (dstFd == -1) {
error_and_exit();
}

char buffer[16];
while (1) {
int num = read(srcFd, buffer, 16);
if (num == 0) {
break;
}
write(dstFd, buffer, num);
}
close(srcFd);
close(dstFd);
return 0;
}
```

主な変更は、エラー内容を出力して終了する関数 `print_error_and_exit` を追加した部分だ。ここで、唐突に `errno` という変数が出現しているが、これは
多くの標準ライブラリの関数やシステムコールが、処理が正常でないとき、エラーの種別をこの変数に代入するようになっている。また、この値はスレッドごとに固有の値を持つことが保証されている。また、`strerror` はエラー番号を受け取り、対応した文字列を返す。>[Man page of STRERROR](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/strerror.3.html)

`exit` はプログラムを終了させる関数で、引数には終了ステータスを指定する。ここでは、「異常終了」を示す `EXIT_FAILURE` を用いている。[Man page of EXIT](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/exit.3.html)

また、エラーメッセージを画面に表示するのにしれっとおなじみの `printf` を使ったが、これもれっきとしたI/Oの関数だ。だが、これは本章で説明したシステムコールのラッパー関数よりもやや高いレイヤに位置する、「ストリーム」と呼ばれるオブジェクトを対象とする関数である。次章で、その詳細について見ていこう。

[^1]: 厳密には、同じファイルが複数のファイルディスクリプタによって参照されることもある。「プロセスとスレッド」の章でその例を示してある。

[^2]: この他にも機能が少し異なる関数がある。詳しくは、下のリンクに示されているマニュアルを参照。以降、ほぼすべての標準ライブラリの関数等の説明をするときにはマニュアルへのリンクを付けている。詳しい仕様を知りたいときは適宜参照して欲しい。
