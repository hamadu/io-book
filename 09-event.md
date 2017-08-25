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

多重I/Oを実現する仕組みの一つに、イベント駆動I/Oというものがある。考え方は、「複数のファイルディスクリプタを登録しておき、そのどれかからイベントが起こったら然るべき処理を行う」である。ここでいう「イベント」とは、ファイルディスクリプタから `read` を呼ぶ準備ができた、もしくは `write` を呼び出せる状態になったことを指す。

この仕組みをこれを `select`, `poll` という仕組みで実現してきた。ここでは、それらを効率化したLinux独自の仕組みである `epoll` を紹介する。

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
    // error
  }
  for (int i = 0 ; i < nfds ; i++) {
    int fd = events[i].data.fd;
    // process fd here
  }
}
```

## 例: echoサーバ(epoll版)


```c

```
