# 一、GPT模块简介
## 1.1 术语解释
GPT模块位于微控制器抽象层（MCAL），负责初始化并控制MCU内部通用定时器（General Purpose Timer）外设。<p>
## 1.2 GPT模块提供的服务
* 开启或关闭硬件定时器
* 获取定时器值（当前计数值、剩余计数值）
* 控制定时器触发中断通知（若硬件支持）
* 控制定时器触发唤醒中断（若硬件支持）
## 1.3 模块细节
1. 定时器计数时间取决于两方面
    + **Tick值**，此值由GPT模块控制,是定时器的向上计数上限或向下计数下限
    + **Tick间隔**，Tick间隔由GPT模块和MCU模块共同控制,其中在GPT模块中对定时器通道的时钟源进行选择，MCU模块则可以配置时钟源的时钟频率。
2. 并非所有定时器硬件都必须被GPT模块控制，一些定时器外设可能被AUTOSAR操作系统控制或被复杂驱动控制，被GPT模块控制的定时器通道的数量取决于硬件，MCAL软件代码实现和系统配置。
3. GPT模块除了可以配置单个定时器通道的属性外，还定义了一些自由运行的计数器 —— **预定义定时器（GPT Predef Timers）**。这些定时器被预定义了Tick值和Tick间隔，即定时的物理时间。预定义定时器由Time Service模块使用。
4. GPT驱动仅用于产生时基，定时器外设的一些更高级的功能由MCAL的其他模块控制。
   **eg:**
    - PWM驱动（脉宽调制 pulse width modulation）
    - ICU驱动（输入捕捉 input capture unit）
    - OCU驱动（输出比较 output compare unit）
5. GPT通道和定时器通道。两者实际是一个东西，定时器通道是定时器外设中的实际通道，GPT通道是逻辑上的通道，通道号是逻辑值。
# 二、GPT模块API
## 2.1 AUTOSAR规定API列表
|API|功能|
|:--|:--|
|`Gpt_GetVersionInfo`|获取GPT模块版本信息|
|`Gpt_Init`|初始化GPT模块|
|`Gpt_DeInit`|GPT模块反初始化|
|`Gpt_GetTimeElapsed`|获取当前计数值|
|`Gpt_GetTimeRemaining`|获取剩余计数值|
`Gpt_StartTimer`|启动定时器通道开始计时
`Gpt_StopTimer`|`停止定时器通道计时
`Gpt_EnableNotification`|使能定时器通道中断通知
`Gpt_DisableNotification`|禁用定时器通道中断通知
`Gpt_SetMode`|设置GPT模块模式
`Gpt_DisableWakeup`|禁用定时器通道的唤醒中断
`Gpt_EnableWakeup`|使能定时器通道的唤醒中断
`Gpt_CheckWakeup`|检查唤醒源
`Gpt_GetPredefTimerValue`|获取预定义定时器当前值
## 2.2 API详细解读
### 2.2.1 Gpt_GetVersionInfo
#### 2.2.1.1 函数基本信息表
|||
:-|:-
函数声明|`void Gpt_GetVersionInfo(Std_VersionInfoType* VersionInfoPtr)`
描述|读取模块版本信息
函数id|0x00
同步\异步|同步
是否可重入|可重入
输入参数|None
输出参数|**VersionInfoPtr**，指向存储GPT模块版本信息的指针
返回值|None
#### 2.2.1.2 函数细节
* 若使能开发错误检测，且当函数传入指针为空指针时，会调用DET模块函数`Det_ReportError`上报错误`GPT_E_PARAM_POINTER`
### 2.2.2 Gpt_Init
#### 2.2.2.1 函数基本信息表
|||
:-|:-
函数声明|`void Gpt_Init(const Gpt_ConfigType* ConfigPtr)`
描述|初始化GPT模块
函数id|0x01
同步\异步|同步
是否可重入|不可重入
输入参数|**ConfigPtr**，指向初始化配置结构体的指针
输出参数|None
返回值|None
#### 2.2.2.2 函数细节
* 此函数可根据**ConfigPtr**指向的配置集对定时器外设进行初始化
* 此函数会禁用所有GPT模块控制的中断通知
* 此函数会禁用所有GPT模块控制的唤醒中断
* 此函数只初始化可配置资源，对配置文件中不可配置的资源不进行初始化
* 若使能开发错误检测(`STD_ON == GPT_DEV_ERROR_DETECT`)，且调用此函数时GPT模块不处于未初始化状态，会调用DET模块函数`Det_ReportError`上报错误`GPT_E_ALREADY_INITIALIZED`
* 此函数将GPT模块运行模式置为`normal mode`，效果等同于调用函数`Gpt_SetMode`,并传参为`GPT_MODE_NORMAL`
* 若想调用此函数对已完成初始化的GPT模块重新进行初始化，应先调用函数`Gpt_DeInit`进行反初始化
* 此函数将开启所有已使能的预定义定时器，并使之从0开始计时
### 2.2.3 Gpt_DeInit
#### 2.2.3.1 函数基本信息表
|||
:-|:-
函数声明|`void Gpt_DeInit(void)`
描述|GPT模块反初始化
函数id|0x02
同步\异步|同步
是否可重入|不可重入
输入参数|None
输出参数|None
返回值|None
#### 2.2.3.2 函数细节
* 此函数对GPT模块控制所有定时器外设进行反初始化，使之恢复到上电复位状态，不可写的寄存器除外
* 此函数会禁用所有GPT模块控制的中断通知和唤醒中断
* 此函数只对由静态配置分配的外设起作用
* 如果配置方式选择PB，且调用过`Gpt_Init`函数，此函数将会对`Gpt_Init`函数初始化的外设进行反初始化
* 此函数受宏`GptDeInitApi`控制，仅当`STD_ON == GptDeInitApi`时此函数可用
* 此函数将GPT模块运行状态置为**uninitialized**未初始化状态
* 当调用此函数时，仍存在定时器通道处于**running**运行状态，此函数将调用DET模块函数`Det_ReportRuntimeError`上报运行错误`GPT_E_BUSY`,此种错误属于运行错误，不需要使能开发错误检测
* 若使能开发错误检测，且调用此函数时GPT模块处于未初始化状态，会调用DET模块函数`Det_ReportError`上报错误`GPT_E_UNINIT`
* 此函数将使所有已使能的预定义定时器停止计数
### 2.2.4 Gpt_GetTimeElapsed
#### 2.2.4.1 函数基本信息表
|||
:-|:-
函数声明|`Gpt_ValueType Gpt_GetTimeElapsed(Gpt_ChannelType Channel)`
描述|获取指定定时器通道当前计数值
函数id|0x03
同步\异步|同步
是否可重入|可重入
输入参数|**Channel**，要查询的GPT通道（即定时器通道的逻辑通道号）
输出参数|None
返回值|**Gpt_ValueType**，定时器通道的当前计数值（即以Tick为单位的计时时间）
#### 2.2.4.2 函数细节
* 函数返回值为定时器通道的当前计数值，是相对于该通道**最近一次开始计时**事件的**当前值**，无论该通道计时模式为**one-shot mode**单次计时模式，还是**continuous mode**连续计时模式（一次计时到时后，计数寄存器立即清零开启下一次计时）此规则均适用。
* 当要查询的定时器通到处于**initialized**仅初始化并未开始计时状态时，函数返回值为0
* 此函数是可重入的，并且完全可重入，即使传入参数为同一个通道
* 此函数受宏`GptTimeElapsedApi`控制，仅当`STD_ON == GptTimeElapsedApi`时此函数可用
* 若使能开发错误检测，且调用此函数时GPT模块处于未初始化状态，会调用DET模块函数`Det_ReportError`上报错误`GPT_E_UNINIT`
* 若使能开发错误检测，且传入参数GPT通道无效时，会调用DET模块函数`Det_ReportError`上报错误`GPT_E_PARAM_CHANNEL`
### 2.2.5 Gpt_GetTimeRemaining
#### 2.2.5.1 函数基本信息表
|||
:-|:-
函数声明|`Gpt_ValueType Gpt_GetTimeRemaining(Gpt_ChannelType Channel)`
描述|获取指定定时器通道剩余计数值
函数id|0x04
同步\异步|同步
是否可重入|可重入
输入参数|**Channel**，要查询的GPT通道（即定时器通道的逻辑通道号）
输出参数|None
返回值|**Gpt_ValueType**，定时器通道的剩余计数值（即以Tick为单位的剩余计时时间）
#### 2.2.5.2 函数细节
* 函数返回值为定时器通道的剩余计数值，即为目标计数值减去当前计数值。
* 定时器通道处于**initialized**仅初始化并未开始计时状态时，返回值为0；定时器通道处于**stopped**停止计时状态时，返回值为定时器通道停止计时时的剩余计数值；定时器通道处于**expired**达到目标计数值状态，且定时器通道被配置为**one-shot mode**单次计时模式时，返回值为0
* 此函数是可重入的，并且完全可重入，即使传入参数为同一个通道
* 此函数受宏`GptTimeRemainingApi`控制，仅当`STD_ON == GptTimeRemainingApi`时此函数可用
* 若使能开发错误检测，且调用此函数时GPT模块处于未初始化状态，会调用DET模块函数`Det_ReportError`上报错误`GPT_E_UNINIT`
* 若使能开发错误检测，且传入参数GPT通道无效时，会调用DET模块函数`Det_ReportError`上报错误`GPT_E_PARAM_CHANNEL`
### 2.2.6 Gpt_StartTimer
#### 2.2.6.1 函数基本信息表
|||
:-|:-
函数声明|`void Gpt_StartTimer(Gpt_ChannelType Channel,Gpt_ValueType Value)`
描述|指定GPT通道开始计时
函数id|0x05
同步\异步|同步
是否可重入|可重入(对同一GPT通道不可重入)
输入参数|**Channel**，要启动计时的GPT通道（即定时器通道的逻辑通道号）
输入参数|**Value**，目标计数值（即以Tick为单位的计时时间）
输出参数|None
返回值|None
#### 2.2.6.2 函数细节



