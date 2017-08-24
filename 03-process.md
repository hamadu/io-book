# プロセス

本章では、プロセスとスレッドに関して説明する。I/Oとは直接関係ないが、プロセス・スレッド間で何を共有、あるいは共有しないかを把握することは重要であるため本章を設けた。これらの概念について十分に習熟している読者はこの章を飛ばしてもかまわない。

## プロセス

プロセスとは、プログラムの実行単位である。プロセスは一連のプログラム（機械語）を順番に書かれたとおりに実行する。

### プロセスのアドレス空間

各プロセスは独立した仮想的なメモリ空間を持つ。これを仮想アドレス空間と呼ぶ。おかげで、各プロセスが他のプロセスのメモリの内容や、データの保存場所（メモリにあるのか、それともページアウトされてディスクに書き出されているのか）を気にする必要がなくなる。

プロセスの仮想アドレス空間には、いくつかのセグメント（連続する領域）に分かれている。

| 種類 | 説明 |
| :---  | :--- |
| Text   | 機械語の列（命令列） |
| Data   | 静的変数のうち、明示的な初期値が入っているもの |
| bss    | 静的変数のうち、初期値が指定されてないもの(初期値として0が入る) |
| Heap   | malloc等で動的にメモリ確保するときに使う領域 |
| Shared | 共有ライブラリ(libc等)が配置される領域 |
| Stack  | ローカル変数や関数の引数、関数からの戻り先が置かれる領域 |

これらは、概ね以下の図のように配置されている。このうち、Text領域は同じプログラムが走るときは共有される。Shared領域は、同じライブラリが使われるプロセス同士で共有される。

![](/assets/mem.png)

### プロセスが持つ情報

各プロセスは以下の情報をそれぞれ独立して持つ。

- プロセスID
- 親プロセスID
- 仮想アドレス空間
  - プログラム本体(命令列)
  - 環境変数
  - スタック領域、ヒープ領域
- プロセスファイルテーブル
- シグナルハンドラ
- 統計情報(使用メモリ量,実行時間など)

したがって、例えばあるプロセスの環境変数が変化しても、他のプロセスに影響することはない。

### プロセスのフォーク

プロセスは、ただ一つの親プロセスを持ち、複数の子プロセスを持つことができる。プロセスの親子関係は木構造を成している。親プロセスが子プロセスを持つには、**フォーク / fork** と呼ばれる自身のプロセスのコピーを作るシステムコールを発行する。コピーされた子プロセスは、親プロセスと同じプログラムを実行する。もしこれを変更したい場合は、`exec(2)` を使って自身のプロセスを他のプログラムで置き換える。

子プロセスは、親プロセスから以下の情報をコピーした上で引き継ぐ。

- 仮想アドレス空間
- プロセスファイルテーブル
-

実際には fork した時点でデータがまるごとコピーされるのではなく、親か子のデータに変更があった場合に初めてコピーされる。この仕組みを **コピーオンライト　/ copy on write** と呼ぶ。このおかげで、fork 直後に exec した場合でも、fork の段階で余計なコピーコストがかかることはない。また、仮想アドレス空間の中でもテキスト領域は親子で使いまわせるため、参照だけコピーして中身は使いまわすようになっている。

## forkとexec

### fork

子プロセスを生やすには、fork関数を使う。

> ```c
> #include <unistd.h>
> pid_t fork(void);
> ```
>
> [Man page of FORK](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/fork.2.html)

奇妙に思えるかもしれないが、fork関数からは、2回返る。1つは呼び出し元のプロセスで、もう片方が新しいプロセスとなる。これらは返り値で判別できる。
子プロセスは親プロセスとアドレス空間は共有しない。したがって、以下のプログラムは期待通りの動作をする。

```c
#include <unistd.h>
#include <errno.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void print_error_and_exit() {
  printf("error(%d): %s\n", errno, strerror(errno));
  exit(1);
}

int global_num = 100;

int main(int argc, char* argv[]) {
  int num = 42;
  pid_t result = fork();
  if (result == -1) {
    print_error_and_exit();
  }

  if (result == 0) {
    num += 10;
    global_num++;
    printf("hi. this is child process : %d, %d\n", num, global_num);
  } else {
    sleep(3);
    num += 4;
    global_num += 2;
    printf("hi. this is parent process : %d, %d\n", num, global_num);
  }
  return 0;
}
```

プログラムが起動すると、まず `fork()` で自身のコピーを作ろうとする。成功すると、`fork()` からは値が 2度 返却され、子プロセスには 0 が、親プロセス(元のプロセス)には 子プロセスのID が渡される。親プロセス側は、`sleep()` 関数で子プロセスの実行終了を待ち[^2]、その後 `num` と `global_num` の値を出力する。子プロセスの方が先に `num` と `global_num` の操作を行うことになるが、この値はお互い別のアドレス空間に存在するため、一方の操作が他に影響をあたえることはない。したがって、このプログラムを実行すると以下のように表示される。

```
$ ./a.out
hi. this is child process : 52, 101
(3秒後)
hi. this is parent process : 46, 102
```

forkを実行すると値が2回返ったり、データがコピーされる仕組み（カーネルの実装）については [はじめてのOSコードリーディング ~UNIX V6で学ぶカーネルのしくみ (Software Design plus)](https://www.amazon.co.jp/dp/4774154644) が参考になる。

### exec

子プロセス側で、親と異なるプログラムを実行したいときもある。実行中のプロセスを他のプログラムで置き換えたい場合、`exec` 関数群を使う。ここでは、`execl` 関数を紹介する。

> ```c
> #include <unistd.h>
> int execl(const char *path, const char *arg, ...);
```
> [Man page of EXEC](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/exec.3.html)

`path` には実行したいプログラムのパス、`arg` にはそのプログラムに与える引数を指定する。`execl` の呼び出しに成功した場合、値は返却されず、実行中のプロセスのテキストセグメントが新しいもので置き換わる。つまり、`execl` 以降に書かれているプログラムは無視され、

#### 例

子プロセスで `ls` を実行する例を示す。

```c
#include <unistd.h>
#include <errno.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void print_error_and_exit() {
  printf("error(%d): %s\n", errno, strerror(errno));
  exit(1);
}

int main(int argc, char* argv[]) {
  pid_t result = fork();
  if (result == -1) {
    print_error_and_exit();
  }

  if (result == 0) {
    execl("/bin/ls", ".", (char*)NULL);
  } else {
    sleep(3);
    printf("hi. this is parent process\n");
  }
  return 0;
}
```

## プロセスの待ち合わせ

上の例では子プロセスの実行を待つのに `sleep` を用いたが、推奨される使い方ではない。`sleep` 中に他のプロセスが実行されていて、sleep の指定秒数経過後に戻ってきた場合、次に走るのは親か子かの保証はできない。（プロセスの実行状況とスケジューラのアルゴリズムに依存する）そのため、子プロセスの状態を監視・通知できる機能が用意されている。

### waitpid

waitpid関数を使うと、子プロセスの状態変化を待つことができる。

> ```c
> #include <sys/types.h>
> #include <sys/wait.h>
>
> pid_t waitpid(pid_t pid, int *status, int options);
> ```
> [Man page of WAIT](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/wait.2.html)

`pid` には対象のプロセス、`options` には、`status` に状態を格納する変数へのポインタを渡す。

例えば親プロセスが fork で生成した子プロセスの終了を待ちたいときは、次のように親側で fork 関数から返ってきた子のプロセスIDを使って waitpid 関数を呼び出せば良い。

```c
#include <unistd.h>
#include <errno.h>
#include <stdio.h>
#include <string.h>
#include <stdlib.h>

void print_error_and_exit() {
  printf("error(%d): %s\n", errno, strerror(errno));
  exit(1);
}

int main(int argc, char* argv[]) {
  int status;
  pid_t result = fork();
  if (result == -1) {
    print_error_and_exit();
  }

  if (result == 0) {
    printf("hi. this is child process\n");
  } else {
    waitpid(result, &status, 0);
    printf("hi. this is parent process\n");
  }
  return 0;
}
```

このプログラムを実行すると、以下のように表示される。

```sh
$ ./a.out
hi. this is child process
hi. this is parent process
```



##　forkとファイルディスクリプタ

複数のプロセスで同じファイルを開いたときはどうなるだろうか？以下のプログラムで実験してみよう。

```c
#include <unistd.h>
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
  pid_t result = fork();
  if (result == -1) {
    print_error_and_exit();
  }

  int fd = open("./src.txt", O_RDONLY);
  if (fd == -1) {
    print_error_and_exit();
  }

  if (result == 0) {
    char buf[6];
    read(fd, buf, 6);
    printf("hi. this is child process : %s\n", buf);
  } else {
    sleep(3);
    char buf[6];
    read(fd, buf, 6);
    printf("hi. this is parent process : %s\n", buf);
  }
  return 0;
}
```

`fork()` をした後、それぞれのプロセスで同じ名前のファイル `./src.txt` を開き、6バイト読んで内容を書き出している。このとき、親プロセスには子プロセスの実行を待つため3秒待つ。
実行結果は次のようになる。（確認してみよう。）

```
$ ./a.out
hi. this is child process : abcdef
(3秒後)
hi. this is parent process : abcdef
```

ファイルを開く操作をしたのは別のプロセスなので、それぞれ先頭から6バイトが読み込まれる、という自然な結果である。では、`fork()` する前にファイルを開いていた場合はどうなるだろうか？

```c
#include <unistd.h>
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
  int fd = open("./src.txt", O_RDONLY);
  if (fd == -1) {
    print_error_and_exit();
  }

  pid_t result = fork();
  if (result == -1) {
    print_error_and_exit();
  }

  if (result == 0) {
    char buf[6];
    read(fd, buf, 6);
    printf("hi. this is child process : %s\n", buf);
  } else {
    sleep(3);
    char buf[6];
    read(fd, buf, 6);
    printf("hi. this is parent process : %s\n", buf);
  }
  return 0;
}
```

今度は `fork()` を実行する前にファイルをオープンし、そのファイルディスクリプタを使って6バイトずつ読む。結果は次の通り。

```sh
$ ./a.out
hi. this is child process : abcdef
(3秒後)
hi. this is parent process : ghijkl
```

今度は、親プロセスの方の6バイトが、子プロセスからの続きになった。これはファイルのオフセットがプロセス間で共有されているからである。どうしてこのようなことが起こるのか？種明かしをすると実は `fork()` のときに、現在開いているファイルディスクリプタがコピーされるのだが、このコピーは同じファイルテーブルエントリを指しているためである。ファイルエントリーテーブルにはそのファイルのオフセットと、open時のモード情報が含まれているので、プロセス間でのオフセットは共通になる。（詳しくは、「ファイルI/O」の章を参照。）
