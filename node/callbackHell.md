## 浅浅的谈一下回调地狱的问题
以前编写c/c++的时候,真心不知道啥是回调地狱 , 为啥呢? 
因为以前编程的时候 , 代码的编写顺序就是执行顺序。
比如我去读取一个文件 (代码简写)

    std::ifstream t;
    t.open("file.txt");
    buffer = new char(length);
    t.read(buffer , length);


在这里 , 代码执行到read的时候,会阻塞 , 直到文件读完,不管失败还是成功都会有一个结果 , 这时候代码才会继续执行.</br>
现在写js的时候 , 读取一个文件是这样的:

    const fs = require('fs');
    fs.readFile("file.txt",function(err , result){
        // 获取结果 , 执行相关的业务代码
    })
    code...

可以看出,在js里,当执行读取文件的代码后,没有去等文件的执行结果,代码直接向下执行 , 当读取文件有结果的时候,在那个回调函数中执行相关的业务代码。</br> 
对于一个一直编写同步代码,秉承着面向对象就是上帝的程序员小白,看到这段代码内心是崩溃的😢

1:这里代码的编写顺序竟然不是代码的执行顺序！</br>
2:调用函数 , 还能传一个函数为参数(难道是函数指针,但是为毛线要这样做,这都是什么鬼??！）</br>
3:为什么在那个回调函数里,能获取到读取文件结果的信息?? </br>

## 什么是函数式编程 ??
简单说,函数式编程是一种“编程范式”,最直观的感觉就是,函数是一种对象类型,可以作为参数传给别的函数,也可以作为结果return; </br>
崩溃的我在学习js的路上停滞不前,心里一直鄙视这种js这种解释性语言，搞什么函数式编程,有毛线用??</br>
直到有一天碰到了函数式编程的上帝 , 他语重心长的和我说:</br>
我们函数式编程天生是为并发编程而生的啊，你看看函数没有side effect，不共享变量，可以安全地调度到任何一个CPU core上去运行，没有烦人的加锁问题，多好啊。</br> 
现在想想自己真的很小白 , 只知道面向对象编程 , 面向过程编程 , 对函数式编程完全无感。。
## 爱上了函数式编程的我又遇到了新的麻烦 (回调地狱)
在编写js的过程中, 事件得到了广泛的应用,配合异步I/O,将事件点暴露给业务逻辑。</br>
事件的编程方式具有轻量级,松耦合,只关注事物点等优势。</br> 但是在多个异步任务的情景下,事件与事件之间如何独立,如何协作是一个问题。</br>
在node中,多个异步调用的情景有很多.比如遍历一个目录

    fs.readdir(path.join(__dirname,'..'),function(err , files){
        files.forEach(function(filename , index){
            fs.readFle(filename , function(){
                ....
            })
        })
    })

这是异步编程的典型问题 , 嵌套过深 , 最难看的代码诞生了... 
对于一个对代码有洁癖的人 , 这个真心不能忍!! 

## 目前我所知道回调地狱的解决方式
Promise

协程 

eventEmitter(事件发布订阅模式)

## promise
promise , 感觉就是把要执行的回调函数拿到了外面执行 , 使代码看起来很"同步"~ </br>
看下promise如何实现的吧 

    let promise = new Promise(function(resolve , reject){
        // 执行异步代码的调用 
        async(function(err , right){
            // 完全是可以根据返回的数据 , 直接执行相应的逻辑 , 不过为了让代码看着"好看同步" , 决定把数据当作参数传递给外面,</br>
            去外面(then的回调函数里 , 或者catch的回调函数里)执行 
            // 根据返回的数据 , 来确定该调用哪个接口 
            if(right){
                resolve("data"); 
            }
            if(err){
                reject('err') 
            }
        })
    })  
    // 如果执行了resolve() , 就走到这里 
    .then(function(data){
        coding..
    })
    //如果执行了reject , 就走到了这里 
    .catch(function(err){
        coding..
    })

这里可以看出 , 调用异步代码之后 , 已经获取了返回的数据 。</br>
为什么执行了resolve('data'), 或者reject('err')后, then的回调函数, 或者catch的回调函数就知道 , '该到我执行的时候到了呢' , 说白了就是有人“通知”我了呗!</br>
resolve , reject 本就是promise自己定义的方法 , 内部实现大概是这样</br>

当调用resolve('data')的时候 , 去通知.then里绑定的回调函数 , 通知你一下 , 你该执行了 , 这是参数   this.emit(‘resolve’ , 'data')

当调用reject('err')的时候 , 去通知.catch里绑定的回调函数 , 通知你一下 , 你该执行了 , 这是参数   this.emit('reject' , 'err')

在调用.then(callback)的时候 , callback , 你监听下‘resolve’ , 有人通知(emit)你的时候 , 你就执行 

在调用.catch(callback)的时候 , callback , 你监听下‘reject’ , 有人通知(emit)你的时候 , 你就执行 

简单说 , 就是把回调函数拿到了外面执行 , 让代码看着'同步' 
自己简单实现了下promise , 源码在这里 <a href="https://github.com/Tankas/Implement-it-myself-with-code/blob/master/promise/promise.js">Promise的简单实现</a>

## 协程
首先说下协程的定义 : 协程是一个无优先级的子程序调度组件 , 允许子程序在特定的地方挂起和恢复.</br>
线程包含于进程，协程包含于线程。只要内存足够，一个线程中可以有任意多个协程，但某一时刻只能有一个协程在运行，多个协程分享该线程分配到的计算机资源。</br>
协程要做的是啥 , 写同步的代码却做着异步的事儿。
### 何时挂起?? 何时恢复呢??
挂起 : 在协程发起异步调用的时候挂起</br>
恢复 : 其他协程退出并且异步操作完成时。
## Generator --> 协程在js中的实现
写个🌰先 : 

    function* generator(x){
        var a = yield x+2;
        var b = yield a+3;
        var c = yield b+2;
        return;
    }

最直观的感觉: 当调用generator(1)时,其实返回了一个链表.每一个单元里装一些函数片段 , 以yield为界线 , 向上面的例子 </br>
(x+2;)  --> (a+3) ---> (b+2) ---> (return;);</br>
每次都通过next()方法来移动指针到下一个函数片段,执行函数片段(eval) , 返回结果.

    var gen = generator(2);
    gen.next(); // 当调用next(),会先走第一个代码段 , 然后就不执行了 , 交出控制权 .直到啥时候再执行next(),会走下一个代码段.


这里可以看出来 , 我们完全可以在每个代码段都封装一个异步任务 , 反正在异步任务执行的时候 , 我已经交出了控制权 , js主线程的代码继续往下走 , 啥也不耽误 , 等到异步任务完成的时候, 通知我一下 , 我这边看看等到其他协程也都退出的时候 , 就调用next() , 继续往下走.. 这样下来 ， 看看代码多"同步" , 是不是～～～ </br>
继续看下  , 当调用next("5")时 , 里面是可以传入参数 , 而且传入的参数是上一个yield的异步任务的返回结果 .</br>
可以说这个特性非常有用,就像上面说的,当异步任务完成的时候,就再调用next() , 走下面的代码 , 但是没法获取到上一个异步任务的结果的 , 所以这个特性就是做这个的 , next('异步任务的结果');

## async/awit
说到async/awit , 最直观的感觉 , 不就是对gennerator的封装 , 改个名么?? 

    let gen = async function(){      
        let f1 = await readFile("one");
        let f2 = await readFile2(123123);       
    }

简单说 , async/awit 就是对上面gennerator自动化流程的封装 , 让每一个异步任务都是自动化的执行 , 当第一个异步任务readFile("one")执行完 , async内部自己执行next(),调用第二个任务readFile2(123123),以此类推...
### 这里也许有人会困惑 , 为什么wait 后面返回的必须是promise ?? 
是这样 , 上面说了当第一个异步完成时通知我一下 , 我在调用next() , 继续往下执行 , 但是我什么时候完成的, 怎么通知你??</br> promise就是做这件事的 , async内部会在promise.then(callback),回调函数里调用 next()... (还有用Thunk的, 也是为了做这个事的);

## eventEmitter(事件发布订阅模式)
事件发布订阅模式广泛应用于异步编程的模式,是回调函数的事件化 , 可以很好的解耦业务逻辑 , 也算是一种解决回调地狱的方式 , 不过和promise,async不同的是 , promise,async就是为了解决回调地狱而设计出来的 , 而eventEmit是一种设计模式 , 正好可以解决这个问题～～

    // 订阅 
    emitter.on('event1',function(message){
        console.log(message);
    })
    // 发布
    emitter.emit('event1',data);

事件发布订阅模式可以实现一个事件与多个回调函数的关联，这些回调函数又称为事件监听器.听过emit发布事件后 , 消息会立即传递给当前事件的所有监听器执行.

事件发布订阅模式常常用来解耦业务逻辑,事件的发布者无需关注订阅的监听器如何实现业务逻辑 , 数据通过消息的方式灵活传递。

    fs.readFile("file.txt",function(err , result){
        // 获取结果
        emmitter.emit('event1' , result)
    })
    emmiter.on('event' , function(result){
        // 在这里执行相关业务代码 
    })

## 事件发布订阅模式咋实现的呢 ??? 
简单说就是在上层维护一个私有的callback回调函数队列 , 每次emmit的时候都会遍历队列 , 把相应的事件拿出来执行。<a href="https://github.com/Tankas/Implement-it-myself-with-code/blob/master/eventEmitter/event.js">eventEmit的简单实现</a>

## 总结 
这里可以看出,不管是哪一种处理回调地狱的方式 , 都是要处理回调函数的, 只不过是真正调用的位置不同而已~ ,上面三种方式做的都是如何组织回调函数链的执行位置 , 如何让代码看着更好看 ~~ 
    






