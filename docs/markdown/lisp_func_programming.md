大体来说，编程范式可能有三种：函数式（Functional）、命令式（Imperative）、面向对象（OOP）。  
还有所谓函数式编程语言、命令式编程语言、面向对象编程语言等说法，意为某种语言是某种编程范式的，但这样的论断并不成立，很多语言是支持多范式的，比如Java8引入了`lambda`从而支持部分函数式编程范式；Scheme语言里引入了赋值语句，完全可以写出纯命令式风格的程序，且无需像C++和Java那样增加语法糖即可进行面向对象编程。  
编程领域很多术语太不严谨，完全没有数学里“$N-\epsilon$定义”那样优美的东西。  

**复习一下：设$X_n$为一无穷数列，若${\exists}$常数$a$，对于${\forall} \epsilon > 0$，${\exists}$正整数$N$，使得当$n>N$时，均有$|X_n-a|<\epsilon$成立，则称$a$为数列$X_n$的极限。**    

以下粗略的给出三种编程范式的主要特征：  
1. **函数式编程：**  函数是第一公民，你可以在程序的任何地方定义函数，并把它作为参数传递。这里的函数是“无状态的”，给定一个输入，无论何时执行它，其输出永远相同。“纯函数式编程语言”是没有赋值语句的（如haskell语言），不允许变量或对象有“状态”，函数的输出只和输入有关，和什么时间执行它无关。（生成随机数的`rand()`不是函数，而是一个程序）。  
2. **命令式编程：** 程序由一串有序的子程序（procedure）组成，每个子程序的执行顺序有严格的先后之分。通过赋值语句引入了“状态”，交换赋值语句的顺序可能会得到不同的输出，这个特性叫副作用（side effect）。  
3. **OOP编程：** 程序基于一系列类型（术语是`type`，C++和Java里叫`class`）构建，类型之间有继承、包含等关系。类型把数据（也叫变量）和子程序（也叫方法）绑定到一起，用重载实现多态。站在OOP的角度，会把命令式编程叫做“过程式编程”。举个不恰当的例子（这个梗来自于Steve Yegge的《程序员的呐喊》）：命令式编程是动词世界，而OOP是名词世界。倒垃圾这一函数在C语言里是`clean(trash)`，而在Java里是`trash.clean()`。把大象放进冰箱在C语言里是`put(elephant, refrigerator)`，而在神圣的原教旨Java教堂里需开会讨论到底应该搞成`refigerator.put(elephant)`还是`elephant.putInto(refrigerator)`，会议不欢而散并各自开发了两套不同的类库。不得不说还是大主教英明神武，在十年后拍板，必须写成`putter.addObject(elephant)`,然后`putter.addDestination(refrigerator)`，最终`putter.run()`。又过了一年，教廷颁布了一部名为《设计模式》的法典，规定一切不这样写的都是异端。所以把大象装进冰箱真的需要三步。：）  

**小贴士：也有人认为OOP只是在命令式范式上增加了“把过程和数据绑定到一起”的语法糖，从有无引入副作用这个角度来分析，OOP也是命令式的。当然，你完全可以使用OOP模拟出无副作用的编程风格，编程界的术语体系并不严谨。**

近年来，函数式编程越炒越热，好像不“函数式”就代表着落后。但其实函数式编程始于1960年，MIT跑在IBM 704的Lisp语言的第一版就非常的“函数式”。它只比命令式语言的祖宗Fortran II晚了两年。  现在网上热炒的函数式编程概念和一些代码范例仍然是1960s~1970s的东西，如果一定要说其“先进”，倒不如说是“不再犯蠢”。  
虽然我并非程序语言专家，但经常听到“我把这个3000行代码算法里的重复逻辑提取为15个函数，主程序现在只有20行，逻辑清晰，便于维护，函数式编程果然有些道理”，驳也不是，忍也不是，实在让人哭笑不得。这些初尝编程快乐的孩子们呵，不是有函数就是“函数式”哦！    
本文试图用一个实例带你简单体验函数式编程的一点皮毛，一来避免大家“鸡同鸭讲”，说得口干舌燥；二来悟道全靠缘分，理解正确与否并不影响一日三餐，苟延残喘。  
该例子来自《计算机程序的构造和解释（SICP）》一书，代码和原书不完全相同。友情提示：已做完此神书习题者将浪费生命约20分钟，勿谓言之不预也。
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 1px 2px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/lisp_func_programming_1.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
               display: block;
               color: #999;
               padding: 2px;">
        图1《SICP》的封面 </div>
</center>

**重要的话说三遍：函数式编程的“函数”指的是“纯函数”！函数式编程的“函数”指的是“纯函数”！函数式编程的“函数”指的是“纯函数”！**

## 1. 序对类型
在Lisp语言家族，存在一个基本的数据结构（也是一个类型）——序对（`pair`）。它有严格数学定义：“对${\forall}$的两个事物$a$和$b$，若${\exists}$三个映射分别满足$p=cons(a,b)$、$a=car(p)$和$b=cdr(p)$，则称$p$为一个序对”。  

*注：cons是构造（construct）这个动词的简写，car和cdr则来自Lisp最初在IBM 704机器上的实现。这种机器提供一个接口，使用户能够访问寄存器(register)地址(address)和它的"减量"(decrement)。car表示“Contents of Address part of Register”，cdr（读作“could-er”）表示"Contents of Decrement part of Register"，它们分别是寄存器的内容和寄存器减量的内容。*  

序对之于Lisp，正如碱基之于生物，01之于电子计算机，阴阳之于算命先生，板砖之于房子......若非要把程序的数据类型分为`float`、`char`、`int`、`bool`等“基本类型”和`struct`、`class`、`vector`、`list`、`map`等“高阶类型”的话，那么`pair`就是高阶类型里最基本的类型。下图演示了`pair`的几种组合，其中之妙尽在三缪三菩提处。
<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 1px 2px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="/img/lisp_func_programming_2.png">
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
               display: block;
               color: #999;
               padding: 2px;">
        图2 几种序对组合成的数据结构，最后一个细思极恐 </div>
</center>

如何用编程语言来实现`pair`呢？显然只要实现了`cons`、`car`和`cdr`即可。这里我们用Scheme语言（一种语法最严谨的Lisp方言）来实现它。  

**约定：后文的“数据”这一术语可暂时指代某个作用域的变量，而函数和程序的意思都是“纯函数”。**  

## 2. 数据导向的序对实现
Scheme语言提供了原生的（primitive）`cons`、`car`和`cdr`函数。我们暂且假设Scheme没有它们，而只提供了数组（`vector`）这个类型。  
按最自然的思维方式，`cons`函数返回2个泛型元素的数组，`car`函数返回数组的第1个元素，`cdr`返回第2个。代码如下：
```lisp
;;用数据型思维来实现的pair类型
(define cons                    ;定义 cons  为
  (lambda (a b)                 ;  一个2参数a,b的函数， 函数体（body）为
    (vector a b)))              ;     构造一个数组{a,b}并返回它

(define car                     ;定义 car   为
  (lambda (p)                   ;  一个单参数p的函数， 函数体为
    (vector-ref p 0)))          ;     返回对数组p取下标为0的元素      

(define cdr                     ;定义 cdr   为
  (lambda (p)                   ;  一个单参数p的函数， 函数体为
    (vector-ref p 1)))          ;     返回对数组p取下标为1的元素

;;测试代码
(define a-pair (cons 1 2))      ;定义 a-pair 为 (cons 1 2) 生成的pair
(display a-pair)                ;打印a-pair的值，应该是#(1 2)，数组简记符
(newline)                       ;打印换行符
(display (car a-pair))          ;打印(car a-pair)的值，应为1
(newline)                       ; ......
(display (cdr a-pair))          ;打印(cdr a-pair)的值，应该2      
(newline)                       ; ......
```
我们暂不考虑参数`p`万一不是数组或元素个数非2的情况，此时Scheme会在运行时抛出异常。执行以上程序，屏幕输出应为：  
```
#(1 2)
1
2
```
Lisp原生`cons`函数构造的`pair`的值一般打印为`(1 . 2)`（可理解为Java的`pair.ToString()`），而我们重新实现的`cons`函数返回值打印为`#(1 2)`。也就是说，从数据的组织形式看，我们实现的`cons`和原生`cons`是不同的东西，但二者在语义上却是完全相同的。我们的`cons`、`car`和`cdr`函数完全满足`pair`的严谨数学定义。新的`pair`类型和原生类型一样可以组合出图2那样的链表、二叉树等高阶类型。  
在函数式编程范式里，还有一个更可怕的`pair`实现方式，可以完全不考虑任何数据组织形式，只用纯函数来实现`pair`。
## 3. 函数式的序对实现
如果程序猿从图2来理解`pair`，他会很自然的认为每个方块是一个数据，每个小黑点是一个指针。第一反应就用数据导向思维构建代码：`cons`一定是对数据打包(packaging)，而`car`和`cdr`一定是打包的数据解包（unpackaging）。  
但如果程序猿一开始就心中无图，而是直接考察`pair`的数学定义本身，它完全可以得到一个结论，实现`pair`就意味着如何满足三个映射关系。《SICP》书中给出了这种函数式的`pair`实现：
```lisp
;;纯函数实现的pair类型
;;修改了`cons`的定义，取消了原代码内部的一个`define dispatch`语句
;;改为匿名函数
(define cons                     ;定义 cons 为
  (lambda (a b)                  ;  两个参数a，b的函数， body为
    (lambda (m)                  ;    返回一个单参数m的匿名函数， 其body为
      (if (= m 0) a              ;       当m=0时，返回a， 否则
                  b))))          ;              返回b

(define car                      ;定义 car  为
  (lambda (p)                    ;  单参数p的函数， body为
    (p 0)))                      ;    返回(p 0)的值，p是函数，0是p的参数

(define cdr                      ;定义 cdr  为
  (lambda (p)                    ;  单参数p的函数， body为
    (p 1)))                      ;    返回(p 1)的值，p是函数，1是p的参数

;;测试代码
(define a-pair (cons 1 2))      ;定义 a-pair 为 (cons 1 2)
(display a-pair)                ;打印a-pair的值，应该是什么？
(newline)                       
(display (car a-pair))          ;打印(car a-pair)的值，应为1
(newline)
(display (cdr a-pair))          ;打印(cdr a-pair)的值，应该2      
(newline)
```
解释或编译执行以上代码，屏幕输出为：
```
#<procedure>
1
2
```
这里我们发现`(cons 1 2)`并没有对数据打包，而是生成了一个神秘的函数（`#<procedure>`）。新的`cons`、`car`和`cdr`仍满足`pair`类型的严谨数学定义。用它们替换原生`pair`，仍然可以完美支持类型组合。以上的编程范式即函数式编程，此时函数和数据界限是模糊的，函数也是一种数据。    
为便于理解这一“奇技淫巧”，可用代换模型把测试用例展开：
```lisp
;;对(cons 1 2)的代换
((lambda (a b)                 ; 把(cons 1 2)的cons替换为它的定义
   (lambda (m)                 ; 从(lamba...这里开始
     (if (= m 0) a             ;   
                 b))))         ;   ...到此结束是cons的定义
  1  2)                        ; 把前面的匿名函数应用到 a=1, b=2的求值
                               ;=>
(lambda (m)                    ; 对lambda表达式的求值规则是：
  (if (= m 0) 1                ;   其值为匿名函数的body，并在body中
              2))              ;      用1，2替换形参a，b
                               ;
;;对(car (cons 1 2))的代换
(car (lambda (m)               ; 新的car函数的参数是一个匿名函数
       (if (= m 0) 1           ; 这下知道什么叫做“函数式编程”了吧：）
                   2)))        ;
                               ;=>
((lambda (m)                   ; 替换为(p 0)
   (if (= m 0) 1               ; p是(cons 1 2)返回的匿名函数
               2))             ; 
  0)                           ; 0是参数
                               ;=>
(if (= 0 0) 1                  ; 返回body，把body里的m替换为0
            2)                 ; if有短路特性，谓词为真只展开true分支，
			                   ; 反之展开false分支
							   ;=>
1                              ; true分支替换为1
;;对(cdr (cons 1 2))的代换类似，请在纸上写一下。注意边写边思考哦，一个纯函数式语言
;;的解释器大致就是这么工作的（没有赋值语句）。
;;再回想一下图2的最后一个例子，就会知道Lisp语言的语法为什么会设计成这样。
```
通过代换模型展开，神秘的“无种生有”的`pair`类型其实不过是用映射关系代替了数据打包。数据导向的实现关注于数据的组织和存储，而函数式编程关注的是映射关系逻辑。  
现在可能你已经稍微理解了函数式编程的特点：可以在程序的任何地方创建函数，并且把它作为参数传递。“函数”和“数据”本身没有严格的区分。一切有为法，如梦幻泡影，如露亦如电，应作如是观。  

**小贴士：虽然我们只用函数就能实现序对类型，但原生的序对从效率上考虑仍采用了数据导向实现。本节的奇技淫巧看起来好像是无中生有的魔法，于是有人说这反映了老子道生一，一生二，二生三，三生万物的哲学思想。“科学家门经过几千年的努力爬到了山顶，却发现老子和释迦摩尼已等候多时了。”如果有人这么说，请帮我唾他一脸。：）**  

对于此例，函数式编程的威力在于它根据`cons`的两个参数，自动生成了一个新函数，并把这个新函数传参、求值。对`cons`输入不同的参数会生成不同的匿名函数。这其实在运行时中用程序来编写程序，此能力常规语言大多无法做到。
## 4. 一点思考
告诉你一个恐怖故事，函数式的`pair`实现方式还有很多种，比如《SICP》的练习2.4：
```lisp
(define cons                 ; 
  (lambda (a b)              ;(cons 1 2) 会代换为
    (lambda (m) (m a b))))   ;    (lambda (m) (m 1 2))

(define car                  ;(car (cons 1 2)) 
  (lambda (p)                ; => ((lambda (m) (m 1 2)) (lambda (x y) x))
    (p (lambda (x y) x))))   ; => ((lambda (x y) x) 1 2)   
                             ; => 1

(define cdr                  ;(cdr (cons 1 2)) 
  (lambda (p)                ; => ((lambda (m) (m 1 2)) (lambda (x y) y))
    (p (lambda (x y) y))))   ; => ((lambda (x y) y) 1 2)
                             ; => 2
```
有人会说编程语言只是工具，任何成熟编程语言之间都是互相等价的。比如C语言大萨满祭祀宣称：“一切编程语言都能翻译成C”，“没有用C语言写不了的算法”。  
对于`pair`这个支持泛型的数据结构来说，你总可以用`void *`类型来传参，并且把数据类型的`tag`和内存绑定已实现一个数据导向版本。比如是这样：
```c
typedef struct{
    void *a; 
    void *b;
} PAIR;

typedef struct{                     //你可以构建一个 DATA 的实例
    char type[32];                  //   int a=1;      int b="hello";
    void *data;                     //   DATA x, y;
} DATA;                             //   x.type="int"; y.type="string";
                                    //   x.data=a;     y.data=b;
                                    //
PAIR* cons(DATA *a, DATA *b) {      //你可以这样调用 cons
    PAIR* p = malloc(sizeof(PAIR)); //   PAIR *p = cons(&x, &y);
    p->a = a;                       //
    p->b = b;                       //
    return p;                       //
}                                   //
                                    //
DATA* car(PAIR *p) {                //你可以得到p的 car
    return p->a;                    //   DATA *car_p = car(p);
}                                   //   但是返回的是一个带type的数据体指针
                                    //
DATA* cdr(PAIR *p) {                //你可以得到p的 cdr
    return p->b;                    //   DATA *cdr_p = cdr(p)
}                                   //   同样返回的是一个带type的数据体指针
```
以上代码的`cons`函数不能组合。你没有办法使用cons(a, (cons(c,d)))来生成更高阶的类型。可这样修改代码：  
```c
DATA* cons(DATA *a, DATA *b) {
    DATA *p = malloc(sizeof(DATA));
    p->type="pair";
    PAIR *p_data = malloc(sizeof(PAIR));
    p_data->a = a;
    p_data->b = b;
    p->data = p_data;
    return p;
}
```
在使用C语言版本的`pair`时，每个数据都必须手动加上`type`信息。如果考虑用函数式风格实现`pair`，C语言框架下基本上没戏。C语言虽然有函数指针，但其表达能力和应用场景太局限，有兴趣的话可以尝试把前面的Lisp代码翻译成C语言，我们甚至可以降低要求，不要求支持泛型，假设所有数据都是`int`类型的整数。    
当然，从另外一个意义上说，“C语言无所不能”这句话也没有错。你仍然可以在C语言的基础上实现以上所有函数式序对，方法是首先用C语言写一个Lisp解释器，然后执行以上所有Lisp代码。 
 
**有一个特别扯蛋的梗："任何C或Fortran程序复杂到一定程度之后，都会包含一个临时开发的、只有一半功能的、不完全符合规格的、到处都是bug的、运行速度很慢的Common Lisp实现。"——格林斯潘第十定律（Greenspun's Tenth Rule）**
