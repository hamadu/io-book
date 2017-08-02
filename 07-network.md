#
ネットワークI/O

本章では、同一ホストから世界を広げ、ネットワーク越しのコンピュータとデータをやり取りする方法について説明する。
ここで紹介するのは、普段我々が使っているプログラムが内部で用いている。

## ソケット

サーバ側でも、クライアント側でも、まずは通信を行うためのソケットを作る必要がある。ソケットとは、

ソケットを作るには、socket関数を用いる。

### socket

> ```c
> #include <sys/socket.h>
>
> int socket(int domain, int type, int protocol);
> ```
> [Man page of SOCKET](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/socket.2.html)

`domain` には通信に使うプロトコル、`type` には通信方式、
成功するとソケットを示すファイルディスクリプタが返る。

## 接続・サーバ編

サーバサイドでは、ソケットを作ったら、bind関数を用いてローカルアドレスを割り当てる。その後、listen関数を呼び出して、接続キューを用意する。その後 `accept` を呼ぶ度に、キューに溜まっている接続要求があれば。

### bind

> ```c
> #include <sys/socket.h>
> int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
> ```
> [Man page of BIND](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/bind.2.html)

`sockfd` には `socket` で作成したファイルディスクリプタ、`addr` にはアドレス、`addrlen` にはアドレス構造体の大きさを指定する。

### listen

> ```c
> #include <sys/socket.h>
>
> int listen(int sockfd, int backlog);
> ```
> [Man page of LISTEN](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/listen.2.html)

`sockfd` には `socket` で作成したファイルディスクリプタ、`backlog` には接続を待ち受けるキューの長さを指定する。

### accept

> ```c
> #include <sys/socket.h>
>
> int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
> ```
> [Man page of ACCEPT](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/accept.2.html)

`sockfd` には `socket` で作成したファイルディスクリプタを指定する。成功した場合、`addr` および `addrlen` に接続相手のアドレス情報が、関数からはソケットファイルディスクリプタが返る。このソケットはクライアントと接続されている状態であり、`read` または `write` で読み書き要求を行うことができる。

## 接続・クライアント編

クライアントは、接続先が分かってるはずのサーバのアドレスを指定して `connect` 関数を呼ぶことでサーバと接続できる。

### connect

> ```c
> #include <sys/socket.h>
>
> int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
> ```
> [Man page of CONNECT](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/connect.2.html)

`sockfd` には `socket` で作成したファイルディスクリプタ、`addr` には接続相手のアドレス、`addrlen` にはアドレス構造体の大きさを指定する。成功すれば `0` が返り、やりとりができる状態になる。

## データの送受信

ソケットファイルディスクリプタを使って、

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

###


## UNIXドメインソケット
