# FPAA使用手册——以实现混沌电路为例

初版编写人：JT Xu

（由于能力原因，手册中不可避免会存在错误）

（后续修改的朋友，修改完记得把自己的名字写在这）



芯片、开发版型号：

- 本手册案例中所使用的芯片型号为AN231E04
- 所使用的软件为AnadigmDesigner2

公司官网：https://www.anadigm.com/sup_downloadcenter.asp?tab=ad2



## 一些使用小技巧（欢迎后续的朋友补充）

  - 在英文输入的前提下，按下 “ i ” 可以放大界面，按下 “ o ” 可以缩小界面。
  - 本软件没有撤回这一操作，但保留 Ctrl + C/V。
  - 当完成第一次较长时间的仿真后，软件出现操作卡顿，可以通过重启来解决问题。

## 一些设计原则（欢迎后续的朋友补充）
  - 在实际的设计电路的过程中，为了使设计过程思路更清晰，通常遵循逐行实现的原则。

- 由于受到集成运算放大器动态范围的限制，FPAA开发板产生的模拟信号范围为 -3V ~ 3V。如果混沌系统的取值范围超出了-3V ~ 3V，FPAA开发板就会进入到积分饱和状态，此时混沌电路不会表现出混沌现象，而是退化为平衡点。因此，必须将混沌系统状态变量的变化范围控制在-3 ~ 3 以内。
- 可以先在MATLAB等软件中，绘制系统的时序图或向轨图来判断系统电压是否始终在 -3V ~ 3V间，若超出范围，则进行适当的缩放。缩放时应注意：将系统中的某个状态变量除以缩放因子，等效于将该状态变量的变化率乘以缩放因子的倒数。
- 所设计的系统中各CAM器件之间的信号相位可在设计完毕后集中处理。
- 当设计的电路较复杂且需要涉及到芯片的资源时，应该进行简单的排序和资源占用计算，以到达电路设计简洁有序的目的。
- 当连接线过多时，可以右击改变线的颜色来辨别线路。
- （个人习惯）绿色代表变量 X ，黄色代表变量 Y ，白色代表变量 Z ，褐色代表变量 U  

- ------

  - 不同于传统的说明书，本手册将通过一些实际的应用来教授FPAA的使用方法

# 新建开发版

- 首先根据实际开发版上标明的芯片型号在 AnadigmDesigner2 (以下简称AD2)中选择对应的芯片型号。操作方法为：

  - 点击工具栏第六个选项。

    <img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240108171738457.png" alt="image-20240108171738457"/>

  - 在弹出的对话框选择对应的芯片型号。

    <img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240108171847885.png" alt="image-20240108171847885"/>

# 案例一

在FPAA中实现系统（1）：

$$
\begin{cases}
\frac{\partial x}{\partial t} = 0.667az\\
\frac{\partial y}{\partial t} =-by+z\tag{1}\\
\frac{\partial z}{\partial t} =-1.5x+y+8my^2+0.125c\\
\end{cases}
$$

$$
a=b=m=2，c=-5.333
$$

根据逐行实现的原则首先要实现公式的是：

$$
\frac{\partial x}{\partial t}=0.667az
$$

可以把上式抽象成公式$`\frac{\partial x}{\partial t}=kz`$，要想实现它，需要用到AD2器件库中的CAM器件GainHalf。

# GainHalf的使用

## 器件调用

- 点击菜单栏下方的工具栏第六个按键

  [^找不到工具栏的方法]: 点击菜单栏第八个选项“View”，勾选“Toolbar”

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240106180930405.png"/>

- 在“CAM”一栏中，找到“GainHalf”，双击选择

  [^注]: 注意左侧芯片型号是否选择正确

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240106181113292.png" alt="image-20240106181113292" style="zoom: 50%;" />

- 放到框里（外），会弹出一个页面：

  [^注]: 也可以左键双击器件打开

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240106181236765.png" alt="image-20240106181236765"  />

## 器件说明

​	GainHalf 创建了一个半周期增益相位，输入电压按可编程增益的值（Gain）进行缩放，并实现反相或非反相的输出，且输出电压在其有效输出相位具有放大器输入失调补偿。

## 参数设置

- 此页面可以设置器件的参数，选项从上到下依次为：

  1. ### 名称（Instance Name）：
  	​	设置了不会显示，可用于报错时查找问题器件。

  2. ### 时钟频率（Clock）：

     ​	指器件用于访问存储器的时钟。当所需要实现的系统中需要用到乘法器时，因为乘法器的 ClockB 要求是 ClockA 的16倍，而所使用的芯片 AN231E04 内部时钟的频率默认为 4000kHz，其十六分之一为 250kHz，所以在这里将器件的 ClockA 设置为250kHz。

  3. ### 选项（options）：

     1. ####  Polarity（极性）：
     
          ​	GainHalf可以简单理解为起到了$`g(x)=kf(x)`$中系数$`k`$的作用，那么Polarity就可以理解为系数$`k`$的正负。
     
        ​		- Non-inverting : 表示系数$`k`$为正
     
        ​		- Inverting : 表示系数$`k`$为负
     
     2. #### Input Sampling Phase(输入采样相位)：
     
        ​	此项主要取决于输入信号的相位，保持与输入信号的相位相同即可，若不相同会出现信号失真的情况。
     
     3. #### Opamp Chopping（斩波）：
     	
     	​	一种用于减小运算放大器的1/f噪声的技术。在运算放大器中，1/f噪声（低频噪声）是一种频谱密度随频率减小而增加的噪声。这种噪声在低频范围内较为显著，特别是在直流和低频应用中可能会对性能产生不良影响。
     
  4. ### Parameters（参数）：
  
     ​	以简单理解为参数$`k`$的绝对值。

[^注]: Gain的值收时钟的值的影响

- #### GainHalf设置为如下：

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240106201707386.png" alt="image-20240106201707386"/>

至此，设置好了$`\frac{\partial x}{\partial t}=0.667az`$中的系数$`k`$。下一步是实现整个式子的功能，这里需要使用到积分器。

# 积分器（Integrator）的使用

## 器件调用

- 按照上述方法在器件库中找到积分器（Integrator）

  <img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240106212245889.png" alt="image-20240106212245889"/>

- 弹出设置界面

  <img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240106213001961.png" alt="image-20240106213001961"/>

## 参数设置

- 设置界面可修改器件的相关参数，自上到下依次为：
  
  1. ### 名称（Instance Name）
  
  2. ### 时钟（Clock）
  
     ​	设置为250kHz（理由参见GainHalf）
  
  3. ### 选择（Options）：
  
     1. #### 极性（Polarity）：
  
        ​	通过下图理解：
  
        <img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107090432184.png" alt="image-20240107090432184"/>
  
     2. #### Input Sampling Phase（输入采样相位）：
  
        ​	此项主要取决于输入信号的相位，保持与输入信号相同即可。
  
     3. #### Compare Control To（比较控制和复位）
  
        1. ##### No Reset（不重置）：
  
           ​	没有复位功能。如果不需要使用到比较和复位功能勾选此项。
  
        2. ##### Signal Ground（地信号）：
  
           ​	当输入的控制信号高于（低于）地信号时积分器复位。同时，应保持内部时钟 ClockB 是 ClockA 的两倍以及保持从比较器“+”输入的信号的频率应该与 ClockB 相同，且输入信号频率应与 ClockB 相同。
  
           （以Non-Inverting， ClockA _250kHz， ClockB _500kHz，Countrol High为例。两个正弦波发生器设置的频率与振幅相同，时钟频率不相同，n1线为250kHz时钟，n2线为500kHz时钟）
  
           <img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107095212643.png" alt="image-20240107095212643"/>
  
        3. ##### Dual Input（双路输入信号）：
  
           ​	当输入的控制信号高于（低于）比较信号时，积分器将复位。同时，应保持内部时钟 ClockB 是 ClockA 的两倍以及保持从比较器“+”输入的信号的频率应该与 ClockB 相同，且输入信号频率应与 ClockB 相同。
  
           （ 以Non-Inverting， ClockA ：250kHz， ClockB ：500kHz，Control High为例。）
  
           <img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107110504269.png" alt="image-20240107110504269"/>
  
           - 图中正弦波发生器（Oscillator Sine）的配置
  
           |      Name       |          Clocks           |                   Parameters                    |
           | :-------------: | :-----------------------: | :---------------------------------------------: |
           | OscillatorSine1 | ClockA ：250kHz（Clock3） |  Osc.Frequency：6kHz <br />Peak Amplitude：2V   |
           | OscillatorSine2 | ClockA ：500kHz（Clock4） |  Osc.Frequency：6kHz <br />Peak Amplitude：2V   |
           | OscillatorSine3 | ClockA ：500kHz（Clock4） | Osc.Frequency：12kHz <br />Peak Amplitude：2.5V |
  
        4. ##### Variable Reference（变量引用）：
  
           ​	当输入的控制信号高于（低于）所选择的参考变量的值时，积分器将复位。同时，应保持内部时钟 ClockB 是 ClockA 的两倍，但此时的从比较器“+”输入信号的频率应该与 ClockA 相同。
  
           （以Non-Inverting， ClockA ：250kHz， ClockB ：500kHz，Control High为例。OscillatorSine1设置为6kHz，2V， ClockA 为250kHz）
  
           <img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107130027888.png" alt="image-20240107130027888"/>
  
     4. #### Opamp Chopping（斩波）：
  
        ​	一种用于减小运算放大器的1/f噪声的技术。在运算放大器中，1/f噪声（低频噪声）是一种频谱密度随频率减小而增加的噪声。这种噪声在低频范围内较为显著，特别是在直流和低频应用中可能会对性能产生不良影响。
  
  4. ### Parameters（参数）
  
     - ##### Integration Const（积分常数）：
  
       ​	积分常数的值取决于时钟频率，一般选取最小的积分常数，表示积分的间隔时间。
  
  - #### Integrator->$`z`$的设置如下：
  
    [^注]: 为了方便确定从Integrator中输出的值对应的变量，规定Integrator->表示输出的变量。
    
    <img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107131358178.png" alt="image-20240107131358178"/>

至此，完成了$`\frac{\partial x}{\partial t}=0.667az`$所需器件的配置，下一步是完成线路的连接。

# 线路连接

## 连接方式（欢迎后续 朋友补充）

​	器件间的线路连接只能在框内进行

​	通过点击端点与端点之间，或者点击后按照鼠标左键拖动，右键可切换连接形式。

## 删除方式（欢迎后续 朋友补充）

​	单击连接线，按下“delete”。

​	关闭器件

## 垮芯片连接（欢迎后续 朋友补充）

​	当输入输出的I/O单元都是 “Bypass” 的模式时，因为FPAA采用的是差模输入与差模输出的方式，所以接口之间的正负极一定要一一对应，不能随便连接。

## 	一些问题（欢迎后续 朋友补充）

### 		出现虚线

​	器件之间时钟频率不相同。

​	器件之间连接对应的相位$`\Phi`$不同。

​	器件之间的传输信号出现周期残缺。

### 		无法连接

​	一条输入线上输入多个信号。

​	输入量当做输出量使用，反之亦然。	

​	

至此已经实现了$`\frac{\partial x}{\partial t}=0.667az`$的功能，此时框内情况应该如下图：

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107140654757.png" alt="image-20240107140654757"/>

下一步，需要实现的公式是：

$$
\frac{\partial y}{\partial t} =-by+z
$$

可以将上式抽象成为$`g(h)=k_1f_1(y)+k_2f_2(z)`$的形式，要实现它，需要用到加法器。

# 加法器（SumDiff）的使用

## 器件调用

- 在器件库中找到SumDiff（加法器）

  <img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107142622566.png" alt="image-20240107142622566"/>

## 参数设置

- 打开参数设备面板

  <img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107142801455.png" alt="image-20240107142801455"/>

- 设置界面可修改器件的相关参数，自上到下依次为：

  1. ### Instance Name（名称）：
  
  2. ### Clocks（时钟）：

	   ​	设置为250kHz（理由参见GainHalf）
  
  3. ### Options（选项）：
  
     #### 	Output Phase（输出相位）：
  
     ​	此项主要取决于输入信号的相位，保持与输入信号相反即可。
  
     #### 	Input 1~4：
  
     ​	此加法器可实现$`G(x)=k_1f_1(x_1)+k_2f_2(x_2)+k_3f_3(x_3)+k_4f_4(x_4)`$的功能，所以可以将Input 1~4,看做是系数$`k_1、k_2、k_3、k_4`$的正负值，Non-Inverting 表示正，Inverting 表示负。
     
     
     ​	需要注意的是：只有在Input 3开启后才能开启Input 4
     
  
  #### 	Opamp Chopping：
  
  ​		一种用于减小运算放大器（op-amp）的1/f噪声的技术。在运算放大器中，1/f噪声（低频噪声）是一种频谱密度随频率减小	而增加的噪声。这种噪声在低频范围内较为显著，特别是在直流和低频应用中可能会对性能产生不良影响。
  
  4. ### Parameters（参数）：
  
     ​	Gain 1~4（其中Gain3与Gain 4要在打开对应的Input后才可以使用）
  
     ​	可以将Gain看做是系数$`k_1、k_2、k_3、k_4`$的绝对值。同时，Gain值范围受器件时钟的影响。
  

​	回到$`\frac{\partial y}{\partial t} =-by+z`$，这里用到了一个两输入的加法器，因此不需要打开加法器的Input 3、Input 4。上述公式第一个输入项为$`-by`$，其中b=2，而加法器输出的结果是$`y`$，所以将加法器的输出端接一个积分器（参数配置同上）后连到加法器的Input 1端，并设置Input 1为Inverting，Gain 1设置为2，将 ClockA 设置为250kHz。
​	至此电路连接应该如下图：

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107160200985.png" alt="image-20240107160200985"/>

​	而加法器的另一个输入项为$`z`$，需要通过$`\frac{\partial z}{\partial t} =-1.5x+y+8my^2+0.125c`$来实现。

​	观察公式，可以看到公式中有一个$`y^2`$，实现它需要使用到乘法器。

# 乘法器（Multiplier）的使用

## 器件调用

- 在器件库中找到Multiplier（乘法器）

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107160748753.png" alt="image-20240107160748753"/>

## 器件说明

​	这个器件创造了一个乘数。左输入电压X乘以8位量化的底部输入电压Y和乘法系数。输出在其有效输出相位具有放大器输入失调补偿。这个器件对输入Y连接有限制。

## 参数设置

- 打开参数设置面板

  <img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107164735278.png" alt="image-20240107164735278"/>

- 设置界面可修改器件的相关参数，自上到下依次为：

  1. ### Instance Name（名称）

  2. ### Clocks（时钟）

     ​	这里将 ClockA 设置为250kHz、 ClockB 设置为4000kHz

     ​	需要注意的是，要想使乘法器正常工作，需要保证的是 ClockB 频率是 ClockA 频率的16倍，而使用的芯片AN231E04的最高时钟频率为4000kHz，将其设置为 ClockB ，则 ClockA 为250kHz，这也是为什么上述的加法器，加法器，GainHalf设置的频率为250kHz。

  3. ### Options（选项）

     1. #### 	Sample and Hold（取样与保持）：
  
          ​	要想理解这个设置，需要明白乘法器的作用机理。乘法器的输出结果为左输入电压 X 乘以底部以8位量化的输入电压 Y 和乘法系数(Multiplication Factor)。

          ​	当关闭 Sample and Hold 时，乘法器在输入电压 Y 采样后，对输入电压 X 采样最多一个时钟周期，这意味着输入信号 X 的采样是在输入电压 Y 的采样之后的一个时钟周期内完成的，这表明系统中存在某种时序关系，即 X 的采样稍晚于 Y 的采样。且输入电压Y完成采样后的一个周期内，其与输入电压X的乘积将作为是一个时钟周期的有效输出。

          ​	当开启 Sample and Hold（Input X）时，在输入电压 X 的信号路径中放置了与样本保持（Sample and Hold）电路相同的电路。乘法器将同时对输入电压 X 与输入电压 Y 进行采样，即两者不存在时序关系，且当输入电压 Y 完成采样后的一个时钟周期内两者乘积将作为有效的输出。

          ​	根据以上的原理，可以得出：当两个输入电压 X Y 相同，即在相同的时间点上具有相同的数值时，可以考虑不开启 Sample and Hold 。在这种情况下，两个输入信号是一致的，不存在采样时序上的差异，因此没有必要通过 Sample and Hold 来确保采样的时序关系。反之，当两个输入电压不相同时，会存在时序上的差异，因此需要开启开启 Sample and Hold（Input X）来确保输出结果的准确性。

          
  
     2. #### Opamp Chopping（斩波）：
  
          ​	一种用于减小运算放大器的1/f噪声的技术。在运算放大器中，1/f噪声（低频噪声）是一种频谱密度随频率减小而增加的噪声。这种噪声在低频范围内较为显著，特别是在直流和低频应用中可能会对性能产生不良影响。
  
  4. ### Parameters（参数）
  
     1. #### Multiplication Factor（乘法系数）：
  
        乘法器的输出电压 = 输入电压 X * 输入电压 Y * 乘法系数
  
## 相关说明

- 在使用乘法器的时候需要考虑一个问题：即使两个输入电压绝对值均小于3V，但是无法保证他们的乘积落在-3~3V，所以这里我们需要将乘法系数设置为0.333，以保证两者乘积一定落在-3~3V的范围内，当然后面的运算中需要在一个合适的地方乘以3以保证系统的准确性。
- 通过观察乘法器的图标可知乘法器的输出在相位1无波形损失，在相位2存在波形损失，为保障信号在垮芯片传输或连接到只接受相位2的信号的CAM器件时的稳定性，应将其通过CAM器件Hold后再进行操作。

​	了解了乘法器的使用方法后，可以实现公式中$`y^2`$的功能，继续观察公式，可以发现公式末尾有一个常数$`0.125c`$，可以通过CAM器件Voltage（电压）来实现。

# 电压（Voltage）的使用

## 器件调用

- 在CAM器件库中找到Voltage（电压）

  <img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107201907591.png" alt="image-20240107201907591"/>

## 参数设置

- 打开器件设置面板

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107202024434.png" alt="image-20240107202024434"/>

- 设置界面可修改器件的相关参数，自上到下依次为：

  1. ### Instance Name（名称）：

  2. ### Options（选项）：

     1. #### Polarity（电压）：

        可以将Voltage看成是一个常数±2，可以通过加法器等器件来调整其值。	

至此可以实现公式$`\frac{\partial z}{\partial t} =-1.5x+y+8my^2+0.125c`$，这里用到了一个四输入的加法器。并按如下要求配置：

| Name     | ClockA | Options                                                      | Parameters                                                   |
| :------- | :----- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| SumDiff2 | 250kHz | Out Phase ：Phase 2<br />Input 1 ：Inverting<br />Input 2 ：Non-inverting<br />Input 3 ：Non-inverting<br />Input 4 ：Inverting | Gain 1：1.5<br />Gain 2：1<br />Gain 3：16<br />Gain 4：0.333 |

将乘法器的输出端连至Input 3，这里需要将加法器的输出相位（Out Phase）改成相位2（$`\Phi2`$），否则会因为$`\Phi`$的值不同而报错，同样的与加法器连接的积分器的输入信号相位也需要修改为相位2；

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107210036229.png" alt="image-20240107210036229"/>

将Integrator->y，连接到四输入加法器的 Input 2

将Integrator->x，连接到四输入加法器的 Input 1

引入Voltage，连接到四输入加法器的 Input 4

将Integrator->z，连接到而输入加法器的 Input 2和 GainHalf

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107210815335.png" alt="image-20240107210815335"/>

为方便检查电路连接，可以右击选择线路的颜色，观察信号的路线。

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107211058149.png" alt="image-20240107211058149"/>

至此，成功在芯片内部实现了电路的功能。但我们需要通过一些芯片的外接接口将电压信号输出至示波器观察。因此需要用到芯片的I/O单元部分。

# I/O 单元的使用

- AN231E04芯片提供了4个模拟 I/O 单元和4个可配置 I/O 单元，这里以左侧的4个模拟I/O单元为例。

- 双击 I/O 单元，打开设置界面

- 选择 I/O Made 中的 Output，并选择 Output Type 中的 Bypass，即 I/O 单元不对输出量做任何改动，其他功能当涉及时在做简述。

- 系统一共有三个变量，因此要开启三个 I/O 单元，并将 Integrator->x、Integrator->y、Integrator->z 分别连接到 I/O 单元，如下图。

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107212523942.png" alt="image-20240107212523942"/>

至此，完成了电路的设计，下一步将进行仿真。

# 电路仿真

​	首先，我们需要设置仿真的时长。点击菜单栏中的“Simulate”下的“Setup Simulation”。

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240108193924269.png" alt="image-20240108193924269"/>

​	将End Time设置为50ms。仿真的时长不宜多久，过久会导致软件崩溃。

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240108193949105.png" alt="image-20240108193949105"/>

​	点击工具栏的 “BeginSimulation” 或者按下F5，通过观察底部来判断模拟进度，需要注意的是软件单核运算，所以仿真时间较长，可在仿真过程中按下 “ESC” 来停止仿真。同时，即使底部显示仿真完成，电路仿真结束也需要一点时间。一般提前将探针放到积分器的输出端，来观察输出波形。

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240108194253244.png" alt="image-20240108194253244"/>

输出波形如下：

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107214314205.png" alt="image-20240107214314205"/>

​	因为软件仿真的时间较长，每次观察波形会比较麻烦，可以点击右下角的 Save 导出. csv 数据文件并在 MATLAB 中绘制图像。当软件进行了一次仿真后，改变器件参数时，软件可能会未响应，可以重启软件以解决这个问题。

# 案例二：Chua’s电路

- 系统方程如下：

$$
  \begin{cases}
  \dot x = c \big [y-x+bx+ \frac{1}{2}(a-b)(|x+1|-|x-1|) \big ]\\
  \dot y = x-y+z\tag{2}\\
  \dot z = -dy\\
  \end{cases}
  $$
  
  $$
  典型参数：a =\frac{8}{7},b=\frac{5}{7},c=9,d=\frac{100}{7}
  $$

MATLAB仿真的向轨图和时序图如下：

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120014727641.png" alt="image-20240120014727641" style="zoom:50%;" />

观察系统公式，可以发现有一个函数 $`|x+1|-|x-1|`$ ，这个函数在 AD2 中没有对应的器件可以实现。因此需要用到 AD2 中的一个可以实现自定义函数的器件。

# 转移函数（TransferFuntion）的使用

## 器件调用

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107222611875.png" alt="image-20240107222611875" >

## 使用原理

​	实验所使用的AN231E04X芯片内部所能提供的电压值为 -3V ~ 3V，可以把这一段视为某个函数的定义域，双击器件进入设置界面。看到右侧的Lookup Table，点击进入后可以看到将-3 ~ 3这个区间平均划分为256个区间，每个区间都对于一个输入值和一个输出值。因此，我们可以通过这个器件来实现自定义函数，俗称“打表”。

## 以实现函数$`|x+1|-|x-1|`$为例：

- 在MATLAB中，输入以下代码：

```
x1 = linspace(-3, 3, 256);\\将-3~3平分为256份

y1 = zeros(1, 256);\\创建一个1行256列的O矩阵

\\重复256次，将xi对应的tanh值赋给yi
for i = 1:256
    y1(i) = abs(x(i)+1)-abs(x(i)-1);
end

data = [y1' x1'];
writematrix(data, 'output.csv');\\输出一个[y1,x1]，256行的矩阵，并将其存放到名为“output”的csv文件中
```

​	点击Lookup Table中的Load，导入生成的output.csv文件，完成函数$`|x+1|-|x-1|`$的定义。这里需要注意的是 TransferFuntion 只会读取文件中第一列的256个数值，所以需要检查导入的文件是否符合要求。

当我们完成了自定义器件 TransferFuntion 的设置，连接电路时，会遇到如下图虚线的问题：

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120120551700.png" alt="image-20240120120551700"/>

这是因为器件默认只会在相位1内完成信号的采集和输出，相位2内没有信号输出，即出现波形的半周期损失。要想解决这个问题，可以打开 TransferFuntion 的 “Out put Hold”设置或者后接一个 Hold 器件，以保证在相位1和2都有输出。如下图： 
<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120121627187.png" alt="image-20240120121627187" style="zoom:50%;" />  <img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120121737537.png" alt="image-20240120121737537" style="zoom:50%;" /> 

器件参数配置如下：

| Name             | Clocks                            | Options                                                      | Parameters                                         |
| ---------------- | --------------------------------- | ------------------------------------------------------------ | :------------------------------------------------- |
| SumDiff-> $`x`$    | ClockA 250kHz                     | Out Phase ：Phase 1<br />Input 1 ：Non-inverting<br />Input 2 ：Inverting<br />Input 3 ：Non-inverting<br />Input 4 ：Off | Gain 1：10<br />Gain 2：3.5<br />Gain 3：3.1<br /> |
| SumDiff-> $`y`$    | ClockA 250kHz                     | Out Phase ：Phase 1<br />Input 1 ：Non-inverting<br />Input 2 ：Inverting<br />Input 3 ：Non-inverting<br />Input 4 ：off | Gain 1：1<br />Gain 2：1<br />Gain 3：2<br />      |
| GainHalf         | ClockA 250kHz                     | Polarity ：Inverting<br />Input Sampling Phase ：Phase 1<br /> | Gain ：7.435                                       |
| Integrator->$`x`$  | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 2<br />Compare Control To ：No Reset | Integration Const ：0.025                          |
| Integrator->$`y`$  | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 1<br />Compare Control To ：No Reset | Integration Const ：0.025                          |
| Integrator->$`z`$  | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 2<br />Compare Control To ：No Reset | Integration Const ：0.025                          |
| TransferFunction | ClockA 250kHz<br />ClockB 4000kHz | Output Hold ：Off                                            | -                                                  |

仿真如下：

![image-20240120135348818](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120135348818.png)

# 案例三：Lorenz系统

- 系统方程如下：

$$
\begin{cases}
\dot x = \sigma(y-x)\\
\dot y = -xz+ \rho x-y \tag{3}\\
\dot z = xy- \beta z\\
\end{cases}
$$

$$
典型参数：\sigma = 10,\rho = 28, \beta = \frac{8}{3}
$$

​	

- MATLAB仿真结果如下：
  <img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120142842623.png" alt="image-20240120142842623"/>

​	通过观察系统方程，发现实现这个系统需要3个加法器，3个积分器，2个乘法器，同时考虑到使用乘法器时可能会出现相位问题而需要使用器件 “Hold” ，实现系统所需的器件较多。一般当器件数大于 8 个时，会涉及到器件的资源占比问题。

# 器件资源占比问题

​	将器件拖入框内时，点击框右侧的圆形按键，可以观察器件的资源占用情况。

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120141618419.png" alt="image-20240120141618419"/>

​	以AN231E04为例，一个芯片可供调配的资源可分为4个区域，每个区域有一个SAR（逐次逼近式模拟数字转换器）、八个电容、两个运放、一个比较器，一个芯片总计有4个SAR、32个电容、8个运放、4个比较器，且一个器件只能放在一个区域里，每个区域内的资源不能混用。

- 这里汇总一下之前所提及的一些器件的资源占用情况：

|         Name          | SAR  | cap  | opamp | comp |
| :-------------------: | :--: | :--: | :---: | :--: |
|      Multiplier       |  1   |  4   |   1   |  1   |
| Multiplier（Input X） |  1   |  6   |   2   |  1   |
|  SumDiff（2 input）   |  X   |  3   |   1   |  X   |
|  SumDiff（3 input）   |  X   |  4   |   1   |  X   |
|  SumDiff（4 input）   |  X   |  5   |   1   |  X   |
|        Voltage        |  X   |  X   |   X   |  X   |
|      Integrator       |  X   |  2   |   1   |  X   |
|       GainHalf        |  X   |  2   |   1   |  X   |
|   TransferFunction    |  1   |  5   |   1   |  1   |

​	当考虑到器件的资源占比问题之后，可以选择使用两块芯片来完成这个系统的设计。下图给出一种设计方法，设计思想是将线性项与非线性项分开处理：

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120143443058.png" alt="image-20240120143443058"/>

​	但当用此电路进行模拟时，会发现输出结果都为0，这是因为系统为激发，可以通过引入一个脉冲刺激来激活系统，如下图：

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120144341027.png" alt="image-20240120144341027"/>

​	右击选择 “Create Signal Generator”，双击后选择 “Pluse” 模式，其余选项默认。

- 器件参数配置如下：

| Name               | Clocks                            | Options                                                      | Parameters                                                 |
| ------------------ | --------------------------------- | ------------------------------------------------------------ | ---------------------------------------------------------- |
| Multiplier（$`xy`$） | ClockA 250kHz<br />ClockB 4000kHz | Sample and Hold : Off                                        | Multiplication Factor : 0.333                              |
| Multiplier（$`xz`$） | ClockA 250kHz<br />ClockB 4000kHz | Sample and Hold : Off                                        | Multiplication Factor : 0.333                              |
| SumDiff->$`x`$       | ClockA 250kHz                     | Out Phase ：Phase 1<br />Input 1 ：Non-inverting<br />Input 2 ：Inverting<br />Input 3 ：Off<br />Input 4 ：Off | Gain 1：1<br />Gain 2：1                                   |
| SumDiff->$`y`$       | ClockA 250kHz                     | Out Phase ：Phase 1<br />Input 1 ：Non-inverting<br />Input 2 ：Inverting<br />Input 3 ：Inverting<br />Input 4 ：Non-inverting | Gain 1：2.8<br />Gain 2：0.1<br />Gain 3：6<br />Gain 4：1 |
| SumDiff->$`z`$       | ClockA 250kHz                     | Out Phase ：Phase 1<br />Input 1 ：Non-inverting<br />Input 2 ：Inverting<br />Input 3 ：Off<br />Input 4 ：Off | Gain 1：1.5<br />Gain 2：0.267                             |
| Integrator->$`x`$    | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 2<br />Compare Control To ：No Reset | Integration Const ：0.025                                  |
| Integrator->$`y`$    | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 1<br />Compare Control To ：No Reset | Integration Const ：0.025                                  |
| Integrator->$`z`$    | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 2<br />Compare Control To ：No Reset | Integration Const ：0.025                                  |
| Hold ($`xy`$)        | ClockA 250kHz                     | Input Sampling Phase ：Phase 1                               |                                                            |
| Hold ($`xz`$)        | ClockA 250kHz                     | Input Sampling Phase ：Phase 1                               |                                                            |

- 仿真波形如下：

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120145618438.png" alt="image-20240120145618438"/>

# 案例四：Rössler系统

- 系统方程如下：

$$
  \begin{cases}
  \dot x = -(y+z)\\
  \dot y = x+ay \ tag{4}\\
  \dot z = b+z(x-c)\\
  \end{cases}
  $$

  $$
  典型参数为：a=0.2,b=0.2,c=5.7
  $$

- MATLAB仿真的时序图和向轨图如下：

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120145536149.png" alt="image-20240120145536149"  />

​	该系统未涉及到新的内容，故不做补充。

- 电路连接图：
  <img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120151610446.png" alt="image-20240120151610446"  />

- 器件参数配置如下：

| Name               | Clocks                            | Options                                                      | Parameters                                        |
| ------------------ | --------------------------------- | ------------------------------------------------------------ | ------------------------------------------------- |
| Multiplier（$`xz`$） | ClockA 250kHz<br />ClockB 4000kHz | Sample and Hold : Off                                        | Multiplication Factor : 0.333                     |
| SumDiff->$`x`$       | ClockA 250kHz                     | Out Phase ：Phase 1<br />Input 1 ：Inverting<br />Input 2 ：Inverting<br />Input 3 ：Off<br />Input 4 ：Off | Gain 1：1<br />Gain 2：1                          |
| SumDiff->$`y`$       | ClockA 250kHz                     | Out Phase ：Phase 1<br />Input 1 ：Non-inverting<br />Input 2 ：Non-inverting<br />Input 3 ：Off<br />Input 4 ：Off | Gain 1：1<br />Gain 2：0.2                        |
| SumDiff->$`z`$       | ClockA 250kHz                     | Out Phase ：Phase 2<br />Input 1 ：Non-inverting<br />Input 2 ：Non-inverting<br />Input 3 ：Inverting<br />Input 4 ：Off | Gain 1 ：0.128<br />Gain 2 ：30<br />Gain 3 ：5.7 |
| Integrator->$`x`$    | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 2<br />Compare Control To ：No Reset | Integration Const ：0.025                         |
| Integrator->$`y`$    | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 1<br />Compare Control To ：No Reset | Integration Const ：0.025                         |
| Integrator->$`z`$    | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 2<br />Compare Control To ：No Reset | Integration Const ：0.025                         |
| GainHalf           | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 2<br /> | Gain ：0.4                                        |
| Voltage            | ClockA 250kHz                     | Polarity ：Positive（+2V）                                   |                                                   |

- 仿真波形如下：

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120152126152.png" alt="image-20240120152126152"  />


- 其他欢迎后续的朋友继续补充其他例子

- 在软件中仿真成功后就可以将程序烧录到开发板上，因此下面介绍烧录方法。


# 程序烧录介绍

- 将对应的电源线和数据线，连接至电脑和FPAA，打好对应的硬件驱动。


- 点击菜单栏中的“Setting”选择“Preferences”，点击“Port”，在“Select Port”的下拉菜单中找到对应的端口。


<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240108200923214.png" alt="image-20240108200923214" style="zoom:50%;" />

- 完成之后，点击工具栏里的“Write Configuration Data to Serial Port”，听到叮叮两声后，表示烧录成功。

# 非零初始值设定

​	混沌系统对系统的初始值十分敏感，因此通过修改系统初始值可以使系统产生不同的动力学行为。在FPAA中由于各积分器的默认初始值为0，因此对应系统初始值不为0的混沌系统需要额外配置电路给积分器充电以到达所需要的初始值，并利用FPAA可动态配置线路的特性，将相关的芯片下载到板卡以实现初始值非0的混沌系统。

以混沌偏置系统（5）为例

$$
\begin{cases}
\dot x = 1-az\\
\dot y = -bz^2-y+d_2\tag{5}\\
\dot z =x-y+d_1\\
\end{cases}
$$

$$
a=3.9,b=3.5,d_1=-5,d_2=0\\
$$

系统初始值为$`[-1- d_1 +d_2, d_2, 0]`$

其相轨图和时序图如下：

![1](https://fpaa.oss-cn-shanghai.aliyuncs.com/1.png)

按照$`x\rightarrow5x,y\rightarrow5y,z\rightarrow2z`$调幅后，得到系统（6）

$$
\begin{cases}
\dot x = \frac{1}{5}-\frac{2az}{5}\\
\dot y = \frac{-4bz^2}{5}-\frac{y}{5}+\frac{d_2}{5}\tag{6}\\
\dot z =\frac{5x}{2}-\frac{5y}{2}+\frac{d_1}{2}\\
\end{cases}
$$

缩放后的初始值为[0.8,0,0]

其相轨图和时序图如下：

![2](https://fpaa.oss-cn-shanghai.aliyuncs.com/2.png)

按照上述思路，绘制下图。整个系统分为两个部分，积分器充电电路和混沌系统实现电路。两部分电路在CAM器件参数，类型，数量，外部连线等方面完全相等，唯一的区别在于systemB_ic与systemB_ic的内部结构上，systemB_ic电路单独独立不与systemA_ic相连且与内部的加法器相连，而systemB则相反。对于积分器充电电路可以理解为将三个积分器独立出来，对于混沌系统实现电路可以理解为将三个积分器放到另一个芯片中。FPAA的动态线路配置功能可以理解为修改FPAA的内部连线，在实际的实验过程中，如果两部分电路中CAM器件的参数，类型，数量不完全相同，会导致系统不稳定，无法出现混沌现象。因此，需要保障两部分的CAM器件完全相同。

![](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240426002541479.png)

systemA_ic(systemA)

| Name       | Options                                                      | Parameters                                    | Clocks                          |
| ---------- | ------------------------------------------------------------ | --------------------------------------------- | ------------------------------- |
| Multiplier | Sample and Off  Hold                                         | Multiplication 1.00  Factor                   | ClockA 250kHz<br />ClockB 4 MHz |
| SumDiff    | Output Phase Phase 1 <br />Input 1 Non-inverting <br />Input 2 Inverting <br />Input 3 Off <br />Input 4 Off | Gain 1 0.1<br />Gain 2 1.56                   | ClockA 250 kHz                  |
| SumDiff    | Output Phase Phase 1<br />Input 1 Inverting<br />Input 2 Inverting<br />Input 3 Off<br />Input 4 Off | Gain 1 2.80<br />Gain 2 1.00                  | ClockA 250 kHz                  |
| SumDiff    | Output Phase Phase 2<br />Input 1 Non-inverting<br />Input 2 Inverting<br />Input 3 Inverting<br />Input 4 Off | Gain 1 2.50<br />Gain 2 2.50<br />Gain 3 1.25 | ClockA 250 kHz                  |
| Voltage    | Polarity Positive  (+2V)                                     |                                               |                                 |
| Hold       | Input Sampling Phase : Phase 1                               |                                               | ClockA 250 kHz                  |
| Hold       | Input Sampling Phase : Phase 2                               |                                               | ClockA 250 kHz                  |

 systemB_ic(systemB)

| Name       | Options                                                      | Parameters                        | Clocks         |
| ---------- | ------------------------------------------------------------ | --------------------------------- | -------------- |
| Integrator | Polarity Non-inverting<br />Input Sampling Phase Phase 1<br />Compare Control To No Reset | Integration Const. 0.0025  [1/us] | ClockA 250 kHz |
| SumDiff    | Output Phase Phase 1<br />Input 1 Non-inverting<br />Input 2 Inverting<br />Input 3 Off<br />Input 4 Off | Gain 1 0.400<br />Gain 2 1.00     | ClockA 250 kHz |
| SumDiff    | Output Phase Phase 1<br />Input 1 Non-inverting<br />Input 2 Inverting<br />Input 3 Off<br />Input 4 Off | Gain 1 1.00<br />Gain 2 1.00      | ClockA 250 kHz |
| SumDiff    | Output Phase Phase 1<br />Input 1 Non-inverting<br />Input 2 Inverting<br />Input 3 Off<br />Input 4 Off | Gain 1 1.00<br />Gain 2 1.00      | ClockA 250 kHz |

对于积分器充电电路systemB_ic，采用如下图的统一格式：

![image-20240426003953756](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240426003953756.png)

其中，Gain1=$`\frac{x_0}{2}`$

​	通过这种方法可以使积分器的值达到并维持在所设定的初始值上。这里只展示$`x`$初始值的设置方式，其余变量同理。

![image-20240426102127924](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240426102127924.png)

​	这里需要注意芯片左上角的地址信息“Adder1:x Adder2:y LOAD ORDER:z”，由于后续程序下载的板卡的方式为逐个下载，所以此时y和z的值无需在意，需要注意的是x的值，考虑到FPAA的动态参数配置特性，需要保障积分器充电电路和混沌系统实现电路对应的芯片模块的x相同。可通过单击x，y，z来改变其值。

​	类比于keil5生成的.hex文件，AnadigmDesigner2生成打可以下载到板卡上的文件的后缀为.ahf，AHF文件有两种类型，“Primary”（主配置文件）和“Dynamic”（动态配置文件）。结合FPAA的动态线路配置特性，可以理解为主配置文件存放是系统中CAM器件参数等基本配置，可以形象理解为电路的基本框架；而动态配置文件中存放的是不同于主配置文件中的线路连接方式，可以形象理解为一种多模式切换的开关，向板卡上依次下载两种文件的过程中，板卡中的线路连接被改变而各CAM器件的值不发生改变，通过这种方法可以实现初始值非零的混沌系统。

## AHF文件配置方式

点击菜单栏中的“Dynamic Configure”下的“State-driven Dynamic Configure”

![image-20240426112024231](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240426112024231.png)

在对应的“Active Chip Address”地址下，点击选择“Complete Chips”选择所需要的芯片，这就是为什么需要保持对应芯片地址x的相同。之后，点击“Transitions”选择主配置AHF文件，因为需要先给积分器充电再实现电路，所以将”systemA_ic”和“systemB_ic”设置为主配置文件。

![image-20240426112600089](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240426112600089.png)

之后点击“Generation”，给每一个芯片生成一个对应的.ahf文件并设置储存地址。之后点击“Generate”生成文件。

![image-20240426112630003](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240426112630003.png)

因为systemA与systemB两个芯片的内部配置完全相同，所以在生成时会出现以下提示：

![image-20240426113013196](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240426113013196.png)

在连接好FPAA，用杜邦线连接好对应的引脚后，开始烧录程序。

点击菜单栏中的“Configure”下的“Write AHF File to Serial Port…”，依次下载对应文件夹下的“(Primary)systemA_ic”、“(Primary)systemB_ic”和“systemB_ic”。完成电路的实现。

![45b0c6d2dc3aad4abb4beb7d2dd8eaf](https://fpaa.oss-cn-shanghai.aliyuncs.com/45b0c6d2dc3aad4abb4beb7d2dd8eaf.jpg)

# 烧录失败原因（欢迎后续的朋友补充）

## 	检测不到端口

- 打开电脑设备管理器，检查对应端口是否存在。
- 重新打驱动
- 检查电脑的蓝牙是否打开，部分型号电脑关闭蓝牙可能导致软件无法检测到端口
- 断开所有蓝牙连接
- 修改数据线对应端口的端口号，在3~9最佳，部分型号的电脑因为端口的端口号较大，软件无法检测到

# 开发版设置

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/IMG_20240404_133147(1).jpg" alt="IMG_20240404_133147(1)" style="zoom:50%;" />
![20240404_133147(1)](https://fpaa.oss-cn-shanghai.aliyuncs.com/IMG_20240404_133147(1).jpg)

如图所示为开发版上的芯片单元。两侧I1P-O3P对应芯片的左侧四个IO口，下方IO5P-IO7N对应芯片的左侧3个IO口，下方为左侧四哥就看的额外拓展接口。搭建电路时，要根据所设计的芯片外部连接方式在开发版上对应连接。同时将输出口接入一个减法电路后再连接示波器，以观察波形。

需要注意的是开发版上的各开关和跳线帽要设置正确，具体可参考官方的手册。

# 版本修改注释

- 初版2024/1/8
- 第一次修改2024/3/29
  - 增加设置非零初始值
  - 实际开发版使用方式
  - 修改一些错误
- 第二次修改2024/6/16
  - 统一文字格式和表述方式
