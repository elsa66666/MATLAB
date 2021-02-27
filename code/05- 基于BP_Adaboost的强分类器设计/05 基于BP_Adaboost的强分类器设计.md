



# 05 基于BP_Adaboost的强分类/预测器设计

## 1 案例背景：公司财务预警系统（分类器）

### 1.1 输入输出

输入数据：10维，分别代表公司的十个指标：

- 成分费用利润率
- 资产营运能力
- 公司总资产
- 总资产增长率
- 流动比率
- 营业现金流量
- 审计意见类型
- 每股收益
- 存货周转率
- 资产负债率

输出：1维，表示公司的财务情况：

- 为1表示财务状况良好
- 为-1表示财务状况出现问题

### 1.2 数据集中数据分配

1000组数据作为输入数据，350组数据作为测试数据。

根据数据维数，采用的BP神经网络结构为10-6-1，共训练生成10个 BP神经网络弱分类器，最后用10个弱分类器组成强分类器对公司财务状况进行分类。

### 1.3 注：测试器使用的案例是预测函数 $y = x_1^2 + x_2 ^2$ 的值



## 2 Adaboost模型简介

算法思想：**合并多个“弱”分类器的输出以产生有效分类**

首先给出弱学习算法和样本空间(𝑋,𝑌)，从样本空间中找出𝑚组训练数据，每组训练数据的权重都是$\frac{1}{m}$。

然后用弱学习算法迭代运算𝑇次，每次运算后都按照分类结果更新训练数据权重分布。

- 分类器：对于  **分类失败的训练个体**  赋予较大权重，下次迭代运算时更加关注这些训练个体。

- 测试器：对于  **预测误差超过阈值个体**  赋予较大权重，下次迭代运算时更加关注这些训练个体。

弱分类器通过反复迭代得到一个分类函数序列，$f_1,f_2, ···f_T$

每个分类函数赋予一个权重，分类结果越好的函数，其对应权重越大。

𝑇次迭代之后，最终强分类函数𝐹由弱分类函数加权得到。



## 3 BP_Adaboost分类器模型

BP_Adaboost模型即BP神经网络作为弱分类器，反复训练BP神经网络预测样本输出，通过Adaboost算法得到多个BP神经网络弱分类器组成的强分类器。

![](https://img2018.cnblogs.com/i-beta/1752446/202002/1752446-20200220094843893-896865310.png)

### 3.1 数据初始化

- 分类器：从样本空间中随机选择𝑚组训练数据，初始化测试数据的分布权值：${D_t}(i) = \frac{1}{m}$

因为训练数据和测试数据已经分好了，所以只需要初始化权重即可

```matlab
%% 下载数据
load data input_train output_train input_test output_test

%% 权重初始化
[mm,nn]=size(input_train);
D(1,:)=ones(1,nn)/nn;
```

- 测试器：

先打乱样本顺序，随机分出1900个测试数据和100个训练数据，再初始化权重。

```matlab
%% 下载数据
load data1 input output

%打乱样本的顺序
k=rand(1,2000);
[m,n]=sort(k);

%训练样本
input_train=input(n(1:1900),:)';
output_train=output(n(1:1900),:)';

%测试样本
input_test=input(n(1901:2000),:)';
output_test=output(n(1901:2000),:)';
```

初始化权重：

```matlab
%权重初始化
[mm,nn]=size(input_train);
D(1,:)=ones(1,nn)/nn;
```

### 3.2 网络初始化

- 分类器：根据样本输入输出维数确定神经网络结构，初始化BP神经网络权值和阈值。

```matlab
%% 弱分类器分类
K=10;
for i=1:K
    
    %训练样本归一化
    [inputn,inputps]=mapminmax(input_train);
    [outputn,outputps]=mapminmax(output_train);
    error(i)=0;
    
    %BP神经网络构建
    net=newff(inputn,outputn,6);
    net.trainParam.epochs=5;
    net.trainParam.lr=0.1;
    net.trainParam.goal=0.00004;
    
    %BP神经网络训练
    net=train(net,inputn,outputn);
    
    %训练数据预测
    an1=sim(net,inputn);
    test_simu1(i,:)=mapminmax('reverse',an1,outputps);
    
    %测试数据预测
    inputn_test =mapminmax('apply',input_test,inputps);
    an=sim(net,inputn_test);
    test_simu(i,:)=mapminmax('reverse',an,outputps);
    
    %统计输出效果
    kk1=find(test_simu1(i,:)>0);
    kk2=find(test_simu1(i,:)<0);
```

- 测试器：

```matlab
%训练样本归一化
[inputn,inputps]=mapminmax(input_train);
[outputn,outputps]=mapminmax(output_train);

K=10;
for i=1:K
    
    %弱预测器训练
    net=newff(inputn,outputn,5);
    net.trainParam.epochs=20;
    net.trainParam.lr=0.1;
    net=train(net,inputn,outputn);
    
    %弱预测器预测
    an1=sim(net,inputn);
    BPoutput=mapminmax('reverse',an1,outputps);
    
    %预测误差
    erroryc(i,:)=output_train-BPoutput;
    
    %测试数据预测
    inputn1=mapminmax('apply',input_test,inputps);
    an2=sim(net,inputn1);
    test_simu(i,:)=mapminmax('reverse',an2,outputps);
```



### 3.3 弱分类器预测

训练第𝑡个弱分类器时，用训练数据训练BP神经网络并且预测训练数据输出，得到预测序列𝑔(𝑡)的预测误差$e_t$.

$$
{e_t} = \sum\limits_i {{D_t}(i)} \;\;\;i = 1,2, \ldots ,m(g(t) \ne y)
$$

- 分类器

```matlab
   aa(kk1)=1;
    aa(kk2)=-1;
    
    %统计错误样本数
    for j=1:nn
        if aa(j)~=output_train(j);
            error(i)=error(i)+D(i,j);
        end
    end
```

- 测试器：没有这一步

### 2.3 计算预测序列权重

- 分类器：

根据预测序列𝑔(𝑡)的预测误差$e_t$计算序列的权重$a_t$，权重计算公式为
$$
{a_t} = \frac{1}{2}\ln \left( {\frac{{1 - {e_t}}}{{{e_t}}}} \right)
$$

```matlab
    %弱分类器i权重
    at(i)=0.5*log((1-error(i))/error(i));
```

- 测试器：

$$
a_t = \frac{0.5}{e^{|e_t|}}
$$

```matlab
    %计算弱预测器i权重
    at(i)=0.5/exp(abs(Error(i)));
```



### 3.4 测试数据权重调整

根据预测序列权重$a_t$挑中下一轮训练样本的权重，调整公式为
$$
{D_{t + 1}}(i) = \frac{{{D_t}(i)}}{{{B_t}}} \cdot {e^{ - {a_t}{y_i}{g_t}({x_i})}}\;\;\;i = 1,2, \ldots ,m
$$
其中，$B_t$为归一化因子，目的是在权重比例不变的情况下使分布权值和为1。

- 分类器：

```matlab
    %更新D值
    for j=1:nn
        D(i+1,j)=D(i,j)*exp(-at(i)*aa(j)*test_simu1(i,j));
    end
    %D值归一化
    Dsum=sum(D(i+1,:));
    D(i+1,:)=D(i+1,:)/Dsum;    
end
```

- 测试器：

```matlab
    %调整D值
    Error(i)=0;
    for j=1:nn
        if abs(erroryc(i,j))>0.2  %较大误差
            Error(i)=Error(i)+D(i,j);
            D(i+1,j)=D(i,j)*1.1;
        else
            D(i+1,j)=D(i,j);
        end
    end
    
    %D值归一化
    D(i+1,:)=D(i+1,:)/sum(D(i+1,:));
end
```

### 3.5 强分类函数

- 分类器：

训练𝑇轮后得到𝑇组弱分类函数$f(g_{t},a_{t})$，由𝑇组弱分类函数组合得到了强分类函数
$$
h(x) = {\rm{sign}}[\sum\limits_{t = 1}^T {{a_t} \cdot f({g_t},{a_t})} ]
$$

```matlab
%% 强分类器分类结果
output=sign(at*test_simu);
```

- 测试器：直接总和➗个数

```matlab
at=at/sum(at);
```



## 4 分类结果统计

- 分类器：

统计强分类器每类分类（共有 **良好kkk1/出现问题kkk2** 两类）错误个数

```matlab
kkk1=0;
kkk2=0;
for j=1:350
    if output(j)==1
        if output(j)~=output_test(j)
            kkk1=kkk1+1;
        end
    end
    if output(j)==-1
        if output(j)~=output_test(j)
            kkk2=kkk2+1;
        end
    end
end
```



- 测试器：

```matlab
%强分离器效果
output=at*test_simu;
error=output_test-output;
plot(abs(error),'-*')
hold on
for i=1:8
error1(i,:)=test_simu(i,:)-output;
end
plot(mean(abs(error1)),'-or')

title('强预测器预测误差绝对值','fontsize',12)
xlabel('预测样本','fontsize',12)
ylabel('误差绝对值','fontsize',12)
legend('强预测器预测','弱预测器预测')
```

<img src="/Users/xixixiao/Desktop/study/MatlabLearning/code/05- 基于BP_Adaboost的强分类器设计/fore_output.png" alt="fore_output" style="zoom:48%;" />

