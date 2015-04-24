title: 闭包(closures)
date: 2015-04-24 19:14:21
tags: [closures, python, javascript, c#, first-class, 闭包, 匿名函数, 函数对象, 自由变量]
categories: 编程基础
---
### 1. 摘要

一直以来，闭包这种编程结构都是一些语言的重要组成部分。在某些场景中使用闭包能够优雅地解决一些棘手的问题。同时闭包的使用有益于模块化编程，它能以简单的方式开发较小的模块，从而提高开发速度和程序的可复用性。

闭包是一门十分有用的技术，但是由于在C#中函数不是一等公民（First-class citizen）的原因，以前在使用C#的时候我没有深入地去关注其中对闭包的支持。而最近一段时间由于工作需要，主要使用 javascript/python 进行开发，在这两种编程语言中函数都是被视为一等对象，实践中大量使用闭包简化编程。例如在javascript中使用闭包模拟对象实例，在python中利用闭包的特性定义功能强大的装饰器等等。

对于这样一门技术，仅仅会使用不是目的，知其然而知其所以然才是。这篇笔记即是出于这样一种目的而整理的。相比较而言我只对C#、javascript、python比较熟悉，而三者中javascript对闭包的支持相对纯粹和完备，所以笔记中我主要用javascript语言表述。但是涉及具体语言的部分，也使用C#和python语言表述，比如对 “闭包陷阱” 的解决，python受限制的闭包实现。

注：笔记相关的代码，主要的目的是描述问题，所以我并没有一一运行过，不能确保都是正确的。但这不影响相关内容。


### 2. 闭包，匿名函数，函数对象，自由变量，好乱的样子

在（一般的）编程语言中，局部变量的作用域仅限于包含它们的函数，脱离了创建它的函数环境后便无法被访问到。但在一些支持嵌套定义函数的语言中，如果内部的函数引用了外部的函数的变量，则可能延长变量的生命周期。例如下面的javascript代码：

```javascript
function foo(x){
    var i = x * 2;
    function(y){
        return i+y;
    }
}

var f1 = foo(1);
var f2 = foo(2);

print(f1(2)); // 4
print(f2(2)); // 6
```

外部 foo 函数执行后返回了foo内部定义的匿名函数，以及不在该内部函数中定义的外部变量i。即使离开了创建i的函数环境，我们依然能够通过f1，f2访问到i。这就是一个典型的闭包（Closure）。

其实闭包并不是一个新概念，而是早在上个世纪60年代高级语言发展初期就已经产生。那么究竟闭包是什么呢？按照维基百科闭包的解释，一般我们有两种定义：

1. 在计算机科学中，闭包（Closure）是词法闭包（Lexical Closure）的简称，指的是引用了自由变量的函数（*自由变量指定的除去函数局部变量之外的其他变量，比如前面例子中的变量i*）。

2. 另一种说法认为闭包是由函数和相关的引用环境组合而成的实体。

从定义上可以看出，这两种对闭包的定义具有完全不同的关注点。对于第一种定义，强调的是闭包是函数，是一类特殊的函数。第二种定义认为闭包是函数和引用环境组成的实体，本质上不再是函数，而是函数对象，能够作为对象使用的函数。术语 **first class function** 是对这个概念的精确描述。函数本质上只是一些可执行的代码，一旦被定义好以后就不会发生变化，没有状态，具有引用的透明性。闭包作为函数对象，可以由同一个函数与不同的引用环境组成不同的实例，有状态，没有引用的透明性，所以闭包不再是单纯的函数。

从理解的角度的来说，第二种定义更为精确和利于理解。

***注：匿名函数与闭包是不同概念，在一般支持闭包的语言中都支持匿名函数，匿名函数可以让我们更容易实现闭包。***

<!-- more -->

### 3. 为什么我们需要闭包

那为什么我们需要闭包呢？这主要是因为在支持嵌套作用域的语言中，有时不能简单直接地确定函数的引用环境。这样的语言一般具有这样的特点：

1. 函数可以嵌套定义，即在一个函数内部可以定义另一个函数。

2. 函数是一阶值（first-class value），函数当作第一类对象（first-class object）——在这些语言中，函数可以被当作参数传递、也可以作为函数返回值、绑定到变量名、就像字符串、整数等简单类型。

还是最上面javascript的例子，我们在执行foo函数返回后，其执行上下文将失效，局部变量i的生命周期也随之结束。后面我们执行foo函数返回的匿名函数时，i不在该匿名函数的作用域范围内，看起来这无法正常工作。也就是说，该匿名函数运行时的引用环境与其定义时的引用环境不同，如果我们按照作用域规则在执行时确定一个函数的引用环境，那这个函数是不能正常工作的。

### 4. 闭包的常见用途

在没有闭包的语言中，变量的生命周期只限于创建它的环境。但在有闭包的语言中，只要有一个闭包引用了这个变量，它就会一直存在。由于闭包的特性，所以一般有以下一些常见的用法：

* 函数式编程语言在内部是无状态的，利用闭包可以实现封装一些状态，实现对象系统。利用同一个函数与不同的引用环境结合起来，我们可能得到不同闭包实例，在javascript中便常常利用闭包来模拟对象实例。

* 多个函数可以使用一个相同的环境，这使得它们可以通过改变那个环境相互交流。

```javascript

var foo = (function() {

    // “闭包” 内的函数可以访问 privateFiled 变量，而 privateFiled 变量对于外部却是隐藏的。privateFiled 就像OO对象实例的私有变量一样。
    var privateFiled = "I'm a private field";

    // "私有" 方法
    function log(msg){
        console.log(msg);
    }

    return {
        getField: function () {
            // 通过暴露的方法来访问 privateFiled
            return privateFiled;
        },
        setField: function (value) {
            // 通过暴露的方法口来修改 privateFiled
            privateFiled = value;
            log("set value.")
        }
    };
}());

foo.getField (); // 得到 'I'm a private field'
foo.privateFiled; // Type error，不能访问
foo.setField ('a new value'); // 修改 privateFiled 的值
foo.getField (); // 得到 'a new value'

```

* 闭包只有在被调用时才执行操作，可用于“惰性求值”或者“延迟加载”。例如,在javascript中模拟python的xrange()函数，我们需要一个固定大小的列表，但是不提前为列表生成所有元素。

```javascript

function xrange(x) {
  var i = 0;
  return {
    hasNext： function(){
        return i < x;
    },
    next: function() {
      return i < x ? (i += 1) : x;
    }
  };
}

var gen = xrange(10000);
while(gen.hasNext()){
    console.log(gen.next());
}

```

* 某些场景下，可以使用闭包对某个函数的参数提前赋值（利用高阶函数，固化已有函数的一个或多个参数，从而产生一个新的函数）。例如python中的偏函数(functools.parial)和javascript的bind函数的功能，从某种角度来看类似高级语言中的重载。

```javascript
// 在js中用js代码简单模拟bind函数本地代码的实现
Function.prototype.bind = function(scope){
    var fn = this;
    args = Array.prototype.slice.call(arguments, 1);
    return function(){
        fn.apply(scope, args.concat(Array.prototype.slice.call(arguments));
    }
}

```

```python
// 在python中简单模拟 functools.parial 的实现
def partial(fn, **out_kwargs):
    def inner(*args, **kwargs):
        for k, v in out_kwargs.items():
            kwargs.setdefualt(k, v)
        return fn(*args, **kwargs)
    return inner
```

### 5. 闭包的陷阱：在循环中创建闭包

闭包的一个常见的陷阱发生于在循环中创建闭包。请看下面的js代码：

```javascript
function foo(count){
    var funcs = [];
    for(var i=0; i < count; i++){
        funcs.push(function() {
        console.log('>>> ' + i);
    })}
    return funcs;
}

var funcs = foo(3);
for(var j=0; j <f1.length; j++) {
    funcs[j]();
}
```

等价的C#实现代码：
```c#
class Program
    {
        static void Main(string[] args)
        {
            Action[] funcs = Foo(3);
            for (int i = 0; i < funcs.Length; i++)
            {
                funcs[i]();
            }
            Console.ReadKey();
        }

        static Action[] Foo(Int32 count)
        {
            Action[] funcs = new Action[count];
            for (int i = 0; i < count; i++)
            {
                funcs[i] = () => Console.WriteLine(">>> {0}", i);
            }

            return funcs;
        }
    }
```

由于自由变量在同一个引用环境（处于同一个作用域）的缘故，上述代码的输出将是：

```javascript
>>> 3
>>> 3
>>> 3
```

而不是

```javascript
>>> 0
>>> 1
>>> 2
```

对于有函数（方法）作用域的c#，我们只需要对上述代码略作修改，给每个自由变量不同的引用环境（不同的作用域），很容易修正这个问题（***多说一句，在c#中闭包的实现是依靠编译器自动生成类来封装变量和函数***）。修改后的代码如下所示：

```c#
class Program
    {
        static void Main(string[] args)
        {
            Action[] funcs = Foo(3);
            for (int i = 0; i < funcs.Length; i++)
            {
                funcs[i]();
            }
            Console.ReadKey();
        }

        static Action[] Foo(Int32 count)
        {
            Action[] funcs = new Action[count];
            for (int i = 0; i < count; i++)
            {
                int j = i; //注意这里，运行时每次循环的j具有不同作用域
                funcs[i] = () => Console.WriteLine(">>> {0}", j);
            }

            return funcs;
        }
    }

```

***但是对于javascript/python这类缺少 "块作用域" 的语言，我们便不能用C#一样的方式来解决这个问题。***  在javascript/python中局部作用域是函数作用域，按照前述对问题的分析，我们修改后的代码如下，为每一个自由变量给予一个函数作用域：

```javascript
function foo(count){
    var funcs = [];
    function print(j) {
        return function(){
            console.log('>>> ' + j);
        }
    }
    for(var i=0; i < count; i++){
        funcs.push(print(i))}
    return funcs;
}

var funcs = foo(3);
for(var j=0; j <f1.length; j++) {
    funcs[j]();
}

```


### 6. python的受限制闭包实现

python 的闭包有一些限制，即 ***“不能对自由变量进行赋值”***，也就是说 python 的闭包是 ***“只读”*** 的。这么说可能不是很好理解，我们比对其他语言的实现来举一个例子来说明这个问题：

现在我们要实现一个累加器函数，它能够生成累加器，即这个函数接受一个参数n，然后返回另一个函数，后者接受参数i，然后返回n增加（increment）了i后的值。


javascript版本：

```javascript
function foo (n) {
    return function (i) {
        return n += i;
    }
}

var f = foo(1);
f(1); //2
f(1); //3
```

等价的C#版本：

```c#
static Func<Int32, Int32> Foo(Int32 n)
{
    Func<Int32, Int32> ret = (i) => n += i;

    return ret;
}
```

参照javascript/C#的实现，你觉得可以用python去实现成这样：
```python
def foo (n):
    def bar(i):
        n = n + i
        return n
    return bar
f = foo(1)
f(1)
f(1)
```

但是上述代码运行时会抛异常：==UnboundLocalError: local variable 'n' referenced before assignment. == 很显然，语句 `n = n + i` 被解释器理解成为（local）局部变量n赋值。(**A name that gets assigned to in a local scope (a function) is always local, unless declared otherwise. While there is the 'global' declaration to declare a variable global even when it is assigned to, there is no such declaration for enclosed variables -- yet. In Python 3.0, there is (will be) the 'nonlocal' declaration that does just that.**)

这就是 python 中闭包的限制，不能完全支持自由变量。
要解决这个问题，我们不得不创造一种数据结构(a mutable container type)，来接受n的值。而且尽管Python确实支持函数数据类型，但是没有一种字面量的表示方式（literal representation）可以生成函数（除非函数体只有一个表达式），所以你需要创造一个命名函数，把它返回。最后的写法如下：
```python
def foo (n):
    s = [n]
    def bar (i):
        s[0] += i
        return s[0]
    return bar
```

或者(虽然实现了功能，但是并不优雅，更像是一种hack手段)：
```python
class foo:
    def __init__ (self, n):
        self.n = n
    def __call__ (self, i):
        self.n += i
        return self.n
```

### 7. 参考资料

* 维基百科关于 [闭包](http://zh.wikipedia.org/wiki/%E9%97%AD%E5%8C%85_(%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%A7%91%E5%AD%A6\) ) 的释义；

* 维基百科关于 [匿名函数](http://zh.wikipedia.org/wiki/%E5%8C%BF%E5%90%8D%E5%87%BD%E6%95%B0) 的释义；

* 维基百科关于 [first class object](http://zh.wikipedia.org/wiki/%E7%AC%AC%E4%B8%80%E7%B1%BB%E5%AF%B9%E8%B1%A1) 的释义；

* [闭包的概念、形式与应用 (IBM DeveloperWorks)](http://www.ibm.com/developerworks/cn/linux/l-cn-closure/)

* MND [闭包（Closures）](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Closures)

* 《黑客与画家》之编程能力 [Appendix: Power](http://www.paulgraham.com/icad.html)

* stackoverflow 中相关问题 [What limitations have closures in Python compared to language X closures?](http://stackoverflow.com/questions/141642/what-limitations-have-closures-in-python-compared-to-language-x-closures)，[How does a javascript closure work ?](http://stackoverflow.com/questions/111102/how-does-a-javascript-closure-work)，[Can you explain closures (as they relate to Python)?](http://stackoverflow.com/questions/13857/can-you-explain-closures-as-they-relate-to-python)