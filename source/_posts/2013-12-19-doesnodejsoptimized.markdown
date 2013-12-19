---
layout: post
title: "Node.jsは予想通りオプティマイズされたか？ (大量タイマー)"
date: 2013-12-19 10:34
comments: true
categories: 
description: 
keywords: 
---

Javascriptで大量タイマーを利用した場合、普通の実装では、
<pre><code>
while(true) {  
  node_js_check_event {  
    すべてのタイマーを一つずつチェック  ===> 一つずつは効率悪い、オプティマイズすべき。  
    タイムアウト付き、他のイベントをチェック (epollなど)  
  }  
  
  イベントをディスパッチ...
}
</code></pre>  

上記「すべてのタイマーを一つずつチェック」をオプティマイズすべきと思います。  
オプティマイズ方法は、B-TREEみたいな構造でタイマーのfireTimeを保存し、  
チェックはB-TREEから最小fireTimeだけをチェックすれば終わり。

Node.jsはどのように実装したのか？ソースを見てみると、確かにオプティマイズしました！：  
<b>[closest timer of all timers]</b>  
https://github.com/joyent/node/blob/master/deps/uv/src/unix/timer.c #120
<pre><code>
RB_MIN(uv__timers, &loop->timer_handles)  
</code></pre>
  
<b>[pass timeout argument to poll api]</b>  
https://github.com/joyent/node/blob/master/deps/uv/src/unix/core.c #276
<pre><code>
timeout = 0;  
if ((mode & UV_RUN_NOWAIT) == 0)  
    timeout = uv_backend_timeout(loop);  

uv__io_poll(loop, timeout); 
</code></pre>

unixのソースですが、Windows系のソースも似ているロジックが入っています。

最高です。
