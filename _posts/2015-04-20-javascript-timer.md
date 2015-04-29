---
layout: post
title: javaScript计时器
tags: [css]
image:
  feature: abstract-7.jpg
---
javaScript中有两个计时器的函数
*	setTimeout(func, delay)
*	setInterval(func, delay)

它们都接受执行函数和延迟时间两个参数,不同的是setTimeout只会调用一次，setInterval会周期调用。
相对的还有清除计时器的函数``clearTimeout(timer) clearInterval(interval)``

计时器不一定在指定的延迟时间后执行，因为js是单线程，一定时间内只能执行一段代码，so...为了控制代码的执行顺序(异步事件)，所以产生了一个执行队列，所有的执行任务都要在队列中排队，等待上一个任务执行完成，再执行，盗的别人的图：
![线程机制](/images/1/thread_01.jpg)](http://hi.baidu.com/isabella_qi/item/766061fd5ac1a0d7a835a2b4)

当计时器被激活开始计时，在时间到了之后才会push入执行队列，那么如果此时队列是空的，计时器的执行函数马上就会执行，那么如果队列不是空的呢？

之前看到的一段暴力的代码，我改动了一下

{% highlight javascript %}
setTimeout(function(){
	console.timeEnd('star');
	alert("放我出来");
} , 1000)
console.time('star');
var now = new Date(),
    nSeconds = new Date().setSeconds(now.getSeconds() + 5);
do{
    now = new Date();
}while(now.getTime() < nSeconds);
{% endhighlight %}

如果你尝试了，会发现``alert``延迟了5s才执行，因为``do while``一直在堵着执行队列。

还有在使用``setInterval``周期性执行某一段代码时，如果执行时间大于delay时间呢，那么下一个interval依旧会进入队列等待执行。
那么如果第一次的执行时间很长，大于多个interval的延迟时间又会怎么样呢？
如果如上所说，那么下面的count1执行的3s中内至少会有两个执行函数进入执行队列(执行时间的误差)，然后几乎无延迟执行。

{% highlight javascript %}
!function(){
	var count = 0,
		timer = null,
		prevDate = null;
	function func(count){
		var now = new Date(),
			interval,
			nSeconds = new Date().setSeconds(now.getSeconds() + 3),
			str = '';

		if(count < 2 || count > 5){
			str =  "";
			do{
				now = new Date();
			}while(now.getTime() < nSeconds)
		}else{
			str = '--无阻塞';
		}

		!prevDate && (prevDate = now);
		interval = now.getTime() - prevDate.getTime();
		prevDate = now;
		console.log("执行第" + count + '次 间隔时间：' + interval + '毫秒 ' + str);
	}

	timer = setInterval(function(){
		console.time("timer");
		func(++count);
		console.timeEnd("timer");
	}, 1000);
}()
{% endhighlight %}

我做了一个delay为1s的interval，在执行函数中做了3s的阻塞，且取消了count2，count3，count4，count5的阻塞，这是在chrome v.42中的测试结果：

![测试结果](/images/1/timer_01.png)

从上图中，可以看到，只有cuont2无延迟执行了(4ms是因为最小延迟)。
所以我猜想：如果执行队列中已经有一个interval在等待执行，那么后面的interval就像是被暂停了一样停止计时，直到上一个等待的interval执行完成再恢复正常。
[我是demo!!](http://jsbin.com/cavipu/1)
PS：**++我并没有找到任何文档来解释上面理论，这只是我的猜测，如果您能帮我证明这个理论是正确的，万分感谢++**

#####计时器的最小延迟
经常能看到这样的面试题

{% highlight javascript %}
function func(){
	var a = 0;
    setTimeout(function(){
    	alert(a);
    }, 0)
    ++a;
}
{% endhighlight %}

问:``alert``会弹出什么? 结果是弹出了1。
关于delay延迟时间，不同的浏览器都会有自己的最小延迟时间，所以就算你写成0，也会按照浏览器的最小延迟时间执行，比如Chrome的最小延迟时4ms,IE的是23毫秒，当然这个时间并不准确，可能会受到硬件的影响。

#####JS线程与GUI线程互斥
这个我写了很久，最后还是觉得没有 [@轻薄](http://qingbob.com/difference-between-settimeout-setinterval/) 写的好，为了不误人子弟，我还是放弃了，不过文章中关于多个`setInterval`会被"忽略"的理论，已经在上面被证实是错误的。


还有在 [@轻薄](http://qingbob.com/difference-between-settimeout-setinterval/) 文章中留的关于“变色”作业的答案的实现是错误的，这是原文的的代码：


{% highlight javascript %}
(function run() {
    var div = document.getElementsByTagName('div')[0]
    for(var i=0xA00000;i<0xBBBBBB;i++) {
        (function (color) {
            setTimeout(function () {
                div.style.backgroundColor = '#' + color.toString(16);    
            });
        })(i);
    }
})()
{% endhighlight %}

这段代码运行起来非常的卡，原因是因为在for循环中几乎同时(这个词不太恰当，但是for循环的速度是非常快的)初始化了非常多个`setTimeout`，且它们的执行时间非常相近，可以计算一下有多少个`setTimeout`在执行。


{% highlight javascript %}
var count = 0;
(function run() {
    var div = document.getElementsByTagName('div')[0]
    for(var i=0xA00000;i<0xBBBBBB;i++) {
        ++count;
        ...
    }
})()
{% endhighlight %}

只是加了个count 字段，在变色开始时，在console面板中打印`count ==> 1817531`，初始化接近两百万个`setTimeout`想不卡都难~~
而且还有一点就是由于这些`setTimtout`的间隔时间非常的短，导致我们看到的颜色变化比较少,这是我重写后的代码


{% highlight javascript %}
!function run2() {
    var div = document.getElementById('crice');
    function fun(color){
        if(color < 0xBBBBBB){
            setTimeout(function(){
                div.style.backgroundColor = '#' + color.toString(16);
                fun(color += 3333);	//增加颜色的区间值
            }, 0)
        }
    }
    setTimeout(function(){
        fun(0xA00000);
    }, 0);
}()
{% endhighlight %}

✿如果你被上面的代码✧(≖ ◡ ≖)闪瞎了\*\*\*眼，就增加以下延迟时间吧，`333`你值得拥有！

我还发现了一个有趣的事情，是关于GUI线程和js线程互斥,我写了这样的代码

{% highlight javascript %}
var ul = document.createElement("ul"), li;
document.body.appendChild(ul);
console.time("timer");
for(var i = 20000; i--;){
    li = document.createElement('li');
    li.innerHTML = i*i;
    ul.appendChild(li);
}
console.timeEnd("timer");
{% endhighlight %}

在Chrome和IE中timeEnd输出都在dom渲染之前发生，但是我又多加了一句

{% highlight javascript %}
var ul = document.createElement("ul"), li;
document.body.appendChild(ul);
console.time("timer");
for(var i = 20000; i--;){
    li = document.createElement('li');
    li.innerHTML = i*i;
    ul.appendChild(li);
}
console.log(ul.offsetHeight);
console.timeEnd("timer");
{% endhighlight %}

没错，就是输出ul的offsetHeight，然后我又试了`clientWH, offsetWH, scrollWH`，发现timeEnd输出的时间和dom渲染完成时间一致。我猜这可能是浏览器为了保证能够及时获取到元素布局所做的调整吧.

#####扩展
在上文中提到过JavaScript是单线程的，还有执行队列，但是实际上经常会遇到异步代码，比如说：

-	计时器
-	浏览器事件触发
-	ajax异步请求

那么这些异步代码是怎么执行的呢？JavaScrip确实是单线程的，但是浏览器是有多个线程的，关于线程详细的可以看这里： [http://hi.baidu.com/isabella_qi/item/766061fd5ac1a0d7a835a2b4](http://hi.baidu.com/isabella_qi/item/766061fd5ac1a0d7a835a2b4)

由此可知异步代码是由事件触发线程负责的，so... 容我偷个懒，请点击这里 [Async JavaScript](http://jser.it/blog/2014/05/18/async-javascript/)

