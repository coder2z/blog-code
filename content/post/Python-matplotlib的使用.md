+++
tags = ["python", "matplotlib"]
categories = ["python"]
description = ""
menu = ""
banner = ""
images = []
title = "Python-matplotlib的使用"
date = "2020-03-15T20:01:49+08:00"
+++


# matplotlib的安装

```sh
pip install -U pip setuptools

pip install matplotlib

```

# matplotlib的使用

## 散点图：scatter

使用:

```python
import numpy as np
import matplotlib.pyplot as plt
N = 1000
x = np.random.randn(N)
y = np.random.randn(N)
plt.scatter(x, y)
plt.show()

```

运行:

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/18/20200718200657.png)

## 折线图：plot

使用:

```python
import numpy as np
import matplotlib.pyplot as plt
x = np.linspace(-10, 10, 100)
y = x**2
plt.plot(x,y)
plt.show()

```

运行:

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/18/20200718201004.png)


## 条形图：bar

### 普通条形图

使用:

```python
import numpy as np
import matplotlib.pyplot as plt
N = 5
y = [4, 6, 2, 1, 6]
index = np.arange(N)
plt.bar(left=index, height=y)
plt.show()

```

运行:

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/18/20200718201110.png)

### 并列条形图

使用：

```python
import numpy as np
import matplotlib.pyplot as plt
index = np.arange(4)
a = [32, 54, 67, 90]
b = [44, 33, 67, 98]
bar_width = 0.3
plt.bar(index, a, bar_width, "b")
plt.bar(index+bar_width, b, bar_width, "r")
plt.show()

```

运行：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/18/20200718201305.png)


## 直方图：hist

使用：

```python
import numpy as np
import matplotlib.pyplot as plt
mu = 100
sigma = 20
x = mu + sigma*np.random.randn(2000)
plt.hist(x, bins=10, color="r", edgecolor="black", normed=True)
plt.show()

```

运行：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/18/20200718201406.png)

## 饼状图：pie

使用：

```python
import numpy as np
import matplotlib.pyplot as plt
labels = "A", "B", "C", "D"
fracs = [15, 29, 89, 54]
explode = [0, 0.1, 0, 0] #设置突出显示的参数部分，此处表示只有B突出显示
plt.axes(aspect=1) #使坐标轴横纵等距
#autopct为在饼状图内部显示出相应的数据（百分比）
plt.pie(x=fracs, labels=labels, autopct="%.0f%%", explode=explode)
plt.show()

```

运行：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/18/20200718201458.png)


## 箱形图（箱线图）：boxplot

使用：

```python
import numpy as np
import matplotlib.pyplot as plt
np.random.seed(100)
data = np.random.normal(size=1000, loc=0, scale=1)
#sym改变异常值的显示形状，whis表示虚线的长度
plt.boxplot(data, sym="o", whis=1.5)
plt.show()

```

运行：

![](https://gitee.com/coder2m/pic/raw/master/img/blog/2020/07/18/20200718201541.png)


这里就简单距离了基础使用具体的设置的参数还有很多，就需要查看官方文档了。

官方文档：[https://matplotlib.org/contents.html](https://matplotlib.org/contents.html)

