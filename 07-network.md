# ネットワークI/O

本章では、同一ホストから世界を広げ、ネットワーク越しのコンピュータのプロセスとデータをやり取りする方法について説明する。

## ソケット

サーバとクライアントが通信するには、まず双方でソケットを作る必要がある。ソケットとは通信方式を取り決めたインタフェースである。ソケットの作成時に、通信に使うプロトコルと、通信形式を予め決められた中から選ぶ。これらはサーバ-クライアント間で同一である必要がある。



### socket

ソケットを作成するには、socket関数を用いる。

> ```c
> #include <sys/socket.h>
>
> int socket(int domain, int type, int protocol);
> ```
> [Man page of SOCKET](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/socket.2.html)

`domain` には通信に使うプロトコルを表す。例えば以下の値が利用可能である。

| 名前 | プロトコル |
| :--- | :--- |
| AF_INET | IPv4 |
| AF_INET6 | IPv6 |
| AF_UNIX | ローカルホスト内での通信 |

`type` には通信方式を指定する。

| 名前 | 通信方式 |
| :--- | :--- |
| SOCK_STREAM | TCP |
| SOCK_DGRAM | UDP |

`protocol` には　`domain` に指定した通信ドメイン固有のプロトコル（もしあれば）を指定する。作成に成功すると、ソケットディスクリプタという通信を一意に識別する数値が帰ってくる。ファイルの読み書きと同じように、通信相手からデータを読んだり、データ送ったりする関数を呼ぶ際にこの値を渡す。また、後述するが `read` や `write` 関数にソケットディスクリプタを渡せば、データの送受信に使うこともできる。


`type`


## 接続・サーバ編

サーバサイドでは、ソケットを作ったら、bind関数を用いてローカルアドレスを割り当てる。その後、listen関数を呼び出して、接続キューを用意する。その後 `accept` を呼ぶ度に、キューに溜まっている接続要求があれば受け入れ、接続可能なソケットを作成する。

### bind

`bind` 関数は、ソケットにアドレスを割り当てる。

> ```c
> #include <sys/socket.h>
> int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
> ```
> [Man page of BIND](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/bind.2.html)

`sockfd` には `socket` で作成したファイルディスクリプタ、`addr` にはアドレス、`addrlen` にはアドレス構造体の大きさを指定する。`addr` に指定する構造体は、ソケットを作る時に指定したアドレスファミリーに依存する。例えば、

### listen

`listen` 関数は、指定したソケットを待ち受け状態にする。

> ```c
> #include <sys/socket.h>
>
> int listen(int sockfd, int backlog);
> ```
> [Man page of LISTEN](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/listen.2.html)

`sockfd` には `socket` で作成したファイルディスクリプタ、`backlog` には接続を待ち受けるキューの長さを指定する。

### accept

`accept` 関数は、待ち受けキューに溜まっている接続要求を受け入れ、クライアントと通信可能なソケットディスクリプタを作成する。

> ```c
> #include <sys/socket.h>
>
> int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
> ```
> [Man page of ACCEPT](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/accept.2.html)

`sockfd` には `socket` で作成したファイルディスクリプタを指定する。成功した場合、`addr` および `addrlen` に接続相手のアドレス情報が、関数からはソケットファイルディスクリプタが返る。このソケットは socket関数で作成したものとは異なり、クライアントと接続されている状態の新しいものだ。これに対して `read` および `write` または後述する関数群を用いるとデータの送受信が行える。

## 接続・クライアント編

クライアントは、サーバと同じ通信方式のソケットを作成し、サーバのアドレスを指定して `connect` 関数を呼ぶことでサーバと接続できる。

### connect

> ```c
> #include <sys/socket.h>
>
> int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
> ```
> [Man page of CONNECT](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/connect.2.html)

`sockfd` には `socket` で作成したファイルディスクリプタ、`addr` には接続相手のアドレス、`addrlen` にはアドレス構造体の大きさを指定する。成功すれば `0` が返り、やりとりができる状態になる。

## データの送受信

通信が確立した後は、ソケットディスクリプタを使ってデータの送受信が可能になる。1章で紹介した `read` 関数や `write` 関数も使えるが、詳細なオプションを指定できる関数が用意されている。

### send

> ```
> #include <sys/types.h>
> #include <sys/socket.h>
>
> ssize_t send(int sockfd, const void *buf, size_t len, int flags);
> ```
> [Man page of SEND](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/send.2.html)

### recv

> ```c
> #include <sys/types.h>
> #include <sys/socket.h>
>
> ssize_t recv(int sockfd, void *buf, size_t len, int flags);
> ```
> [Man page of RECV](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/recv.2.html)


## 例

さて、ここまで紹介した関数を使ってサーバ-クライアント間の通信を行うプログラムを作ってみよう。今回はクライアントから接続があると、適当なメッセージを送り返して通信を終了させるサーバを書こう。

### サーバプログラム

```c
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <string.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>

void print_error_and_exit() {
  printf("error(%d): %s\n", errno, strerror(errno));
  exit(1);
}

int main(int argc, char* argv[]) {
  int fd = socket(AF_INET, SOCK_STREAM, 0);
  if (fd == -1) {
    print_error_and_exit();
  }

  struct sockaddr_in addr;
  memset(&addr, 0, sizeof(addr));
  addr.sin_family = AF_INET;
  addr.sin_port = htons(8080);
  inet_aton("127.0.0.1", &addr.sin_addr);
  if (bind(fd, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
    print_error_and_exit();
  }

  if (listen(fd, 100) < 0) {
    print_error_and_exit();
  }

  while (1) {
    int peerFd = accept(fd, NULL, NULL);
    char* data = "Good day!";
    write(peerFd, data, strlen(data));
    close(peerFd);
  }

  return 0;
}
```

まず、ソケットを作成する。アドレスファミリーには `AF_INET`(IPv4)、通信方式には `SOCK_STREAM`(TCP) を指定する。

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);
if (fd == -1) {
  print_error_and_exit();
}
```

次に、待ち受け用のアドレスを設定して、`bind` でソケットディスクリプタに紐付ける。今回はIPv4での接続を待ち受けるため、構造体 `struct sockaddr_in` に情報を詰める。`sin_family`, `sin_port`, `sin_addr` にはそれぞれアドレスファミリー、ポート番号、IPアドレスを指定する。

bind の第二引数には作成したアドレス情報を渡すのだが、`struct sockaddr *` を渡す必要があるのでキャストしている。

```c
struct sockaddr_in addr;
memset(&addr, 0, sizeof(addr));
addr.sin_family = AF_INET;
addr.sin_port = htons(8080);
inet_aton("127.0.0.1", &addr.sin_addr);

if (bind(fd, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
  print_error_and_exit();
}
```

次に、`listen` を呼び出してソケットを待ち受け状態にする。ここではキューの長さを適当に100に設定している。

```c
if (listen(fd, 100) < 0) {
  print_error_and_exit();
}
```

最後に、クライアントからの接続を `accept` で受け、クライアント向けのソケットディスクリプタを受け取り、それを使ってデータを送信する。ちなみに `accept` 関数は接続がない場合処理をブロックするので、この実装のままだと同時に1クライアントしか処理ができない。

```c
while (1) {
  int peerFd = accept(fd, NULL, NULL);
  char* data = "Today is another good day!";
  write(peerFd, data, strlen(data));
  close(peerFd);
}
```

### クライアントプログラム

```c
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <string.h>
#include <errno.h>
#include <stdio.h>
#include <stdlib.h>

void print_error_and_exit() {
  printf("error(%d): %s\n", errno, strerror(errno));
  exit(1);
}

int main(int argc, char* argv[]) {
  int fd = socket(AF_UNIX, SOCK_STREAM, 0);

  struct sockaddr_un addr;
  memset(&addr, 0, sizeof(addr));
  addr.sun_family = AF_UNIX;
  strcpy(addr.sun_path, "foo.sock");
  if (connect(fd, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
    print_error_and_exit();
  }

  char buf[120];
  read(fd, buf, 120);
  printf("== response from server ==\n%s\n", buf);

  close(fd);

  return 0;
}
```

まず、ソケットを作成し、接続先のアドレスを構造体 `sockaddr_in` に詰める。

```c
int fd = socket(AF_INET, SOCK_STREAM, 0);

struct sockaddr_in addr;
memset(&addr, 0, sizeof(addr));
addr.sin_family = AF_INET;
addr.sin_port = htons(8080);
inet_aton("127.0.0.1", &addr.sin_addr);
```

`connect` 関数を呼んでサーバに接続する。接続に成功すると、ソケットディスクリプタを使って読み書きが利用可能になる。

```c
if (connect(fd, (struct sockaddr*)&addr, sizeof(addr)) < 0) {
  print_error_and_exit();
}
```

最後に、サーバから送られたデータを読んでその内容を出力する。その後、`close(fd)` を呼び出して通信を終了する。

```c
char buf[120];
read(fd, buf, 120);
printf("== response from server ==\n%s\n", buf);

close(fd);
```

### まとめ

次章では本章で説明したソケットの仕組みを用いて、同ホスト上のプロセス間通信を行う方法を見ていく。
