---
title: "python numpy pandas"
date: 2019-08-08 17:05:12
tags: [Python,numpy]
categories: [Python]
---

[TOC]

## python的numpy库

numpy（Numerical Python）提供了python对多维数组对象的支持：ndarray，具有矢量运算能力，快速、节省空间。numpy支持高级大量的维度数组与矩阵运算，此外也针对数组运算提供大量的数学函数库。

[NumPy](http://www.scipy.org/NumPy) 是非常有名的Python 科学计算工具包，其中 包含了大量有用的思想，比如数组对象（用来表示向量、矩阵、图像等）以及线性 代数函数。数组对象可以帮助你实现数组中重要的操作，比如**矩阵乘积**、**转置**、 **解方程系统**、**向量乘积**和**归一化**，这为**图像变形**、**对变化进行建模**、**图像分类**、 **图像聚类**等提供了基础。

- 图像灰度处理

![](http://pic.fenghong.tech/1.jpg?raw=true)

![](http://pic.fenghong.tech/2.jpg?raw=true)

```
from PIL import Image
import numpy as np

a = np.array(Image.open('1.jpg').convert('L'))
print(a.shape, a.dtype)
# b = (100/255)*a +150
b = 255 * (a/255)**2 # 图像的灰度很重

im = Image.fromarray(b.astype('uint8'))
im.save('2.jpg')
```

- 图像的手绘处理

![](http://pic.fenghong.tech/varto.jpg?raw=true)

![](http://pic.fenghong.tech/transfor.jpg?raw=true)

```
# # 相片变为手绘效果图片
# # 黑白灰色 & 便捷线条较重 & 相同或相近色彩趋于白色 & 略有光源效果
# 梯度和虚拟深度值对图像进行重构


from PIL import Image
import numpy as np

name = input("please input your jpg Path: ")

a = np.asarray(Image.open(name).convert('L')).astype('float')    # convert('L')灰度图片

depth = 10.  # (0-100)
grad = np.gradient(a)  # 取图像灰度的梯度值
grad_x, grad_y = grad  # 分别取横纵图像梯度值
grad_x = grad_x * depth / 100.
grad_y = grad_y * depth / 100.
A = np.sqrt(grad_x ** 2 + grad_y ** 2 + 1.)
uni_x = grad_x / A
uni_y = grad_y / A
uni_z = 1. / A

vec_el = np.pi / 2.2  # 光源的俯视角度，弧度值
vec_az = np.pi / 4.  # 光源的方位角度，弧度值
dx = np.cos(vec_el) * np.cos(vec_az)  # 光源对x 轴的影响
dy = np.cos(vec_el) * np.sin(vec_az)  # 光源对y 轴的影响
dz = np.sin(vec_el)  # 光源对z 轴的影响

b = 255 * (dx * uni_x + dy * uni_y + dz * uni_z)  # 光源归一化
b = b.clip(0, 255)  # 为避免数据越界，将生成的灰度值裁剪至0-255区间

im = Image.fromarray(b.astype('uint8'))  # 重构图像
im.save("images/new_name.jpg")
```

如何批量生成素描文件,这里写了一个for循环即可解决问题.

```
from PIL import Image
import numpy as np
import os

# 传入的一个是文件全路径,一个是获取文件名称
def generate(name, txt):
    a = np.asarray(Image.open(name).convert('L')).astype('float')  # convert('L')灰度图片
    depth = 10.  # (0-100)
    grad = np.gradient(a)  # 取图像灰度的梯度值
    grad_x, grad_y = grad  # 分别取横纵图像梯度值
    grad_x = grad_x * depth / 100.
    grad_y = grad_y * depth / 100.
    ax = np.sqrt(grad_x ** 2 + grad_y ** 2 + 1.)
    uni_x = grad_x / ax
    uni_y = grad_y / ax
    uni_z = 1. / ax
    vec_el = np.pi / 2.2  # 光源的俯视角度，弧度值
    vec_az = np.pi / 4.  # 光源的方位角度，弧度值
    dx = np.cos(vec_el) * np.cos(vec_az)  # 光源对x 轴的影响
    dy = np.cos(vec_el) * np.sin(vec_az)  # 光源对y 轴的影响
    dz = np.sin(vec_el)  # 光源对z 轴的影响
    b = 255 * (dx * uni_x + dy * uni_y + dz * uni_z)  # 光源归一化
    b = b.clip(0, 255)  # 为避免数据越界，将生成的灰度值裁剪至0-255区间
    im = Image.fromarray(b.astype('uint8'))  # 重构图像
    im.save("images/" + txt[-1])

# 获取当前目录下的所有以jpg结尾的文件
def get_files(path='D:\\text', rule=".jpg"):
    allfile = []
    for fpathe, dirs, fs in os.walk(path):  # os.walk是获取所有的目录
        for f in fs:
            filename = os.path.join(fpathe, f)
            if filename.endswith(rule):  # 判断是否是"xxx"结尾
                allfile.append(filename)
    return allfile

# 入口函数
if __name__ == "__main__":
    b = get_files(r"D:\text")
    for i in b:
        txt = i.split("\\")
        generate(i, txt)
```



## python的matplotlib库

> 简单来说，Matplotlib 是 Python 的一个绘图库。它包含了大量的工具，你可以使用这些工具创建各种图形，包括简单的散点图，正弦曲线，甚至是三维图形。Python 科学计算社区经常使用它完成数据可视化的工作。

很容易画出一张图

![yuxuan](http://pic.fenghong.tech/yuxuan.png?raw=true)

```
import numpy as np
import matplotlib.pyplot as plt

def f(t):
    return np.exp(-t) * np.cos(2*np.pi*t)

a = np.arange(0.0, 5.0, 0.02)
plt.subplot(211)
plt.plot(a, f(a))
plt.subplot(212)
plt.plot(a, np.cos(2*np.pi*a), 'r--')
plt.savefig('images/yuxuan', dpi=600)
cn_matp
```

- .plot函数

```
plt.plot(x, y, format_string, **kwargs): x为x轴数据，可为列表或数组；y同理；format_string 为控制曲线的格式字符串， **kwargs 第二组或更多的（x, y,format_string） 

format_string: 由 颜色字符、风格字符和标记字符组成。 
	颜色字符：‘b’蓝色 ；‘#008000’RGB某颜色；‘0.8’灰度值字符串 
	风格字符：‘-’实线；‘–’破折线； ‘-.’点划线； ‘：’虚线 ； ‘’‘’无线条 
	标记字符：‘.’点标记 ‘o’ 实心圈 ‘v’倒三角 ‘^’上三角

rcParams 的属性
	font.family 用于显示字体的名字 'SimHei'\ 'Kaiti'
	font.style  字体风格，'normal'或者 'italic'
	font.size    字体大小， 'large' 或者'x-small'
	fontproperties fontsize
	

```

- pyplot的文本显示函数说明:

```
plt.xlabel()：对x轴增加文本标签
plt.ylabel()：同理
plt.text(): 在任意位置增加文本
plt.title(2,1,r'$\mu=100$') 对图形整体增加文本标签,2,1为横轴坐标轴。
plt.annotate() 在图形中加入注解
plt. annotate(s, xy = arrow_crd, xytext = text_crd, arrowprops = dict)	
```

在图形中加中文注解，正式的余弦函数如下:

![cn_matp](http://pic.fenghong.tech/cn_matp.png?raw=true)

![cn_matp](http://pic.fenghong.tech/cn_matp_arrow.png?raw=true)

```
import numpy as np
import matplotlib.pyplot as plt

# import matplotlib

# matplotlib.rcParams['font.family']='SimHei'
# matplotlib.rcParams['font.size']=20

a = np.arange(0.0, 5.0, 0.02)
plt.plot(a, np.cos(2*np.pi*a), 'r--')
plt.xlabel('横轴: 时间', fontproperties='SimHei', fontsize=15, color='green')
plt.ylabel('纵轴: 振幅', fontproperties='SimHei', fontsize=15)

plt.title(r'余弦波实例$y=cos(2\pi x)$', fontproperties='SimHei', fontsize=25)
# plt.text(2, 1, r'$\mu=100$',fontsize=15)    # 
plt.annotate(r'$\mu=100$', xy=(2, 1), xytext=(3, 1.5),
             arrowprops=dict(facecolor='black', shrink=0.1, width=2))
plt.axis([-1, 6, -2, 2])
plt.grid(True)

plt.savefig('images/cn_matp_arrow', dpi=600)
```

- plot的图标函数：

```
plt.plot(x,y , fmt) ：绘制坐标图 
plt.boxplot(data, notch, position): 绘制箱形图 
plt.bar(left, height, width, bottom) : 绘制条形图 
plt.barh(width, bottom, left, height) : 绘制横向条形图 
plt.polar(theta, r) : 绘制极坐标图 
plt.pie(data, explode) : 绘制饼图 
plt.scatter(x, y) :绘制散点图 
plt.hist(x, bings, normed) : 绘制直方图
```

## python的pandas库

Pandas 是基于 NumPy 的一个非常好用的库，正如名字一样，人见人爱。之所以如此，就在于不论是读取、处理数据，用它都非常简单。

Pandas 有两种自己独有的基本数据结构。读者应该注意的是，它固然有着两种数据结构，因为它依然是 Python 的一个库，所以，Python 中有的数据类型在这里依然适用，也同样还可以使用类自己定义数据类型。只不过，Pandas 里面又定义了两种数据类型：Series 和 DataFrame，它们让数据操作更简单了。

- Series类型

这种样式我们已经熟悉了，不过，在有些时候，需要把它竖过来表示：

| index | data |
| ----- | ---- |
| 0     | 9    |
| 1     | 3    |
| 2     | 8    |
- Dateframe类型

DataFrame 是一种二维的数据结构，非常接近于电子表格或者类似 mysql 数据库的形式。它的竖行称之为 columns，横行跟前面的 Series 一样，称之为 index，也就是说可以通过 columns 和 index 来确定一个主句的位置。


```

# series 类型会自动对齐索引，且基于索引, 补齐后运算, 运算默认产生浮点数.
import pandas as pd
import numpy as np

a = pd.DataFrame(np.arange(12).reshape(3, 4))
b = pd.DataFrame(np.arange(20).reshape(4, 5))
c = a.mul(b, fill_value=0)
d = pd.Series(np.arange(4))
print(c)
print(d-10)
print(b-d)

# optput:
#      0     1      2      3    4
# 0   0.0   1.0    4.0    9.0  0.0
# 1  20.0  30.0   42.0   56.0  0.0
# 2  80.0  99.0  120.0  143.0  0.0
# 3   0.0   0.0    0.0    0.0  0.0
# 0   -10
# 1    -9
# 2    -8
# 3    -7
# dtype: int32
#       0     1     2     3   4
# 0   0.0   0.0   0.0   0.0 NaN
# 1   5.0   5.0   5.0   5.0 NaN
# 2  10.0  10.0  10.0  10.0 NaN
# 3  15.0  15.0  15.0  15.0 NaN
```

- pandas的Dateframe数据丢弃

```
import pandas as pd

d1 = {'城市': ['北京', '上海', '深圳'],
     '环比': [101.5, 101.2, 103.6],
     '同比': [110.2, 113.8, 115.3]}
d = pd.DataFrame(d1, index=['c1', 'c2', 'c3'])
nd = d.drop('c1')

print(nd)
# output: 
#     城市     环比     同比
# c2  上海  101.2  113.8
# c3  深圳  103.6  115.3

```

- pandas数据排序

```
import pandas as pd
import numpy as np

# 排序，有索引和数值排序

b = pd.DataFrame(np.arange(20).reshape(4, 5), index=['c', 'a', 'b', 'd'])
# print(b)
#
# print(b.sort_index())
# print(b.sort_index(ascending=False))
# print(b.sort_index(axis=1, ascending=False))
# print(b.sort_values(2, ascending=False))

# cumsum:就是当前列之前的和加到当前列上
c = b.cumsum(axis=0)
print(b)
print(c)
# print(b.describe().ix['max'])

# output:
#     0   1   2   3   4
# c   0   1   2   3   4
# a   5   6   7   8   9
# b  10  11  12  13  14
# d  15  16  17  18  19
#     0   1   2   3   4
# c   0   1   2   3   4
# a   5   7   9  11  13
# b  15  18  21  24  27
# d  30  34  38  42  46
```

- 利用正相关来谈论房价.

这里有协方差公式：

$$cov(X,Y) =E[X-E[X]]E[Y-E[Y]]=E[X*Y] - E[X]E[Y]$$

`pearson`相关系数：

因为$\mu _{X}=E(X)$;

$\sigma_{X}^{2}=E(X^{2})-E^{2}(X)$

同样地，对于$Y$，可以写成:

$$\rho _{X,Y}={\frac {E(XY)-E(X)E(Y)}{{\sqrt {E(X^{2})-E^{2}(X)}}~{\sqrt {E(Y^{2})-E^{2}(Y)}}}}$$

```
# cov()函数：协方差 
# corr()函数： pearson相关系统函数。

import pandas as pd

# 房价增幅与m2增幅的相关性。
hprice = pd.Series([3.04, 22.93, 12.75, 22.6, 12.33], index=['2008','2009', '2010', '2011', '2012'])
m2 = pd.Series([8.18, 18.38, 9.13, 7.82, 6.69], index=['2008','2009', '2010', '2011', '2012'])

r1 = hprice.corr(m2)
print(r1)

# output:
# 0.5239439145220387
```

## 参考

- [wiki-correlation](https://en.wikipedia.org/wiki/Correlation_and_dependence)
- [慕课昊天](http://www.icourse163.org/learn/BIT-1001870002?tid=1001963001#/learn/content)

<script type="text/javascript" async
  src="https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.5/MathJax.js?config=default">
</script>
