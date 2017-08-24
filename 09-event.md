# イベント駆動I/O

前の章で紹介したサーバの実装はクライアントを単一のキューで待ち受け、一つずつ順番に実行するものであった。つまり、もし長い処理が必要なクライアントがあった場合、後続を待たせてしまう。

(まちぼうけのず)

なので、実際にはクライアントの処理そのものは、待受処理とは異なるプロセスもしくはスレッドで行う等の策が取られる。しかし、それでもI/Oの処理は「待ち」が発生するため、ボトルネックになりがちである。本章では、同時接続可能なechoサーバの実装を通じて、イベント駆動I/Oの仕組みを紹介する。


## 例: echoサーバ



```c
```


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

`epoll` は複数のファイルディスクリプタを管理する。まず初期化をする。

> ```c
> #include <sys/epoll.h>
>
> int epoll_create(int size);
> ```
> [Man page of EPOLL_CREATE](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/epoll_create.2.html)

呼び出しに成功すると、ファイルディスクリプタが返される。


### epoll_ctl(2)

`epoll_ctl` 関数を使うと、指定したファイルディスクリプタをepollオブジェクトに登録したり、できる。

> ```c
> #include <sys/epoll.h>
>
> int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);   
> ```
> [Man page of EPOLL_CTL](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/epoll_ctl.2.html)


`epfd` には `epoll_create` で作成した epollオブジェクトのファイルディスクリプタを渡す。`fd` には対象のファイルディスクリプタを渡す。`op` は操作内容を指定する。

|||

たとえば、ファイルを追加する時は次のようにする。

```c

```

### epoll_wait(2)

登録済のファイルディスクリプタからのI/Oイベントを待つ。

> ```c
> #include <sys/epoll.h>
>
> int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
> ```
> [Man page of EPOLL_WAIT](https://linuxjm.osdn.jp/html/LDP_man-pages/man2/epoll_wait.2.html)

`epfd` には epollオブジェクトのファイルディスクリプタを渡す。




## 例: echoサーバ(epoll版)


```c
```




