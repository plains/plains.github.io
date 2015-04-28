---
layout: post
title: JavaScript中的浮点数运算
tags: [JavaScript, 浮点数]
image:
  feature: abstract-4.jpg
---
浮点数：是指一个数值包含一个小数点，且小数点后面至少有以一个数字，但是由于ECMAScript的松散特性，所以这样写也不会有问题

{% highlight html %}
var n = 1.; ==> 1
var n2 = 5.0; ==> 5
{% highlight html %}

但是输出结果n变成了整数，这是因为浮点数需要的内存空间是整数的两倍，所以如果该数值本身就表示一个整数ECMAScript就会将该数值转化为整数。

科学计数法：
一些很大的数值或者很小的数值，可以使用(e)科学计数表示：表示的数值等于 e前面的数值乘以10的指数次幂

{% highlight html %}
var n = 2.333e7; //==> 2.333 * 10000000 = 23330000000
var n2 = 0.2e-5; //==> 0.2 * -100000 = -0.000002
//e并不区分大小写
{% endhighlight %}

精度：

>浮点数值的最高精度是17 位小数，但在进行算术计算时其精确度远远不如整数。这是使用基于IEEE754 数值的浮点计算的通病，ECMAScript 并非独此一家；其他使用相同数值格式的语言也存在这个问题。———— JavaScript高级程序设计

以及更详细可以看这里：[JavaScript中的数字](http://blog.jobbole.com/74199/)

所以经常会遇到简单的预算却发生出乎意料的结果，比如说：

{% highlight html %}
1.0 - 0.9 = 0.09999999999999998
1.0 - 0.8 = 0.19999999999999996
1.0 - 0.7 = 0.30000000000000004
...
0.1 + 0.3 = 0.30000000000000004
0.1236 * 1000 = 123.60000000000001
{% endhighlight %}

类似上面的事情很多，如果不在乎精度的情况下，可以使用 `toFixed()`四舍五入保留n为小数。

{% highlight html %}
Number((1.0 - 0.8).toFixed(2)) ==> 0.2
Number((1.0 - 0.8).toFixed(2)) ==> 0.2
Number((1.0 + 0.3).toFixed(2)) ==> 1.3
{% endhighlight %}

PS：`toFixed()` 会将结果变成字符串.

上面的处理方法有些不尽人意，既然小数在计算时会有误差，那么可以将小数转化为整数再将计算后的结果转化回去.
我封装了一个用于`+-*/`运算的的函数

{% highlight html %}
//@浮点数运算
//@config arguments[0] {stirng} 运算符 +-*/
//@config arguments[0...n] {number} 操作数
//@return result {number}
//@case floatCount('*', 0.1236, 100, 0.1)
function floatCount() {
    var res, len, result, i = m = 0,
            arg = [].slice.call(arguments),
            type = "+-*/".indexOf(arg[0]) > -1 ? arg.shift() : "";
    //@取数组中最大小数位数
    //@config arr       {array}
    //@return result    {number}
    function maxPointLen(arr){
        var  n, row, len = 0, res = 0;
        for(n = arr.length; n--;){
            row = arr[n].toString().split('.');
            len = row[1] ? row[1].length : 0;
            res = Math.max(len, res);
        }
        return res;
    }
    //@转化整数,使用乘法转化也会产生进度问题比如 0.1236 * 1000 ==> 123.60000000000001
    //@config num {string}  操作数
    //@config pow {number}  精度
    //@return result {number}
    //@case transInt(0.1236, 4)
    function transInt(num, pow){
        var len, res;
        num = num.toString();
        len = num.split('.');
        len = len[1] ? len[1].length : 0;
        return Number(num.replace('.', '')) * Math.pow(10, pow - len);
    }
    switch (type) {
        case "+":
            m = maxPointLen(arg);
            for (len = arg.length; len--;) {
                res = transInt(arg[len], m);
                result = result ? result + res : res;
            }
            result = result / Math.pow(10, m);
            break;
        case "-":
            m = maxPointLen(arg);
            for (i, len = arg.length; i < len; i++) {
                res = transInt(arg[i], m);
                result = result ? result - res : res;
            }
            result = result / Math.pow(10, m);
            break;
        case "*":
            for (len = arg.length; len--;) {
                res = arg[len] = arg[len].toString();
                m += res.indexOf(".") > -1 ? res.split('.')[1].length : 0;
            }
            m = Math.pow(10, m);
            for (len = arg.length; len--;) {
                res = Number(arg[len].replace(".", ""));
                result = result ? result * res : res;
            }
            result = result / m;
            break;
        case "\/":
            var pointLen = 0, resPointLen, row, pow;
            for (i, len = arg.length; i < len; i++) {
                row = arg[i];
                res = row.split('.');
                pointLen = res[1] ? res[1].length : 0;

                if (result != void 0) {
                    resPointLen = (resPointLen = result.toString().split('.')[1]) ? resPointLen.length : 0;
                    m = Math.max(pointLen, resPointLen);    
                    result = transInt(result, m) / transInt(row, m);                      
                } else {
                    result = row;
                }
            }
            break;
        default:
            throw new Error("第一个参数必须是操作字符");
            break;
    }
    return Number(result);
}
{% endhighlight %}

[demo在这里](/demo/flaotNumber/index.html)

注意：浮点数的最大精度是17位，如果操作数的精度超出的话会导致计算结果有误差，还有在进行除法操作时出现循环小数的时候结果会被四舍五入.
