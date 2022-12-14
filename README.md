## 利用FPGA处理Sobel图像处理算法



### 实验设备

```
 芯片 ZYNQ7010
 板卡 博辰精芯ZYNQ2022A版
 笔记本电脑
 Type-c数据线 x 2 （用于串口通信）
```

![image-20221109191436320](D:\Atencent\Typora_image\image-20221109191436320.png)

---



### sobel算法简介

- 算法目标：实现图像边缘检测（边缘检测：用黑白两色勾勒图像整体轮廓，如下图示意）

  处理前图像					![image-20221109190848602](D:\Atencent\Typora_image\image-20221109190848602.png)                          

  处理后图像					![image-20221109191001260](D:\Atencent\Typora_image\image-20221109191001260.png)

  

- 算法原理

  对于图像而言，依次取3行3列的图像数据，将图像数据与对应位置的算子的值相乘再相加，得到x方向的Gx，和y方向的Gy，将得到的Gx和Gy，平方后相加，再取算术平方根，得到Gxy，近似值为Gx和Gy绝对值之和，将计算得到的Gxy与我们设定的阈值相比较，Gxy如果大于阈值，表示该点为边界点，此点显示黑点，否则 显示白点。对于一个100x100的图像处理后得到98x98

![image-20221109191649013](D:\Atencent\Typora_image\image-20221109191649013.png)

![image-20221109191927603](D:\Atencent\Typora_image\image-20221109191927603.png)



---



### FPGA实现流程

- 预处理：对于任意一张图像，首先利用python-cv2库进行预处理。将图像调整为100x100的灰度图，并且将图像数组的十进制数转换成为八位十六进制数，依次写入到文件里面（图像点的范围为0-255，恰好为八位宽）

```python
import cv2
import numpy as np

img = cv2.imread('5.png')
img = cv2.cvtColor(img,cv2.COLOR_RGB2GRAY) 
img = cv2.resize(img,(100,100))
print(img.shape)
cv2.imwrite('tx.png',img)
file = open ('tx.txt' ,mode='w')
for i in range(100):
    for j in range (100):
        if(img[i][j]<16):
            file.write('0')
        file.write(hex(img[i][j])[2:])
        file.write(' ')
```

- 通过串口进行数据的发送（串口波特率115200，由于FPGA时延固定，在尝试高波特率460800时也没有发生数据错误）
- FPGA通过串口处理数据，处理完成之后传出给电脑（具体Verilog代码可联系作者，实现过程在取绝对值时要注意，当两个不同位宽的有符号数相加时，空的高位会被赋值成1）

![image-20221109193331699](D:\Atencent\Typora_image\image-20221109193331699.png)

- 电脑接受到数据，通过CV2的库显示出图片

```python
import numpy
import cv2
file2 = open ('rx.txt',mode='r')
a=file2.read().split()
img = []
for x in a:
    img.append(int(x,16))
img=numpy.array(img,dtype=int).reshape(98,98)

cv2.imwrite('image.png',img)
```



---



### FPGA实现的优点

- **并行执行任务**

![image-20221028182207438](D:\Atencent\Typora_image\image-20221028182207438.png)

对于时间轴time，除了刚开始的时间 和 结束的几个时间 ，我们只能做一些预处理和收尾的工作，其他时间，我们都在有多个任务并行执行。这些任务彼此不互相依赖。

对应于Sobel算法来说：我们要做以下一些事情（下面我们按照时间的顺序来进行描述）

时间 1- 时间 8 ：我们等待数据一个一个的到来，并且把他放到寄存器里

时间 9 ：我们一旦有了九个数据，我们就可以对1-9这些数据运算他进行输出。同时，将新来的数据9放到缓存器的任务也没有停止……

时间10 ： 我们此时依然有2-10，九个数据运算输出。同时，将新来的数据10放到缓存器的任务也没有停止……

时间11 - 时间 n ：我们这时候每来一个数据，我们都能进行输出。同时做新来数据预处理的任务。



这样看似我们只在同时执行两个任务。然而，实际情况可能比这复杂的多。我们只要有足够的资源，就能轻易的把这些资源搭成我们想要的任务处理器。

- **固定的时延**

对于一个任务需要多少时间，我们清楚的知道。因为我们对每个时钟周期进行了规划。

- **更快的速度**

对于本实验而言，工作的流程是PC--->FPGA，这样的串行首发数据是很占时间的。

对于真正的处理时间，我们只花了几个时钟的时间。

这样比较FPGA肯定是吃亏的。

但是如果工作场景是

传感器前端 ---> PC --->控制模块 和 传感器--->FPGA--->控制模块进行比较

那么FPGA肯定是完胜的。这也就是一些工厂设备，例如线阵相机采样，数据传输给嵌入式设备做识别是否是次品，再控制控制模块剔除次品的这样一个工作流程中，FPGA肯定是首选。当然这些都是一些成熟的技术了。

- **一些关于CNN加速的猜想**

sobel算子本质上就也是CNN的一层3x3的filter，CNN对于数据的重复利用率很高，也就是说，数据传输所占的时间并不会很影响整体的效率。数据的复用性得到极大的提高之后,即使是pc ---> FPGA的工作场景，FPGA依然有他的胜算。

- 具体代码
