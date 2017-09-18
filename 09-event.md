# イベント駆動I/O

前の章で紹介したサーバの実装はクライアントを単一のキューで待ち受け、一つずつ順番に実行するものであった。つまり、もし長い処理が必要なクライアントがあった場合、後続を待たせてしまう。

(まちぼうけのず)

なので、実際にはクライアントの処理そのものは、待受処理とは異なるプロセスもしくはスレッドで行う等の策が取られる。しかし、それでもI/Oの処理は「待ち」が発生するため、ボトルネックになりがちである。本章では、同時接続可能なechoサーバの実装を通じて、イベント駆動I/Oの仕組みを紹介する。


echoサーバとは、以下の機能を実現するサーバプログラムである。


- 複数のクライアントからの接続を待ち受ける。
- クライアントから文字列が送られたら、それをそのまま返す。


## 多重I/OとC10K問題

現実には、サーバは多数のクライアントと通信を行う必要がある。

[The C10K problem](http://www.kegel.com/c10k.html)

[TheC10kProblem - 「C10K問題」（クライアント1万台問題）とは、ハードウェアの性能上は問題がなくても、あまりにもクライアントの数が多くなるとサーバがパンクする問題のこと](http://www.hyuki.com/yukiwiki/wiki.cgi?TheC10kProblem)


多重I/O / IO Multiplexing

## イベント駆動I/O

多重I/Oを実現する仕組みの一つに、イベント駆動I/Oというものがある。考え方は、「複数のファイルディスクリプタを登録しておき、そのどれかからイベントが起こったら然るべき処理を行う」というものである。ここでいう「イベント」とは、ファイルディスクリプタから `read` を呼ぶ準備ができた、もしくは `write` を呼び出せる状態になったことを指す。

歴史的には、この仕組みは `select`, `poll` というシステムコールを用いて実現してきたが、ここではその紹介はしない。代わりに、それらを効率化したLinux独自の仕組みである `epoll` を紹介する。

[Man page of EPOLL](https://linuxjm.osdn.jp/html/LDP_man-pages/man7/epoll.7.html)

### epoll_create(2)

`epoll` は複数のファイルディスクリプタを管理するオブジェクトである。`epoll` を使うには、まず初期化をする。

> ```c
> #include <sys/epoll.h>
>
> int epoll_create(int size);
> ```
> [Man page of EPOLL_CREATE](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/epoll_create.2.html)

呼び出しに成功すると、`epoll` オブジェクトを表すファイルディスクリプタが返される。


### epoll_ctl(2)

`epoll_ctl` 関数を使うと、指定したファイルディスクリプタをepollオブジェクトに登録したり、外したりできる。

> ```c
> #include <sys/epoll.h>
>
> int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);   
> ```
> [Man page of EPOLL_CTL](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/epoll_ctl.2.html)


`epfd` には `epoll_create` で作成した epollオブジェクトのファイルディスクリプタを渡す。`fd` には対象のファイルディスクリプタを渡す。`op` は操作内容を指定する。

| op | event | 説明 |
| :--- | :--- | :--- |
| EPOLL_CTL_ADD | `fd` で指定したファイルディスクリプタを `epoll` オブジェクトに追加する。`event` には監視内容を指定する。 |
| EPOLL_CTL_MOD | `event` をファイルディスクリプタ `fd` に関連付ける。 |
| EPOLL_CTL_DEL | `fd` で指定したファイルディスクリプタを `epoll` オブジェクトから外す。 |

`epoll_event` 構造体の内容は以下の通り。

> ```c
> typedef union epoll_data {
>   void        *ptr;
>   int          fd;
>   uint32_t     u32;
>   uint64_t     u64;
> } epoll_data_t;
>
> struct epoll_event {
>   uint32_t     events;      /* epoll イベント */
>   epoll_data_t data;        /* ユーザーデータ変数 */
> };
> ```
> [Man page of EPOLL_CTL](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/epoll_ctl.2.html)

`events` には監視したいイベントをビットの論理和で表す。

| events | 説明 |
| :--- | :--- |
| EPOLLIN | 関連付けられたファイルに対して、`read` 操作が可能である。 |
| EPOLLOUT | 関連付けられたファイルに対して、`write` 操作が可能である。 |

たとえば、ファイル `fd` の読み込み操作を epollオブジェクト `epollfd` に監視させる時は次のようにする。

```c
struct epoll_event ev;
ev.events = EPOLLIN;
ev.data.fd = fd;
if (epoll_ctl(epollfd, EPOLL_CTL_ADD, listen_sock, &ev) == -1) {
  // error
}
```

### epoll_wait(2)

登録済のファイルディスクリプタからのI/Oイベントを待つには、`epoll_wait` を使う。

> ```c
> #include <sys/epoll.h>
>
> int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
> ```
> [Man page of EPOLL_WAIT](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/epoll_wait.2.html)

`epfd` には epoll オブジェクトのファイルディスクリプタ、`events` にはイベント内容を格納する領域を渡す。`maxevents` には最大イベント数、`timeout` は最大待ち時間を指定する。呼び出すと、イベントの個数が帰ってくる。

`epoll_wait` の大まかな使用例を以下に示す。

```c
#define MAX_EVENTS 10

struct epoll_event events[MAX_EVENTS];
while (1) {
  int nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
  if (ndfs == -1) {
    // エラー処理
  }
  for (int i = 0 ; i < nfds ; i++) {
    int fd = events[i].data.fd;
    // ファイルディスクリプタのイベントを処理
  }
}
```

## echoサーバ、ver1

これらの関数を使って echoサーバを実装した例を以下に示す。長いプログラムだが、順番に解説していく。

```c
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <string.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>
#include <sys/epoll.h>
#include <signal.h>

#define MAX_EVENTS 100

void print_error_and_exit(const char* s) {
  perror(s);
  exit(1);
}

int main(int argc, char** argv) {
  signal(SIGPIPE, SIG_IGN);

  int sockfd = socket(AF_INET, SOCK_STREAM, 0);
  if (sockfd == -1) {
    print_error_and_exit("create socket");
  }

  struct sockaddr_in addr;
  memset(&addr, 0, sizeof(addr));
  addr.sin_family = AF_INET;
  addr.sin_port = htons(8080);
  inet_aton("127.0.0.1", &addr.sin_addr);
  if (bind(sockfd, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
    print_error_and_exit("bind address to the socket");
  }

  if (listen(sockfd, 10000) < 0) {
    print_error_and_exit("listen to the socket");
  }

  int epollfd = epoll_create1(0);

  struct epoll_event ev;
  ev.events = EPOLLIN;
  ev.data.fd = sockfd;
  if (epoll_ctl(epollfd, EPOLL_CTL_ADD, sockfd, &ev) == -1) {
    print_error_and_exit("registering the socket descriptor to epoll");
  }

  struct epoll_event events[MAX_EVENTS];
  while (1) {
    int nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
    if (nfds == -1) {
      print_error_and_exit("waiting events");
    }

    for (int i = 0 ; i < nfds ; i++) {
      int fd = events[i].data.fd;
      if (fd == sockfd) {
        int peerfd = accept(fd, NULL, NULL);
        ev.events = EPOLLIN | EPOLLET;
        ev.data.fd = peerfd;
        if (epoll_ctl(epollfd, EPOLL_CTL_ADD, peerfd, &ev) == -1) {
          print_error_and_exit("registering the peer socket descriptor to epoll");
        }
      } else {
        char* data = "boom";
        int write_result = write(fd, data, strlen(data));
        if (write_result == EPIPE) {
          if (epoll_ctl(epollfd, EPOLL_CTL_DEL, fd, NULL) == -1) {
            print_error_and_exit("closing peer socket descriptor to epoll");
          }
        }
      }
    }
  }
}
```

```
signal(SIGPIPE, SIG_IGN);
```

まず、main関数の1行目はシグナルハンドラを変更するシステムコールである。ここでは、SIGPIPE シグナルを無視(SIG_IGN) するよう変更している。これが必要な理由については後述する。

```c
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
if (sockfd == -1) {
  print_error_and_exit("create socket");
}

struct sockaddr_in addr;
memset(&addr, 0, sizeof(addr));
addr.sin_family = AF_INET;
addr.sin_port = htons(8080);
inet_aton("127.0.0.1", &addr.sin_addr);
if (bind(sockfd, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
  print_error_and_exit("bind address to the socket");
}

if (listen(sockfd, 10000) < 0) {
  print_error_and_exit("listen to the socket");
}
```

待受用のソケットを作成する部分である。ここは前章と同様なので、説明は省く。

```c
int epollfd = epoll_create1(0);
```

epollオブジェクトを作成する。epoll_create1関数はepollオブジェクトのファイルディスクリプタを返す。

```c
struct epoll_event ev;
ev.events = EPOLLIN;
ev.data.fd = sockfd;
if (epoll_ctl(epollfd, EPOLL_CTL_ADD, sockfd, &ev) == -1) {
  print_error_and_exit("registering the socket descriptor to epoll");
}
```

`struct epoll_event` に監視したいイベント(EPOLLIN: `read` 操作)とファイルディスクリプタ(sockfd: ソケット)を詰めて、epoll_ctl関数に渡して監視を開始する。


以降は `while (1)` でイベント処理を繰り返す。この部分が所謂「イベントループ」である。

```c
int nfds = epoll_wait(epollfd, events, MAX_EVENTS, -1);
if (nfds == -1) {
  print_error_and_exit("waiting events");
}
```

epoll_wait 関数を呼び出して監視中のイベントが起こったかどうか調べる。もし、未処理のイベントがあれば、そのイベントの数を返す。

```c
for (int i = 0 ; i < nfds ; i++) {
  int fd = events[i].data.fd;
  if (fd == sockfd) {
    int peerfd = accept(fd, NULL, NULL);
    ev.events = EPOLLIN | EPOLLET;
    ev.data.fd = peerfd;
    if (epoll_ctl(epollfd, EPOLL_CTL_ADD, peerfd, &ev) == -1) {
      print_error_and_exit("registering the peer socket descriptor to epoll");
    }
  } else {
    char* data = "boom\n";
    int write_result = write(fd, data, strlen(data));
    if (write_result == EPIPE) {
      if (epoll_ctl(epollfd, EPOLL_CTL_DEL, fd, NULL) == -1) {
        print_error_and_exit("closing peer socket descriptor to epoll");
      }
    }
  }
}
```

各イベントに対して処理をする。もしイベントがソケットであったならば、それは新しい接続要求なのでaccept関数を呼び、相手と通信している状態のソケット(`peerfd`)を監視対象として追加する。もしそうでなければ、それはクライアントからの要求なので、適切に処理をする。（ここではデータを読まずに固定長の文字列を送り返しているだけだが。）


ここでのソケットに対する `write` はクライアントの切断によって失敗することがある。もしそうなった場合、プロセスは `SIGPIPE` シグナルを受け取ることになっている。プログラム冒頭で `SIGPIPE` を無視したのはその為で、その場合は `errno` に `EPIPE` が設定されるため、ソケットを監視対象から外す。


試しに実行してみよう。Linuxでないとコンパイルできないので注意。

```
$ gcc -std=c99 server.c
$ ./a.out
```

別のセッションを開いて、telnetでの接続を試みる。

```
$ telnet localhost 8080
```

### 性能を評価する

さて、作ったプログラムがどれほどのクライアントを捌けるか試そう。

これを並行で実行するため、



[^1]: BSD系のOSには kqueue という同様の仕組みが用意されている。
