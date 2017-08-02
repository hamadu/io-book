# 非同期I/Oとイベント駆動I/O

## 多重I/OとC10K問題

現実には、サーバは多数のクライアントと通信を行う必要がある。

[The C10K problem](http://www.kegel.com/c10k.html)

[TheC10kProblem - 「C10K問題」（クライアント1万台問題）とは、ハードウェアの性能上は問題がなくても、あまりにもクライアントの数が多くなるとサーバがパンクする問題のこと](http://www.hyuki.com/yukiwiki/wiki.cgi?TheC10kProblem)

この問題を解決するには、二つの考え方がある。I/Oを非同期にするか、イベント駆動にするかだ。以下、それぞれについて説明する。

## 非同期I/O

そこで、非同期I/Oなる仕組みが規定された。

[Man page of AIO](https://linuxjm.osdn.jp/html/LDP_man-pages/man7/aio.7.html)

## イベント駆動I/O

イベント発火のタイミングは2つの考え方があり、それぞれ **レベルトリガー / level triggered** と **エッジトリガー / edge triggered** と呼ばれている。

[Man page of EPOLL](https://linuxjm.osdn.jp/html/LDP_man-pages/man7/epoll.7.html)
