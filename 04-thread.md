# スレッド

## スレッドとは

スレッドとは、同じプログラムを見かけ上並行に実行する仕組みである。プロセスは複数のスレッドを持ちうる。プロセスは起動時に1つだけスレッドを持ち、このスレッドのことをメインスレッドと呼ぶ。同じプロセスに属するスレッドは、以下の内容を共有する。複数のスレッドで共有されている領域を触る可能性がある場合、排他制御の仕組みが必要になる。

- ヒープ領域
- データ領域
- プロセスエントリテーブル

実行中のスレッドの切り替えはカーネル(OS)により行われる。

### pthread_create

スレッドを生やすには、pthread_create関数を使う。

> ```c
> #include <pthread.h>
>
> int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
> void *(*start_routine) (void *), void *arg);
> ```
> [Man page of PTHREAD_CREATE](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/pthread_create.3.html)

`start_routine` には作ったスレッドで実行したい関数を指定する。`arg` にはその関数の引数を渡せる。作成されたスレッドは `thread` に代入される。`attr` にはスレッドのオプションを指定する。

### 例: スレッドとヒープ・データ領域

スレッド間で共有する情報とそうでないものを確かめるため、いくつか実験をしよう。

```c
#include <unistd.h>
#include <stdio.h>
#include <pthread.h>

int global_num = 100;

void* another(void *arg) {
  global_num = 200;
  printf("hello from another thread: %d\n", global_num);
  global_num = 300;
}

int main(int argc, char* argv[]) {
  pthread_t thread;
  pthread_create(&thread, NULL, *another, NULL);

  sleep(3);

  printf("hello from main thread: %d\n", global_num);

  return 0;
}
```

このプログラムは別スレッドを立ち上げ `another` を実行させる一方、自分は３秒待ってから変数の中身を確認する。`another` では、変数の操作をしてデータ出力し、再度変数の操作を行う。この操作がメインスレッドの出力内容に反映されるか確かめるのが目的だ。実行結果は次の通りで、別スレッドでの操作内容が反映されていることが分かる。理由は `global_num` がデータ領域に属していて、スレッド間で共有されるからだ。

```
```

## スレッドの待ち合わせ

###



## スレッドとファイル操作


### 例: スレッドとファイル操作

forkの節で実験したのと同様に、別々のスレッドで同じファイルを開いたときの挙動について確認しておこう。まず、ファイルを開く前に別スレッドを作り、各スレッドの処理の中でファイル操作を行う場合。

```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <pthread.h>

void* another(void *arg) {
  int fd = open("./src.txt", O_RDONLY);
  char buf[6];
  read(fd, buf, 6);
  printf("hello from another thread: %d, %s\n", fd, buf);
}

int main(int argc, char* argv[]) {
  pthread_t thread;
  pthread_create(&thread, NULL, *another, NULL);

  sleep(3);

  int fd = open("./src.txt", O_RDONLY);
  char buf[6];
  read(fd, buf, 6);

  printf("hello from main thread : %d, %s\n", fd, buf);

  return 0;
}
```

プログラムは `fork_open_read.c` をスレッド版に置き換えただけである。これを実行すると、次のような結果を得るだろう。

```
$ ./a.out
hello from another thread: 3, abcdef
(3秒後)
hello from main thread : 4, abcdef
```

fork版と同等の結果を得た。ファイルディスクリプタも別々である。また、ファイルを開いてからスレッドを作り、それぞれ読む場合も見ておこう。


```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>
#include <pthread.h>

int fd;

void* another(void *arg) {
  char buf[6];
  read(fd, buf, 6);
  printf("hello from another thread: %s\n", buf);
}

int main(int argc, char* argv[]) {
  fd = open("./src.txt", O_RDONLY);

  pthread_t thread;
  pthread_create(&thread, NULL, *another, NULL);

  sleep(3);
  char buf[6];
  read(fd, buf, 6);

  printf("hello from main thread : %s\n", buf);

  return 0;
}
```

これもfork版と同じ結果となる。同プロセスの同じファイルディスクリプタを通じて操作を行っているので、当然の結果といえる。

```
$ ./a.out
hi. this is child process : abcdef
(3秒後)
hi. this is parent process : ghijkl
```

## 余談: Linuxにおけるスレッドの実装

Linuxにおけるスレッドの実装について軽く触れておく。内部的には、`clone()` システムコールが使われている。`clone` とは、`fork` 　で子プロセスを作るシステムコールである。`fork` と異なる点は、子プロセスで実行する関数を引数で指定できることと、親と何を共有するかを細かく指定できることである。詳しくはマニュアルを参照。

> ```
> #include <sched.h>
>
> int clone(int (*fn)(void *), void *child_stack,
>           int flags, void *arg, ...
>           /* pid_t *ptid, struct user_desc *tls, pid_t *ctid */ );
> ```
> [Man page of CLONE](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/clone.2.html)

また、`fork` の内部の実装にも `clone` が使われている。Linuxにおけるスレッドとはプロセスと基本的に変わりはなく、実際プロセスとスレッドは同じ仕組みでスケジューリングされる。そのため、Linuxにおけるスレッドは通常のプロセスに対して軽量プロセス / Lightweight processと呼ばれることがある。

## 排他制御

複数のスレッドから同じデータ領域を触るとき、一連の操作のアトミック性を担保するには何かしらの排他制御の仕組みが必要になる。このうち、mutex, read/write lockを紹介する。

### mutex

mutexは排他制御の仕組みの一つで、「アンロック状態」「ロック状態」のいずれかの状態を持つオブジェクトである。スレッドが共有資源上の何かを触るときは、必ず「ロック獲得」「操作」「アンロック」の順番で処理が行われるようにする。mutexがロック状態のときは、他のスレッドからのロック獲得をブロックするので、同時に走る操作は1スレッドのみであることが保証される。


mutex を使うには、mutex オブジェクトを初期化する必要がある。mutexオブジェクトを初期化するには `pthread_mutex_init` 関数を使うか、定数 `PTHREAD_MUTEX_INITIALIZER` を使う。今回は定数を使った使用例のみを示す。当然、大域変数として宣言しないと意味が無いので注意。


mutexのロックを獲得するには、`pthread_mutex_lock` 関数を使う。また、アンロックするには `pthread_mutex_unlock` 関数を使う。排他制御をしたい操作が、これらの関数で囲まれるようにする。


使用例を以下に示す。

```c
#include <unistd.h>
#include <stdio.h>
#include <pthread.h>

int x;
pthread_mutex_t mut = PTHREAD_MUTEX_INITIALIZER;

void add_x(int a) {
  pthread_mutex_lock(&mut);
  x += a;
  pthread_mutex_unlock(&mut);
}

void* doit(void *arg) {
  for (int i = 0 ; i < 100 ; i++) {
    add_x(1);
  }
  printf("hello from another thread: %d\n", x);
}

int main(int argc, char* argv[]) {
  pthread_t thread;
  for (int cur = 0 ; cur < 100 ; cur++) {
    pthread_create(&thread, NULL, *doit, NULL);
  }
  sleep(3);
  printf("total: %d\n", x);
  return 0;
}
```

上のプログラムは、大域変数 `x` に 1 を加える操作を100回行う動作、を 100 スレッド並列で行う。
以下を実行すると、以下の結果を得るだろう。

```
$ ./a.out
hello from another thread: 100
hello from another thread: 200
hello from another thread: 300
hello from another thread: 400
hello from another thread: 500
(略)
hello from another thread: 9996
hello from another thread: 9997
hello from another thread: 9998
hello from another thread: 9999
hello from another thread: 10000
total: 10000
```

環境によって動作は変わるかもしれない。見ての通り、各スレッドが報告する数値は 100 の倍数とは限らない。が、合計が 10000 であることは mutex によって保証されているのである。


では、mutex のロック/アンロック操作無しでプログラムを実行した場合はどうなるだろうか？関数 add_x を以下のように書き換えて再度実行してみよう。

```
void add_x(int a) {
  x += a;
}
```

筆者の環境では、以下のようになった。

```
$ ./a.out
hello from another thread: 100
hello from another thread: 200
hello from another thread: 300
hello from another thread: 400
hello from another thread: 500
hello from another thread: 600
(略)
hello from another thread: 9568
hello from another thread: 9668
hello from another thread: 9768
hello from another thread: 9868
hello from another thread: 9968
total: 9968
```

### read/write lock

そもそも、関心のある領域に変更がないことを保証できるなら、複数スレッドが同時に読んでも問題ない。ただし、「書き込み」を行う動作は1つだけのスレッドで走り、その間「読み込み」を行う動作が行われないことを保証したい。

これを実現するため、「読み」「書き」用それぞれのロックを獲得する仕組みが考案された。
読み込みロックが獲得されている間は、さらなる「読み込みロック」は獲得可能だが、「書き込みロック」は獲得不可。「書き込み」のロックが獲得されている間は、「読み込みロック」「書き込みロック」共に獲得不可。

これをまとめると以下の表のようになる。

| ロックの種類 | 読み込みロック | 書き込みロック |
| :---  | :--- | :--- |
| 読み込みロック中 | o | x |
| 書き込みロック中 | x | x |


read/write lock オブジェクトを初期化する必要がある。read/write lock オブジェクトを初期化するには `pthread_rwlock_init` 関数を使うか、定数 `PTHREAD_RWLOCK_INITIALIZER` を使う。今回は定数を使った使用例のみを示す。当然、大域変数として宣言しないと意味が無いので注意。

読み込みロックを獲得するには、`pthread_rwlock_rdlock` 関数を使う。この関数は対象の書き込みロックが獲得されている間はブロックする。読み込みロックには内部的にカウンタが用いられていて、この関数を実行すると 1 増える。1 以上の値を持つときは読み込みロックが獲得されている、とみなす。


書き込みロックを獲得するには、`pthread_rwlock_wrlock` 関数を使う。この関数は対象の読み込みロック、または書き込みロックが獲得されている間はブロックする。


読み込み・書き込みロックの解放には `pthread_rwlock_unlock` 関数を使う。

### 例

```c

```


[^1]: これらは本章で説明するスレッド(POSIXスレッド)に対して、グリーンスレッドと呼ばれることがある。
[^2]: [sleep(3)](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/sleep.3.html)は現在実行中のスレッドを指定秒数停止させる関数。
