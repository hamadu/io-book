# パイプとFIFOとシグナル

本章では、複数のプロセス間でデータをやり取りするための方法として、パイプ、FIFO、シグナルを紹介する。

## パイプ

パイプとは\(読み込み口,書き込み口\)のファイルディスクリプタの組である。片方にデータを入れると、もう片方から読み出せるようになる。

> [Man page of PIPE](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/pipe.2.html)

forkと組み合わせることで、子プロセスとデータをやり取りできる。

### pipe

> ```
> #include <unistd.h>
>
> int pipe(int pipefd[2]);
> ```
> [Man page of PIPE](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/pipe.2.html)

pipeを呼び出すと、入力用、出力用のファイルディスクリプタの組が返される。出力用のファイルディスクリプタに対して `write` を呼び出すことで、入力用から `read` でそのデータを読み込める。


### 例

`pipe` と `fork` を使ってデータのやりとりを行う例を示す。

```c

```

まず、親プロセス側で `pipe` を呼び出してファイルディスクリプタの組を作る。その後、`fork` して子プロセスを作る。子プロセス側では、データ列を入力口に100バイト書き込む。親プロセス側では、出力口からデータを5バイトずつ読み出して、改行しながら表示する。

### popen

上に示した pipe してからの fork、その後 exec はよくあるパターンなので便利な関数が用意されている。

```
```
> [Man page of POPEN](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/popen.3.html)




### シェルにおけるパイプ

bashでは、あるコマンドの標準出力を、別のコマンドの標準入力に渡すのにパイプを用いることができる。

```sh
ls -al dir | grep hoge
```

## FIFO

FIFOとは、ファイルシステム上に作成される特殊なパイプである。パス名を指定して作成でき、複数のプロセスからそのファイルを通じてデータのやり取りが可能になる。名前付きパイプと呼ばれることもある。

通常のパイプと異なり、パス名さえ取り決めておけば親子関係にないプロセス間でも通信が可能になる。

### mkfifo

> ```
> #include <sys/types.h>
> #include <sys/stat.h>
>
> int mkfifo(const char *pathname, mode_t mode);
> ```
> [Man page of MKFIFO](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/mkfifo.3.html)

FIFOを作成する。作成されたFIFOは、指定パス名を通じて `open` でき、書き出し・読み込みができるようになる。

### 例

FIFOを使ったプログラムの例を示す。3つのプログラムが登場する。ひとつはFIFOを作って以降待つだけのプログラム。残りの2つはデータの読み込みを行う側と、FIFOをオープンしてデータの書き出しを行う側。


```c
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>

int main(int argc, char* argv[]) {
mkfifo("fifo", S_IRWXU);
while (1) {
sleep(1);
}
return 0;
}
```

FIFOを作って待つだけ。

```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>

int main(int argc, char* argv[]) {
int fd = open("fifo", O_RDONLY);
char buf[6];
while (1) {
int num = read(fd, buf, 5);
if (num == 0) {
break;
}
printf("%s\n", buf);
}
return 0;
}
```

読み込み側。


```c
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>

int main(int argc, char* argv[]) {
int fd = open("fifo", O_WRONLY);
char data[3] = {'c', 'a', 't'};
for (int i = 0 ; i < 100 ; i++) {
write(fd, data, 3);
}
return 0;
}
```

書き込み側。


それぞれのプログラムをコンパイルして、それぞれ実行しよう。まず、FIFOを作って待つだけのプログラムを実行する。


```
$ gcc fifo.c
$ ./a.out
```

次に、


## シグナル

シグナルとは、プロセスが受け取ることができる通知である。シグナルは、カーネルがプロセスに送ってくるものもあれば、ユーザの端末経由で送られることも、単にプロセスが他のプロセスを指定して送ることもある。

各シグナルごとに、プロセスのデフォルトの動作が定められているが、[signal\(2\)](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/signal.2.html) によって変更したり、シグナルそのものを無視したりすることができる。但し、SIGSTOP\(一時停止\)およびSIGKILL\(強制終了\)ではできない。

よく使われるシグナルは以下の通り。

| 名前 | 値 | デフォルトの動作 | 説明 |
| :--- | :--- | :--- | :--- |
| SIGHUP | 1 | 終了 | デーモンが設定ファイルを再読込する時に使われていることが多い。nohupで無視できる。 |
| SIGINT | 2 | 終了 | プロセスの中断。 |
| SIGKILL | 9 | 終了\(**変更不可**\) | プロセスの強制終了 |
| SIGSEGV | 11 | 終了、コアダンプ | Segmentation fault |
| SIGSTOP | 17,19,23 | 一時停止\(**変更不可**\) | プロセスの一時停止 |
| SIGTSTP | 18,20,24 | 一時停止 | 端末より入力された一時停止。Ctrl+z |


その他については、[Man page of SIGNAL](https://linuxjm.osdn.jp/html/LDP_man-pages/man7/signal.7.html) を見よ。
