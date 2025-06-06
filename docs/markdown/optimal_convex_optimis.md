本篇是常规地球物理反演方法的入门级笔记，没有什么高深的理论，写这些东西也只是想把地球物理反演领域比较混乱的术语体系尽量协调，以改善目前各大牛人讨论起来鸡同鸭讲的现状。所有推导尽量用我自己的方法而有别于经典论文。如有争议，你都对。      
对于一个观测数据集$\boldsymbol b$（用此符号以避免和下降方向$\boldsymbol d$冲突），且存在一个精确的、非常复杂的、难以求反函数（或反函数不存在）的函数$F$，使得任一合理地球物理介质模型$\boldsymbol m$的正演响应$\boldsymbol b= F(\boldsymbol m)$，则根据数据$\boldsymbol b$求取真实地球物理介质$\boldsymbol m_{true}$的过程就叫地球物理反演。  
显然，基本上无法直接解出$\boldsymbol m_{true}$，所以只能退而求其次，寻找某个最优解使得$F(\boldsymbol m_{opt}) - \boldsymbol b$的某种范数极小。  
这里只讨论二范数的情况，定义函数：  

$$
\Phi(\boldsymbol m)  = \frac {1}{2} \Vert {F(\boldsymbol m) - \boldsymbol b} \Vert_2 ^2
\tag{1}\label{eq1}
$$  

式$\eqref{eq1}$通常被称为“目标函数”。假定对模型$\boldsymbol m$的定义域不做要求，即$\forall$编号为$i$的模型参数都满足$m_i \in [-\infty, +\infty]$，那么反演过程等价于在$N$维实空间中寻找目标函数$\Phi(\boldsymbol{m})$的极小值点$\boldsymbol m_{opt}$（$N$是向量$\boldsymbol m$的大小），这是典型的无约束最小化问题。  

**小贴士：目标函数取L2范数的平方是为避免对根号求导，乘以1/2是为了让梯度系数更简单。**   

**声明：实际反演问题中目标函数往往还需要加入正则项，以让反演稳定且使模型解更符合某种期望。很多地球物理专家称这种方法是约束反演。但由于解空间本身没有被限制，因此这仍是无约束问题。地球物理上讲的“正则化约束”和“约束最优化”的概念不同。我个人希望大家能把“正则化”、“约束”、“期望”等术语进行区分。**   

式$\eqref{eq1}$的最小化问题也被称为非线性最小二乘问题，通常用迭代法求解。第$k$次迭代的模型更新公式为：  

$$
\boldsymbol m_{k+1}  = \boldsymbol m_{k} + \alpha_k \boldsymbol d_{k} = \boldsymbol m_{k} + \boldsymbol s_{k}
\tag{2}\label{eq2}
$$  

其中，向量$\boldsymbol{d}_{k}$被称为下降方向(descent direction)，$\alpha_k > 0$被称为步长(step length)，步长和下降方向共同确定了模型修正量$\boldsymbol s_k$。  
若对于$\forall$模型$\boldsymbol m$，除某个极小点$\boldsymbol m_*$之外，都有$\Phi(\boldsymbol m) > \Phi(\boldsymbol m_*)$，那么函数$\Phi$被称为凸函数(convex function)。凸函数的最小化问题当然被称为凸优化，所用的算法自然叫凸优化算法。此时极小点$\boldsymbol m_*$也被称为全局极小点，且满足$\boldsymbol m_* = \boldsymbol m_{opt} = \boldsymbol m_{true}$。  
但地球物理反问题由于数据不完备(多解性)和正演非双射（等值性）的特点，导致目标函数非凸。此时沿用凸优化算法只能找到迭代初始模型$\boldsymbol m_0$某个邻域内目标函数的极小（局部是凸函数），这造成了不同初始模型得到不同反演结果的现象（初始模型依赖性）。  

**小贴士：（1）反演结果的可信度很大意义上由目标函数性质决定，最好的情况是目标函数为凸函数；（2）对于存在唯一全局极小的非凸问题，一切常规反演方法都只能保证局部极小；（3）然鹅，几乎所有地球物理反演问题都有无穷多个全局极小，所以无论你用什么算法，找到全局极小也不等于模型可信；（4）很明显，可信的做法是在合理初始模型的基础上迭代，或在一个可信域内进行凸优化（约束反演）。**   

废话已扯了太多，无论你是否理解都不影响后文内容，从现在开始我们只讨论如何构建一系列算法去迭代寻优，也就是构造下降方向$\boldsymbol d_{k}$和步长$\alpha_k$。  
当然，我们必须要进行一些约定：（1）目标函数是光滑可导的；（2）每次迭代的目标函数梯度向量$\boldsymbol g$可以求出来，其每个元素为$g_i = dF(\boldsymbol m)/dm_i$（注意大小写，下标$i$表示向量的第$i$个元素）。  
好了，现在请深吸一口气，回忆一下高等代数第一册，我们所作的只不过是把一元函数推广到多维空间而已，那么开始吧.....  

## 1. 梯度法（Gradient Method）
首先，回忆一下泰勒展开式：  

$$
f(x)  = f(x_0) + f^{'}(x_0)(x-x_0) +  \frac{1}{2} f^{''}(x_0) (x-x_0) ^ 2 + R_n(x)
\tag{3}\label{eq3}
$$  

这个伟大的东西很重要，它可以把一个复杂的函数用某点函数值和导数值展开。把泰勒公式推广到高维，对于迭代式$\eqref{eq2}$，$\boldsymbol m_{k+1}$的目标函数值可表示为：  

$$
\begin{align} 
\Phi(\boldsymbol m_{k+1}) &= \Phi(\boldsymbol m_k) + \boldsymbol g_k^T (\boldsymbol m_{k+1} - \boldsymbol m_k) \\
&+ \frac{1}{2} (\boldsymbol m_{k+1} - \boldsymbol m_k)^T \boldsymbol B_k (\boldsymbol m_{k+1} - \boldsymbol m_k) + R_n( \boldsymbol m_{k+1})
\end{align}
\tag{4}\label{eq4}
$$  

其中，$\boldsymbol B_k$是对应模型$\boldsymbol m_k$的海塞矩阵，这里没有使用符号$\boldsymbol H_k$是为了和拟牛顿法经典论文的符号统一，我们会用$\boldsymbol H_k$表示海塞矩阵的逆矩阵，即$\boldsymbol H_k = \boldsymbol B_k^{-1}$。海塞矩阵的具体形式见$\eqref{eq15}$。  
按照国际惯例，忽略高阶项，目标函数的一阶逼近如下：  

$$
\Phi(\boldsymbol m_{k+1})  \approx \Phi(\boldsymbol m_k) + \boldsymbol g_k^T (\boldsymbol m_{k+1} - \boldsymbol m_k) 
\tag{5}\label{eq5}
$$  

把约等号改为等号，对于第$k$次迭代的模型修正量，存在关系式：  

$$
\Phi(\boldsymbol m_{k+1})  = \Phi(\boldsymbol m_k) + \boldsymbol g_k^T  \boldsymbol s_k 
        = \Phi(\boldsymbol m_k) + \lvert {\boldsymbol g_k} \rvert \cdot \lvert {  \boldsymbol s_k } \rvert \cdot cos\theta
\tag{6}\label{eq6}
$$  

其中，$\theta$是向量$\boldsymbol g_k$和$\boldsymbol s_k$的夹角。很显然，$cos \theta = -1$时$\Phi(\boldsymbol m_{k+1})$取得当前迭代的最小目标函数值。也就是说，模型修正的方向与梯度方向成180夹角时目标函数下降最快。故而可构建下降方向$\boldsymbol d_k = - \boldsymbol g_k$，它被称为最速下降方向(steepest descent direction)，因此梯度法也被称为“最速下降法”，其模型修正量可表示为$\boldsymbol s_k = - \alpha_k \boldsymbol g_k$。  
步长$\alpha_k$可通过泛函$\Phi(\boldsymbol{m}_{k} + \alpha \boldsymbol{d}_k)$关于变量$\alpha$的极小点确定，根据极值条件有：  

$$
\frac {\partial \Phi(\boldsymbol m_k + \alpha \boldsymbol d_k)  } {\partial {\alpha}} = 0
\tag{7}\label{eq7}
$$  

根据$\Phi(\boldsymbol m_k + \alpha \boldsymbol d_k)$构建关于式$\eqref{eq7}$的解析表达式，形成一元方程，即可求解出最优的$\alpha_k$，这样的步长叫做“精确线搜索步长”。  
但$\Phi(\boldsymbol m_k + \alpha \boldsymbol d_k)$的表达式往往无法解析给出，所以只能对$\alpha_k$进行某种近似得到$\alpha_k^{(0)} \approx \alpha_k$，再以它为初值搜索能使目标函数充分下降（请自行搜索Wolfe条件）的“不精确线搜索步长”。  
不精确线搜索的方法有很多（如二次插值法、二分回溯法、黄金分割法等），但良好的初始尝试步长$\alpha_k^{(0)}$ 对算法稳定性和收敛速度的影响巨大。实用中可基于式$\eqref{eq4}$进行二阶泰勒展开得到目标函数的近似解析表达式，如下：  

$$
\begin{align}
 \Phi(\boldsymbol m_k + \alpha \boldsymbol d_k) 
    &= \Phi(\boldsymbol m_k) + \boldsymbol g_k^T \boldsymbol s_k  + \frac{1}{2} \boldsymbol s_k^T \boldsymbol B_k \boldsymbol s_k \\
    &= \Phi(\boldsymbol m_k) + \alpha \boldsymbol g_k^T \boldsymbol d_k  + \frac{\alpha ^ 2 }{2} \boldsymbol d_k^T \boldsymbol B_k \boldsymbol d_k \\
\end{align}
\tag{8}\label{eq8}
$$  

对$\alpha$求导后代入式$\eqref{eq7}$，得到：  

$$
\frac {\partial \Phi(\boldsymbol m_k + \alpha \boldsymbol d_k)  } {\partial {\alpha}} = \boldsymbol g_k^T \boldsymbol d_k + \alpha \boldsymbol d_k^T \boldsymbol B_k \boldsymbol d_k = 0
\tag{9}\label{eq9}
$$  

显然，$\alpha_k^{(0)}$的表达式如下：  

$$
\alpha_k^{(0)} = \frac { - \boldsymbol g_k^T \boldsymbol d_k} {\boldsymbol d_k^T \boldsymbol B_k \boldsymbol d_k} 
               = \frac { \boldsymbol g_k^T \boldsymbol g_k} {\boldsymbol g_k^T \boldsymbol B_k \boldsymbol g_k}
\tag{10}\label{eq10}
$$  

**小贴士：从式（6）和式（8）中不难看出，下降方向必须满足$\boldsymbol g_k^T \boldsymbol{d}_k < 0$，这是线搜索步长存在且能让目标函数减小的必要条件。**

当然，式$\eqref{eq10}$中的$\boldsymbol{B}_k$也难以求解且计算特别耗时。但针对不同的地球物理问题，往往有巧妙方法进行快速近似。你可以查看[《解线性方程组的CG和PCG方法》](optimal_cg_and_nlcg.md)的第1节，步长表达式与式$\eqref{eq10}$相似。  
梯度法的本质是一阶泰勒近似，实用问题中由于这种逼近误差较大，即使对于一个凸函数，只要问题的条件数很大，梯度法就会因较差的初始点而出现震荡现象（见上面链接的图1）。    

**吐槽：曾被某知名反演专家问到Rodi在2001年发表的经典NLCG大地电磁反演论文的初始步长是怎么来的。我说这难道不是非常obvious的吗？然后就得了个骄傲自满装腔作势的恶名。（手动摊手）**   

我们无需给出具体的算法步骤甚至流程图（后文也一样），因为经典凸优化问题的核心问题只有两个——步长和下降方向，其它的都很obvious。：）

## 2. 牛顿法（Newton Method）

第1节证明了一阶泰勒近似条件下，目标函数下降最快的方向必须和梯度方向成180度（负梯度方向），这是非常粗略的一种估计。Obviously，你一定会想用更高阶的泰勒展开去试试，就像这样：  

$$
\begin{align} 
\Phi(\boldsymbol m_{k+1}) &= \Phi(\boldsymbol m_k) + \boldsymbol g_k^T (\boldsymbol m_{k+1} - \boldsymbol m_k) \\
&+ \frac{1}{2} (\boldsymbol m_{k+1} - \boldsymbol m_k)^T \boldsymbol B_k (\boldsymbol m_{k+1} - \boldsymbol m_k)
\end{align}
\tag{11}\label{eq11}
$$  

和梯度法的分析过程不同，式$\eqref{eq11}$右边有三项：反演问题中大多第1、3项为正，第2项为负。由于模型修正量$\boldsymbol s_k = \boldsymbol m_{k+1} - \boldsymbol m_k$包含在第3项之中，此时负梯度方向不再保证最高的下降效率。
首先，$\boldsymbol g_k^T \boldsymbol s_k < 0$仍须满足，同时还要让第3项数值尽量的小。  
精准的描述为：$\boldsymbol s_k$让式$\eqref{eq11}$取最小值的必要条件为$\Phi(\boldsymbol{m}_{k+1})$对$\boldsymbol s_k$的偏导数为0。即：  

$$
\begin{align} 
\frac{ \partial \Phi (\boldsymbol{m}_{k+1}) }
     {\partial \boldsymbol s_k}
     &= \frac{ \partial \Phi (\boldsymbol{m}_k + \boldsymbol{s}_k ) } {\partial \boldsymbol s_k} \\
     &= \frac{ \partial \Phi (\boldsymbol{m}_k) } {\partial \boldsymbol s_k} + \frac{ \partial  (\boldsymbol{g}_k^T \boldsymbol{s}_k) } {\partial \boldsymbol s_k}
        + \frac{1}{2} \frac{ \partial  (\boldsymbol{s}_k^T \boldsymbol{B}_k \boldsymbol{s}_k) } {\partial \boldsymbol s_k} \\
     &= 0 + \boldsymbol{g}_k + \boldsymbol{B}_k \boldsymbol{s}_k
\end{align}
\tag{12}\label{eq12}
$$  

根据$\eqref{eq12}$可得到线性方程组：  

$$
\boldsymbol{B}_k \boldsymbol{s}_k = \alpha_k \boldsymbol{B}_k \boldsymbol{d}_k = - \boldsymbol{g}_k
\tag{13}\label{eq13}
$$  

由于$\alpha_k$是标量，为简化问题可规定$\alpha_k = 1$（线搜索简单了），下降方向可用以下方程构建：  

$$
\boldsymbol{B}_k \boldsymbol{d}_k = - \boldsymbol{g}_k
\tag{14}\label{eq14}
$$  

$\boldsymbol{d}_k = - \boldsymbol{B}_k^{-1} \boldsymbol{g}_k$被称为牛顿方向或牛顿步，它是如此牛逼，以至于必须用牛逼的物理学家命名，以它为核心的凸优化方法唤作“牛顿法”。线性二次型牛顿法是一步收敛的，非线性问题中牛顿法的效率（以迭代次数衡量）普遍远高于梯度法。  
**你可能会疑惑：**
既然牛顿法的下降效率更快，那为什么还要把负梯度方向称为“最速下降方向”呢？这是因为“距离”的概念有不同的定义（对应不同的范数），收敛速度是距离和时间之比，距离不同自然速度不同。
负梯度方向基于一阶近似（直线方程），而牛顿方向基于二阶近似（二次曲线），如果真实目标函数的形状为碗状，进行折线近似时要走更长的路。  
梯度法以曲线长度为距离，牛顿法以曲线在模型轴上的投影为距离。数学上可解释为：梯度法是欧式距离（L2范数）下的最速下降，而牛顿法是$\boldsymbol{B}$范数下的最速下降。当然，欧式距离也可认为是$\boldsymbol{I}$范数，这样二者就统一了。：）    

**小贴士：牛顿方向利用海塞矩阵$\boldsymbol{B}$对原问题的线性空间进行了仿射变换，所以有一种说法是牛顿法是梯度法经过“缩放(scaling)”后的效果，实际上就是让负梯度方向在各子维度都有最佳步长。**  

虽然牛顿法很牛，但海塞矩阵$\boldsymbol{B}$有两个缺陷：（1）不保证正定，导致式$\eqref{eq14}$不一定有解；（2）地球物理反问题海塞矩阵的计算和存储成本过大。第（2）个问题可进行“退化”式逼近以实用化（参考后文），第（1）个问题可能存在两个解决方案：  
**其一**，
引入足够大的正标量$\lambda$使矩阵$\boldsymbol{B}_k + \lambda \boldsymbol{I}$正定，下降方向变为$\boldsymbol{d}_k = - (\boldsymbol{B}_k + \lambda \boldsymbol{I})^{-1} \boldsymbol{g}_k$，**（注意这里的$\lambda$是下降方向方程正则解的正则因子，而非正则反演问题的正则因子）**
由于$\lambda$的引入，步长$\alpha_k = 1$可能导致目标函数上升，需以$\alpha_k^{(0)}=1$为初始尝试步进行不精确线搜索构建稳定迭代。
也可令步长不变但随迭代调整$\lambda$的大小，若目标函数增大就减小$\lambda$，否则增大$\lambda$，由此发展出一个新算法体系——**信赖域方法。**  
**其二**，
用迭代法（如CG）求$\eqref{eq14}$的某个截断解$\boldsymbol{d}_k^*$，配合$\alpha_k^{(0)}=1$的不精确线搜索完成反演迭代。这一技巧被称为不精确牛顿法（Inexact Newton Method）或截断牛顿法（Truncated Newton Method）。  
但无论如何改进，在大型地球物理反演问题中，由于计算成本和问题复杂性的限制，牛顿法基本无法实用。但你先别着急喷我：“推导那么久，搞了半天弄个没用的算法”。只有熟知梯度法和牛顿法的基本原理才能理解其它实用凸优化方法为何是那个样子，沙滩上无法建高楼，这两个方法就是地基。  

**吐槽：有人居然又又又又要创新，说如果进行三阶四阶泰勒展开不就可以开发出更快的算法吗？我一脸懵逼：大哥，三阶导四阶导存在吗？算它们不要钱啊？**  

**小贴士：梯度法和牛顿法就像线段的两个端点，就像C语言与Lisp语言是编程语言设计的两个端点一样，中间方法或中间语言都在端点之间寻求计算成本和收敛效率的平衡。**  

## 3. 高斯牛顿法（Gauss-Newton Method，GN）
牛顿法实现困难是因为海塞矩阵里面有二阶偏导。但如果目标函数是如式$\eqref{eq2}$的非线性最小二乘形式，则$\boldsymbol{B}_k$存在一个特殊的近似手段。假定模型$\boldsymbol{m}$的维度为$N$，数据$\boldsymbol{b}$的维度为$M$，那么$\boldsymbol{B}_k$的表达式为：  

$$
\boldsymbol{B}_k = 
\begin{bmatrix}
    { \frac {\partial^2 \Phi(\boldsymbol{m}_k)} {\partial m_1^2} } 
       & { \frac {\partial^2 \Phi(\boldsymbol{m}_k)} {\partial m_1 \partial m_2} }  
       & {\cdots}
       & {\frac {\partial^2 \Phi(\boldsymbol{m}_k)} {\partial m_1 \partial m_N} }  \\
    {\frac {\partial^2 \Phi(\boldsymbol{m}_k)} {\partial m_2 \partial m_1} } 
       & { \frac {\partial^2 \Phi(\boldsymbol{m}_k)} {\partial m_2^2 } }  
       & {\cdots}
       & {\frac {\partial^2 \Phi(\boldsymbol{m}_k)} {\partial m_2 \partial m_N} }  \\
    {\vdots }  & {\vdots } & {\vdots } & {\vdots } \\
    {\frac {\partial^2 \Phi(\boldsymbol{m}_k)} {\partial m_N \partial m_1} } 
       & { \frac {\partial^2 \Phi(\boldsymbol{m}_k)} {\partial m_N \partial m_2 } }  
       & {\cdots}
       & {\frac {\partial^2 \Phi(\boldsymbol{m}_k)} {\partial m_N^2} }  \\
\end{bmatrix}
\tag{15}\label{eq15}
$$  

没错，它是对称矩阵。不但如此，它的每个元素$B_{ij}$还可统一表示为：  

$$
\begin{align}
B_{ij} &=  \frac {\partial^2 \Phi(\boldsymbol{m}_k)} {\partial m_i \partial m_j} 
       = \frac{1}{2} \sum_{l=1}^{M} \frac {\partial^2  [F(\boldsymbol{m}_k)_l - \boldsymbol{b}_i]^2   } {\partial m_i \partial m_j} \\
       &= \sum_{l=1}^{M} \frac{\partial \{  [F(\boldsymbol{m}_k)_l - \boldsymbol{b}_i] \frac{\partial F(\boldsymbol{m}_k)_l} {\partial m_i} \}} {\partial m_j} \\
       &= \sum_{l=1}^{M} \{ \frac{\partial F(\boldsymbol{m}_k)_l} {\partial m_i} \frac{\partial F(\boldsymbol{m}_k)_l} {\partial m_j}
          + \frac{\partial^2 F(\boldsymbol{m}_k)_l} {\partial m_i \partial m_j}   [F(\boldsymbol{m}_k)_l - \boldsymbol{b}_i] \} \\
\end{align}
\tag{16}\label{eq16}
$$  

国际惯例，忽略二阶偏导项，得到：  

$$
\hat{\boldsymbol{B}_{ij}}  = \sum_{l=1}^{M} \frac{\partial F(\boldsymbol{m}_k)_l} {\partial m_i} \frac{\partial F(\boldsymbol{m}_k)_l} {\partial m_j} \approx \boldsymbol{B}_{ij}
\tag{17}\label{eq17}
$$  

如此，$\boldsymbol{B}_k$便用$\hat{\boldsymbol{B}_k}$代替，为简化记号，用$\partial F_l$作为$\partial F(\boldsymbol{m}_k)_l$的简写，则有：  

$$
\hat{\boldsymbol{B}_k} = 
\begin{bmatrix}
    { \sum\limits_{l=1}^{M} (\frac{\partial F_l} {\partial m_1})^2  } 
       & { \sum\limits_{l=1}^{M} \frac{\partial F_l} {\partial m_1}  \frac{\partial F_l} {\partial m_2}}  
       & {\cdots}
       & { \sum\limits_{l=1}^{M} \frac{\partial F_l} {\partial m_1}  \frac{\partial F_l} {\partial m_N} }  \\
    {\sum\limits_{l=1}^{M} \frac{\partial F_l} {\partial m_2}  \frac{\partial F_l} {\partial m_1}  } 
       & {\sum\limits_{l=1}^{M} (\frac{\partial F_l} {\partial m_2})^2 }  
       & {\cdots}
       & { \sum\limits_{l=1}^{M} \frac{\partial F_l} {\partial m_2}  \frac{\partial F_l} {\partial m_N} }  \\
    {\vdots }  & {\vdots } & {\vdots } & {\vdots } \\
    {\sum\limits_{l=1}^{M} \frac{\partial F_l} {\partial m_N}  \frac{\partial F_l} {\partial m_1}  } 
       & { \sum\limits_{l=1}^{M} \frac{\partial F_l} {\partial m_N}  \frac{\partial F_l} {\partial m_2}}  
       & {\cdots}
       & { \sum\limits_{l=1}^{M} (\frac{\partial F_l} {\partial m_N})^2 }  \\
\end{bmatrix}
\tag{18}\label{eq18}
$$  

式$\eqref{eq18}$是主对角元不小于0的对称$N \times N$半正定矩阵，且每个元素都是$M$项偏导乘积的求和。它可写为$\boldsymbol{J}_k^T \boldsymbol{J}_k$的形式，其中$\boldsymbol{J}_k$是一个$M \times N$矩阵，如下：  

$$
\boldsymbol{J}_k = 
\begin{bmatrix}
    {  \frac{\partial F_1} {\partial m_1}  } 
       & { \frac{\partial F_1} {\partial m_2} }  
       & {\cdots}
       & { \frac{\partial F_1} {\partial m_N}  }  \\
    {  \frac{\partial F_2} {\partial m_1}  } 
       & { \frac{\partial F_2} {\partial m_2} }  
       & {\cdots}
       & { \frac{\partial F_2} {\partial m_N}  }  \\
    {\vdots }  & {\vdots } & {\vdots } & {\vdots } \\
    {  \frac{\partial F_M} {\partial m_1}  } 
       & { \frac{\partial F_M} {\partial m_2} }  
       & {\cdots}
       & { \frac{\partial F_M} {\partial m_N}  }  \\
\end{bmatrix}
\tag{19}\label{eq19}
$$  

**小贴士：这个神奇的偏导数矩阵$\boldsymbol{J}_k$是第$k$次迭代正演响应$F(\boldsymbol{m}_k)$对模型$\boldsymbol{m}_k$的雅可比（Jacobian）矩阵，也被称为灵敏度矩阵。**  

对于非线性最小二乘问题，不仅海塞矩阵可以用雅可比矩阵内积表示，梯度$\boldsymbol{g}_k$也可表示为雅可比转置与残差向量乘积，如下：  

$$
\boldsymbol{g}_k = \boldsymbol{J}_k^T [F(\boldsymbol{m}_k) - \boldsymbol{b}] = 
\begin{bmatrix}
    {  \sum\limits_{l=1}^{M} \frac{\partial F_l} {\partial m_1} (F_l - b_l)  }  \\
    {  \sum\limits_{l=1}^{M} \frac{\partial F_l} {\partial m_2} (F_l - b_l)  } \\
    {\vdots }   \\
    {  \sum\limits_{l=1}^{M} \frac{\partial F_l} {\partial m_N} (F_l - b_l)  } \\
\end{bmatrix}
\tag{20}\label{eq20}
$$  

利用式$\eqref{eq14}$牛顿法下降方向的方程，近似后可得到：  

$$
 \boldsymbol{J}_k^T \boldsymbol{J}_k \boldsymbol{d}_k = - \boldsymbol{J}_k^T [F(\boldsymbol{m}_k) - \boldsymbol{b}] = - \boldsymbol{g}_k
\tag{21}\label{eq21}
$$  

新方法中$\boldsymbol{d}_k = - (\boldsymbol{J}_k^T \boldsymbol{J}_k)^{-1} \boldsymbol{J}_k^T [F(\boldsymbol{m}_k) - \boldsymbol{b}]$，步长同牛顿法一样取1，这样便得出高斯牛顿法（Gauss-Newton Method，简称GN）。   
当然，在有些“经典”地球物理论文里，GN方向可能采用另一种推导方式。从偏导数定义出发，在$\boldsymbol{d}_k$“足够小”时，存在$F(\boldsymbol{m}_k + \boldsymbol{d}_k) = F(\boldsymbol{m}_k) + \boldsymbol{J}_k \boldsymbol{d}_k$的近似，极小点必要条件为：  

$$
 \frac{\partial \Phi(\boldsymbol{m}_{k+1})}{\partial \boldsymbol{m}_{k+1}} 
   = [F(\boldsymbol{m}_k + \boldsymbol{d}_k) - \boldsymbol{b}]\frac{\partial F(\boldsymbol{m}_{k+1})}{\partial \boldsymbol{m}_{k+1}} = 0
\tag{22}\label{eq22}
$$  

显然，若右边的一阶偏导项不为0，则要求残差向量为0，即：  

$$
 F(\boldsymbol{m}_k + \boldsymbol{d}_k) - \boldsymbol{b} = F(\boldsymbol{m}_k) + \boldsymbol{J}_k \boldsymbol{d}_k - \boldsymbol{b}= 0
\tag{23}\label{eq23}
$$  

也就是：  

$$
  \boldsymbol{J}_k \boldsymbol{d}_k = - [F(\boldsymbol{m}_k)  - \boldsymbol{b}]
\tag{24}\label{eq24}
$$  

因为$\boldsymbol{J}_k$不是方阵，所以在方程两边左乘$\boldsymbol{J}_k^T$以求取$\boldsymbol{d}_k$的线性最小二乘解，从而得到式$\eqref{eq21}$。  

**吐槽：我个人不喜欢第二个推导方式，因为式$\eqref{eq23}$用向量差代替了偏微分，而从牛顿法降阶逼近去理解可以明确得知这种近似在何种情况下成立。显然对于式$\eqref{eq16}$而言，GN近似成立的条件是二阶偏导项求和的数值必须尽量小于其它项的求和。**  

GN这个名字把数学大爷高斯和伟大的牛爵爷放到一起，足以表明这一算法相当牛逼，具体表现为：（1）海塞逼近的主对角元一定不小于0；（2）充分利用了非线性最小二乘的特性；（3）不需计算二阶偏导，絕大多数情况下收敛效率和牛顿法相当；（4）即使原问题二阶偏导项求和后数值很大，GN方向配合不精确线搜索仍然能稳定收敛。  
不过对于更大规模的问题，显式计算并存储$\boldsymbol{J}_k$和$\boldsymbol{J}_k^T \boldsymbol{J}_k$仍不可接受（何况还要求逆），而有些地球物理问题可用隐式方法高效小内存地迭代求解方程$\eqref{eq21}$（如大地电磁法）得到 $\boldsymbol{d}_k$的截断解，这种方法被称为截断高斯牛顿法（TGN）或不精确高斯牛顿法（IGN）。  

**小贴士：若$\boldsymbol{J}_k^T \boldsymbol{J}_k$不正定，在显式GN方法中，仍然可以像牛顿法一样引入$\boldsymbol{J}_k^T \boldsymbol{J}_k + \lambda \boldsymbol{I}$做正定化处理，进而得出一个非常出名的信赖域方法，被称为列文伯格-马奎特法（Levenberg-Marquardt method，简称L-M），诨名唤作阻尼最小二乘法。**  

## 4. 拟牛顿法（Quasi-Newton Method，QN）
虽然GN方法已经足够牛逼，但它只保证非线性最小二乘问题的有效性，并且涉及雅可比矩阵的复杂计算。于是人们开始寻找更普适、更快捷的牛顿法逼近形式。  
当然，我们仍然要从泰勒展开这个神奇的东西出发。按照国际惯例，我们先照抄式$\eqref{eq12}$，不过加入了一点小小改变，现在要用$\boldsymbol{m}_k$的目标函数去逼近$\Phi(\boldsymbol{m}_{k-1})$，有：  

$$
\begin{align}
 \Phi(\boldsymbol{m}_{k-1}) 
    & \approx \Phi(\boldsymbol m_k) + \boldsymbol g_k^T (\boldsymbol{m}_{k-1} - \boldsymbol{m}_k)  \\
    & + \frac{1}{2} (\boldsymbol{m}_{k-1} - \boldsymbol{m}_k)^T \boldsymbol B_k (\boldsymbol{m}_{k-1} - \boldsymbol{m}_k)   \\
\end{align}
\tag{25}\label{eq25}
$$  

两边对$\boldsymbol{m}_{k-1}$求偏导，令$\boldsymbol{y}_{k} = \boldsymbol{g}_{k+1} - \boldsymbol{g}_{k}$得到：  

$$
\begin{align}
\frac{\partial \Phi(\boldsymbol{m}_{k-1}) }{\partial \boldsymbol{m}_{k-1}} = \boldsymbol{g}_{k-1}
    & \approx  \boldsymbol{g}_k  +  \boldsymbol{B}_k (\boldsymbol{m}_{k-1} - \boldsymbol{m}_k) \\
\Rightarrow  \boldsymbol{g}_{k-1} - \boldsymbol{g}_{k}  
    & \approx   \boldsymbol{B}_k (\boldsymbol{m}_{k-1} - \boldsymbol{m}_k) \\
\Rightarrow  \boldsymbol{g}_{k} - \boldsymbol{g}_{k-1}  
    & \approx    \boldsymbol{B}_k (\boldsymbol{m}_{k} - \boldsymbol{m}_{k-1}) \\
\Rightarrow  \boldsymbol{y}_{k-1}  
    & \approx    \boldsymbol{B}_k \boldsymbol{s}_{k-1}
\end{align}
\tag{26}\label{eq26}
$$  

把约等于改为等于（常规操作），根据递推关系，牛顿法第$k+1$次迭代满足以下方程：  

$$
\begin{align}
& \boldsymbol{B}_{k+1}  \boldsymbol{s}_{k} = \boldsymbol{y}_{k} \\
& \boldsymbol{H}_{k+1}  \boldsymbol{y}_{k} = \boldsymbol{s}_{k}    
\end{align}
\tag{27}\label{eq27}
$$  

其中，$\boldsymbol{H}=\boldsymbol{B}^{-1}$，式$\eqref{eq27}$被称为拟牛顿（Quasi-Newton）条件，它建立了当前迭代的海塞矩阵与上一次迭代信息（模型修正量和梯度改变量）之间的关系，使得利用之前迭代的信息逼近当前迭代的海塞矩阵变得可能。  
基于拟牛顿条件（这是个必要条件）逼近$\boldsymbol{H}$和$\boldsymbol{B}$的方法被称为拟牛顿法（QN）。和Quasi的字面意思对应，
**QN方法实际上是一类方法的统称**。有一些QN算法对$\boldsymbol{H}$逼近并可高效计算它和同维列向量的乘积，非常适应大规模最优化问题。  
### 4.1 SR1算法
SR1的全称为对称秩-1（Symmetric Rank-1）方法，它对海塞的逼近方式比较简单，如下：  

$$
\hat{\boldsymbol{H}}_{k+1} = \hat{\boldsymbol{H}}_{k} + \Delta  \hat{\boldsymbol{H}}_{k} = \hat{\boldsymbol{H}}_{k} + \beta  \boldsymbol{u} \boldsymbol{u}^T
\tag{28}\label{eq28}
$$  

注意，这里我们在矩阵符号上加了一个尖帽子以表明对海塞或其逆矩阵的逼近。式$\eqref{eq28}$在第0次迭代取$\hat{\boldsymbol{H}}_0 = \boldsymbol{I}$，之后所有次迭代的$\boldsymbol{H}$从编程角度均可视为不断更新得到的，故而后文的所有QN算法也被成为“某某更新公式”。  
SR1名字中的S表示对称，要求更新后$\Delta  \hat{\boldsymbol{H}}_{k}$保持对称性；而R1的意思则是表示秩-1更新，因为$\Delta \hat{\boldsymbol{H}}_{k}=\beta \boldsymbol{u} \boldsymbol{u}^T$，向量$\boldsymbol{u}$乘以它的转置后得到的对称阵的秩当然是1。  

**吐槽：我个人认为SR1这个名字极其贴切，其它QN方法的发明人显然想不出更好的名字，只好用发明人的姓名首字母连起来。：）**  

**趣闻：SR1是William C. Davidon在1956年提出的，但由于他的论文里没有做算法收敛性分析，导致被无情拒稿。因此SR1没能成为公认的第一个QN算法。审稿大人牛逼（震声）！**  

为得出标量$\beta$和向量$\boldsymbol{u}$的表达式，首先在式$\eqref{eq28}$两边同时乘以$\boldsymbol{y}_{k}$，再代入式$\eqref{eq27}$，即：  

$$
\begin{align}
\hat{\boldsymbol{H}}_{k+1} \boldsymbol{y}_{k} 
   &= \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_{k} 
   + \beta \boldsymbol{u} \boldsymbol{u}^T \boldsymbol{y}_{k}= \boldsymbol{s}_{k} \\
\Rightarrow \beta \boldsymbol{u} (\boldsymbol{u}^T \boldsymbol{y}_{k}) 
   &= \beta  (\boldsymbol{u}^T \boldsymbol{y}_{k}) \boldsymbol{u} 
    = \boldsymbol{s}_{k} - \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_{k} \\
\end{align}
\tag{29}\label{eq29}
$$  

由于$\boldsymbol{u}^T \boldsymbol{y}_{k}$是标量（所以上式括号内的东西能左右移动），$\beta \boldsymbol{u}^T \boldsymbol{y}_{k}$也是标量，所以：  

$$
\boldsymbol{u} = \frac{\boldsymbol{s}_{k} - \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_{k}}{\beta \boldsymbol{u}^T \boldsymbol{y}_{k}}
\tag{30}\label{eq30}
$$  

令$\gamma = 1 / (\beta \boldsymbol{u}^T \boldsymbol{y}_{k})$，把式$\eqref{eq30}$再代入式$\eqref{eq29}$，即：  

$$
\begin{align}
\boldsymbol{s}_{k} - \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_{k} 
       &= \beta  \boldsymbol{u} \boldsymbol{u}^T \boldsymbol{y}_{k}   \\
       &= \beta \gamma^2 (\boldsymbol{s}_{k} - \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_{k}) (\boldsymbol{s}_{k} - \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_{k})^T \boldsymbol{y}_{k}  \\
       &= [\beta \gamma^2 (\boldsymbol{s}_{k} - \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_{k})^T \boldsymbol{y}_{k} ](\boldsymbol{s}_{k} - \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_{k})
\end{align}
\tag{31}\label{eq31}
$$  

显然，若式$\eqref{eq31}$成立，则要求中括号内的标量值为1，即：  

$$
\beta \gamma^2 = \frac{1}{(\boldsymbol{s}_{k} - \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_{k})^T \boldsymbol{y}_{k}}
\tag{32}\label{eq32}
$$  

将式$\eqref{eq32}$继续回代到式$\eqref{eq28}$，利用一样的套路可得到：  

$$
\begin{align}
\hat{\boldsymbol{H}}_{k+1} 
    &= \hat{\boldsymbol{H}}_{k} + \beta  \boldsymbol{u} \boldsymbol{u}^T \\
    &= \hat{\boldsymbol{H}}_{k} + \beta \gamma^2 (\boldsymbol{s}_{k} - \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_{k}) (\boldsymbol{s}_{k} - \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_{k})^T\\
    &= \hat{\boldsymbol{H}}_{k} + \frac{(\boldsymbol{s}_{k} - \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_{k}) (\boldsymbol{s}_{k} - \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_{k})^T}{(\boldsymbol{s}_{k} - \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_{k})^T \boldsymbol{y}_{k}} 
\end{align}
\tag{33}\label{eq33}
$$  

式$\eqref{eq33}$即$\boldsymbol{H}$的SR1更新公式。设定$\hat{\boldsymbol{H}}_{0} = \boldsymbol{I}$，随着迭代进行，逼近会越来越精确。但不要高兴太早，SR1并不保证$\boldsymbol{H}_{k}$正定，这可能违反$\boldsymbol{H}_{k}^{-1} = \boldsymbol{B}_{k}$的规定（自动摊手）。所以人们开始寻找对称正定的秩-2更新公式。  

**小贴士：你可能会担心大规模问题中海塞逆矩阵需要消耗巨量内存，但实用中几乎所有的QN算法都存在更经济、高效的计算$\boldsymbol{H}$或$\boldsymbol{B}$与同维列向量的乘积的技巧，请参考Byrd和Nocedal等人于1994年发表的文章《Representation of quasi-Newton matrices and their use in limited memory methods》。**  
### 4.2 DFP算法
从现在开始，QN算法的命名就开始无厘头了，DFP是算法发明人Davidon、Fletcher和Powell三个人的名字首字母排列，是第一个“公认”的QN方法。其基本思想和SR1差不多，不同在于引入了正定秩-2更新。更新公式为：  

$$
\hat{\boldsymbol{H}}_{k+1} =  \hat{\boldsymbol{H}}_{k} + \beta  \boldsymbol{u} \boldsymbol{u}^T + \gamma \boldsymbol{v} \boldsymbol{v}^T
\tag{34}\label{eq34}
$$  

和SR1一样的套路，先两边同时乘以$\boldsymbol{y}_k$，得到：  

$$
\begin{align}
\hat{\boldsymbol{H}}_{k+1} \boldsymbol{y}_k 
    &= \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_k + \beta  \boldsymbol{u} \boldsymbol{u}^T \boldsymbol{y}_k + \gamma \boldsymbol{v} \boldsymbol{v}^T \boldsymbol{y}_k = \boldsymbol{s}_k  \\
\Rightarrow  \boldsymbol{s}_k - \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_k 
    &= \beta  \boldsymbol{u} \boldsymbol{u}^T \boldsymbol{y}_k + \gamma \boldsymbol{v} \boldsymbol{v}^T \boldsymbol{y}_k 
\end{align}
\tag{35}\label{eq35}
$$  

秩-2更新给予$\Delta \hat{\boldsymbol{H}}_{k}$更大的自由度，$\beta$、$\gamma$、$\boldsymbol{u}$和$\boldsymbol{v}$不再像SR1那样通过一顿骚操作求出唯一解，需要首先进行一些规定。  
DFP方法规定$\boldsymbol{u}=\boldsymbol{s}_k$，$\boldsymbol{v}=\boldsymbol{H}_k \boldsymbol{y}_k$，至于为什么这么规定，鬼才知道。不要问为什么，问就是为了推导方便，公式简单且收敛性好证明。：（   
但即便这样规定后，$\beta$和$\gamma$仍然求不出来，于是更骚的手段出现了，为了方便，干脆再令$\beta \boldsymbol{u} \boldsymbol{u}^T \boldsymbol{y}_k= \boldsymbol{s}_k$, $\gamma \boldsymbol{v} \boldsymbol{v}^T \boldsymbol{y}_k= - \boldsymbol{H}_k \boldsymbol{y}_k$。  
很明显，这完全是在凑。但“凑”这个词儿在数学上往往还有个高大上的同义词——“构造”。事实上，只需要凑出来的东西满足式$\eqref{eq27}$的QN条件，且能证明算法收敛，那么它就是一种有效的QN方法，**这也是最优化领域水文章的套路。**  
实际上，DFP公式还有另外一个推导方式，不过太复杂了，基本思想是求如下约束最小化问题：  

$$
\begin{align}
& \mathop{\min}\limits_{\hat{\boldsymbol{H}}} \Vert \boldsymbol{W}^{-T} (\hat{\boldsymbol{H}} - \hat{\boldsymbol{H}}_k) \boldsymbol{W}^{-1}  \Vert \\
& \mathrm{s.t. } \hat{\boldsymbol{H}} = \hat{\boldsymbol{H}}^T, \hat{\boldsymbol{H}} \boldsymbol{y}_k = \boldsymbol{s}_k\\
\end{align}
$$  

前文的规定刚好满足上式的一个特解，这里我们不执拗于数学家已经证明的结论，继续往后推导。首先针对向量$\boldsymbol{u}$，如下：  

$$
\begin{align}
 & \boldsymbol{u} = \boldsymbol{s}_k = \beta \boldsymbol{u} \boldsymbol{u}^T \boldsymbol{y}_k = (\beta \boldsymbol{u}^T \boldsymbol{y}_k) \boldsymbol{u}\\
 & \Rightarrow  \beta \boldsymbol{u}^T \boldsymbol{y}_k = 1 \\
 & \Rightarrow  \beta = \frac{1}{\boldsymbol{u}^T \boldsymbol{y}_k} = \frac{1}{\boldsymbol{s}_k^T \boldsymbol{y}_k}  
\end{align}
\tag{36}\label{eq36}
$$  

再分析对$\boldsymbol{v}$的假设，得到：  

$$
\begin{align}
 & \boldsymbol{v} = \boldsymbol{H}_k \boldsymbol{y}_k = - \gamma \boldsymbol{v} \boldsymbol{v}^T \boldsymbol{y}_k = - (\gamma \boldsymbol{v}^T \boldsymbol{y}_k) \boldsymbol{v} \\
 & \Rightarrow  \gamma \boldsymbol{v}^T \boldsymbol{y}_k = -1 \\
 & \Rightarrow  \gamma = - \frac{1}{\boldsymbol{v}^T \boldsymbol{y}_k} = - \frac{1}{\boldsymbol{y}_k^T \boldsymbol{H}_k^T \boldsymbol{y}_k} = - \frac{1}{\boldsymbol{y}_k^T \boldsymbol{H}_k \boldsymbol{y}_k} 
\end{align}
\tag{37}\label{eq37}
$$  

将凑出来的$\beta$、$\gamma$、$\boldsymbol{u}$和$\boldsymbol{v}$代入式$\eqref{eq34}$，得到：  

$$
\begin{align}
 \hat{\boldsymbol{H}}_{k+1} 
    &= \hat{\boldsymbol{H}}_{k} + \beta  \boldsymbol{u} \boldsymbol{u}^T + \gamma \boldsymbol{v} \boldsymbol{v}^T \\
    &= \hat{\boldsymbol{H}}_{k} + \frac{ \boldsymbol{s}_k \boldsymbol{s}_k^T}{\boldsymbol{s}_k^T \boldsymbol{y}_k }  
                                - \frac{ \boldsymbol{H}_k \boldsymbol{y}_k \boldsymbol{y}_k^T \boldsymbol{H}_k^T  }{\boldsymbol{y}_k^T \boldsymbol{H}_k \boldsymbol{y}_k } \\
    &= \hat{\boldsymbol{H}}_{k} + \frac{ \boldsymbol{s}_k \boldsymbol{s}_k^T}{\boldsymbol{s}_k^T \boldsymbol{y}_k }  
                                - \frac{ \boldsymbol{H}_k \boldsymbol{y}_k \boldsymbol{y}_k^T \boldsymbol{H}_k  }{\boldsymbol{y}_k^T \boldsymbol{H}_k \boldsymbol{y}_k } \\
\end{align}
\tag{38}\label{eq38}
$$  

DFP算法一经推出即受到关注，作为第一个秩-2的QN算法，它即保证了逼近阵的正定性，又具有极好的收敛性。但很快有人凑出来了更好的QN更新公式。  

**小贴士：你可能会怀疑会不会有秩-3的QN更新公式。但如果注意到$\boldsymbol{u}$和$\boldsymbol{v}$分别是关于$\boldsymbol{s}$和$\boldsymbol{y}$的表达式之后，我想你有可能会打消这个念头，除非上一次迭代除了模型修正量、梯度改变量还能提供其它的有用信息。但我不排除秩-3更新存在的可能性，祝你成功！（坏笑）**  

### 4.3 BFGS算法
BFGS的名字继续发扬无厘头风格，它的四个发明人是Broyden、Fletcher、Goldfarb和Shanno，这里我们从$\boldsymbol{B}$的更新进行推导，因为公式会简单很多，而且和DFP方法有一种神秘的联系（见第4.4节）。更新公式这样构造：  

$$
\hat{\boldsymbol{B}}_{k+1} =  \hat{\boldsymbol{B}}_{k} + \beta  \boldsymbol{u} \boldsymbol{u}^T + \gamma \boldsymbol{v} \boldsymbol{v}^T
\tag{39}\label{eq39}
$$  

规定$\boldsymbol{u} = \boldsymbol{y}_k$和$\boldsymbol{v} = \boldsymbol{B}_k \boldsymbol{s}_k$，经过和DFP算法一样的推导过程，我们可以得到：  


$$
 \hat{\boldsymbol{B}}_{k+1} 
    = \hat{\boldsymbol{B}}_{k} 
    + \frac{ \boldsymbol{y}_k \boldsymbol{y}_k^T}{\boldsymbol{y}_k^T \boldsymbol{s}_k }  
    - \frac{ \boldsymbol{B}_k \boldsymbol{s}_k \boldsymbol{s}_k^T \boldsymbol{B}_k  }{\boldsymbol{s}_k^T \boldsymbol{B}_k \boldsymbol{s}_k } \\
\tag{40}\label{eq40}
$$  

这个公式和DFP更新公式非常相似，为验证它和DFP不是一种算法，不妨对式$\eqref{eq40}$求逆得到$\hat{\boldsymbol{H}}_{k+1}$的BFGS更新公式。在此之前需要引入Shermann-Morison-Woodbury求逆定理：
对于可逆的$N \times N$矩阵$\boldsymbol{A}$和两个$N \times 1$向量$\boldsymbol{u}$和$\boldsymbol{v}$，下式成立：  

$$
 (\boldsymbol{A} + \boldsymbol{u} \boldsymbol{v}^T )^{-1} 
    = \boldsymbol{A}^{-1} 
    - \frac{\boldsymbol{A}^{-1} \boldsymbol{u} \boldsymbol{v}^T \boldsymbol{A}^{-1}}
           {1 + \boldsymbol{v}^T \boldsymbol{A}^{-1} \boldsymbol{u}}
\tag{41}\label{eq41}
$$  

对式$\eqref{eq40}$嵌套使用两次Shermann-Morison-Woodbury求逆公式，再经过一顿暴躁的演算之后，可以得到：  

$$
 \hat{\boldsymbol{H}}_{k+1} 
    = \hat{\boldsymbol{H}}_{k} 
    + (1+\frac{ \boldsymbol{y}_k^T \boldsymbol{H}_k \boldsymbol{y}_k} {\boldsymbol{s}_k^T \boldsymbol{y}_k } ) \frac{ \boldsymbol{s}_k \boldsymbol{s}_k^T}{\boldsymbol{y}_k^T \boldsymbol{s}_k }  
    - \frac{ \boldsymbol{s}_k \boldsymbol{y}_k^T \boldsymbol{H}_k + \boldsymbol{H}_k \boldsymbol{y}_k \boldsymbol{s}_k^T } {\boldsymbol{s}_k^T \boldsymbol{y}_k } \\
\tag{42}\label{eq42}
$$  

显然BFGS和DFP是完全不同的两个QN方法，实践证明BFGS比DFP相对更为高效和稳定。  

### 4.4 SR1、DFP和BFGS的关系以及Broyden族算法
在SR1算法中，如果$\hat{\boldsymbol{H}}_{k+1}$正定，对式$\eqref{eq33}$也应用Shermann-Morison-Woodbury求逆公式，可得：  

$$
 \hat{\boldsymbol{B}}_{k+1} 
    = \hat{\boldsymbol{B}}_{k} 
    + \frac{(\boldsymbol{y}_k - \boldsymbol{B}_k \boldsymbol{s}_k)(\boldsymbol{y}_k - \boldsymbol{B}_k \boldsymbol{s}_k)^T}
           {(\boldsymbol{y}_k - \boldsymbol{B}_k \boldsymbol{s}_k)^T \boldsymbol{s}_k}
\tag{43}\label{eq43}
$$  

将式$\eqref{eq33}$、式$\eqref{eq43}$、式$\eqref{eq38}$和式$\eqref{eq40}$写到一起会发现一个特别有趣的现象，数学家觉得这“美得很”（陕西话）：  

$$
\begin{align}
 \hat{\boldsymbol{H}}_{k+1}^{SR1} 
   &= \hat{\boldsymbol{H}}_{k} 
    + \frac{(\boldsymbol{s}_{k} - \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_{k}) (\boldsymbol{s}_{k} - \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_{k})^T}
           {(\boldsymbol{s}_{k} - \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_{k})^T \boldsymbol{y}_{k}}  \\
 \hat{\boldsymbol{B}}_{k+1}^{SR1} 
   &= \hat{\boldsymbol{B}}_{k} 
    + \frac{(\boldsymbol{y}_k - \boldsymbol{B}_k \boldsymbol{s}_k)(\boldsymbol{y}_k - \boldsymbol{B}_k \boldsymbol{s}_k)^T}
           {(\boldsymbol{y}_k - \boldsymbol{B}_k \boldsymbol{s}_k)^T \boldsymbol{s}_k} \\
 \hat{\boldsymbol{H}}_{k+1}^{DFP} 
    &= \hat{\boldsymbol{H}}_{k} + \frac{ \boldsymbol{s}_k \boldsymbol{s}_k^T}{\boldsymbol{s}_k^T \boldsymbol{y}_k }  
                                - \frac{ \boldsymbol{H}_k \boldsymbol{y}_k \boldsymbol{y}_k^T \boldsymbol{H}_k  }{\boldsymbol{y}_k^T \boldsymbol{H}_k \boldsymbol{y}_k } \\
 \hat{\boldsymbol{B}}_{k+1}^{BFGS} 
    &= \hat{\boldsymbol{B}}_{k} 
    + \frac{ \boldsymbol{y}_k \boldsymbol{y}_k^T}{\boldsymbol{y}_k^T \boldsymbol{s}_k }  
    - \frac{ \boldsymbol{B}_k \boldsymbol{s}_k \boldsymbol{s}_k^T \boldsymbol{B}_k  }{\boldsymbol{s}_k^T \boldsymbol{B}_k \boldsymbol{s}_k } \\
\end{align}
\tag{44}\label{eq44}
$$  

可见，SR1更新公式的$\boldsymbol{H}$和$\boldsymbol{B}$形式具有一样的结构，只是互换了$\boldsymbol{s}$和$\boldsymbol{y}$的位置并把$\boldsymbol{H}$替换为$\boldsymbol{B}$。
DFP和BFGS更新的两个公式互换规律类似。**因此，数学家称SR1算法自对偶，而DFP和BFGS算法互为对偶，但DFP和BFGS都不是自对偶。**  
由于DFP和BFGS都是两个凑出来的更新公式，数学上无法证明谁更好，于是有些贪心鬼把两个算法合二为一，取一正数$\phi \in [0,1]$，得到新的加权更新公式：  

$$
 \hat{\boldsymbol{B}}_{k+1}^{\phi} = \hat{\boldsymbol{B}}_{k+1}^{DFP} + (1-\phi_k)  \hat{\boldsymbol{B}}_{k+1}^{BFGS}
\tag{45}\label{eq45}
$$  

让$\phi_k$在$[0,1]$中遍历就可以得到一系列$\hat{\boldsymbol{B}}_{k+1}^{\phi_k}$，所有的$\hat{\boldsymbol{B}}_{k+1}^{\phi_k}$组成的集合被称为Broyden族（这个名字也很无厘头）。由于PDF和BFGS都是正定更新，Broyden族的所有更新矩阵也都保持正定。  
### 4.5 有限内存的BFGS（LBFGS）
第4.1～4.4节的所有算法虽然都有压缩内存的实现技巧，但只要问题规模变得很大，由于更新公式需保存之前所有次迭代的$\boldsymbol{u}$和$\boldsymbol{v}$向量（或者存储整个海塞矩阵），计算成本仍然变得不可接受。  
因此，一个在大规模最优化问题中应用最广泛的退化版BFGS更新公式在1980年被Nocedal提出，叫做有限内存BFGS算法（Limited-memory BFGS，简称LBFGS）。    
LBFGS基于一个Obvious的事实，影响第$k$次迭代海塞矩阵逼近效果的关键成分一定是最临近的前$m$次迭代信息（一般取$3 \leq m \leq 20$）。为了推导LBFGS，首先把BFGS更新公式$\eqref{eq39}$改写为：  

$$
\hat{\boldsymbol{B}}_{k+1}^{BFGS} =  \hat{\boldsymbol{B}}_{k}^{(0)} + \sum\limits_{i=0}^{k} [\boldsymbol{a}_i \boldsymbol{a}_i^T - \boldsymbol{b}_i \boldsymbol{b}_i^T]
\tag{46}\label{eq46}
$$  

一般情况下，和SR1一样，设定$\hat{\boldsymbol{B}}_{k}^{(0)} = \boldsymbol{I}$（**下一节会给出更好的方法**），
根据公式$\eqref{eq39}$可知，$\boldsymbol{a}_i$和$\boldsymbol{b}_i$可用以下公式求取并存储(注：符号$:=$表示编程中的赋值)：  

$$
\begin{align}
    & \boldsymbol{a}_i := \frac{\boldsymbol{y}_i}{\sqrt{\boldsymbol{y}_i^T \boldsymbol{s}_i}}   \\
    & \boldsymbol{b}_i := \hat{\boldsymbol{B}}_{i} \boldsymbol{s}_i 
                       = \hat{\boldsymbol{B}}_{k}^{(0)} \boldsymbol{s}_i 
                       +  \sum\limits_{i=0}^{k} [ (\boldsymbol{a}_i^T \boldsymbol{s}_i) \boldsymbol{a}_i + (\boldsymbol{b}_i^T \boldsymbol{s}_i) \boldsymbol{b}_i  ]  \\
    & \boldsymbol{b}_i := \frac{\hat{\boldsymbol{B}}_{i} \boldsymbol{s}_i} {\sqrt{\boldsymbol{s}_i^T \hat{\boldsymbol{B}}_{i} \boldsymbol{s}_i}}
                       = \frac{\boldsymbol{b}_i}{\sqrt{\boldsymbol{s}_i^T  \boldsymbol{b}_i}}  \\
\end{align}
\tag{47}\label{eq47}
$$  

所谓LBFGS，就是最多只保存最近$m$次迭代的模型修正量和梯度改变量(不足$m$次迭代傻子都知道怎么办)$\{\boldsymbol{s}_{k-1}, \boldsymbol{s}_{k-2}, \cdots, \boldsymbol{s}_{k-m}\}$
和$\{\boldsymbol{y}_{k-1}, \boldsymbol{y}_{k-2}, \cdots, \boldsymbol{y}_{k-m}\}$，并只用这些信息得到更新公式，它可利用公式$\eqref{eq46}$改写得到：  

$$
\hat{\boldsymbol{B}}_{k+1}^{LBFGS} =  \hat{\boldsymbol{B}}_{k+1}^{(0)} + \sum\limits_{i=p}^{k} [\boldsymbol{a}_i \boldsymbol{a}_i^T - \boldsymbol{b}_i \boldsymbol{b}_i^T], 
p = max(k-m+1, 0)
\tag{48}\label{eq48}
$$  

**小贴士：本文的推导方式和Nocedal不同，经典LBFGS是直接逼近海塞逆矩阵$\boldsymbol{H}$的，并且这位老先生还推导出了一个牛逼克拉斯的双循环递归算法，可直接高效计算$\boldsymbol{H}$和梯度向量的乘积，求下降方向的速度几乎和NLCG算法差不多。这活干得......让后人绝了创新的念头。**  
### 4.6 自调节(self-scaling)的BFGS和LBFGS
第4.5节的BFGS更新和LBFGS更新都从单位阵$\boldsymbol{I}$开始，在迭代早期的逼近效果较差，从而导致收敛较慢或不稳定。  
因此Shanno（香农！没错，又是他！）和Phua于1975年提出一种改进方法，
其思想是每次迭代都从一个经过标量缩放的$\sigma_k \boldsymbol{I}$矩阵重新建立逼近。原论文是从Broyden族的$\boldsymbol{H}$更新推导的，看得我一脸懵逼，这里我用自己的推导方式，版权所有，爱看不看。：）  
无论是BFGS还是LBFGS，海塞更新可统一写为：  

$$
\hat{\boldsymbol{B}}_{k+1} =  \hat{\boldsymbol{B}}_{k} + \Delta \hat{\boldsymbol{B}}_{k}
\tag{49}\label{eq49}
$$  

在第0次迭代时，所有QN算法都是梯度法，即$\hat{\boldsymbol{B}}_{(0)} = \boldsymbol{I}$，经过不精确线搜索后得到向量$\boldsymbol{s}_0$和$\boldsymbol{y}_0$。
所以常规BFGS中$\hat{\boldsymbol{B}}_{1}=\boldsymbol{I} + \Delta \hat{\boldsymbol{B}}_{0}$，$\Delta \hat{\boldsymbol{B}}_{0}$的构建只考虑了拟牛顿条件$\boldsymbol{B}_1 \boldsymbol{s}_0 = \boldsymbol{y}_0$而不保证高精度逼近，
如果把更新公式改写为$\hat{\boldsymbol{B}}_{1}=\sigma_0 \boldsymbol{I} + \Delta \hat{\boldsymbol{B}}_{0}$，且$\sigma_0 \boldsymbol{I} \approx \hat{\boldsymbol{B}}_{1}$，那就能极大提升$\hat{\boldsymbol{B}}_{1}$的逼近精度，以此类推。  
根据拟牛顿条件，在第$k$次迭代有：  

$$
\begin{align}
& \hat{\boldsymbol{B}}_{k} \boldsymbol{s}_{k-1} = \boldsymbol{y}_{k-1}   \\
& \Rightarrow \sigma_k \boldsymbol{I} \boldsymbol{s}_{k-1} \approx \boldsymbol{y}_{k-1} \\
& \Rightarrow \boldsymbol{y}_{k-1}^T \sigma_k  \boldsymbol{I} \boldsymbol{s}_{k-1} =  \sigma_k \boldsymbol{y}_{k-1}^T \boldsymbol{s}_{k-1} \approx \boldsymbol{y}_{k-1}^T \boldsymbol{y}_{k-1} \\
& \Rightarrow \sigma_k \approx \frac{\boldsymbol{y}_{k-1}^T \boldsymbol{y}_{k-1}} {\boldsymbol{y}_{k-1}^T \boldsymbol{s}_{k-1}} \\
\end{align}
\tag{50}\label{eq50}
$$  

对式$\eqref{eq46}$和$\eqref{eq48}$在每次迭代中设定$\hat{\boldsymbol{B}}_{k}^{(0)} = \sigma_k \boldsymbol{I}$后，由于$\sigma_k$是标量，根据DFP、BFGS算法中凑出更新公式的过程（见式$\eqref{eq34}$～$\eqref{eq38}$），所有$\boldsymbol{a}_{k}$和$\boldsymbol{b}_{k}$的形式仍可保持不变。  
新算法保证每次迭代的更新公式都自适应的从一个逼近当前海塞阵的标量阵出发，因此也被称为自调节（self-scaling）的QN更新。  

**小贴士：经典LBFGS实现是对$\boldsymbol{H}$更新的，此时自调节标量为$1 / \sigma_k$。但如果所解问题存在更好的逼近方案，你可以每次迭代计算一个主对角元素大于0的对角阵，用来代替$\hat{\boldsymbol{B}}_{k}^{(0)}$或$\hat{\boldsymbol{H}}_{k}^{(0)}$。**  

## 5. 非线性共轭梯度法（Non-linear Conjugate Gradient Method，NLCG）
在前几节中，为了适应大规模问题，我们一步步从牛顿法退化到了QN方法，最终构建了只需要保存$m$个向量对的LBFGS方法。但还存在一个极限方式，只利用当前梯度和上一次模型修正量以尽量改善梯度法的收敛效率。  
在[《解线性方程组的CG和PCG方法》](optimal_cg_and_nlcg.md)的第2.1节中我们论述了在线性二次型中如何以下降方向$\boldsymbol{d}_{k-1}$和梯度向量$\boldsymbol{g}_k$构建靠近牛顿步的$\boldsymbol{d}_k$，并且证明了线性CG是一种共轭方向法 。
也就是说CG中任意不等于$k$的第$j$次迭代，一定满足$\boldsymbol{d}_k^T \boldsymbol{B} \boldsymbol{d}_j = 0$。**注意：这里的海塞矩阵没有下标，因为线性二次型的海塞矩阵不变**。  
而NLCG的实质就是把CG算法“强行”推广到非线性最小化问题。我们首先复习一下线性CG的步长和下降方向（此时残差向量$\boldsymbol{r}_k= - \boldsymbol g_k$），如下：  

$$
\begin{align}
& \alpha_k = \frac { - \boldsymbol g_k^T \boldsymbol d_k} {\boldsymbol d_k^T \boldsymbol B_k \boldsymbol d_k} 
           = \frac { \boldsymbol r_k^T \boldsymbol d_k} {\boldsymbol d_k^T \boldsymbol B \boldsymbol d_k} \\
& \boldsymbol d_k = - \boldsymbol g_k + \beta_k \boldsymbol d_{k-1} = \boldsymbol r_k + \beta_k \boldsymbol d_{k-1}  \\
& \beta_k = \frac { \boldsymbol g_k^T \boldsymbol g_k} {\boldsymbol g_{k-1}^T \boldsymbol g_{k-1}} 
          = \frac { \boldsymbol r_k^T \boldsymbol r_k} {\boldsymbol r_{k-1}^T \boldsymbol r_{k-1}}  \\
\end{align}
\tag{51}\label{eq51}
$$  

最早的NLCG方法可能是由Fletcher和Reeve在1964年提出的FR-NLCG，它的形式基本和式$\eqref{eq51}$一致，如下：  

$$
\begin{align}
& \alpha_k^{(0)} \approx \frac { - \boldsymbol g_k^T \boldsymbol d_k} {\boldsymbol d_k^T \boldsymbol B_k \boldsymbol d_k} \\
& \boldsymbol d_k = - \boldsymbol g_k + \beta_k \boldsymbol d_{k-1} \\
& \beta_k^{FR} = \frac { \boldsymbol g_k^T \boldsymbol g_k} {\boldsymbol g_{k-1}^T \boldsymbol g_{k-1}} \\
\end{align}
\tag{52}\label{eq52}
$$  

其中，$\alpha_k^{(0)}$是不精确线搜索的初始步长，需对$\boldsymbol d_k^T \boldsymbol B_k \boldsymbol d_k$进行某种快速近似（因为我们不想求海塞矩阵）。但这种暴力的推广是否还能保证迭代的收敛性呢？我们需要进行一些细致的分析，以对比CG和NLCG的不同。  
首先，讨论第$k$次精确线搜索迭代产生的梯度$\boldsymbol g_{k+1}$。利用二阶泰勒展开式$\eqref{eq11}$并对$\boldsymbol m_{k+1}$求导，可得：  

$$
\begin{align}
& \boldsymbol g_{k+1} = \frac{\partial \Phi(\boldsymbol m_{k+1}) }{\partial \boldsymbol m_{k+1}} = \boldsymbol g_{k} + \alpha_k \boldsymbol B_{k} \boldsymbol d_{k} \\
& \Rightarrow \boldsymbol{g}_{k+1}^T \boldsymbol{d}_{k} 
    = \boldsymbol{g}_{k}^T \boldsymbol{d}_{k} + \alpha_k \boldsymbol d_{k}^T \boldsymbol B_{k} \boldsymbol d_{k}
    = \frac{\partial \Phi(\boldsymbol m_{k+1}) }{\partial \alpha_k} = 0           \\
\end{align}
\tag{53}\label{eq53}
$$  

**这表明在精确线搜索条件下NLCG和CG同样都保证$\boldsymbol{g}_{k+1}^T \boldsymbol{d}_{k} = 0$。**
继续考察$\boldsymbol{g}_{k+1}^T \boldsymbol{d}_{k+1}$如下：  

$$
\boldsymbol{g}_{k+1}^T \boldsymbol{d}_{k+1} = - \boldsymbol{g}_{k+1}^T \boldsymbol{g}_{k+1} + \beta_{k+1}^{FR} \boldsymbol{g}_{k+1}^T \boldsymbol{d}_{k} \\
\tag{54}\label{eq54}
$$  

若后续迭代步长均由不精确线搜索得到，则不能保证式$\eqref{eq54}$右边第二项为0，有可能造成破坏$\boldsymbol{g}_{k+1}^T \boldsymbol{d}_{k+1} < 0$的目标函数下降必要条件。幸运的是，数学家们已经证明了只要每次迭代的不精确线搜索满足Wolfe条件，下降必要条件就可以成立。  
**然鹅，当我们考察NLCG的共轭性时出了问题。**
共轭方向法至少需要保证$\boldsymbol{d}_{k}^T \boldsymbol{B}_{k} \boldsymbol{d}_{k-1} = 0$，把左边展开后得到：  

$$
\boldsymbol{d}_{k}^T \boldsymbol{B}_{k} \boldsymbol{d}_{k-1} = - \boldsymbol{g}_{k}^T \boldsymbol{B}_{k} \boldsymbol{d}_{k-1} + \beta_k \boldsymbol{d}_{k-1}^T \boldsymbol{B}_{k} \boldsymbol{d}_{k-1}
\tag{55}\label{eq55}
$$  

显然，把$\beta_k^{FR}$代入$\eqref{eq55}$中并不保证前后两次迭代关于海塞矩阵共轭。除非$\beta_k^{Daniel} = \frac{\boldsymbol{g}_{k}^T \boldsymbol{B}_{k} \boldsymbol{d}_{k-1} }{\boldsymbol{d}_{k-1}^T \boldsymbol{B}_{k} \boldsymbol{d}_{k-1}}$，
这个$\beta$值由Daniel在1967年提出，但它需要计算海塞矩阵，所以并不适用于大规模问题（你都能算海塞了，还搞啥NLCG？）。**退一万步说，即使保证$\eqref{eq55}$等于0，由于非线性问题的海塞阵随迭代变化，NLCG的下降方向并不满足共轭方向法中“不同迭代的下降方向两两关于海塞共轭”的要求（和线性CG不同）。**  
$\beta_k^{FR}$在实用中收敛较慢，$\beta_k^{Daniel}$计算又太耗时，因此很多学者提出了不同的NLCG变种，如下：  

$$
\begin{align}
 \beta_k^{PRP} &= \frac{\boldsymbol{g}_{k}^T (\boldsymbol{g}_{k} - \boldsymbol{g}_{k-1} )}{\boldsymbol{g}_{k-1}^T \boldsymbol{g}_{k-1}} \\
 \beta_k^{SW} &= \frac{\boldsymbol{g}_{k}^T (\boldsymbol{g}_{k} - \boldsymbol{g}_{k-1} )}{\boldsymbol{d}_{k-1}^T (\boldsymbol{g}_{k} - \boldsymbol{g}_{k-1} )} \\
 \beta_k^{Dixon} &= \frac{\boldsymbol{g}_{k}^T \boldsymbol{g}_{k}}{\boldsymbol{d}_{k-1}^T  \boldsymbol{g}_{k-1} } \\
 \beta_k^{DY} &= \frac{\boldsymbol{g}_{k}^T \boldsymbol{g}_{k}}{\boldsymbol{d}_{k-1}^T (\boldsymbol{g}_{k} - \boldsymbol{g}_{k-1} ) } \\
\end{align}
\tag{56}\label{eq56}
$$  

其中，PRP在地球物理中较为常用，DY则是由我国学者袁亚湘提出的。但这些值代入公式$\eqref{eq55}$里都不能保证共轭性（其中$\beta_k^{SW}$在满足拟牛顿条件时局部共轭，有兴趣请自己推导），它们不可能从本质上解决问题。

**小贴士：线性CG是一种共轭方向法，但NLCG的诸多变种均不保证不同迭代的下降方向互相关于某个固定$\boldsymbol{Q}$矩阵共轭，这就是NLCG失去二次截止性的本质原因。CG是一种共轭方向法，但严谨一点来说NLCG并不是一种共轭方向法。**  

无论如何，NLCG是一种比梯度法好得多的算法，如果你觉得LBFGS的$m$个向量对还是太占内存，那NLCG可能是你最后的合理选择。与第4.6节介绍的情况一样，为尽量提高收敛效率，你也可以引入类似self-scaling的方法对常规NLCG进行改进，这种技术在NLCG中被称为“预条件”，下降方向变为：  

$$
 \boldsymbol{d}_k = - \hat{\boldsymbol{H}}_k \boldsymbol{g}_{k} + \beta_k \boldsymbol{d}_{k-1}
\tag{57}\label{eq57}
$$  

其中，$\hat{\boldsymbol{H}}_k$是对海塞逆矩阵的某个高效近似。当然，你也可以用LBFGS来实现预条件，但这种NLCG的实际收敛效率可能和LBFGS相当，但却失去了QN方法线搜索初始步长可设定为1的良好特性。  

## 总结
我们介绍了几乎所有的经典非线性凸优化方法，你可以看到每个方法都是从泰勒公式开始的，所以说泰勒公式很牛逼。  
如果目标函数和初始点一样，那么以上所有方法理论上会找到同样的局部极小点（对于有些问题，可能不稳定的算法会迭代失败）。**也就是说，如果不考虑计算时间，对于同样的目标函数，无约束反演的结果不由反演方法决定。**  
有时候你会在论文中看到某个算法“具有全局收敛性”，这并不是说它能收敛到全局极小，**而是它对于任意初始点都能产生一个收敛的迭代点列，和全局极小完全是两码事（手动摊手）！！！**  
一般情况下，经典凸优化算法按照收敛效率（指迭代多少次收敛）由慢到快的排序是：梯度法、NLCG、LBFGS、BFGS、GN、牛顿法。按内存需求和单次迭代计算时间的排序则刚好相反。对于不同的实际问题，总收敛时间不一定谁快谁慢。但综合衡量一般NLCG和LBFGS会是最省时、省内存的选择。  
曾经不知道哪位审稿人非要教育我，说QN方法是共轭方向法。我搜索了一下，好像这个提法来自于《最优化导论（An introduction to optimization）》。证明如下：  

$$
\begin{align}
 \boldsymbol{d}_{k}^T \boldsymbol{B}_{k} \boldsymbol{d}_{k-1} 
    &= - \boldsymbol{g}_{k}^T \hat{\boldsymbol{H}}_{k} \boldsymbol{B}_{k} \boldsymbol{d}_{k-1} \\
    &= - \boldsymbol{g}_{k}^T \hat{\boldsymbol{H}}_{k} \boldsymbol{B}_{k} \boldsymbol{s}_{k-1} / \alpha_{k-1} \\
    &= - \boldsymbol{g}_{k}^T \hat{\boldsymbol{H}}_{k} \boldsymbol{y}_{k-1} / \alpha_{k-1} \\
    &= - \boldsymbol{g}_{k}^T \boldsymbol{s}_{k-1} / \alpha_{k-1} \\
    &= - \boldsymbol{g}_{k}^T \boldsymbol{d}_{k-1} = 0 \\
\end{align}
$$  

但这只表明QN方法因为要满足式$\eqref{eq27}$的拟牛顿条件，所构建的相邻两次迭代下降方向关于当前海塞矩阵共轭（比NLCG更共轭，蛤蛤！）。
原文是用线性二次型问题来推导，规定了$\boldsymbol{B}_{k} \equiv \boldsymbol{Q}$，进而可推出对任意不等于$k$的迭代$j$，都存在$\boldsymbol{d}_{k}^T \boldsymbol{Q} \boldsymbol{d}_{j} = 0$，并声称“QN本质上是共轭方向法”。  
但很遗憾(日语好像是“残念Da”)，对于非线性问题这一点并不成立。  
**如果你坚持QN是一种共轭方向法，那你都对。：）毕竟在第5节我们已经证明了大部分NLCG都不共轭，那么就只好让QN来共轭了。共轭万岁，Oh Yeah！！！**   

最后，我们没有给出各个算法的伪代码或流程图，因为那很Obvious（狗头保命）。



