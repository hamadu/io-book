# イベント駆動I/O

前の章で紹介したクライアント／サーバ間の通信は、1

現実には、サーバは多数のクライアントと通信を行う必要がある。


## 多重I/OとC10K問題

現実には、サーバは多数のクライアントと通信を行う必要がある。

[The C10K problem](http://www.kegel.com/c10k.html)

[TheC10kProblem - 「C10K問題」（クライアント1万台問題）とは、ハードウェアの性能上は問題がなくても、あまりにもクライアントの数が多くなるとサーバがパンクする問題のこと](http://www.hyuki.com/yukiwiki/wiki.cgi?TheC10kProblem)


## イベント駆動I/O



イベント発火のタイミングは2つの考え方があり、それぞれ **レベルトリガー / level triggered** と **エッジトリガー / edge triggered** と呼ばれている。

[Man page of EPOLL](https://linuxjm.osdn.jp/html/LDP_man-pages/man7/epoll.7.html)
