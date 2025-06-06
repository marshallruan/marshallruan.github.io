C语言采用典型的静态类型命令式编程风格，是当前主流操作系统（如Windows,Unix,Unix-like等)的构造语言。从某种意义上说，可以认为操作系统本身是C语言的解释器，当然这个解释器实现得极度糟糕、肮脏、愚蠢、功能残缺、bug不断。
在科学计算场景下，经常有以下需求（假设都是`double`型数据）：  
1. 存在一个通用操作，比如一个线性方程组求解器用于求解$Ax=b$;  
2. 该求解器需要输入一个矩阵$A$，一个右端项$b$，一个输出向量$x$;  
3. 输入矩阵可能是CSR压缩存储的三元组，也可能是致密矩阵的一元组;  
4. 假定外部还可能要提供一个求解预条件问题的函数快速求取$Pu=v$;  

## 1. 面向对象的C++解决方式
要定义一个通用函数解决这个问题，在C++语言框架下可利用OOP的多态，针对CSR压缩的稀疏矩阵求解器很可能如下：
```c++
class CsrSolver {
    public:
        CsrSolver();
        void set_A(int n, int *ia, int *ja, double *a);
        //P的ILU分解下三角部分
        void set_PL(int n, int *il, int *jl, double *l); 
        //P的ILU分解上三角部分
        void set_PU(int n, int *iu, int *ju, double *u);
        void set_b(double *b);
        void compute_Av(double *v);  //A乘以v向量
        void solve_Pv(double *v);    //求解预条件问题
        double* get_x();        
        run();
    private:
        ~CsrSolver();
        ...... //这里是需要的数据
}
```
针对致密阵的求解器可能为：
```c++
class DenceSolver{
    public:
        DenceSolver();
        void set_A(int n, double *a);
        //P的ILU分解下三角部分
        void set_PL(int n, double *L); 
        //P的ILU分解上三角部分
        void set_PU(int n, double *U);
        void set_b(double *b);
        void compute_Av(double *v);  //A乘以v向量
        void solve_Pv(double *v);    //求解预条件问题
        double* get_x();        
        run();
    private:
        ~DenceSolver();
        ...... //这里是需要的数据
}

```
对代码进行“重构”，让`CsrSolver`和`DenceSolver`均继承于一个抽象类`Solver`，在子类中重载虚函数实现多态:
```c++
class Solver{
    public:
        ......
        virtual void set_A();
        virtual void comput_Av();
        virtual void solve_Pv();
        ......
}
```
这种OOP求解器若能设计得很好，其外部调用方式可能为：
```c++
    ......
    //稀疏矩阵：已经对矩阵、向量等分配内存，并初始化完毕
    CsrSolver s1();
    s1.set_A(n, ia1, ja1, a1);
    s1.set_PL(n, il1, jl1, l1);
    s1.set_PU(n, iu1, ju1, u1);
    s1.run();
    ......
    //致密矩阵:已经对矩阵、向量等分配内存，并初始化完毕
    DenseSolver s2();
    s2.set_A(n, a2);
    s2.set_PL(n, l2);
    s2.set_PU(n, u2);
    s2.run();
    ......
```
以上OOP实现的最大优点在于代码“解耦”，尽量实现“模块化”，修改`DenseSolver`的代码不会影响`CsrSolver`的运行。但也要看到如果修改了`Solver`类，所有子类都会受到影响。
在大项目中，由于继承、包含、接口的关系非常复杂，需要非常“资深（你当我说反话好了）”的架构师小心的设计类图，否则还不如老老实实的用纯命令式风格来写。  
OOP对于我这样的菜鸟来说，常规体验是一次重构等价于推倒重来。以前网上有一篇亚马逊员工的文章吐槽维护C++代码好像要爬到一个巨型“屎山”的正中心，在此深表同情。  
但无论如何，OOP策略是比较合乎工业“规范”的做法。这一套方法论在如今大行其道，无人质疑，想必是有其深刻的文化和历史原因。    
对于“科研界”的科学计算代码，需求往往变化太快，算法经常要需要推倒重来。假设我们需要让求解器支持复线性方程组，很多C++程序员第一印象就敲下了`template`。但矩阵和向量的乘法、三角矩阵的回代解法等函数又往往是第三方C语言库提供的，在`template`机制下各种类型爆炸，绕来绕去。即便花了很长时间写了一个勉强能用的版本，其可读性（这里指的是便于团队成员的理解）和可维护性实在堪忧。  
一旦需求稍作改动，比如现在要加入复杂的代数多重网格法（AMG），原来的基类可能需要很大的改动（需增加成员变量或`getter`、`setter`），造成子类也全部要动手术。我想，可能正因如此，“基于接口”（基类全是虚函数，不要成员变量）的OOP工业原则近两年越来越受强调。  
总之，数学上很流畅的计算步骤，一到OOP里，似乎突然间会变得支离破碎，特别像黑客帝国里浑身插着(plugin)营养管的人体电池。  
我感觉有时间去OOP一个求解器，还不如简单粗暴的把`dense_solve`的代码复制一份，改名`csr_solve`，然后修改让其运行正确。如果要支持复数类型，那么就再复制两份，慢慢的把`double`改为`double _Complex`，再修改相关的取范数等特殊函数，得到`z_dense_solve`和`z_csr_solve`。  
科研编程活动往往着眼于用最快的速度实现算法，早日测试算法选型是否合理，能否高效得到正确结果。一个算法随时可能被证明无用而需被替换，替换算法的数据组织和形参极可能与老算法完全不同，我实在不知道OOP如何高效应对这种场景。  
话虽如此，有一条原则是放之四海而皆准的——将“通用”过程抽象为函数可让命令式（或“过程式”）代码拥有模块性并易于维护。  

**声明：我无意争论OOP、命令式或函数式编程哪个好。以我目前的菜鸟水平，只能hold得住以下的思路。我喜欢简单直接的办法，只要语句清晰，词法丑陋一些尚在可以接受范围，毕竟操作系统本身就设计得特别丑陋，：）。**

## 2. C99模拟函数式的方式
若不考虑计算效率，仅从逻辑清晰程度和代码维护成本来看，Lisp语言的高阶函数功能很适合这种求解器的编程场景。
以scheme语言（Lisp的一种方言）为例，求解器（外部函数）可定义为：  
```lisp
(define solve
    (lambda (compute-av solve-pv a l u b) ;compute-av和solve-px是两个函数
    ;;;do-something
    ))
```
调用`solve`函数需实现`compute-av`和`solve-pv`两个函数作为前两个参数:  
```lisp
;;CSR压缩矩阵有关的函数
(define compute-av-csr
    (lambda (a v)
        .....
        ;;返回Av向量
        ))
(define solve-pv-csr
    (lambda (l u v)
        .....
        ;;返回Pu=v的解向量
        ))
;;致密矩阵有关的函数
(define compute-av-dense
    (lambda (a v)
        .....
        ;;返回Av向量
        ))
(define solve-pv-dense
    (lambda (l u v)
        .....
        ;;返回Pu=v的解向量
        ))
;;求解器函数`solve`的调用方式
(solve 
    compute-av-csr
    solve-px-csr
    a1 l1 u1 b1)
(solve 
    compute-av-dense
    solve-px-dense
    a2 l2 u2 b2)

```
由于Lisp语言支持动态类型，形参里`a`、`l`、`u`可为不同类型的数据结构，`solve`函数天然具有多态性。函数作为“一等公民”的编程方式被称为“函数式编程”，这种抽象能力在C语言中没有提供。  
以上的例子只是函数式编程的小式牛刀，Lisp把函数式编程玩出了花，比如“流模式”、“无数据cons”等，这些话题今后再谈。  
现阶段用C或Fortran来实现算法仍是现实需求。利用C语言的函数指针也可尽量模拟Lisp语言的“高阶函数”，从而实现以上Lisp代码的清晰逻辑。
假设求解器算法为对称拟最小残差法（SQMR），我们最低的的需求是“对于不同存储格式的`double`型矩阵和向量，都能调用同一个`sqmr_solve`函数求解”，这可视为一种不用OOP的“多态”。

### 2.1 形参类型化
参考Scheme语言的高阶函数实现方式，首先必须要拥有“wishful thinking”的思维方式，自顶向下进行设计。  
可以乐观的认为调用`sqmr_solve`函数之前一切数组内存都被正确分配。矩阵、向量等都是一种“数据类型”，而非`double`型动态数组那样底层的东西。  
对于CSR压缩存储的矩阵，可以用C语言结构体定义`CSR_MATRIX`类型表达：
```C
typedef struct {
    int *row_ptr;   //行指针
    int *col_ind;   //列下标
    double *val;    //非0值
} CSR_MATRIX;
```
$A$、$L$和$U$的CSR压缩矩阵都可用`CSR_MATRIX`类型传参（$P \approx LU$）。右端项$b$和待解项$x$不妨仍使用`double*`类型表示。  
另外，对于SQMR算法，需要$(r,d,p,s,t,v,\hat{v})$7个辅助向量作为中间存储，为便于内存管理，`malloc`和`free`等函数应在外部实现，以让`sqmr_solve`函数更“纯粹”（类似于Unix哲学中的“一个程序只作一件事”的原则）。  
这些辅助向量可用以下数据结构表示：
```C
typedef struct {
    double *r;   
    double *d;  
    double *p;
    double *s;
    double *t;
    double *v;
    double *vhat;
} SQMR_BUFFER;

```
当然，也可用`double* buff[7]`来声明一个7列的二维动态数组表示`buff`，这和`SQMR_BUFFER`结构体是等价的。

### 2.2 函数指针
仍然假设张三或李四已经实现好了CSR的乘法运算、范数计算等辅助函数（wishful thinking）。具体一点，某矩阵运算C语言库提供了计算$b=Ax$、求解$LUx=b$的以下函数：
```C
//稀疏矩阵
void dcsrmv(int n, int *ia, int *ja, double *a, double *x, double *b);
void dcsrlusv(int n, int *il, int *jl, double *l, 
              int *iu, int *ju, double *u, 
              double *b, double *x);
//致密矩阵
void dmv(int n, double *x, double *b);
void lusv(int n, double *l, double *u, 
              double *b, double *x);

```
为适应通用类型传参，需进行一次简单“封装”：
```C
//注：这里a使用void*类型是因为compute_ax和solve_px需要对csr和dense两种矩阵生效
//  以下av和pv函数无论对于稀疏矩阵还是致密矩阵，其形参类型都是统一的
void compute_av_csr(int n, void *a, double *v, double *av) {
    CSR_MATRIX *m = (CSR_MATRIX *)a;  //强制类型转换
    dcsrmv(n, m->row_ptr, m->col_ind, m->val, v, av);
}

void solve_pv_csr(int n, void *l, void *u, double *v, double *pv) {
    CSR_MATRIX *pl = (CSR_MATRIX *)l; //强制类型转换
    CSR_MATRIX *pu = (CSR_MATRIX *)u;
    dcsrlusv(n, 
             pl->row_ptr, pl->col_ind, pl->val,
             pu->row_ptr, pu->col_ind, pu->val,
             v, pv);
}

void compute_av_dense(int n, void *a, double *v, double *av) {
    dmv(n, (double*)a, v, av);       //强制类型转换
}

void solve_pv_dense(int n, void *l, void *u, double *v, double *pv) {
    lusv(n, (double *)l, (double *)u, v, pv);      //强制类型转换      
}
```
以上两个“通用”函数可以用函数指针作为`sqmr_solve`高阶函数的参数。最简单的`sqmr_solve`函数实现方式为：
```C
void dsqmr_solve(int n,       //方阵的行列数
                 int quiet,   //是否打印中间迭代信息
                 int setx0,   //是否使用已初始化的x为x0开始迭代
                 int max_iter,//最大迭代次数
                 double tol,  //最小相对残差
                 //求Ax的函数，这里的void*是为了适应其他数据结构
                 void(*func_av)(int, void*, double *, double *),   
                 //求解LUx=b的函数，这里的void*是为了适应其他数据结构
                 void(*func_pv)(int, void*, void*, 
                 double *, double *), //inv(LU)x
                 void *a; //A, L,U三个矩阵
                 void *l;
                 void *u;
                 SQMR_BUFFER *buffer; //七个辅助变量
                 double *b,   
                 double *x, 
                 double *err); //最终残差
```
其调用形式为：
```C
    ......
    CSR_MATRIX a1, l1, u1;
    double *a2;
    double *l2;
    double *u2;
    SQMR_BUFFER buff1, buff2;
	......
    //分配buffer的内存，初始化x和b等预备工作
    ......
	//设误差为大值
	double err1 = 1.0;
    double err2 = 1.0;
    //求解稀疏问题
    sqmr_solve(n, 0, 0, 100, 1.0E-8,
               compute_av_csr, solve_pv_csr,
               (void *)a1, (void *)l1, (void *)u1,
               &buff1, b1, x1, &err1);
    //求解致密问题
    sqmr_solve(n, 0, 0, 100, 1.0E-8,
               compute_av_csr, solve_pv_csr,
               (void *)a2, (void *)l2, (void *)u2,
               &buff2, b2, x2, &err2);
    ......
    //按需求决定是否释放buffer的空间
    ......
    
```

### 2.3 进一步封装
前文的简单`sqmr_solve`函数需传递15个参数，不利于代码维护及第三方调用，可利用结构体减少形参，并增加`setter`接口函数使其外部调用方式类似C++的OOP代码。    
另外，之前对预条件矩阵使用了两个型参，必须有一个`l`和一个`u`，无法适应某些上、下三角写到一个矩阵中的第三方库。从通用性考虑，可进一步高阶抽象。  
首先，定义参数结构体：
```C
typedef struct {
    char name[16]; //本次求解的名称
    int  n;
    int  quite;
    int  use_x0;
    int  max_iter;
    void (*fun_av) (int n, void *, double *, double *);
    void (*fun_pv) (int n, void *, double *, double *);
    double *buffer[7];
    void *matrix_a;
    void *matrix_lu;
    double err;
    double tol;
    double *b;
    double *x;
} SQMR_ARGS;

//定义几个辅助函数
void sqmr_init(int n, SQMR_ARG *args) {
    args->n = n;
    args->name = "Noname";
    args->err = 1.0;
    //设定一些默认值
    ........
}
//用函数接口的方式添加setter
void sqmr_set_maxiter(int max, SQMR_ARG *args);
void sqmr_set_tol(double tol, SQMR_ARG *args);
void sqmr_set_func_av(void (*fun_ax) (int n, void *, double *, double *), 
                      SQMR_ARG *args);
void sqmr_set_func_pv(void (*fun_ax) (int n, void *, double *, double *), 
                      SQMR_ARG *args);
void sqmr_run(SQMR_ARG *args);
......
```
假设现在有两种矩阵，CSR和按列存储的致密阵，且它们的LU分解分别是两个CSR和一个致密阵，如下:
```C
    CSR_MATRIX *csr_a   //稀疏A  
    CSR_MATRIX *csr_l;  //稀疏L
    CSR_MATRIX *csr_u;  //稀疏U
    double *den_a;      //致密A
    double *den_lu;     //致密LU，且存到一个方阵里面
    
```
为了SQMR求解器能够调用这两种形式，需统一函数的参数表，实现两组函数如下：
```C
void csr_av(int n, void *a, double *v, double *av) {
    CSR_MATRIX *m = (CSR_MATRIX *)a;
    //CSR的乘法
    ......
}

typedef struct {
    CSR_MATRIX *l;
    CSR_MATRIX *u;
} CSR_LU;

void csr_pv(int n, void *lu, double *v, double *pv) {
    CSR_LU *lu = (CSR_LU*)lu;
    CSR_MATRIX *l = lu->l;
    CSR_MATRIX *u = lu->u;
    //LUx=b的解法
    ......
}

void dense_av(int n, void *a, double *v, double *av) {
    double *dense_a = (double *)a;
    //致密矩阵乘法
    ......
}

void dense_pv(int n, void *lu, double *v, double *pv) {
    double *dense_lu = (double *)lu;
    //致密LU存在一起的解法
}
```
改进版的通用高阶`sqmr_run`函数外部调用方式如下：
```C
    
    SQMR_ARG csr_arg, dense_arg;
    //CSR求解器
    sqmr_init(&csr_arg);
    sqmr_set_name("CSR", &csr_arg);
    sqmr_set_tol(1.0E-8, &csr_arg);
    sqmr_set_func_av(csr_av, &csr_arg);
    sqmr_set_func_pv(csr_pv, &csr_arg);
    sqmr_set_matrix_a((void*)csr_a, &csr_arg);
    CSR_LU lu;
    lu->l = csr_l;
    lu->u = csr_u;
    sqmr_set_matrix_lu((void*)(&lu), &csr_arg);
    ....
    //DENSE求解器
    sqmr_init(&dense_arg);
    sqmr_set_name("DENSE", &dense_arg);
    sqmr_set_tol(1.0E-8, &dense_arg);
    sqmr_set_matrix_a((void*)dense_a, &csr_arg);
    sqmr_set_matrix_lu((void*)dense_lu, &csr_arg);
    sqmr_set_func_av(dense_av, &dense_arg);
    sqmr_set_func_pv(dense_av, &dense_arg);
    //开始求解
    sqmr_run(&csr_arg);
    sqmr_run(&dense_arg);
    ........
```
### 2.4 高阶求解函数的实现
按前文的封装方式，`sqmr_run`函数的所有参数已用`SQMR_ARG`类型封装完毕，其实现的关键部分在于结构体解析和函数指针调用，代码关键部分如下：
```C
void sqmr_run(SQMR_ARG *arg) {
    int n = args->n;
    double *b    = arg->b;
    double *x    = arg->x;
    //设置7个辅助向量
    double *r    = arg->buffer[0];
    double *d    = arg->buffer[1];
    double *p    = arg->buffer[2];
    double *s    = arg->buffer[3];
    double *t    = arg->buffer[4];
    double *v    = arg->buffer[5];
    double *vhat = arg->buffer[6];
    //其他临时变量
    double tow, tow0, cc, cc0, theta, theta0, re, im;
    double alpha, beta, delta;
    double eta, epsilon;
    //1.计算初始残差
    double rnorm, bnorm;
    bnorm = dnorm2(n, b);
    if(bnorm <= 1.0E-10) { bnorm = 1.0; }
    if(arg->use_x0 == 0) {
        //求取r=Ax
        arg->func_av(n, arg->matrix_a, x, r);
        .....
        arg->err = rnorm / bnorm;
    } else {
        for(int i = 0; i < n; ++i) {
            x[i] = 0.0;
            r[i] = b[i];
        }
        rnorm = bnorm;
        arg->err = 1.0;
    }
    ......
    //开始循环
    for(int iter=0; iter < arg->max_iter; ++iter) {
        //如果不是静默模式则输出迭代信息
        if(arg->quiet != 0) {
            printf("solver:%s iter:%d ||r||/||b||:%e\n",
                   arg->name, iter, arg->err);
        }
        ....
        //求解LU×vhat = v
        arg->func_pv(n, arg->matrix_lu, v, vhat);
        ....

    }
}
```

## 总结
把代码里有通用操作的部分提取出来，适当的抽象有利于模块化。  
不要过度抽象而陷入语法陷阱，很多OOP语言提供了高阶抽象语法，但往往带来了“心智包袱”，让你产生必须这么写的错觉。从这个意义来说，我特别不喜欢C++。写C++的时候随便干点什么都有好几种模式，好几种语法，以目前我的菜鸟水平，还是老老实实写C代码好了。  
`void *`这个类型是精髓，在C语言极其简陋的类型系统下，仍然能做到少许的函数式编程模拟。  
世界是不完美的，没有能解决一切的银弹（silver bullet）。

