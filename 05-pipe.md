# パイプとFIFOとシグナル

本章では、複数のプロセス間でデータをやり取りするための方法として、パイプ、FIFO、シグナルを紹介する。

## パイプ

パイプは単方向のデータの流れ道である。片方にデータを入れると、もう片方から読み込めるようになる。具体的には、\(読み込み口,書き出し口\)のファイルディスクリプタの組として実装されている。パイプに書き出されたデータは、読み込まれるまでバッファに貯まる。

(TODO:図)

> [Man page of PIPE](https://linuxjm.osdn.jp/html/LDP_man-pages/man7/pipe.7.html)

パイプを使うと、親子プロセス間でデータをやり取りできる。親プロセス側でパイプを作り、`fork` を呼び出せば子プロセスにファイルディスクリプタがコピーされるので、例えば親プロセス側で書き込んだ内容を子プロセス側から読み出せる。その逆ももちろん可能。


### pipe

> ```
> #include <unistd.h>
>
> int pipe(int pipefd[2]);
> ```
> [Man page of PIPE](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/pipe.2.html)

pipeを呼び出すと、`pipefd[0]` に入力用、 `pipefd[1]` に出力用のファイルディスクリプタが返される。出力用のファイルディスクリプタに対して書き出しを行うと、入力側からそのデータを読み込める。このままだと、単に同じプロセスの片方から `write` して、別の箇所で `read` でそのデータを読むだけだが、`fork` と組み合わせることで親子間のプロセスでのデータのやり取りが実現できる。

#### 例

`pipe` と `fork` を使ってデータのやりとりを行う例を示す。

```c
#include <unistd.h>
#include <stdio.h>

int main(int argc, char* argv[]) {
  int pipefd[2];
  pipe(pipefd);

  int readFd = pipefd[0];
  int writeFd = pipefd[1];

  pid_t result = fork();
  if (result == 0) {
    close(readFd);
    char data[100];
    for (int i = 0 ; i < 100 ; i++) {
      data[i] = (char)('a' + (i % 26));
    }
    write(writeFd, data, 100);
  } else {
    close(writeFd);
    char buf[5];
    while (1) {
      int num = read(readFd, buf, 5);
      if (num == 0) {
        break;
      }
      printf("%s\n", buf);
    }
  }
  return 0;
}
```

まず、親プロセス側で `pipe` を呼び出してファイルディスクリプタの組を作る。その後、`fork` して子プロセスを作る。このとき、オープン中のファイルディスクリプタがコピーされるので、`readFd`, `writeFd` が親側、子側でそれぞれ同じパイプの読み込み口、書き出し口を参照している状態になる。

（ず）

ちなみに `fork` 後、子プロセスで読み込み側、親プロセスで書き込み側のファイルディスクリプタをクローズしている理由は、それぞれにとって不要であるため。


子プロセス側では、データ列を入力口に100バイト書き込む。親プロセス側では、出力口からデータを5バイトずつ読み出して、改行しながら表示する。このとき、`read` 関数はパイプの書き込み口に十分な量(5byte)のデータが溜まるか閉じるまで後続の処理をブロックすることに注意。なので、子プロセス側の処理を `sleep` 等で遅延させてもデータは正しく読み込める。

#### 例2

子プロセスが別のコマンドを実行する際、その出力結果を親プロセス側で受け取りたい場合がよくある。親プロセスでパイプを作成し、書き込み口のファイルディスクリプタを子プロセスの標準出力としてコピーすることで、親プロセスは子プロセスの標準出力の内容をパイプから読むことができる。

ファイルディスクリプタを指定した番号にコピーするには、`dup2` システムコールを用いる。

> ```c
> #include <unistd.h>
>
> int dup2(int oldfd, int newfd);
> ```
> [Man page of DUP](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/dup.2.html)


実装は例えば次のようにする。ここでは、`/bin/ls` の実行結果を親側で受け取り、5文字ごとに改行して表示している。

```c
#include <unistd.h>
#include <stdio.h>

int main(int argc, char* argv[]) {
  int pipefd[2];
  pipe(pipefd);

  int readFd = pipefd[0];
  int writeFd = pipefd[1];

  pid_t result = fork();
  if (result == 0) {
    dup2(writeFd, STDOUT_FILENO);
    close(readFd);
    close(writeFd);
    execl("/bin/ls", "al", (char *)NULL);
  } else {
    close(writeFd);
    char buf[5];
    while (1) {
      int num = read(readFd, buf, 5);
      if (num == 0) {
        break;
      }
      printf("%s\n", buf);
    }
  }
  return 0;
}
```

まず親プロセス側でパイプを作成し、`fork` して子プロセスを作る。

子プロセス側は、まずパイプの書き込み口を標準出力としてコピーする。ここで `execl` を実行すると、いま実行しているプログラムが `/bin/ls` で置き換わる。このとき、プロセスがオープンしていたファイルディスクリプタはそのまま残るため、標準出力はパイプの書き込み口を指している。その結果、`/bin/ls` の標準出力がパイプに書き込まれることになる。

親プロセス側の処理は例1と同じため説明は省く。

### popen

例2で示したような `pipe` で作ったファイルディスクリプタを子プロセスにコピーして `exec` するパターンは頻出なので便利な関数が用意されている。

> ```c
> #include <stdio.h>
>
> FILE *popen(const char *command, const char *type);
> ```
> [Man page of POPEN](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/popen.3.html)

popen 関数を実行すると、パイプを作成し、フォークをして、子プロセス側で `/bin/sh` に `command` で指定したコマンドを実行する。`type` には読み込み・書き出しのどちらかを指定する。成功するとストリームオブジェクトが返ってくるので、ストリーム操作関数を用いて読み込み・書き込み操作が行える。

#### 例

```c
#include <unistd.h>
#include <stdio.h>

int swapcase(int ch) {
  if ('a' <= ch && ch <= 'z') {
    return 'A' + (ch - 'a');
  } else if ('A' <= ch && ch <= 'Z') {
    return 'a' + (ch - 'A');
  } else {
    return ch;
  }
}

int main(int argc, char* argv[]) {
  FILE* reader = popen("/bin/ls -al", "r");
  while (1) {
    int ch = fgetc(reader);
    if (ch == EOF) {
      break;
    }
    fputc(swapcase(ch), stdout);
  }
  pclose(reader);
  return 0;
}
```

この例では、`/bin/ls -al` の実行結果を大文字／小文字を逆転させて表示している。コマンドの標準出力からデータを読むため、`popen` の第二引数には 「読み込み」を表す `r` を渡す。


## FIFO

FIFOとは、ファイルシステム上に作成される特殊なパイプである。パス名を指定して作成でき、複数のプロセスからそのファイルを通じてデータのやり取りが可能になる。名前付きパイプと呼ばれることもある。

通常のパイプと異なり、パス名さえ事前に取り決めておけば親子関係にないプロセス間でも通信できる。

### mkfifo

> ```
> #include <sys/types.h>
> #include <sys/stat.h>
>
> int mkfifo(const char *pathname, mode_t mode);
> ```
> [Man page of MKFIFO](https://linuxjm.osdn.jp/html/LDP_man-pages/man3/mkfifo.3.html)

FIFOを作成する。作成されたFIFOは、指定パス名を通じて `open` でき、書き出し・読み込みができるようになる。`mode` にはFIFOのアクセス許可モードを指定する。

### 例

FIFOを使ったプログラムの例を示す。3つのプログラムが登場する。ひとつはFIFOを作って以降待つだけのプログラム。残りの2つはFIFOを通じてデータの読み込み、書き出しを行うプログラムである。

#### create_fifo.c

FIFOを作って待つだけのプログラム。

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

#### read.c

FIFOが作成済である前提で `open` し、通常のファイルと同じように読み込み処理を行うプログラムである。
ここでは、読み取ったデータを5byteごとに改行して出力している。

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


#### write.c

FIFOが作成済である前提で `open` し、通常のファイルと同じように書き出し処理を行うプログラムである。
ここでは、'c', 'a', 't' の 3文字を 100回書き込んでいる。


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


#### 3つのプログラムの実行

プログラムをコンパイルして、それぞれ実行しよう。まず、FIFOを作って待つだけのプログラムを実行してから、読み込み、書き込みを行うプログラムを起動する。

```
$ gcc create_fifo.c -o create_fifo
$ gcc read.c -o read
$ gcc write.c -o write
$ ./create_fifo &
$ ./write &
$ ./read
```

読み込みを行うプログラムの実行結果は次のとおりとなった。

```
$ gcc read.c -o read
$ gcc write.c -o write
```

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


詳しい仕様については、[Man page of SIGNAL](https://linuxjm.osdn.jp/html/LDP_man-pages/man7/signal.7.html) を参照。
