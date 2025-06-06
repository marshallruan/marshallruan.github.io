我一直对C++没有好感，总觉得每写下一句C++就像是亲手埋了一个爆炸时间为random()的延迟脏弹。断断续续的学了一点Lisp的皮毛之后，这种感觉更甚。很多人对说C++坏话的人嗤之以鼻：  
“理论上你是对的，然而你行你上呗？”  
“世界上有两种语言，人人骂的和没人用的。”  
“C++既给了你OOP，又给了你过程式，你还要怎样？”  
然鹅，近日偶然在某个404的视频应用里面看到了一些C++的hacking例子，觉得特别有意思。我要把它记录在小本本上面，以后喷人用得着。：）  

**声明1：本文所有程序实例均修改自某404视频网站播主@Creel，原视频的标题是Object Oriented Programming is a Dirty，Rotten，Low-down Trick。嗯，我觉得Bjarne Stroustrup肯定欠他很多钱。**  

**声明2：以下的一些hacking代码仅以学术讨论和互喷为目的，无论如何，你都不应该把这种hacking技巧用在实际C++项目中。**  

## 1. 私有变量不私有
几乎所有的OOP语言都有私有成员，我们都会被教导说：“私有的好处是类的内部世界和外部世界会被隔离开来，实现封装。如果你想要在外部世界获取或修改私有成员，你必须用`getter`和`setter`函数来给外部提供接口。”比如这样：  

```c++
#include <iostream>

class Ptest {
public:
    Ptest()  {   //构造函数
        _a = 0;  //初始化为0
    }
    ~Ptest() { }

    int getA() {  //getter
        return _a;
    }

    void setA(int a) { //setter
        _a = a;
    }

private:    
    int _a;  //私有成员_a
};

int main() {
    Ptest ptest = Ptest();  //私有变量_a为0
    ptest.setA(5);          //用setter修改_a为5
    std::cout << ptest.getA() << std::endl; //用getter获取_a
    return 0;
}
```

这个例子看起来没有必要用`setter`和`getter`，因为成员变量只有一个`_a`。但资深人士总是说：“当项目复杂到一定程度这样做是非常必要的，因为一个类很复杂的时候，需要把一些肮脏的，不参与交互的变量囚禁在自己的世界，而只允许很少数的必要的变量和外界交互。”原教旨Java奴们更是规定：“一切类的成员变量都必须私有！”    
好吧，虽然看起来很麻烦，但这个观点看起来是可以接受的。然而，现代的OOP又告诉我们：“当一个类复杂的时候，应该写成很多简单的子类型的组合。”那么问题来了，这些简单的子类型是否有必要这么搞呢？  
很多事情不需要论证的，Java可以说是纯种OOP的语言，但貌似现代Java奴们也开始受不了无休止的`setter`和`getter`了，比如这些苦工们用Lombok库搞了个`@Getter`、`@Setter`注解。C#搞了更诡异的方式，如下：  

```c#

public class Ptest {
    //外部可以c=Ptest.a,不能Ptest.a=c，因为setter私有
    public int a {get; private set;} 

    //等价于一个public变量b
    public int b {get; set;}         
}
```  

这两种方式都是语法糖，本质上仍然是最初的那种老老实实的代码，正所谓“大巧不工，以拙...那个...本来就很拙“。不过有一点可以肯定，一旦成员变量被声明是`private`的，那么无论如何，你在外部世界都应该不允许直接修改它，这一点在Java和C#里成立或部分成立。  
至于C++嘛......说什么好呢？如果我们试图在外部函数里修改`Ptest`类的私有变量`_a`（比如，你写`ptest._a = 4`），在clang++编译器下会报错：error: '_a' is a private member of 'Ptest'。嗯，看起来不错，编译器确实阻止我们这么干，但你也可以这样：  

```c++
...

int main() {
    Ptest ptest = Ptest();    //私有变量_a为0
    int *pa = (int*) &ptest;  //定义一个int*的指针指向ptest地址
    *pa = 123；                //把这个地址改为int型的123
    std::cout << ptest.getA() << std::endl; //能编译通过吗？
    return 0;
}
```  

按照OOP的规矩，这么干应该是绝对被禁止的。但以上的程序完全合法，正确的在外部把`ptest._a`变量修改为123，并打印了出来。

**小贴士：C++的private在挂羊头，买狗肉。：）**  

当然，即使成员变量不止一个，你同样可以这么干：  

```c++
#include <iostream>

class Ptest {
public:
    void printA() { std::cout << _a << std::endl; }
    void printB() { std::cout << _b << std::endl; }
private:    
    int  _a;  //私有成员_a 4个字节
    char _c;  //私有成员_c 1个字节+3个对齐用的空字节
    int  _b;  //私有成员_b 4个字节
};

int main() {
    Ptest ptest;
    int *p = (int *) &ptest;
    *p = 123;   //OK，到这里和前面一样
    p++;        //现在p指向了_c，地址+4，刚好越过了后面3个空字节
    p++;        //现在p指向了_b，地址+4
    *p = 456;   //    成功的把_c赋值为456
    ptest.printA();
    ptest.printB();
    return 0;
}
```

自从有了指针类型，C++给了你C语言操作内存的能力，现在什么`public`,`private`之类的修饰形同虚设。  
当然，上面我们两个`p++`调用是利用了C++类的“对齐”机制。如果你是个内存强迫症患者，为了省掉一些空字节，把变量顺序写为`char _c`在前，两个`int _a, _b`在后，`p++`就不能用了。但那也不能阻挡`main`函数世界里破坏分子的脚步。他们会用强制指针类型转换获得每个变量的地址。  
一言以弊之，在伟大的C++世界里，只要你给别人看了头文件，那么`private`和`public`就变得毫无意义。C++的所谓OOP实际上只是语法糖而已，在它眼里所有的数据都是内存里的一团浆糊。就“封装”的字面意义来说，C++看似搞了那么多花样，其实只是无用功。  

## 2. 诡异的继承和多态
继承和多态应该是OOP中最基础的概念。在读书的时候（1999～2006），总觉得继承和多态简直是神技。同样一个函数名，居然可以即有三个参数，又有四个参数，行为还可以完全不同。比如下面这个猫狗叫的例子：  
```c++
#include <iostream>

class Animal {  //基类：动物
public:
    virtual void say() {  //基类的say
        std::cout << "Animal sound!" << std::endl;
    }
};

class Dog : public Animal { //狗：is a 动物
public:
    virtual void say() { //改变了say的行为
        std::cout << "Wang!" << std::endl;
    }
};

class Cat : public Animal { //猫：is a 动物
public:
    virtual void say() { //改变了say的行为
        std::cout << "Miao!" << std::endl;
    }
};

int main() {
    Cat cat;
    Dog dog;    
    cat.say();
    dog.say();
    return 0;
}

```
这个程序看起来非常自然，你绝对不会认为他有任何问题，而事实上它确实也没有什么问题（当然，这是废话）。同样一个`say`的名字，通过继承绑定到不同的动作。很可能蛮荒时代的编程专家这么干的原因就是想让程序看起来像英语。`cat.say()`的确看起来非常像"cat (which is a Cat type) say"。在C语言里我们就不得不取很多互不相同的名字，如`cat_say`、`dog_say`、`cat_say_more`、`dog_say_more`等。而且随着调用层次的增加，每个不同的类型会加上更长的前缀。到目前为止，一切都还好，以上程序的输出为：  

```
Miao!
Wang!
```

即使有个变态非要让猫狗都做动物叫，仍然OK（调用父类的say函数）：  

```c++
int main() {
    Cat cat;
    Dog dog;    
    ((Animal) cat).say(); //强制把cat的类型转换为Animal然后再say
    ((Animal) dog).say(); //强制把dog的类型转换为Animal然后再say
    return 0;
}
```

程序的输出是：  

```c++
Animal sound!
Animal sound!
```

但用指针创建类的实例时，情况变得不同（诡异的是，这反而是正确的处理方法）：  

```c++
int main() {
    Cat* cat = new Cat();   
    Dog* dog = new Dog();    
    ((Animal*) cat)->say(); //猜猜发生啥？
    ((Animal*) dog)->say(); //猜猜发生啥？
    delete cat;     //记得释放内存
    delete dog;     //记得释放内存
    return 0;
}
```

虽然我们把`cat`和`dog`的类型都强制转换为`Animal*`了，但程序的输出却是：  

```
Miao!
Wang!
```

Java版本的情况也是如此，这种设计可能是为了处理容器（C++是不是叫做泛型来着？），比如你有一个`std::vector<Animal*>`类型的动态数组，里面放上100个猫的指针，100个狗的指针，那你就可以用一个循环让200个猫猫狗狗发出各自不同的叫声，像这样：  

```c++
    std::vector<Animal*> dogs_and_cats;
    Dog *dog1 = new Dog();
    dogs_and_cats.push_back( (Animal*) dog1);
    ...   //一些往数组dogs_and_cats里面添加猫猫狗狗的代码
    for(int i = 0; i < dogs_and_cats.size(); ++i) {
        dogs_and_cats[i]->say(); //猫猫狗狗发出各自愉快的叫声
    }
```

造成指针版本和非指针版本不同输出的原因在于C++的蹩脚语法。在C++中虚函数会在一个类刚刚实例化的时候被绑定到特定的地址。也就是说，无论是`Dog *pd = new Dog();`还是`Dog d;`，在初始化以后，`pd`和`d`的`say()`函数都会绑定到`Dog`类型的`say()`上，且C++声称此后永远不能更改（看后文你就知道它在撒谎，嘿嘿）。  
而`((Animal) d).say();`和`((Animal*) pd)->say();`却是完全不同的东西，它们被简写了，真正的等价代码是：  

```c++
    // ((Animal) d).say()的等价代码
    Animal a = (Animal) d;  //这里新建了一个Animal类型的实例
    a.say();                //say被绑定到Animal类型

    // ((Animal*) pd)->say()的等价代码
    Animal* pa = (Animal*) pd;  //这里没有新建Animal类型的实例
    pa->say();                  //只是指针指向了pd,改变不了say的行为
```

看吧！只要存在类型转换，OOP的行为就变得令人迷惑：从语义上说，我既然转换了类型，成员函数就应该改变到那个类型之下；但从容器角度来说函数行为又不应该被转换。真是两头堵的逻辑。  

**小贴士：问题的根本在于数据和函数就不应该强行绑定在一起。现代OOP又教导我们要基于接口编程，于是你看到大家写一个只有数据的类，再写一个只有函数的类，然后再组合成一个新类。：）组合优先于继承嘛！**  

OOP程序的行为已经足够难以预测了，C++的OOP更是高深莫测（贬义），刚才我说`Dog d`一旦实例化，`d.say()`函数就被指向了`Dog`类型的`say()`函数，且不能更改。但你完全可以干出任何邪乎的事情来，比如这样：  

```c++
int main() {
    Cat* cat = new Cat();   
    Dog* dog = new Dog();    
    unsigned long long* vtable_c = (unsigned long long*)cat;
    unsigned long long* vtable_d = (unsigned long long*)dog;
    //交换两个vtable
    unsigned long long tmp = *vtable_c;
    *vtable_c = *vtable_d;
    *vtable_d = tmp;
    //现在狗做猫叫，猫做狗叫了
    cat->say(); 
    dog->say(); 
    delete cat;     //记得释放内存
    delete dog;     //记得释放内存
    return 0;
}
```

这个程序的输出是：  

```
Wang!
Miao!
```

在上面的代码里，只要你有了指针和强制转换，就能把`say`函数指向任何地方。让猫做狗叫还算是有良心的，如果把`creatData()`函数指向`eraseDataBase()`才叫真的狠毒。  
你要相信我，这绝对不是危言耸听，因为C++的实例根本不知道自己的函数是个什么东西，它只是根据地址去干他看起来应该去做的事情，任何类、实例、数据、函数等概念对于C++来说就是一团二进制泥巴。难怪有个笑话说：“C语言给了你一把手枪自杀，而C++给了你一把冲锋枪，让你先把邻居突突了，然后再自杀。”

## 3.筛子一样的C++
在C++中，一旦一个成员函数被`virtual`修饰，它就等于一手拿刀，一手执筛子（没错，不是盾牌）的战士，第一轮箭雨就已必死无疑。上一个程序`vtable`对虚成员函数的管理就是简单粗暴的地址数组，它甚至不会检查函数名称、参数类型等。在C++眼里，只要你给我一个地址，告诉我它是个函数，那我就执行。比如下面这个稍微复杂一点的例子：  

```c++
#include <iostream>

//这个类是不需要注释的
class Cat { 
public:
    virtual void say() {
        std::cout << "Cat say: Miao!" << std::endl;
    }
    virtual void eat(char* food) {
        std::cout << "Cat eat: " << food << std::endl;
    }
    virtual void sleep() {
        std::cout << "Cat sleep!" << std::endl;
    }
};

//我们在外面定义了三个函数，试图在运行时替换掉Cat类内部的函数
void func1() {
    std::cout << "We can not make sound!" << std::endl;
}

//这里要注意一下：Cat的eat函数只有一个参数，但其实第一个this指针是被默认隐藏的
//所以为了匹配Cat::eat函数的参数表，func2需要在前面加一个void*的指针
void func2(void* this_ptr, char* shit_type) {
    std::cout << "We give " << shit_type << " shit!" << std::endl;
}

void func3() {
    std::cout << "We are totally fucked!" << std::endl;
}

int main() {
    Cat* cat = new Cat();   
    //定义一个无参函数指针类型pfunc
    typedef void(*pfunc) ();
    //我们建立一个vtable,类型就是无参函数指针
    //注意func2需要首先强制转换为无参函数指针，因为其函数体是2参数的
    pfunc vtable[3] = {func1, (pfunc)func2, func3};
    //现在把cat的vtable替换为定义的vtable数组
    *((unsigned long long*) cat) = (unsigned long long)vtable;
    //后来的事情就精彩了：猜猜会发生什么？
    cat->say();
    cat->eat("rat");
    cat->sleep();
    delete cat;     //记得释放内存
    return 0;
}
```

你可能希望这个程序会输出：

```
Cat say: Miao!
Cat eat: rat
Cat sleep!
```

但实际上它会输出：

```
We can not make sound!
We give rat shit!
We are totally fucked!
```

也就是说，类的成员函数在C++里面并没有什么特别的地位，只要有指针和强制类型转换这种东西，所有关键字都变得毫无意义。而CPU在执行这个程序的时候，根本不会管`cat->say()`里面的`say()`是不是`Cat`类里面的函数。  
之前只是私有变量变得不私有，好了，现在类的成员函数也可以在运行时改换门庭。更恐怖的是，这种操作可以支持函数的参数传值！  
不记得哪个文章说过了，C++的对象在运行期根本不知道自己是什么东西，而这样做的目的是为了执行效率。没错，运行越快，死得越快。  
给我一个类库的说明书，我就可以在主函数做任何事情！  

## 总结
当然，C++是一门伟大的语言，从C++98、C++03、C++11一直到C++23，它持续不断的增加各种愚蠢语义和强行塞入各种全世界没有任何人能全部学会的功能，而奇迹之处是：直到今天，世界居然还没有因此而毁灭。  
也有人说，指针和强制类型转换是C带来的历史包袱，不能让C++背锅。但C可没宣称自己是高大上的OOP语言，C也没说自己安全（手动摊手）。  
如果有人在你的项目里用本文的方式写C++，赶紧开除。  
是的，经过很长时间我才明白，WE ARE TOTALLY FUCKED!!!!!!
