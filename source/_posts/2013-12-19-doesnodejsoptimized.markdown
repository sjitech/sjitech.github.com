---
layout: post
author: Jin Qian
title: "Node.jsは予想通りオプティマイズされたか？ (大量タイマー)"
date: 2013-12-19 10:34
comments: true
categories: [Tips]
tags: [Node.js JavaScript]
description: Node.jsの内部実現の確認
keywords: Node.js, JavaScript, NodeJS, 性能
---

Node.jsのタイマーの使用を調査しました。
<!-- more -->

Javascriptで大量タイマーを利用した場合、普通の実装では、

{% codeblock lang:javascript %}
while(true) {  
  node_js_check_event {  
    すべてのタイマーを一つずつチェック  ===> 一つずつは効率悪い、オプティマイズすべき。  
    タイムアウト付き、他のイベントをチェック (epollなど)  
  }  
  
  イベントをディスパッチ...
}
{% endcodeblock %}

上記「すべてのタイマーを一つずつチェック」をオプティマイズすべきと思います。  
オプティマイズ方法は、B-TREEみたいな構造でタイマーのfireTimeを保存し、  
チェックはB-TREEから最小fireTimeだけをチェックすれば終わり。

Node.jsはどのように実装したのか？ソースを見てみると、確かにオプティマイズしました！：  

{% codeblock lang:c closest timer of all timers start:120 https://github.com/joyent/node/blob/master/deps/uv/src/unix/timer.c  timer.c %}
  RB_MIN(uv__timers, &loop->timer_handles)
{% endcodeblock %}

{% codeblock lang:c pass timeout argument to poll api start:276 https://github.com/joyent/node/blob/master/deps/uv/src/unix/core.c core.c%}
timeout = 0;  
if ((mode & UV_RUN_NOWAIT) == 0)  
    timeout = uv_backend_timeout(loop);  

uv__io_poll(loop, timeout); 
{% endcodeblock %}

unixのソースですが、Windows系のソースも似ているロジックが入っています。

最高です。
