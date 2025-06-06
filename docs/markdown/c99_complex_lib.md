C99标准引入了复数库，如果安装的C语言编译器支持c99、gnu99、c11或gnu11参数（gcc-4.8.0或更高版本）
，在源文件中包含`<complex.h>`即可使用此标准库。`<complex.h>`引入了`_Complex`关键字，定义如下：
```c
#define complex  _Complex
#define _Complex_I  ((const float _Complex)__I__)
#define I  _Complex_I
```
C99的复数类型有3种，需要将实、虚部的浮点类型和`_Complex`关键字连用，声明方式如下:
```c
float _Complex c1;
double _Complex c2;
long double _Complex c3;
```
关键字的顺序可以不遵守，也可以写为：
```c
_Complex float c1;
_Complex double c2;
_Complex long double c3;
```
此头文件里还声明了复数运算的常用数学函数（用c开头）：  
1. 三角函数: `ccos`, `csin`, `ctan`, `cacos`, `casin`, `catan`  
2. 双曲函数: `ccosh`, `csinh`, `ctanh`, `cacosh`, `casinh`, `catanh`  
3. 指数、对数、绝对值、幂函数等: `cexp`, `clog`, `cabs`, `cpow`, `csqrt`  
4. 幅角主值、取实部、取虚部、共轭、Riemann球投影等: `carg`, `cimag`,`creal`, `conj`, `cproj`  

## 1. 应用实例
定义三个复数$1+i$、$2+3i$和$0+i$，进行一些简单测试，代码如下：
```c
//main.c
#include <complex.h>
#include <stdio.h>
#include <stdlib.h>

int main()  {
    double _Complex a = 1.0 + 2.0 * I;
    _Complex double b = 3.0 + 4.0 * I;
    _Complex double c = a + b;

    printf ("a=(%f, %f)\n", creal(a), cimag(a));
    printf ("b=(%f, %f)\n", creal(b), cimag(b));
    printf ("c=(%f, %f)\n", creal(c), cimag(c));
    printf ("phase angle of c is %f degrees\n", carg(c)*180.0/3.1415926);
    return 0;
}
```

**注意：编译需链接数学库，否则会提示无法链接相关数学函数。**

在FreeBSD下clang编译器的编译命令为：
```shell
clang main.c -lm -o test
```
执行`./test`程序，其输出为：
```
a=(1.000000, 2.000000)
b=(3.000000, 4.000000)
c=(4.000000, 6.000000)
phase angle of c is 56.309933 degrees
```

## 2. 复数动态数组
动态数组在gcc和clang不同版本的编译器上有些诡异，以下两种写法理论上都应正确：
```c
double _Complex *a = (double _Complex*)malloc (sizeof(double _Complex) * 2);
_Complex double *a = (_Complex double*)malloc (sizeof(_Complex double) * 2);
```
但前者在centos7.x的gcc编译器上可能会报错，提示的错误是`sizeof`函数里`double`后面必须是一个变量而不能是一个宏。第二种写法可以编译通过，不知道此bug是否已被修正。  
在FreeBSD 12.1上clang编译器两种方式开辟动态数组都能成功编译。

## 3. 类型定义和宏
`long double _Complex`这样冗长的类型声明不利于代码维护，可利用`typedef`或宏的功能将长字符串类型替换为更简单的Symbol。在以下代码中，`typedef`和宏的执行结果相同：
```c
//test_type.c
#include <complex.h>
#include <stdio.h>
#include <stdlib.h>

typedef double _Complex dcomplex; //类型定义

#define DCOMPLEX double _Complex  //宏

int main()  {
    dcomplex *a = (dcomplex*)malloc (sizeof(dcomplex) * 2);
    DCOMPLEX *b = (DCOMPLEX*)malloc (sizeof(DCOMPLEX) * 2);
    a[0] = 1.0 + 2.0*I;
    a[1] = 2.0 + 3.0*I;
    b[0] = a[0];
    b[1] = a[1];

    printf("a[0]=(%f, %f)\n", creal(a[0]), cimag(a[0]));
    printf("a[1]=(%f, %f)\n", creal(a[1]), cimag(a[1]));
    printf("b[0]=(%f, %f)\n", creal(b[0]), cimag(b[0]));
    printf("b[1]=(%f, %f)\n", creal(b[1]), cimag(b[1]));

    free(a);
    free(b);
    return 0;
}
```
编译并运行以上代码的输出为：
```
a[0]=(1.000000, 2.000000)
a[1]=(2.000000, 3.000000)
b[0]=(1.000000, 2.000000)
b[1]=(2.000000, 3.000000)

```
