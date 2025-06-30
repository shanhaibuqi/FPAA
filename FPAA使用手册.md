# FPAA使用手册——以实现混沌电路为例

编写人：xjt（shanhaibuqi@gmail.com）(q:3278683787)(v:shbq3278683787)

（由于能力原因，手册中不可避免会存在错误，还望见谅）

芯片、开发版型号：

​	本书中所有涉及到FPAA(dpASP)的电路系统，均使用Anadigm公司的EDA软件AnadigmDesigner2搭配AN231K04-QUAL2 TWO FPAA开发版实现，该开发版内置两片AN231E04 dpASP芯片，或使相同规制的4芯片开发版AN231K04-QUAL4，该开发版内置四片AN231E04 dpASP芯片

公司官网：https://www.anadigm.com



**使用技巧**

  - 在英文输入的前提下，按下 “ i ” 可以放大界面，按下 “ o ” 可以缩小界面，也可在工具栏手动调整。
  - 本软件没有撤回这一操作，但保留 Ctrl + C/V。
  - 在完成一次较长时间的仿真后，软件出现操作卡顿，可以通过重启清除数据缓存来解决问题。

# 基本概念

​	现场可编程模拟阵列（Field Programmable Analog Array，FPAA）是一种可编程模拟电路集成器件，类似于数字领域的FPGA，但其主要面向模拟信号的处理与应用。用户可以通过编程的方式灵活配置这些电路单元以实现各种模拟功能，将复杂的模拟信号和处理功能设计到一个集成的、无漂移的、预先测试的设备中。通过FPAA，可以快速完成对模拟电路系统的验证，而无需搭建实物电路。
​	然而，传统FPAA在处理具有初始值的电路时，常面对较高的电路初始化复杂性，它不同实物电路，初始值可以在内部电容的充放电过程中的达到，传统的FPAA默认的电路初始值为0，且较难修改，需要额外设计外加电路去处理，这大大增加了整体的成本和难度，而对于混沌电路系统而言，电路初始值又是非常重要的一部分。为了解决这一问题，引入数字可编程模拟信号处理器(Digital Programmable Analog Signal Processor，dpASP) ，dpASP可以有效降低初始化难度并提升系统性能。dpASP是一种结合数字技术与模拟信号处理能力的器件，它与传统的FPAA类似，但在其基础上引入数字电路技术，增强处理模拟信号时的灵活性和效率。通过dpASP可以实现对信号处理器的动态配置，降低电路初始化复杂性，通过程控，精确快速地配置电路系统的初始值，大大提高了效率。

# 设计思路

  - 在实际的设计电路的过程中，为了使设计过程思路更清晰，通常遵循逐行实现的原则。
- 由于受到集成运算放大器动态范围的限制，FPAA（dpASP）芯片内能进行正常运算的模拟信号范围为 -3V-3V。如果混沌系统中变量，包括运算过程中产生的中间变量的电压值超出了-3V-3V这个范围，该变量就会进入到电压饱和状态，芯片锁定其电压值为3V或者-3V，被电压锁定的变量带入运算时大概率会导致计算出错。因此，必须将混沌系统中变量的电压变化范围控制在-3~3V 以内以避免这种情况。可以先在MATLAB等数值仿真软件中，绘制系统重各变量时序图来判断其电压范围是否合适，若超出范围，则进行适当的尺度变化。
- 设计过程中，各CAM器件之间的信号相位可在设计结束后统一处理，以保证整体上的统一。
- 设计电路时需要考虑到芯片内部的资源占用情况，应在设计前对所使用器件资源进行初步计算，以选择合适的芯片数量，器件数量和器件类型，最优化整体架构。
- 连线时，右击改变线的颜色和添加备注来辨别线路，以确保了解每条线路的功能。
- $\Phi$1表示在时钟高时有效，$\Phi$2表示在时钟低有效
- （个人习惯）本文中，在实现混沌电路时，绿色代表变量 X ，黄色代表变量 Y ，白色代表变量 Z ，褐色代表变量 U  

# 软件基本操作

## 芯片调用

- 点击工具栏第五个选项，通过菜单栏第八个选项“View”，勾选“Toolbar”打开。

![image-20250223172257583](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250223172257583.png)

- 在弹出的对话框选择对应的芯片型号AN231E04，单击空白处安放。

![image-20250223172948386](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250223172948386.png)

## 器件调用

- 点击工具栏第六个选项，打开支持AN231E04芯片的器件库，使用Approved为Yes的器件

![image-20250223173120017](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250223173120017.png)



## 器件库器件

### GainHalf

![image-20250630113910230](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250630113910230.png)

**核心功能：**

​	该 CAM实现了一个 **半周期增益放大级**。它将输入的电压信号按照用户设定的增益值 (`Gain` 参数) 进行放大（或缩小），并且可以选择是**反相放大**还是**非反相放大**。它的输出仅在时钟的一个相位（半个时钟周期）内有效，并且在有效的输出相位期间，会对运放的输入失调电压进行补偿。

**关键特性与选项：**

1.  **极性 (Polarity):**
    *   **非反相 (Non-inverting):** 输出信号与输入信号同相。
        *   **输出延迟：** 输出相对于输入采样点有 **一个相位（半个时钟周期）的延迟**。
    *   **反相 (Inverting):** 输出信号与输入信号反相。
        *   **输出延迟：** 输出相对于输入采样点 **没有额外的相位延迟**。
2.  **输入采样 (Input Sampling):**
    *   决定 CAM 在哪个时钟相位对其输入信号进行采样。
    *   该选项必须确保 CAM 的输入采样相位与输入信号有效的相位一致。输入信号可以是：
        *   在所选相位有效（例如，来自另一个半周期 CAM 的输出）。
        *   连续有效（例如，来自一个全周期 CAM 或外部信号源）。
3.  **运放斩波 (Opamp Chopping):**
    *   此选项可以启用核心运算放大器的斩波技术，以进一步减小其失调电压和低频噪声 (1/f 噪声)。
    *   **重要限制：**
        *   斩波时钟由 CAM 的 `CLOCKA` 派生。
        *   `CLOCKA` 必须使用芯片时钟 0, 1, 2 或 3。
        *   所选芯片时钟的分频系数 **必须能被 4 整除**。
    *   **副作用：** 斩波会在其频率（`CLOCKA` 频率的一半）处产生一个信号分量。通常需要使用滤波器来去除这个分量。
4.  **增益参数 (Gain Parameter):**
    *   设定放大器的电压增益 (V/V)。
    *   **增益范围：** 可设定的最小增益为 0.01 V/V。
    *   **最大增益限制：** 取决于 CAM 的工作时钟频率 (`Fc`)：
        *   `Fc = 4 MHz` 时：最大增益 ≈ **3.92 V/V**
        *   `Fc = 250 kHz` 时：最大增益 ≈ **77.7 V/V**
        *   `Fc = 50 kHz` 时：最大增益 ≈ **100.0 V/V**
    *   *时钟频率越高，能达到的最大增益越低。*

**输出特性:**

*   **输出有效性：** 输出仅在 **一个相位** 有效。
    *   **反相版本：** 输出在 **输入采样相位** 有效（无延迟）。
    *   **非反相版本：** 输出在 **输入采样相位之后的下一个相位** 有效（延迟半个周期）。
*   **非有效相位：** 在输出无效的时钟相位，输出被连接到信号地 (Signal Ground)。
*   **失调补偿：** 在输出有效的相位期间，会对运算放大器的输入失调电压进行补偿。

**相关 CAM 对比:**

*   **GainInv:** 提供 **全周期、反相** 增益。输入和输出都是连续的（总是有效）。**没有** 输出失调补偿相位（可能设计不同或补偿方式不同）。
*   **GainHold:** 提供 **半周期、反相** 增益 **并带输出保持**。
    *   输入是采样的（只在特定相位有效）。
    *   输出是 **全周期有效**（在无效相位保持上一个有效值）。
    *   仅在一个输出相位进行运放失调补偿。
    *   可能支持比 `GainHalf` **更高的最大增益**。

------

### Hold

![image-20250630113937320](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250630113937320.png)

**核心功能**

该 CAM 实现一个 **采样保持电路**。在输入采样相位对输入电压进行采样，并将该值 **保持两个相位（一个完整时钟周期）** 后输出。

**关键特性与选项**

1. **输入采样 (Input Sampling)**  
   - 决定 CAM 在哪个时钟相位采样输入信号：  
     - **相位 1 (Phase 1)**：  
       - 输入信号需在 **相位 1 或连续有效**（如全周期 CAM 输出）。  
       - **输出保持时段**：采样后的相位 2 → 下一周期相位 1（完整时钟周期）。  
     - **相位 2 (Phase 2)**：  
       - 输入信号需在 **相位 2 或连续有效**。  
       - **输出保持时段**：采样后的相位 1 → 下一周期相位 2（完整时钟周期）。  
   - **必须匹配输入信号有效性**：若输入信号仅在特定相位有效（如半周期 CAM 输出），需选择对应采样相位。

**输出特性**

- **采样延迟**：  
  - 输出值 = **上一采样相位结束时的输入电压**。  
  - 例如：  
    - 若在 **相位 1 采样**，则输出从 **相位 2 开始保持**，直至 **下一周期相位 1 结束**。  
    - 若在 **相位 2 采样**，则输出从 **相位 1 开始保持**，直至 **下一周期相位 2 结束**。  
- **输出有效性**：  
  - 输出在 **两个连续相位（完整时钟周期）内有效且稳定**，适合连接需要全周期输入的后续 CAM。

------

### Integrator

![image-20250630114349861](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250630114349861.png)

**核心功能**

该 CAM 实现 **可编程积分常数的模拟积分器**，支持反相/非反相操作，并集成 **比较器控制的复位功能**。输出为全周期有效信号，在输出相位进行运放失调补偿。

**关键特性**

1. **积分模式配置**

| **选项**         | **非反相模式 (Non-inverting)** | **反相模式 (Inverting)** |
| ---------------- | ------------------------------ | ------------------------ |
| **输出延迟**     | 输入采样后延迟半周期           | 输入采样相位即时响应     |
| **输出变化时机** | 输出相位相反时更新             | 输出相位相同时更新       |

2. **复位系统（比较器控制）**

| **配置选项**                      | **功能说明**                   | **复位条件**                 |
| --------------------------------- | ------------------------------ | ---------------------------- |
| **比较对象 (Compare Control To)** |                                |                              |
| - No Reset                        | 禁用复位功能                   | -                            |
| - Signal Ground                   | 控制信号 vs 0V                 | `Control > 0V` 或 `<0V`      |
| - Dual Input                      | 控制信号 vs 参考输入信号       | `Control > Reference` 或 `<` |
| - Variable Reference              | 控制信号 vs 内部可编程参考电压 | `Control > Vref` 或 `<Vref`  |
| **复位条件 (Reset When)**         |                                |                              |
| - Control High                    | 控制信号高于阈值时复位         | Reset=High 时激活复位        |
| - Control Low                     | 控制信号低于阈值时复位         | Reset=High 时激活复位        |

| **参考电压 (Reference Voltage)** |  
| - 范围：±4.0V（仅限 Variable Reference 模式）  
| - 注意：0V 需用 Signal Ground 模式实现，±3V 建议用 Dual Input 模式节省资源  

**时钟系统**

| **时钟**   | **作用**           | **频率关系**   | **关键要求**                |
| ---------- | ------------------ | -------------- | --------------------------- |
| **CLOCKA** | 积分器开关电容时钟 | 基准频率       | 分频系数需被 4 整除（斩波） |
| **CLOCKB** | 比较器采样时钟     | **2 × CLOCKA** | 确保复位与积分同步          |

------

### SumDiff

![image-20250630114011055](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250630114011055.png)

**核心功能**

该 CAM 实现一个 **可配置的模拟信号混合器**，支持最多 **4 个输入通道**，可灵活组合 **求和（同相）** 与 **求差（反相）** 运算。每个输入通道有独立可编程增益，输出为 **半周期有效信号**，并在输出相位进行运放失调补偿。此加法器可实现$G(x)=k_1f_1(x_1)+k_2f_2(x_2)+k_3f_3(x_3)+k_4f_4(x_4)$的功能，所以可以将Input 1~4,看做是系数$k_1、k_2、k_3、k_4$的正负值，Non-Inverting 表示正，Inverting 表示负。

**关键特性**  

1. **输入通道配置（最多 4 路）**  

   - 每个通道独立选择 **极性** 和 **开关状态**：  

     | **极性**                 | **采样相位规则**               | **输出项符号**  |
     | ------------------------ | ------------------------------ | --------------- |
     | **同相 (Non-inverting)** | 在 **输出相位相反** 的相位采样 | `+Gain×V_input` |
     | **反相 (Inverting)**     | 在 **输出相位相同** 的相位采样 | `-Gain×V_input` |
     | **关闭 (Off)**           | 禁用该输入通道                 | -               |

2. **输出相位控制**  

   | **输出相位** | **反相输入采样相位** | **同相输入采样相位**  | **输出示例**          |
   | ------------ | -------------------- | --------------------- | --------------------- |
   | **Phase 1**  | Phase 1（无延迟）    | Phase 2（延迟半周期） | `Vout = ±G1V1 ± G2V2` |
   | **Phase 2**  | Phase 2（无延迟）    | Phase 1（延迟半周期） | `Vout = ±G1V1 ± G2V2` |

3. **运放斩波 (Opamp Chopping)**  

   - 启用条件：CLOCKA 使用芯片时钟 0-3 且分频系数 **能被 4 整除**  
   - 副作用：产生频率为 **CLOCKA/2** 的噪声，需后续滤波  

**输出特性**

| **特性**     | **说明**                                 |
| ------------ | ---------------------------------------- |
| **有效性**   | 仅在选定相位有效（另一相位输出接地）     |
| **延迟特性** | 反相输入：无延迟<br>同相输入：半周期延迟 |
| **失调补偿** | 输出有效期间动态补偿运放偏移             |

**相关 CAM 对比**

| **模块**          | **通道数** | **周期特性** | **特殊功能**   | **适用场景**   |
| ----------------- | ---------- | ------------ | -------------- | -------------- |
| **Sum/Diff Half** | 4          | 半周期       | 极性可编程     | 高速混合运算   |
| **SumFilter**     | 3          | 全周期       | 集成低通滤波器 | 抗混叠求和电路 |
| **SumInv**        | 3          | 全周期       | 固定反相求和   | 基础反相加法器 |

------

### Multiple

![image-20250629175143251](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250629175143251.png)

**核心功能**

该 CAM 实现 **模拟电压乘法器**，计算 `输出 = X × Y × M`：  

- **X**：模拟电压输入（连续或采样）  
- **Y**：**8位量化输入**（通过 SAR ADC 转换，范围 ±3V）  
- **M**：可编程乘法因子（0.33~4.0）  
- **输出特性**：半周期有效（仅相位 1），相位 2 接地  

**关键选项与模式**

1. **输入采样模式 (Sample and Hold)**

| **模式**       | **X 输入特性**                             | **采样同步性**      | **输出延迟**              |
| -------------- | ------------------------------------------ | ------------------- | ------------------------- |
| **Off (默认)** | 直接采样（无缓冲）                         | X 和 Y **异步采样** | Y 采样后 **1 时钟周期**   |
| **On**         | 增加采样保持电路（同 Sample and Hold CAM） | X 和 Y **同步采样** | 同步采样后 **1 时钟周期** |

​	要想理解这个设置，需要明白乘法器的作用机理。乘法器的输出结果为左输入电压 X 乘以底部以8位量化的输入电压 Y 和乘法系数(Multiplication Factor)。

​	当关闭 Sample and Hold 时，乘法器在输入电压 Y 采样后，对输入电压 X 采样最多一个时钟周期，这意味着输入信号 X 的采样是在输入电压 Y 的采样之后的一个时钟周期内完成的，这表明系统中存在某种时序关系，即 X 的采样稍晚于 Y 的采样。且输入电压Y完成采样后的一个周期内，其与输入电压X的乘积将作为是一个时钟周期的有效输出。

​	当开启 Sample and Hold（Input X）时，在输入电压 X 的信号路径中放置了与样本保持（Sample and Hold）电路相同的电路。乘法器将同时对输入电压 X 与输入电压 Y 进行采样，即两者不存在时序关系，且当输入电压 Y 完成采样后的一个时钟周期内两者乘积将作为有效的输出。

​	根据以上的原理，可以得出：当两个输入电压 X Y 相同，即在相同的时间点上具有相同的数值时，可以考虑不开启 Sample and Hold 。在这种情况下，两个输入信号是一致的，不存在采样时序上的差异，因此没有必要通过 Sample and Hold 来确保采样的时序关系。反之，当两个输入电压不相同时，会存在时序上的差异，因此需要开启开启 Sample and Hold（Input X）来确保输出结果的准确性。

2. **运放斩波 (Opamp Chopping)**

- **功能**：抑制运放失调电压  
- **启用条件**：  
  - CLOCKA 必须使用芯片时钟 0~3  
  - CLOCKA 分频系数 **必须被 4 整除**  
- **副作用**：产生频率为 **CLOCKA/2** 的噪声，需后续滤波  

**时钟要求**  

| **时钟**   | **作用**         | **频率关系**    | **信号路径要求**        |
| ---------- | ---------------- | --------------- | ----------------------- |
| **CLOCKA** | 开关电容网络时钟 | 基准频率        | 同信号路径所有 CAM 一致 |
| **CLOCKB** | SAR ADC 工作时钟 | **16 × CLOCKA** | 独立配置                |

**输出特性**  

| **特性**       | **Sample and Hold Off**     | **Sample and Hold On**      |
| -------------- | --------------------------- | --------------------------- |
| **输出有效性** | 仅相位 1 有效               | 仅相位 1 有效               |
| **X 采样延迟** | 依赖 Y 极性：Y>0 延迟半周期 | **无延迟**（与 Y 同步采样） |
| **总输出延迟** | Y 采样后 1 时钟周期         | 同步采样后 1 时钟周期       |

**设计约束**

1. **Y 输入范围**：  
   - 硬件限制：**±3V**（超范围输出饱和）  
   - 信号源要求：必须 **在相位 1 保持稳定**（满足 SAR 的 8 次连续采样需求）  
2. **时钟同步**：  
   - CLOCKB 必须严格为 CLOCKA 的 **16 倍频**  
3. **精度限制**：  
   - Y 输入量化误差（8 位 SAR）  
   - 电容比例匹配精度影响乘法因子 Multiplication Factor 

- 在使用乘法器的时候需要考虑一个问题：即使两个输入电压绝对值均小于3V，但是无法保证他们的乘积落在-3~3V，所以这里我们需要将乘法系数设置为0.333，以保证两者乘积一定落在-3~3V的范围内，当然后面的运算中需要在一个合适的地方乘以3以保证系统的准确性。
- 通过观察乘法器的图标可知乘法器的输出在相位1无波形损失，在相位2存在波形损失，为保障信号在垮芯片传输或连接到只接受相位2的信号的CAM器件时的稳定性，应将其通过CAM器件Hold后再进行操作。
- 因为clockA和clockB默认设置为250kHz和4000kHz，符合16倍的关系。

------

### Transfunction

![image-20250630114049502](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250630114049502.png)

**核心功能**

该 CAM 实现 **用户可编程的电压转换功能**，通过 256 级量化查找表（LUT）将输入电压映射到指定的输出电压。支持两种输出模式，输入信号需满足特定时序要求。

**关键特性**

1. **输出模式 (Output Hold)**

| **模式**       | **输出特性**              | **失调补偿**    | **开关数量** |
| -------------- | ------------------------- | --------------- | ------------ |
| **Off (默认)** | 半周期有效（仅相位 1）    | 相位 1 期间补偿 | 5 个开关     |
| **On**         | 全周期有效（相位 2 保持） | **无补偿**      | 8 个开关     |

2. **电压转换机制**

- **输入处理**：  
  - 8 位 SAR ADC 量化输入电压（±3V 满量程）  
  - 超范围输入按满量程处理  
- **查找表 (LUT)**：  
  - 256 个可编程输出电压值（-3V ~ +3V）  
  - 通过 `setTransferFunctionTable()` 加载配置  
  - 理论精度 ±3V/256 ≈ 11.7mV
- **输出响应**：  
  - **固定延迟**：输入采样后 1 个时钟周期输出  

3. **运放斩波 (Opamp Chopping)**

- **启用条件**：  
  - CLOCKA 使用芯片时钟 0-3  
  - CLOCKA 分频系数 **必须被 4 整除**  
- **副作用**：产生 **CLOCKA/2** 频率噪声，需后续滤波  

**时钟系统**

| **时钟**   | **作用**            | **频率关系**    | **关键要求**        |
| ---------- | ------------------- | --------------- | ------------------- |
| **CLOCKA** | 开关电容网络时钟    | 基准频率        | 同信号路径 CAM 一致 |
| **CLOCKB** | SAR 和 LUT 驱动时钟 | **16 × CLOCKA** | 严格同步            |



**以实现函数$|x+1|-|x-1|$为例**

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

​	点击Lookup Table中的Load，导入生成的output.csv文件，完成函数$|x+1|-|x-1|$的定义。这里需要注意的是 TransferFuntion 只会读取文件中第一列的256个数值，所以需要检查导入的文件是否符合要求。



## 新建开发版

选择菜单栏中的Get a new chip选择芯片并创建

![image-20250629171301659](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250629171301659.png)

## 芯片地址和下载顺序设置

​	点击LOAD ORDER：1后可以修改数字，改变下载的顺序，仅在直接烧录中有用

​	点击Addr可以修改两个的数值，在选择烧录的过程中会有影响

![image-20250629172132828](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250629172132828.png)

## 端口配置

端口设置打开Settings-preferences-port使软件中选择的端口号与资源管理器中的端口号相同。![image-20250629170000037](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250629170000037.png)

## 	检测不到端口

- 打开电脑设备管理器，检查对应端口是否存在。
- 重新打驱动
- 检查电脑的蓝牙是否打开，部分型号电脑关闭蓝牙可能导致软件无法检测到端口
- 断开所有蓝牙连接
- 修改数据线对应端口的端口号，在3~9最佳，部分型号的电脑因为端口的端口号较大，软件无法检测到

## 直接烧录：软件芯片数与板卡芯片数相同

端口配置成功后，通过点击烧录按键实现，电脑会发出叮叮的两声表示代码烧录完成。

![image-20250629150219200](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250629150219200.png)

## 选择烧录：软件芯片数多于板卡芯片数

端口配置成功后，点击Configure-Write Configuration data to a file，选择目标AHF文件进行烧录，详细操作见案例二。

## 线路连接

​	器件间的线路连接只能在框内进行

​	通过点击端点与端点之间，或者点击后按照鼠标左键拖动，右键可切换连接形式。

## 垮芯片连接

​	当输入输出的I/O单元都是 “Bypass” 的模式时，因为FPAA采用的是差模输入与差模输出的方式，所以接口之间的正负极一定要一一对应，不能随便连接。

## 	一些问题

### 		出现虚线

​	主要是器件之间时钟频率不相同、对应的时钟相位$\Phi$不同等时序问题。

### 		无法连接

​	一条输入线上输入多个信号。

​	输入量当做输出量使用，反之亦然。

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

# 器件资源占用问题

将器件拖入框内时，点击框右侧的圆形按键，可以观察当前器件的资源占用情况。

![image-20250629142256600](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250629142256600.png)

​	以AN231E04芯片为例，一个芯片可供调配的资源可分为4个区域，每个区域有1个单位的SAR、8个单位的电容、2个单位的运放、1个单位的比较器，一个芯片总计有4个单位的SAR、32个单位电容、8个单位的运放、4个单位的比较器，且一个器件只能调用一个区域内的资源，每个区域内的资源不能混用。

​	常用器件的资源占用情况：

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
|         Hold          |  X   |  2   |   1   |  X   |

​	一般当单个芯片内器件数大于 8 个时，会涉及到器件的资源占比问题，这需要结合实际的硬件设备和目标设计，来调整器件或芯片数量。特别是需要当信号要在多个芯片中传输时，需要考虑时序设计以及Hold器件的资源占用情况。

# 系统初始值问题

​	混沌系统对于系统的初始值有一定的要求，如果不进行设置那么在软件中系统的初始值均为0。设置初值有两种方法：1.对于只需要微小扰动就能从000状态进入混沌的系统而言可以直接使用软件自带的外部输入模拟中的激励发生器来模拟输入微小扰动，实际电路中这类激励可以由一些引脚上的微小电荷变化实现，无需进行额外的电路搭建，可以参考Lorenz系统；2.对于需要稳定的初始电压的系统，需要使用动态配置，具体步骤参考案例五。

# 案例一

在FPAA中实现混沌系统CS1（1）：
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
可以把上式抽象成公式$\frac{\partial x}{\partial t}=kz$，要想实现它，需要用到器件库中的CAM器件GainHalf，实现$\frac{\partial x}{\partial t}=0.667az$的功能，此时框内情况应该如下图：

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107140654757.png" alt="image-20240107140654757"/>

下一步，需要实现的公式是：
$$
\frac{\partial y}{\partial t} =-by+z
$$
可以将上式抽象成为$g(h)=k_1f_1(y)+k_2f_2(z)$的形式，要实现它，需要用到加法器。这里用到了一个两输入的加法器，因此不需要打开加法器的Input 3、Input 4。上述公式第一个输入项为$-by$，其中b=2，而加法器输出的结果是$y$，所以将加法器的输出端接一个积分器（参数配置同上）后连到加法器的Input 1端，并设置Input 1为Inverting，Gain 1设置为2，将 ClockA 设置为250kHz。
​	电路连接应该如下图：

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240107160200985.png" alt="image-20240107160200985"/>

​	而加法器的另一个输入项为$z$，需要通过$\frac{\partial z}{\partial t} =-1.5x+y+8my^2+0.125c$来实现。

​	观察公式，可以看到公式中有一个$y^2$，实现它需要使用到乘法器。实现公式$\frac{\partial z}{\partial t} =-1.5x+y+8my^2+0.125c$，这里用到了一个四输入的加法器。并按如下要求配置：

| Name     | ClockA | Options                                                      | Parameters                                                   |
| :------- | :----- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| SumDiff2 | 250kHz | Out Phase ：Phase 2<br />Input 1 ：Inverting<br />Input 2 ：Non-inverting<br />Input 3 ：Non-inverting<br />Input 4 ：Inverting | Gain 1：1.5<br />Gain 2：1<br />Gain 3：16<br />Gain 4：0.333 |

将乘法器的输出端连至Input 3，这里需要将加法器的输出相位（Out Phase）改成相位2（$\Phi2$），否则会因为$\Phi$的值不同而报错，同样的与加法器连接的积分器的输入信号相位也需要修改为相位2；

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

至此，完成了电路的设计，下一步将进行仿真，按照前文的仿真设置。

# 案例二：Chua’s电路

系统方程如下：
$$
\begin{cases}
\frac{\part x}{\part t} = c\bigg[y-x+bx+\frac{1}{2}(a-b)(|x+1|-|x-1|)\bigg]\\
\frac{\part y}{\part t} = x-y+z\tag{2}\\
\frac{\part z}{\part t} = -dy\\
\end{cases}
$$

$$
典型参数：a =\frac{8}{7},b=\frac{5}{7},c=9,d=\frac{100}{7}
$$

MATLAB仿真的相轨图和时序图如下：

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120014727641.png" alt="image-20240120014727641" style="zoom:50%;" />

观察系统公式，可以发现有一个函数 $|x+1|-|x-1|$ ，这个函数在 AD2 中没有对应的器件可以实现。因此需要用到 AD2 中的一个可以实现自定义函数的器件Transfunction。 

器件参数配置如下：

| Name             | Clocks                            | Options                                                      | Parameters                                         |
| ---------------- | --------------------------------- | ------------------------------------------------------------ | :------------------------------------------------- |
| SumDiff-> $x$    | ClockA 250kHz                     | Out Phase ：Phase 1<br />Input 1 ：Non-inverting<br />Input 2 ：Inverting<br />Input 3 ：Non-inverting<br />Input 4 ：Off | Gain 1：10<br />Gain 2：3.5<br />Gain 3：3.1<br /> |
| SumDiff-> $y$    | ClockA 250kHz                     | Out Phase ：Phase 1<br />Input 1 ：Non-inverting<br />Input 2 ：Inverting<br />Input 3 ：Non-inverting<br />Input 4 ：off | Gain 1：1<br />Gain 2：1<br />Gain 3：2<br />      |
| GainHalf         | ClockA 250kHz                     | Polarity ：Inverting<br />Input Sampling Phase ：Phase 1<br /> | Gain ：7.435                                       |
| Integrator->$x$  | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 2<br />Compare Control To ：No Reset | Integration Const ：0.025                          |
| Integrator->$y$  | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 1<br />Compare Control To ：No Reset | Integration Const ：0.025                          |
| Integrator->$z$  | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 2<br />Compare Control To ：No Reset | Integration Const ：0.025                          |
| TransferFunction | ClockA 250kHz<br />ClockB 4000kHz | Output Hold ：Off                                            | -                                                  |

仿真如下：

![image-20240120135348818](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120135348818.png)

# 案例三：Lorenz系统

系统方程如下：
$$
\begin{cases}
\frac{\part x}{\part t} = \sigma(y-x)\\
\frac{\part y}{\part t} = -xz+\rho x-y\tag{3}\\
\frac{\part z}{\part t} = xy-\beta z\\
\end{cases}
$$

$$
典型参数：\sigma = 10,\rho = 28, \beta = \frac{8}{3}\\
初始值：\left ( 0.1,0,0 \right)
$$

MATLAB仿真结果如下：
<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120142842623.png" alt="image-20240120142842623"/>

​	不同于前面两个案例，在案例中注意到原系统变量$x$，$y$和$z$的范围均超过既定电压值（±3V），因此需要对系统进行放缩调幅，缩放要求变量$x$，$y$，$z$，$\dot x$，$\dot y$，$\dot z$在电压范围内。放缩倍率设置为$x\to10x$，$y\to10y$，$z\to20z$，此时$x$，$y$，$z$在电压范围内，但$\dot x$，$\dot y$，$\dot z$不在电压范围内，此时可以继续进行缩放，但因为此时$x$，$y$，$z$的缩放倍率较大，继续放大可能会导致$x$，$y$，$z$的信号过小，因此可以对时间分量$t$操作同步10倍缩小$\dot x$，$\dot y$，$\dot z$，得到：
$$
\begin{cases}
\frac{\part x}{\part t} = 0.1\cdot (10\cdot(10\cdot y - 10\cdot x))/10\\
\frac{\part y}{\part t} = 0.1\cdot (-10\cdot x\cdot 20 \cdot z+28\cdot 10 \cdot x-10\cdot y)/10\tag{4}\\
\frac{\part z}{\part t} = 0.1\cdot(10\cdot x \cdot 10\cdot y-\frac{8}{3} \cdot 20 \cdot z)/20\\
\end{cases}
$$
​	简化得到：
$$
\begin{cases}
\frac{\part x}{\part t} =  y - x\\
\frac{\part y}{\part t} = -2xz+2.8x-0.1y\tag{5}\\
\frac{\part z}{\part t} = 0.5xy-\frac{4}{15} z\\
\end{cases}
$$
​	注意(5)式，仅是在电路设计中的参数设置，而直接非代入所提供的MATLAB代码中，因为两者时间分类$t$的设置不一样。上述例子给我们提供电路缩放的两种思路：1.对单一参数进行倍数缩放；2.对时间变化率的调整，但对整个系统的方程同步进行。当参数倍率的缩放倍率较大后且$\dot x$，$\dot y$，$\dot z$的信号幅度较小时，可以考虑对时间变化率进行调整，同时，在遇到一些奇怪的问题时，也可以考虑调整时间变化率。
​	AnadigmDesigner2中的系统架构如下图所示：

![image-20250629133728003](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250629133728003.png)

器件参数配置如下：

| Name               | Clocks                            | Options                                                      | Parameters                                                 |
| ------------------ | --------------------------------- | :----------------------------------------------------------- | ---------------------------------------------------------- |
| Multiplier（$xy$） | ClockA 250kHz<br />ClockB 4000kHz | Sample and Hold : Off                                        | Multiplication Factor : 0.333                              |
| Multiplier（$xz$） | ClockA 250kHz<br />ClockB 4000kHz | Sample and Hold : Off                                        | Multiplication Factor : 0.333                              |
| SumDiff->$x$       | ClockA 250kHz                     | Out Phase ：Phase 1<br />Input 1 ：Non-inverting<br />Input 2 ：Inverting<br />Input 3 ：Off<br />Input 4 ：Off | Gain 1：1<br />Gain 2：1                                   |
| SumDiff->$y$       | ClockA 250kHz                     | Out Phase ：Phase 1<br />Input 1 ：Non-inverting<br />Input 2 ：Inverting<br />Input 3 ：Inverting<br />Input 4 ：Non-inverting | Gain 1：2.8<br />Gain 2：0.1<br />Gain 3：6<br />Gain 4：1 |
| SumDiff->$z$       | ClockA 250kHz                     | Out Phase ：Phase 1<br />Input 1 ：Non-inverting<br />Input 2 ：Inverting<br />Input 3 ：Off<br />Input 4 ：Off | Gain 1：1.5<br />Gain 2：0.267                             |
| Integrator->$x$    | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 2<br />Compare Control To ：No Reset | Integration Const ：0.025                                  |
| Integrator->$y$    | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 1<br />Compare Control To ：No Reset | Integration Const ：0.025                                  |
| Integrator->$z$    | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 2<br />Compare Control To ：No Reset | Integration Const ：0.025                                  |

一些设计细节问题已在前文阐述，这里不在过多赘述。

仿真结果如下图所示：

![image-20250629141924255](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250629141924255.png)



# 案例四：Rössler系统

系统方程如下：
$$
\begin{cases}
\frac{\part x}{\part t} = -(y+z)\\
\frac{\part y}{\part t} = x+ay\tag{6}\\
\frac{\part z}{\part t} = b+z(x-c)\\
\end{cases}
$$

$$
典型参数为：a=0.2,b=0.2,c=5.7
$$

MATLAB仿真的时序图和相轨图如下：

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120145536149.png" alt="image-20240120145536149"  />

​	该系统未涉及到新的内容，故不做补充。

电路架构图：
<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120151610446.png" alt="image-20240120151610446"  />

器件参数配置如下：

| Name               | Clocks                            | Options                                                      | Parameters                                        |
| ------------------ | --------------------------------- | ------------------------------------------------------------ | ------------------------------------------------- |
| Multiplier（$xz$） | ClockA 250kHz<br />ClockB 4000kHz | Sample and Hold : Off                                        | Multiplication Factor : 0.333                     |
| SumDiff->$x$       | ClockA 250kHz                     | Out Phase ：Phase 1<br />Input 1 ：Inverting<br />Input 2 ：Inverting<br />Input 3 ：Off<br />Input 4 ：Off | Gain 1：1<br />Gain 2：1                          |
| SumDiff->$y$       | ClockA 250kHz                     | Out Phase ：Phase 1<br />Input 1 ：Non-inverting<br />Input 2 ：Non-inverting<br />Input 3 ：Off<br />Input 4 ：Off | Gain 1：1<br />Gain 2：0.2                        |
| SumDiff->$z$       | ClockA 250kHz                     | Out Phase ：Phase 2<br />Input 1 ：Non-inverting<br />Input 2 ：Non-inverting<br />Input 3 ：Inverting<br />Input 4 ：Off | Gain 1 ：0.128<br />Gain 2 ：30<br />Gain 3 ：5.7 |
| Integrator->$x$    | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 2<br />Compare Control To ：No Reset | Integration Const ：0.025                         |
| Integrator->$y$    | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 1<br />Compare Control To ：No Reset | Integration Const ：0.025                         |
| Integrator->$z$    | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 2<br />Compare Control To ：No Reset | Integration Const ：0.025                         |
| GainHalf           | ClockA 250kHz                     | Polarity ：Non-inverting<br />Input Sampling Phase ：Phase 2<br /> | Gain ：0.4                                        |
| Voltage            | ClockA 250kHz                     | Polarity ：Positive（+2V）                                   |                                                   |

仿真波形如下：

<img src="https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240120152126152.png" alt="image-20240120152126152"  />

# 案例五

​	混沌系统对系统的初始值十分敏感，因此通过修改系统初始值可以使系统产生不同的动力学行为。在FPAA中由于各积分器的默认初始值为0，因此对应系统初始值不为0的混沌系统需要额外配置电路给积分器充电以到达所需要的初始值，并利用FPAA可动态配置线路的特性，将相关的芯片下载到板卡以实现初始值非0的混沌系统。

以混沌偏置系统（7）为例
$$
\begin{cases}
\dot x = 1-az\\
\dot y = -bz^2-y+d_2\tag{7}\\
\dot z =x-y+d_1\\
\end{cases}
$$

$$
a=3.9,b=3.5,d_1=-5,d_2=0\\
$$

系统初始值为$[-1- d_1 +d_2, d_2, 0]$

其相轨图和时序图如下：

![1](https://fpaa.oss-cn-shanghai.aliyuncs.com/1.png)

按照$x\rightarrow5x,y\rightarrow5y,z\rightarrow2z$调幅后，得到系统（8）
$$
\begin{cases}
\dot x = \frac{1}{5}-\frac{2az}{5}\\
\dot y = \frac{-4bz^2}{5}-\frac{y}{5}+\frac{d_2}{5}\tag{8}\\
\dot z =\frac{5x}{2}-\frac{5y}{2}+\frac{d_1}{2}\\
\end{cases}
$$

缩放后的初始值为$[0.8,0,0]$

其相轨图和时序图如下：

![2](https://fpaa.oss-cn-shanghai.aliyuncs.com/2.png)

​	按照上述思路，绘制下图。整个系统分为两个部分，积分器充电电路和混沌系统实现电路。两部分电路在CAM器件参数，类型，数量，外部连线等方面完全相等，唯一的区别在于systemB_ic与systemB_ic的内部结构上，systemB_ic电路单独独立不与systemA_ic相连且与内部的加法器相连，而systemB则相反。对于积分器充电电路可以理解为将三个积分器独立出来，对于混沌系统实现电路可以理解为将三个积分器放到另一个芯片中。FPAA的动态线路配置功能可以理解为修改FPAA的内部连线，在实际的实验过程中，如果两部分电路中CAM器件的参数，类型，数量不完全相同，会导致系统不稳定，无法出现混沌现象。因此，需要保障两部分的CAM器件完全相同。

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

其中，Gain1=$\frac{x_0}{2}$

​	通过这种方法可以使积分器的值达到并维持在所设定的初始值上。这里只展示$x$初始值的设置方式，其余变量同理。

![image-20240426102127924](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20240426102127924.png)

​	这里需要注意芯片左上角的地址信息“Adder1:x Adder2:y……LOAD ORDER:z”，由于后续程序下载的板卡的方式为逐个下载，所以此时y和z的值无需在意，需要注意的是x的值，考虑到FPAA的动态参数配置特性，需要保障积分器充电电路和混沌系统实现电路对应的芯片模块的x相同。可通过单击x，y，z来改变其值。

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

# 开发版设置

![2024040413314(1)](https://fpaa.oss-cn-shanghai.aliyuncs.com/2024040413314(1).jpg)

​	如图所示为开发版上的芯片单元。两侧I1P-O3P对应芯片的左侧四个IO口，下方IO5P-IO7N对应芯片的左侧3个IO口，下方为左侧四哥就看的额外拓展接口。搭建电路时，要根据所设计的芯片外部连接方式在开发版上对应连接。同时将输出口接入一个**减法电路**后再连接示波器，以观察波形。

需要注意的是开发版上的各开关和跳线帽要设置正确，具体可参考官方的手册。

# 案例六：混沌振荡器Sprott A

$$
\begin{cases}
\frac{\part x}{\part t} = y\\
\frac{\part y}{\part t} = ax+yz\tag{9}\\
\frac{\part z}{\part t} = b+ay^2\\
\end{cases}
$$

$$
a=-1,b=1,\text{IC}=(0.5,0,0)
$$

​	由于dpASP具有±3V的差分输出电压，超过此范围电压值将会被锁定，因此需要分别对系统进行尺度变换，使电压始终处于±3V范围。通过分析，按照对$x\to4x$，$y\to4y$，$z\to2z$系统进行缩放。

| Name                  | Options                                                      | Parameters                        | Clocks                          |
| --------------------- | ------------------------------------------------------------ | --------------------------------- | ------------------------------- |
| systemA_ic（systemA） |                                                              |                                   |                                 |
| Multiplier(yz)        | Sample and Off : Off                                         | Multiplication Factor：1.00       | ClockA:250kHz<br />ClockB:4 MHz |
| Multiplier(yy)        | Sample and Off : Off                                         | Multiplication Factor： 1.00      | ClockA:250kHz   ClockB:4 MHz    |
| SumDiff(y)            | Output Phase : Phase 2<br />Input 1 : Inverting<br />Input 2 : Non-inverting<br />Input 3 : Off<br />Input 4 : Off | Gain 1 0.50<br />Gain 2 4.00      | ClockA:250kHz                   |
| SumDiff(z)            | Output Phase : Phase 1<br />Input 1 : Inverting<br />Input 2 : Non-inverting<br />Input 3 : Off<br />Input 4 : Off | Gain 1 8.00<br />Gain 2 0.250     | ClockA :250kHz                  |
| GainHalf              | Polarity : Non-inverting<br />Input Sampling Phase : Phase 1 | Gain 1 2.00                       | ClockA :250kHz                  |
| Voltage               | Polarity Positive (+2V)                                      |                                   |                                 |
| Hold                  | Input Sampling Phase : Phase 1(2)                            |                                   | ClockA :250kHz                  |
| systemB_ic（systemB） |                                                              |                                   |                                 |
| Integrator            | Polarity : Non-inverting<br />Input Sampling Phase : Phase 2 <br />Compare Control To :  No Reset | Integration Const:0.0025  [1/us]  | ClockA :250kHz                  |
| SumDiff(x)            | Output Phase : Phase 2<br />Input 1 : Non-inverting<br />Input 2 : Inverting<br />Input 3 : Off<br />Input 4 : Off | Gain 1 : 0.125<br />Gain 2 : 1.00 | ClockA :250kHz                  |
| SumDiff(y)            | Output Phase : Phase 2<br />Input 1 : Non-inverting<br />Input 2 : Inverting<br />Input 3 : Off<br />Input 4 : Off | Gain 1 : 1.00<br />Gain 2 : 1.00  | ClockA :250kHz                  |
| SumDiff(z)            | Output Phase : Phase 2<br />Input 1 : Non-inverting<br />Input 2 : Inverting<br />Input 3 : Off<br />Input 4 : Off | Gain 1 : 1.00<br />Gain 2 : 1.00  | ClockA :250kHz                  |
| Voltage               | Polarity Positive (+2V)                                      |                                   |                                 |

FPAA架构图如下：

![image-20250629192102401](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250629192102401.png)

实验结果：

![QQ图片20250629194403](https://fpaa.oss-cn-shanghai.aliyuncs.com/QQ%E5%9B%BE%E7%89%8720250629194403.jpg)

# 案例七：洋葱型振荡器

$$
\begin{cases}
\frac{\part x}{\part t} = y\\
\frac{\part y}{\part t} = z\tag{10}\\
\frac{\part z}{\part t} = -z+ay-x^2y-bx\\
\end{cases}
$$

$$
a=9,b=5,\text{IC}=(1,-1,0)
$$

​	由于dpASP具有±3V的差分输出电压，超过此范围电压值将会被锁定，因此需要分别对系统进行尺度变换，使电压始终处于±3V范围。通过分析，按照对$x\to40x$，$y\to80y$，$z\to400z$系统进行缩放。

器件参数配置

| Name            | Options                                                      | Parameters                                                   | Clocks                       |
| --------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ---------------------------- |
| Multiplier(xx)  | Sample and Off : Off                                         | Multiplication Factor : 4.00                                 | ClockA:250kHz   ClockB:4 MHz |
| Multiplier(xxy) | Sample and Off : Off                                         | Multiplication Factor : 2.00                                 | ClockA:250kHz   ClockB:4 MHz |
| SumDiff(x)      | Output Phase : Phase 1<br />Input 1 : Non-inverting<br />Input 2 : Non-inverting    <br />Input 3 : Off    <br />Input 4 : Off | Gain 1 1.00<br />Gain 2 2.00                                 | ClockA:250kHz                |
| SumDiff(z)      | Output Phase : Phase 1<br />Input 1 : Inverting<br />Input 2 : Non-inverting<br />Input 3 : Inverting<br />Input 4 : Inverting | Gain 1 1.00<br />Gain 2 1.80<br />Gain 3 40.00<br />Gain 4 0.60 | ClockA :250kHz               |
| GainHalf        | Polarity : Non-inverting<br />Input Sampling Phase : Phase 1 | Gain 1 5.00                                                  | ClockA :250kHz               |
| Integrator      | Polarity : Non-inverting<br />Input Sampling Phase : Phase 1(2)<br />Compare Control To :  No Reset | Integration Const:0.0025  [1/us]                             | ClockA :250kHz               |

FPAA架构

![image-20250629192819649](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250629192819649.png)

仿真结果图

![image-20250629192939760](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250629192939760.png)

# 案例八：可调幅分段线性Lorenz振荡器

$$
\begin{cases}
\frac{\part x}{\part t} = y-x\\
\frac{\part y}{\part t} = -\text{sgn}(x)z\tag{11}\\
\frac{\part z}{\part t} = x\text{sgn}(y)-a\\
\end{cases}
$$

$$
a = 1,\text{IC} = (0,1,0)
$$

由于dpASP具有±3V的差分输出电压，超过此范围电压值将会被锁定，因此需要分别对系统进行尺度变换，使电压始终处于±3V范围。通过分析，按照对$x\to2x$，$y\to2y$，$z\to2z$系统进行缩放。

| Name             | Options                                                      | Parameters                        | Clocks                          |
| ---------------- | ------------------------------------------------------------ | --------------------------------- | ------------------------------- |
| systemA          |                                                              |                                   |                                 |
| Multiplier       | Sample and Off : Off                                         | Multiplication Factor：1.00       | ClockA:250kHz<br />ClockB:4 MHz |
| TransferFunction | Output Hold : On                                             | Integration Const:0.0025  [1/us]  | ClockA :250kHz                  |
| SumDiff          | Output Phase : Phase 1<br />Input 1 : Nov-inverting<br />Input 2 : Inverting<br />Input 3 : Off<br />Input 4 : Off | Gain 1 1.00<br />Gain 2 1.00      | ClockA:250kHz                   |
| SumDiff          | Output Phase : Phase 1<br />Input 1 : Inverting<br />Input 2 : Non-inverting<br />Input 3 : Off<br />Input 4 : Off | Gain 1 1.00<br />Gain 2 1.00      | ClockA :250kHz                  |
| Integrator       | Polarity : Non-inverting<br />Input Sampling Phase : Phase 1<br />Compare Control To :  No Reset | Integration Const:0.0025  [1/us]  | ClockA :250kHz                  |
| systemB          |                                                              |                                   |                                 |
| TransferFunction | Output Hold : On                                             | Integration Const:0.0025  [1/us]  | ClockA :250kHz                  |
| SumDiff          | Output Phase : Phase 2<br />Input 1 : Non-inverting<br />Input 2 : Inverting<br />Input 3 : Off<br />Input 4 : Off | Gain 1 : 1.00<br />Gain 2 : 0.250 | ClockA :250kHz                  |
| Integrator       | Polarity : Non-inverting<br />Input Sampling Phase : Phase 1<br />Compare Control To :  No Reset | Integration Const:0.0025  [1/us]  | ClockA :250kHz                  |
| Multiplier       | Sample and Off : Off                                         | Multiplication Factor：1.00       | ClockA:250kHz<br />ClockB:4 MHz |
| Voltage          | Polarity Positive (+2V)                                      |                                   |                                 |

FPAA架构图

![image-20250629193309083](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250629193309083.png)

仿真结果图

![image-20250629193323705](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250629193323705.png)

实验结果

![35e59bbb365957907767ff19d68fae4](https://fpaa.oss-cn-shanghai.aliyuncs.com/35e59bbb365957907767ff19d68fae4.bmp)

![0bd2008be607e501db4ad6bf18ceeb8](https://fpaa.oss-cn-shanghai.aliyuncs.com/0bd2008be607e501db4ad6bf18ceeb8.jpg)

# 案例九：无平衡点混沌振荡器

$$
\begin{cases}
\frac{\part x}{\part t} = -y\\
\frac{\part y}{\part t} = x+z\tag{12}\\
\frac{\part z}{\part t} = 2y^2+xz-a\\
\end{cases}
$$

$$
a=0.35,\text{IC}=(0,0.4,1)
$$

​	由于dpASP具有±3V的差分输出电压，超过此范围电压值将会被锁定，因此需要分别对系统进行尺度变换，使电压始终处于±3V范围。通过分析，按照对$x\to8x$，$y\to3y$，$z\to8z$系统进行缩放。

| Name                  | Options                                                      | Parameters                                      | Clocks                          |
| --------------------- | ------------------------------------------------------------ | ----------------------------------------------- | ------------------------------- |
| systemA_ic（systemA） |                                                              |                                                 |                                 |
| Multiplier(yy)        | Sample and Off : Off                                         | Multiplication Factor :  0.333                  | ClockA:250kHz<br />ClockB:4 MHz |
| Multiplier(xz)        | Sample and Off : Off                                         | Multiplication Factor : 2                       | ClockA:250kHz<br />ClockB:4 MHz |
| SumDiff(y)            | Output Phase : Phase 2<br />Input 1 : Non-inverting<br />Input 2 : Non-inverting<br />Input 3 : Off<br />Input 4 : Off | Gain 1 4.00<br />Gain 2 3.00                    | ClockA:250kHz                   |
| SumDiff(z)            | Output Phase : Phase 1<br />Input 1 : Non-inverting<br />Input 2 : Non-inverting<br />Input 3 : Inverting<br />Input 4 : Off | Gain 1 4.00<br />Gain 2 4.00<br />Gain 3 0.0292 | ClockA :250kHz                  |
| Voltage               | Polarity Positive (+2V)                                      |                                                 |                                 |
| Hold                  | Input Sampling Phase :  Phase 1(2)                           |                                                 | ClockA :250kHz                  |
| systemB_ic（systemB） |                                                              |                                                 |                                 |
| Integrator            | Polarity : Non-inverting<br />Input Sampling Phase : Phase 2<br />Compare Control To : No Reset | Integration Const:0.0025  [1/us]                | ClockA :250kHz                  |
| SumDiff(y)            | Output Phase : Phase 1<br />Input 1 : Non-inverting<br />Input 2 : Inverting<br />Input 3 : Off<br />Input 4 : Off | Gain 1 : 0.10<br />Gain 2 : 1.00                | ClockA :250kHz                  |
| SumDiff(z)            | Output Phase : Phase 1<br />Input 1 : Non-inverting<br />Input 2 : Inverting<br />Input 3 : Off<br />Input 4 : Off | Gain 1 : 0.0625<br />Gain 2 : 1.00              | ClockA :250kHz                  |
| GainHalf              | Polarity : Inverting<br />Input Sampling Phase : Phase 1     | Gain 1 0.250                                    | ClockA :250kHz                  |
| Voltage               | Polarity Positive (+2V)                                      |                                                 |                                 |

FPAA架构

![image-20250629193612380](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250629193612380.png)

实验

![image-20250629193626052](https://fpaa.oss-cn-shanghai.aliyuncs.com/image-20250629193626052.png)

# 版本修改注释

- 初版2024/1/8
- 第一次修改2024/3/29
  - 增加设置非零初始值
  - 实际开发版使用方式
  - 修改一些错误
- 第二次修改2024/6/16
  - 统一文字格式和表述方式

第三次修改2025/7/27
	大修，包括整体格式和描述风格，配图等。
