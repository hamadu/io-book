# スレッド

## スレッドとは

スレッドとは、同じプログラムを見かけ上並行に実行する仕組みである。プロセスは複数のスレッドを持ちうる。プロセスは起動時に1つだけスレッドを持ち、このスレッドのことをメインスレッドと呼ぶ。同じプロセスに属するスレッドは、以下の内容を共有する。複数のスレッドで共有されている領域を触る可能性がある場合、排他制御の仕組みが必要になる。

- ヒープ領域
- データ領域
- プロセスエントリテーブル

実行中のスレッドの切り替えはカーネルにより行われる。

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

### 例: スレッドとファイル操作

forkの節で実験したのと同様に、別々のスレッドで同じファイルを開いたときの挙動について確認しておこう。まず、ファイルを開く前に別スレッドを作り、各スレッドの処理の中でファイル操作を行う場合。

```
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

### 余談: Linuxにおけるスレッドの実装

内部的には、clone() システムコールが使われている。clone とは、fork の亜種でコピープロセスを作るシステムコールである。

> [Man page of CLONE](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/clone.2.html)


## 排他制御

複数のスレッドから同じデータ領域を触るとき、一連の操作のアトミック性を担保するには何かしらの排他制御の仕組みが必要になる。このうち、mutex, read/write lockを紹介する。

### mutex

mutexは排他制御の仕組みの一つで、「アンロック状態」「ロック状態」のいずれかである。スレッドが共有資源上の何かを触るときは、必ず「ロック獲得」「操作」「アンロック」の順番で処理が行われるようにする。mutexがロック状態のときは、他のスレッドからのロック獲得をブロックするので、同時に走る操作は1スレッドのみであることが保証される。

#### pthread_mutex_init(3)

> ```c
> #include <pthread.h>
>
> int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *mutexattr);
> ```
> [Man page of PTHREAD_MUTEX](https://linuxjm.osdn.jp/html/glibc-linuxthreads/man3/pthread_mutex_lock.3.html)


### read/write lock

変更がないなら同時に読んでも問題ないのでは、という考えになってくる。

read lock :
write lock :


### 例



[^1]: これらは本章で説明するスレッド(POSIXスレッド)に対して、グリーンスレッドと呼ばれることがある。
[^2]: [sleep(3)](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/sleep.3.html)は現在実行中のスレッドを指定秒数停止させる関数。
