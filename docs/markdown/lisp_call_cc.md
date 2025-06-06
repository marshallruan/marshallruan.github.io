Scheme语言有一个非常特殊的机制，唤作`call/cc`，它可以在程序的任意位置得到其“延续”，英文是continuation。这货简直是著名的brain fucker，极尽精巧神秘之能事，据说由混沌初开之时远古诸神的祝福而生。  
我可以向你保证，如果不在编程环境中反复测试和hacking，几乎没有人类能一眼看出这个东西的正确解释。  

##1. continuation实例
“延续”（continuation）是计算过程中程序控制状态的一种抽象，它可以用数据结构或函数表示。在C语言中，这一数据结构不允许程序员访问和修改，而某些Lisp方言（如Scheme）则可以在外部显式的操作它们，从而带来了后续一系列神奇的蝴蝶效应。  
稍微更准确的说，在某个程序中，任意位置的continuation可视为计算机完成这个位置的事情之后，后续需要做的事情。  

**吐槽：这个东西后人翻译为“后续”，有人为了提升格调翻译成“延续”，但总觉得差点意思。**   

比如，有以下表达式，它在C语言和Lisp语言下的计算逻辑完全相同：  
```Scheme
(if (null? x)  ;如果x是空 
    '()        ;则返回空链表
	(cdr x))   ;否则返回去掉第一个元素后的新链表
```

这个表达式的任意位置（也就是子表达式）的continuation动作可以用很简单的方式去思考，我们可以用`A`符号来替换掉该位置的代码，那么剩下来要进行的运算就是延续，如下：  
1. `if`语句的延续可以理解为`(A (null? x) '() (cdr x))`；  
2. `(null? x)`的延续是`(if A '() (cdr x))`；  
3. `'()`的延续是`(if (null? x) A (cdr x))`；  
4. 以此类推。  

很明显，延续本身就是一个函数，更精确的表示如下：  
```Scheme
;在注释中，我们用C-x表示x的continuation

; C-if
;  你可以直接在原函数里面挖掉if用k替代，得到(k (null? x) '() (cdr x))，
;  然后再套一个lambda即得到这个函数的Lisp表达式，这个表达式本身就是一个Lisp函数
(lambda (k) 
  (k (null? x) '() (cdr x)))
  
; C-(null? x)
(lambda (k)
  (if k '() (cdr x)))

; C-'()
(lambda (k)
  (if (null? x) k (cdr x)))

; C-'(cdr x)
(lambda (k)
  (if (null? x) '() k))

; 玩点更高级的，C-(null? x)语句中的x
(lambda (k)
  (if (null? k) '() (cdr x)))
  
;C-(cdr x)语句中的x
(lambda (k)
  (if (null? x) '() (cdr k)))

;C-cdr
(lambda (k)
  (if (null? x) '() (k x)))

;......
  
```

没错，用Lisp的S表达式使得问题更加简单，其实continuation就是在抽象语法树中先挖掉当前位置，用参数k替代，然后对替代后的树套上一个lambda根节点，从而让这棵树变成一个新函数，就这么简单。C语言下实在是没法想出这么直接的解释。  
但无论如何，在上面的例子中，continuation的概念已经足够清晰。  

##2. call/cc语句
在C语言等一大票摩登语言中，你无法操作这些`C-k`的函数。而古老的Scheme语言提供了一种可以调用当前位置延续(current continuation)的函数，叫做`call-with-current-continuation`，简称`call/cc`。它可以让你在程序的任意位置得到continuation，进而创造各种神奇的可能性。  
`call/cc`的调用规则是`(call/cc f)`，其中`f`是一个单参函数，也就是`(call/cc (lambda (k) <body>)`。一旦程序中某个位置被这样的语句包裹起来，那么`k`会绑定为当前位置的延续，一旦在`<body>`中存在`(k value)`形式的调用，程序会跳到外面的世界。  
网上很多教程说存在`(k value)`调用时，`call/cc`的返回值为`value`。但实际上这是严重错误的，正确解释需要从调用栈思考，当你内部调用`(k value)`时，CPU会用它覆盖掉整个调用栈，看起来好像从里面的世界开了个洞，直接跳到最外层。   
更细节一点，在当前调用栈中一旦出现了`call/cc`，那么后续行为会被绑定到函数`k`，然后在调用栈之前添加`<body>`的调用，一旦`<body>`中有任何对`k`的传参过程，即`(k value)`，执行完毕后会忽略调用栈后的代码直接返回到下一个调用栈。比如这样（我们用`<=CPU`表示当前指令）：   
```Scheme
;;这个表达式等价于(+ 1 (* 5 3))，即16
(+ 1 (* (call/cc (lambda (k) 
                     k      ;(call/cc (lambda (k) k 5)返回值是
					 5)))   ;这里的最后一个语句 5
        3))

;;我们看看调用栈一步步会如何改变,一开始是
(+ 1 ???)           <=CPU
;;然后是
y = (* ??? 3)
(+ 1 y)
;;然后是
z = (call/cc ???)   <=CPU
y = (* z 3)
(+ 1 y)
;;现在来到call/cc了，它首先要得到栈空间后续动作然后绑定给k
;;也就是说k变成了一个函数(lambda (x) (+ 1 (* x 3)))
;;这个函数存在堆中，准备调用
;;然后调用栈变成
p = k               <=CPU
p = 5
z = p
y = (* z 3)
(+ 1 y)
;;调用栈开始缩小
z = 5               <=CPU
y = (* z 3)
(+ 1 y)
;;很自然的，继续缩小
y = (* 5 3) = 15    <=CPU
(+ 1 y)
;;最后变成
(+ 1 15) = 16

;;现在我们看一下表达式中存在(k value)调用的情况
(+ 1 (* (call/cc (lambda (x) 
                     (x 2)  ;不同之处在于这里
					 5)))   
        3))

;;调用栈和前面一开始是差不多的
z = (call/cc ???)    <=CPU
y = (* z 3)
(+ 1 y)
;;这里的call/cc同样在堆里生成了延续函数
;;即 k = (lambda (x) (+ 1 (* x 3)))
;;当执行到(k 2)时，整个后续的调用栈会被全部清空
p = (k 2)            <=CPU
p = 5                ;后续的都没用了
z = p
y = (* z 3)
(+ 1 y)
;;直接会变换为
(k 2)                <=CPU  ;当然(k 2)会展开新调用栈
;;即
(+ 1 (* 2 3)) ;这里不用再重复展开了，显然是等于7
```

网上很多教程的错误在于，把第二个表达式错误的理解为先`(k 2)`返回`2`，然后再执行外面的`(+ 1 (* (k 2) 3))`。而真实情况是`(k 2)`的值根本就不是`2`，而是`7`。  
具体流程是先从堆里面取`k`函数，求值后直接返回`7`，然后打开一个虫洞，直接穿越到整个表达式的最外层之外。所谓的虫洞穿越，其实就是把外层的`(+ 1 (* ### 3))`等一系列调用栈强行清空，并没有多神秘。  

**注意：一定要记住，一旦某个表达式内部`call/cc`语句中存在其延续函数`k`的调用，则会直接返回这个调用的值，然后就没有然后了。**  

好吧，现在看起来已经足够烧脑了，不过我们可以继续用简单的例子加深理解：  

```Scheme
(define x '(1 2 3 4 5))   ;定义x是链表[1,2,3,4,5]
(define func-k #f)        ;在外面定义func-k来存储延续，一开始初始化为False

(if (call/cc              ;call/cc外面的东西是(if A '() (cdr x)) 
      (lambda (k)         ;k会绑定到(lambda (x) (if x '() (cdr x)))
	    (set! func-k k)   ;上面的这个延续函数会赋值给func-k
		(null? x)))       ;没有看到(k value)的调用，自然会执行这一行，
	'()                   ;
	(cdr x))              ;显然x不为空，所以得到'(2 3 4 5)
	
;如果你执行一次以上的代码，那么它做了两件事
;1. 执行原函数，返回'(2 3 4 5)，即if语句的False分支
;2. func-k被绑定为 (lambda (x) (if x '() (cdr x)))这一延续函数

;检验一下
(func-k #t)
;此时应该返回'()
(func-k #f)
;此时应该返回'(2 3 4 5)
```

复杂一点的`call/cc`就不那么好理解了，比如以下的经典表达式：  
```Scheme
(let ([x (call/cc (lambda (k) k))])
  (x (lambda (ignore) "hi")))

;执行这条语句的输出是"hi"，但是为什么会这样呢？我们来一步步分析，首先不妨揭开let得到
((call/cc (lambda (k) k))
 (lambda (ignore) "hi"))
 
;如果用A表示当前位置，那么k的行为是(A (lambda (ignore) "hi"))
;k的值是函数(lambda (x) (x (lambda (ignore) "hi")))
;由于call/cc包裹的位置中没有出现(k value)这样的调用，所以不会出现中断
((lambda (x) 
  (x (lambda (ignore) "hi")))
 (lambda (ignore) "hi"))

;求值时把x替换为后面的参数（它是一个函数），得到
( (lambda (ignore) "hi")
  (lambda (ignore) "hi") )
  
;如果我们假设(lambda (ignore) "hi")这个匿名函数是f
;那么(let ([x (call/cc (lambda (k) k))])
;      (x f))
;会得到(f f)

;由于f是一个对于任意参数都会返回"hi"的函数，所以(f f)的返回值是"hi"
```

或许你怀疑上面推导的正确性，那不妨按照优先进行`call/cc`展开的顺序再推导一次：  
```Scheme
(let ([x (call/cc (lambda (k) k))])
  (x (lambda (ignore) "hi")))

;首先展开call/cc此时k的值为
(lambda (p) 
  (let ([x p])
    (x (lambda (ignore) "hi"))))
	
;此时let语句会把x绑定为上面的函数，那么表达式的值为
( (lambda (p)
    (let ([x p])
	  (x (lambda (ignore) "hi"))))  ;上面的这部分是x
  (lambda (ignore) "hi"))
  
;我们把上面的x化简得到
( (lambda (p)
    (p (lambda (ignore) "hi")))
  (lambda (ignore) "hi"))
  
;很明显，替换p进行求值后得到
((lambda (ignore) "hi")
 (lambda (ignore) "hi"))
 
;结论不变
```

到现在为止，我们知道了什么叫做continuation，并且知道了`call/cc`语句的用法。不过目前所有的`(call/cc (lambda (k) <body>))`调用中，`<body>`里面少有`(k value)`的语句，其实利用它会产生很多烧脑且神奇的效应。  

**call/cc定理1：(let ([x (call/cc (lambda (k) k))]) (x f)) 等价于(f f)。**  
**call/cc定理2：((call/cc (lambda (k) k)) f)) 等价于(f f)。**  

##3. call/cc的应用场景
### 3.1 非本地退出(nonlocal exits)
Scheme语言里面没有`break`或`return`语句，如果你需要提前返回的话，就可以借助`call/cc`。首先看一个超级简单的例子：
```Scheme
(define (f return)  ;定义f是一个单参函数，参数return
  (return 2)        ;首先执行(return 2)，
  3)                ;显然，return也必须是一个单参函数
                    ;最后返回3

;这个调用的返回值是3
(f (lambda (x) x))  ;相当于(begin 2 3)，
                    ;最后一条语句3的值是返回值
					
;下面的调用方式返回值是2					
(call/cc f)

;由于f是单参函数，所以上面的语句等价于
(call/cc (lambda (k)
           (k 2)    ;此处碰到了(k value)调用，跳出外面的世界
		   3))      ;这条语句不会被执行

;call/cc之外什么都没有,也就是说其延续是啥也不干
;即 k会被绑定为 (lambda (x) x)
;这样(k 2)自然是 2
;注意这和第2节的(k 2)是完全不同的东西
```


**注意：这个东西别问百度或者任何AI，几乎80%以上的回答都是错误的。** 

比如在百度搜索“`call/cc`非本地退出”的结果如下：  
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 1px 2px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/lisp_call_cc1.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">图1 百度的人工智障不知道从哪里抄出来的错误代码</div>
</center>

这个代码简直错得离谱，因为`exit`是一个单参函数，图1中红框的语句根本执行不了。正确的代码如下：  
```Scheme
(call/cc 
  (lambda (exit)
    (display "Hello, ")
    (exit 0)
    (display "World!")))
```

再考虑一个复杂一点的例子，计算链表所有元素乘积的程序如下：  
```Scheme
;常规的非尾递归的程序很简单，但这个程序可能会爆栈，
;更讨厌的是，当有一个元素为0时，本来就可以返回0了，
;但它仍然会老老实实的遍历所有元素
(define (product-1 lst)
  (cond [(null? lst) 1]
	[else (* (car lst) (product-1 (cdr lst)))]))

(product-1 '(1 2 3)) ;输出6
(product-1 '(1 0 3)) ;输出0

;展开后，计算过程是
(* 1 (product-1 '(0 3)))
(* 1 (* 0 (product-1 '(3))))
(* 1 (* 0 (* 3 (prduct-1 '())))))
(* 1 (* 0 (* 3 1)))
(* 1 (* 0 3))
(* 1 0)
0
```

利用`call/cc`的跳出机制可以把程序改成下面的样子：（这里用到了带`name`的`let`语句，它通常用于递归）
```Scheme
(define (product-2 lst)
  (call/cc
   (lambda (return)
     (let iter ((rest lst))
       (cond [(null? rest) 1]
	     [(zero? (car rest)) (return 0)] ;只要rest的首元素为0,程序会执行延续函数并退出
	     [else (* (car rest) (iter (cdr rest)))])))))
;参考前面简单的例子，当rest的第一个元素为0时，程序不会执行后面的递归，而自动退出
;展开后如下
(product-2 '(1 0 3))
(* 1 (iter '(0 3))) ;在下一个步骤中rest的第一个元素是0,程序会直接执行return 0的代码
((lambda (return) return) 0)
0
```

当然，这个程序不是尾递归，它可能爆栈。但写出它的尾递归（循环）版本也不难，这里不多作展开，一切的重点在于我们用`call/cc`实现了return语句。  

### 3.2 回溯(backtracing)
参考下面的代码：  
```Scheme
; fail初始为一个只返回错误信息的无参函数
(define fail                  
   (lambda () 'no-choice))
   
; 这是一个克里化的函数，它可以接受不确定个数的参数
; (choose 1 2 3)时，ls为'(1 2 3)
; (choose)时，ls为'()
(define (choose . ls)
  (if (null? ls)
    (fail)
	(let [(fail0 fail)]
	  (call/cc (lambda (k)
	             (set! fail (lambda ()
				               (set! fail fail0)
							   (k (apply choose (cdr ls)))))
	             (k (car ls)))))))   ;跳出并返回ls的第一个元素

;上面(apply choose (cdr ls))，如果ls是'(1 2 3)
;则该表达式等价于(choose 2 3)

(choose 1 2 3) ;会返回1
(choose)       ;会返回2
(choose)       ;会返回3
(choose)       ;会返回'no-choice
```
这里的`choose`函数实际上实现了一个对参数列表的取首元素的程序，每执行一次，它会生成一个新的`fail`函数调用，`choose`每次执行都改变了自己的行为。  
执行`(choose 1 2 3)`时，返回值是最后一行的`(car '(1 2 3))`，同时生成了一个新的函数`fail`，它的行为是：  
1. 首先把自己赋值为上一个函数状态，这里是`(lambda () 'no-choice)`;  
2. 返回`(choose 2 3)`。  

利用这个特性，我们可以进行非确定性计算，比如找出1到5之间满足勾股定理的三元数：  
```Scheme

(let
  ([sq (lambda (x) (* x x))]  ;平方函数
   [a  (choose 1 2 3 4 5)]    ;a、b、c都是choose的回溯调用
   [b  (choose 1 2 3 4 5)]    ;
   [c  (choose 1 2 3 4 5)])
  (if (equal? (+ (sq a)
		 (sq b))
	      (sq c))             ;判断是否满足勾股定律
      (list a b c)            ;满足就返回a b c这三个数
      (choose)))              ;不满就回溯调用(choose)

;这个程序会返回'(4 3 5)
```
这里的神奇之处在于你无需在主函数中分别对`a`、`b`或`c`分别指定递归调用，在`let`一开始执行了三个`choose`函数之时，它们各自被绑定到一个自动列举的序列函数中，每一次调用(choose)，其中某个勾股数就会改变。  
具体过程不再推导中，因为这个例子远没有协程和阴阳谜题复杂，有兴趣者不妨仿照后文的方式推导一遍。  


### 3.3 协程(coroutines)
协程是一般化的子函数。一个协程可以在某个执行点挂起并在之后从挂起点恢复。与子函数不同的是，协程不需要在它返回前完成整个执行过程。  

```Scheme
(define (one go1)
  (display 1)
  (set! go1 (call/cc go1))   ;第一个挂起点
  (display 2)
  (set! go1 (call/cc go1))   ;第二个挂起点
  (display 3)
  (set! go1 (call/cc go1)))  ;....

(define (two go2)
  (display #\a)
  (set! go2 (call/cc go2))
  (display #\b)
  (set! go2 (call/cc go2))
  (display #\c)
  (set! go2 (call/cc go2)))
```

理解程序为什么在挂起点处暂停的关键在于再次思考`call\cc`会在什么时候“跳出”当前程序。看完本节的推导你会发现所谓挂起只是一种错觉，CPU中发生的事实只不过是不断的清空和生成调用栈而已。  
前文已经提到出现`(k value)`的调用时程序会进行虫洞穿越。但为了顺利推导上面的代码行为，还需要理解以下几种进阶跳出机制：  
```Scheme
;;经典跳出，屏幕输出12
(begin (display 1)
       (call/cc (lambda (x)
	               (x 0)            ;不会输出"-"
				   (display "-")))
       (display 2))
	   
;;调用栈是
(display 1)       <=CPU
(call/cc ...)
(display 2)
;;先输出1，调用栈变为
(call/cc ...)     <=CPU
(display 2)
;;遇到call/cc了，x函数开始绑定为延续
;;即 x = (lambda (p) p (display 2))
;;当发生(x 0)时，清空调用栈，直接返回以下代码
((lambda (p) p (display 2)) 0)  ;中间的p没有任何作用

;;用call/cc跳出，屏幕输出12，这个理解起来就难一些
(begin (display 1)
       (call/cc (lambda (x)
	               (call/cc x)
				   (display "-")))  ;不会输出"-"
	   (display 2))
;;输出1后，在堆里生成延续并绑定x
;;x = (lambda (p) p (display 2))
;;调用栈变为
a = (call/cc x)   <=CPU
(call/cc a)
(display 2)
;这个操作会把p绑定到一个新的延续，形式和x一样
;;p = (lambda (k) k (display 2))
;;调用栈变为
y = (call/cc x)    <=CPU ;后面的代码都会被跳出
y = (display "-")
(call/cc y)
(display 2)
;;注意(call/cc x)在干什么，它首先把x的参数绑定为新的延续
;;也就是说p = (lamba (y) y (display "-") (call/cc y) (display "2"))
;;然后调用x程序, 这相当于在原栈中调用了某个(x f)
;;调用栈瞬间变为
(x f)
;;很自然d的会得到
((lambda (p) p (display 2)) f)
;;然后调用栈变为
f            ;这里直接返回f的lambda表达式,没啥用,会被后面覆盖
(display 2)  ;注意这条语句是x函数中的

;;同理，下面的代码也会在屏幕输出12
(begin (display 1)
       (call/cc (lambda (x)
	               (set! x (call/cc x)) ;同样不会输出"-"
				   (display "-")))      ;而且set!语句不会执行  
       (display 2))

;;不会跳出，屏幕输出1-2
(begin (display 1)
       (call/cc (lambda (x)
	               x
				   (display "-")))
       (display 2)）				   
```

有了以上基础，再来讨论调用`(one two)`会发生什么，详细推导过程如下：  
```Scheme
;;程序的调用栈是
(display 1)        <=CPU  
(set! ...
(display 2)
(set! ...)
(display 3)
(set! ...)

;;输出1之后，调用栈变换为
a = (call/cc two)  <=CPU
(set! two a)
(display 2)
(set! ...)
(display 3)
(set! ...)

;;不妨设two的参数为k0,它是一个延续
;;k0 = (lambda (x) (set! two x)
;;                 (display 2)
;;			  	   (set! ...)
;;				   (display 3)
;;				   (set! ...)))
;;调用栈变为
(display #\a)      <=CPU   ;这里输出a,屏幕上变成1a
p = (call/cc k0)           ;很明显这里调用了延续k0,会跳出
p = (set! k0 a)
p = (display #\b)
p = (set! ...)
p = (display #\c)  
p = (set! ...)             ;这之前是two函数的世界
p = (call/cc p)            ;这之后是one函数的世界
(set! two p)              
(display 2)
(set! ...)
(display 3)  

;;我们看看(call/cc k0)会发生什么?
;;它等价于(call/cc (lambda (x)
;;                    (set! two x) (display 2) (set! ...) ...)
;;在未跳出之前就必须产生一个新的延续并绑定到x0
;;x0 = (lambda (p) (set! k0 p) (display #\b) ... 
;;然后调用k0函数并清空后面的调用栈直接跳出
;;生成的新调用栈为
(set! two x0)      <=CPU   ;现在two函数变为x0
(display 2)                ;执行到这里会输出2,屏幕为1a2
(set! ...)
(display 3)
(set! ...)

;;当屏幕输出2之后,调用栈变为
a = (call/cc two)   <=CPU   ;故事又重演了
(set! two a)                
(display 3)
(set! ...)

;;首先对(call/cc two)产生新的延续
;;k1 = (lambda (x) (set! two x) 
;;                 (display 3)
;;                 (set! ...))
;;(call/cc two)变成
;;(call/cc (lambda (p) (set! k0 p) (display #\b) ...)
;;然后把(call/cc two)展开并产生新调用栈
p = (set! k0 k1)                ;这里没啥用可以忽略
p = (display #\b)     <=CPU     ;输出了b,屏幕为1a2b
p = (call/cc k1)                ;很明显,这里会跳出
p = (set! k1 p)
p = (display #\c)               ;新世界的two函数结束
p = (call/cc p)
(set! two b)
(display 3)
(set! ...)

;;当屏幕输出b时,又需要产生一个新的延续并绑定到x1
;;x1 = (lambda (p) (set! k1 p) (display #\c)
;;                             (set! ...)
;;                             ...)
;;此时调用栈会一下子消灭跳出后的东西,直接变为
(call/cc k1)  

;;也就是
(set! two x1)       <=CPU ;这里two又变成x1函数
(display 3)               ;这里输出3,屏幕为1a2b3
(set! ...)

;;输出3之后,调用栈变成
p = (call/cc two)
(set! two p)

;;同样会首先生成一个延续k2
;;k2 = (lambda (x) (set! two x))
;;再同样用x1替换two展开得到新调用栈
p = (set! k1 k2)    <=CPU ;这里没啥用可以忽略
p = (display #\c)         ;这里输出c,屏幕为1a2b3c
p = (call/cc k2)          ;这里仍然会跳出
p = (set! ...)
...
(set! two p)

;当调用到(call/cc k2)时,同样又会生成一个新的延续
;x2 = (lambda (p) (set! k1 p) ...)
;但这个延续已经没有啥意义了,此时整个调用栈变成
(call/cc (lambda (x) (set! two x)))
;也就是
(set! two x2)      <=CPU
;;然后再没有任何代码会生成新的调用栈,程序结束
```

同理可知，如果我们调用`(two one)`，那输出一定会变成`a1b2c3`，推导方式几乎一摸一样。


**注意：`call/cc`由于拥有了取出当前位置延续和跳出该位置这两个功能，因此高级应用还有很多，比如多线程（协程的进化版），结合scheme宏生成`while`、`loop`，`break`等新控制结构，等等。**  

##4. CPS
CPS是continuation-passing style的简写，可翻译为“延续传递风格”。使用 CPS 编写的函数会带有一个额外参数，它本身也是一个单参函数。当 CPS 函数计算出它的返回值后，它通过调用 continuation 函数来进行“返回”。比如下面的CPS斐波拉契函数：  

```Scheme
; 根据数学定义的非CPS递归版本，它不是尾递归
(define fib
   (lambda (n)
      (cond 
         ((< n 0) #f)
         ((= n 0) 0)
         ((= n 1) 1)
         (else (+ (fib (- n 1)) 
                  (fib (- n 2)))))))
				  
; CPS风格的递归版本，它是尾递归，不会爆栈，注意调用时k是一个call/cc
(define fib-cps
  (lambda (n k)
    (cond
     ((< n 0) (k #f))
     ((= n 0) (k 0))
     ((= n 1) (k 1))
     (else
      (fib-cps (- n 1)
           (lambda (n1)
              (fib-cps (- n 2) (lambda (n2) (k (+ n1 n2))))))))))

```

这两个函数的调用方式并不相同，`(fib 7)`会返回`13`，而等价的CPS函数调用方式为`(fib-cps 7 (lambda (x) x))`。当然，这里我们没有用到`call/cc`函数，但实际上现代Scheme解释器对于`fib`函数会自动采用`call\cc`得到它的CPS变换，然后再执行。自动CPS变换是一个更烧脑的问题，如果你想挑战一下自己的智商可以搜索“王垠的40行代码”。  

##5. call/cc的call/cc
前面我们已经说了，`call/cc`本身也是一个单参函数，它通常的调用方式为`(call/cc (lambda (k) <body>))`。那么问题来了，`(call/cc call/cc)`会得到什么？  
不妨用1和2将两个函数进行区分，推导如下：  

```Scheme
;考虑func是某个函数，执行一下表达式
((call/cc1 call/cc2) func)

;不妨设call/cc2的参数是k
;也就是说，k被绑定为(A func)这个行为
;k = (lambda (x) func)
;原表达式变为
((call/cc1 (lambda (x) func)) func)

;这里x被绑定为当前位置的延续，且没有看到(x value)的调用
;(call/cc1 (lambda (x) func))的值为func
;所以整个表达式的值变为
(func func)

;这也就是说(call/cc call/cc)会返回一个函数，它接受一个单参，
;此单参必须为一个函数，表达式为
(lambda (f) (f f))

```

**call\cc定理3：`((call/cc call/cc) f)`等价于`(f f)`。**  

到这里我们已经看到了三个语句，他们都能把一个单参函数重复两次，不妨回想一下[《Y组合子》](lisp_y_combinator.md)。很显然，使用`(call/cc)`会得到更简单的Y组合子写法：  
```Scheme
;原始写法
(define (Y f)
  ((lambda (x) (x x))
   (lambda (g)
     (f (lambda (y)
	 ((g g) y))))))
	 
;call/cc写法
(define (Y f)
  ((lambda (u)
     (u (lambda (x)
	      (lambda (n) ((f (u x))) n))))
   (call/cc call/cc)))
```

##6. 阴阳谜题
这货是Lisp邪教徒老拿来说事的东西，代码如下：  
```Scheme
(let* 
  ([yin  ((lambda (x) (display #\@) x)
	      (call/cc (lambda (k) k)))]
   [yang ((lambda (x) (display #\*) x)
	      (call/cc (lambda (k) k)))])
  (yin yang))
```
它会得到一个死循环，输出如下：  
`@*@**@***@****@*****@******@*******@********@*********`往后省略。  

但网上几乎所有的解释都不那么清爽，有些干脆是错的，或者是按照输出来凑。这里有必要手动推导一遍，在推导之前，请一定注意Scheme语言求值的规则，这个题目如果你把`let*`语句里的`yang`和`yin`顺序颠倒，那么输出就会不同。此代码并不满足代换模型，`display`语句带来了副作用。  
另外，解释器的求值是一步步进行的，由于`let*`的存在，它首先会对第一条语句的表达式求值，然后再绑定到`yin`。随即再对`yang`进行同样的操作，每次一旦求值结束（表达式变为一个延续函数或一个单值），则立刻进行绑定。而每次在`let*`中执行求值就会在屏幕输出字符。好，现在开始烧脑操作：  
(1) 首先会依次对`yin`和`yang`两个表达式求值，求值过程中会在屏幕中打印出`@*`，同时得到两个延续函数，为了对延续函数表示区分，我们在`k`后面加上编号，以下为代码推导。  

```Scheme
;;对yin进行绑定，此时屏幕上是@
(let* 
  ([yin k0]       ;先输出@，然后返回call/cc处的延续函数k0
   [yang ((lambda (x) (display #\*) x)
          (call/cc (lambda (k1 k1))))])    
  (yin yang))     

;;用####表示调用时的参数,k0函数的行为(函数体)是
(let* 
  ([yin  ((lambda (x) (display #\@) x)
	      ####)]      ;####表示调用k0时的参数
   [yang ((lambda (x) (display #\*) x)
	      (call/cc (lambda (k) k)))])
  (yin yang))
  
;;再对yang进行绑定，此时屏幕上是@*
(let* ([yin  k0]
       [yang k1])
  (yin yang))      ;这东西就是(k0 k1)
  
;;同样在碰到call/cc之前会产生新的延续函数k1,其行为是
(let* 
  ([yin k0]  ;注意，之前yin求值时call/cc已经没有了
   [yang ((lambda (x) (display #\*) x)
          ####)]) 
  (yin yang))
  
;;这样在第1步时做了三件事，
;; 屏幕输出为@*，
;; 生成了k0, k1两个函数
;; 原来的程序变成了(k0 k1)
```

(2) 现在要对`(k0 k1)`进行求值，由于求值后和原来的表达式不同，中途产生的延续也会不同，推导如下。  

```Scheme
;;对(k0 k1)求值，那就是把k1代换到前面的k0代码块里，如下
(let* 
  ([yin  ((lambda (x) (display #\@) x)
	      k1)]
   [yang ((lambda (x) (display #\*) x)
	      (call/cc (lambda (k2) k2)))]) ;表达式变了，延续也会变，用k2
  (yin yang))
  
;;首先对yin求值，屏幕输出@，然后得到
(let* 
  ([yin  k1]
   [yang ((lambda (x) (display #\*) x)
	      (call/cc (lambda (k2) k2)))])
  (yin yang))

;;再对yang求值，屏幕输出*，然后得到
(let* ([yin  k1]
       [yang k2])
  (yin yang))       ;即(k1 k2)
  
;;同时，新的k2延续函数的行为是
(let* 
  ([yin  k1]
   [yang ((lambda (x) (display #\*) x)
	      ####)])
  (yin yang))
  
;;现在对(k1 k2)求值，会得到
(let* 
  ([yin k0]  
   [yang ((lambda (x) (display #\*) x)
          k2)]) 
  (yin yang))
  
;;当然这个表达式一定会先对yang求值，屏幕会输出*
;;同时表达式变为
(let* ([yin  k0]
       [yang k2])
  (yin yang))       ;即(k0 k2)

;;显然，以上做了四件事
;;在屏幕上输出@*
;;调用(k1 k2)
;;在屏幕上输出*
;;原来函数变成了(k0 k2)
```

(3) 现在对`(k0 k2)`继续求值，过程与上面的过程类似，同样要注意，`yang`的延续会继续改变，推导如下。  

```Scheme
;;对(k0 k2)求值，那就是把k2代换到前面的k0代码块里
(let* 
  ([yin  ((lambda (x) (display #\@) x)
	      k2)]
   [yang ((lambda (x) (display #\*) x)
	      (call/cc (lambda (k3) k3)))]) ;表达式变了，延续也会变，用k3
  (yin yang))
  
;;首先对yin求值，屏幕输出@，然后得到
(let* 
  ([yin  k2]
   [yang ((lambda (x) (display #\*) x)
	      (call/cc (lambda (k3) k3)))])
  (yin yang))

;;再对yang求值，屏幕输出*，然后得到
(let* ([yin  k2]
       [yang k3])
  (yin yang))       ;即(k2 k3)
  
;;同时，新的k3延续函数的行为是
(let* 
  ([yin  k2]
   [yang ((lambda (x) (display #\*) x)
	      ####)])
  (yin yang))
  
;;(k2 k3)会变换为
(let* 
  ([yin k1]  ;注意，这里的k2函数在上一个代码块已经给出
   [yang ((lambda (x) (display #\*) x)
          k3)]) 
  (yin yang))
  
;;对yang进行绑定，屏幕会输出*，然后表达式变换为
(let* ([yin  k1]
       [yang k3])
  (yin yang))       ;即(k1 k3)
  
;;对(k1 k3)求值，按照上个代码块的推导应该是
(let* 
  ([yin k0]  
   [yang ((lambda (x) (display #\*) x)
          k3)]) 
  (yin yang))
  
;;对yang进行绑定，会继续输出*，表达式变换为
(let* ([yin  k0]
       [yang k3])
  (yin yang))       ;即(k0 k3)
  
;;所以上面的代码实际上做了如下的事情：
;;输出@*，(k0 k2)变换为(k2 k3)
;;输出*，变换为(k1 k3)
;;输出*，变换为(k0 k3)

```

(4) 对`(k0 k3)`求值与上面基本一样，它会生成一个新的`k4`延续函数，然后经过一系列转换变成`(k0 k4)`的调用。   

以上过程会无穷无尽的执行下去，总体过程可以用以下的简记法表示（k0表示产生k0这个函数）：  
`k0->@->k1->*->(k0 k1)->@->k2->*->(k1 k2)->*->(k0 k2)->@->k3->*->(k2 k3)->*->(k1 k3)->*->(k0 k3)->@->k4->......`

这里尤其要注意的是求值顺序，其实用调用栈的方式去推导更为简单。无论如何，这个问题现在看起来不是那么难了。  

**注意：纸上得来终觉浅，推导完了就洗脸。**  


##7. 吐槽
`call\cc`虽然提供了各种奇技淫巧，但如果你耐心的看到这里，应该明白这东西有多么可怕。  
它在Scheme语言中是一个Low level的API函数，也就是说，能不用尽量不要用。  
这东西说好听点是实现了程序宇宙的虫洞，说难听点就是到处钻眼儿。做一下思维练习还好，要是用来构建一般程序，估计不但会秃头，甚至可能丧命。  
貌似C++也有`call\cc`，我实在难以猜测其语法应该怎么设计，Lisp版本的call/cc已经足够造成心智负担，C++？还是C艹吧。  




