---
title: "HAL库定时器初始化逻辑"
description: "关于HAL库时钟初始化解析"
date: 2026-08-25T10:01:18+08:00
draft: False
tags: [STM32]
---
本文内容基于STM32CubeMX+Keil5的HAL库环境 
## STM32时钟树逻辑
- 外部时钟源开启在SystemCore的RCC，若是晶振选择`Crystal/Ceramic Resonator`
- LSI除了要选择`Crystal/Ceramic Resonator`之外还需在`RTC`里使能RTC![[Pasted image 20260825093819.png]]
- ![[Pasted image 20260825093751.png]]
- CubeMX在上方Clock Configuration可以看到时钟树
- 选择HSE/HSI[^HSE/HSI]时钟源，可进行PLL锁相环调频,得到SYSCLK，再经过AHP Prescaler[^Prescaler]分频得到AHB总线时钟HCLK，最后为各个外设配置时钟。在CubeMX我们一般只需选择需要的时钟源，填写HSE频率,填写所需HCLK频率即可
- 若有RTC功能,可选择LSE/LSI/HSE做RTC时钟源
	![[Pasted image 20260825094202.png]]

## 配置定时器
- STM32F1的定时器分类，根据所需功能选择定时器种类![[Pasted image 20260825094520.png]]
- 
### 配置基本定时器
- 下面以1ms定时中断为例配置一个基本定时器
- 这里选择TIM6，勾选Activated
- 由于有中断需求，在NVIC Settings[^NVIC]勾选Enable使能中断
- ![[Pasted image 20260825094902.png]]
- 配置定时器定时时间
- 主要在于PSC和auto-reload preload(ARR)，即预分频器值和自动重载值
- ![[Pasted image 20260825103216.png]]
- ![[Pasted image 20260825100933.png]]
- 查手册可以知道，除了高级控制定时器，剩下的定时器时钟源来自APB1时钟
- ![[Pasted image 20260825101405.png]]
- 我们配置的是TIM6，所以要找APB2 timer clocks
- ![[Pasted image 20260825101507.png]]
- 那么公式是 定时频率 = (自动重载值+1)/（时钟源/(预分频器值+1))
- 即(写方便观看的公式
- 那么 1ms需要1000Hz的频率，设置好PSC和ARR配置就好了
## 基本定时器初始化逻辑
- 对于CubeMX生成的代码主要过程是`SystemClock_Config`后 `MX_TIM6_Init`调用`HAL_TIM_Base_Init`调用`TIM_Base_SetConfig`设置一系列参数，然后`HAL_TIM_Base_MspInit`设置`NVIC`,最后HAL_TIM_Base_Start_IT启用中断callback
- 下面是MX_TIM6_Init代码
```c
TIM_HandleTypeDef htim6;//timer6句柄

void MX_TIM6_Init(void)
{
  TIM_MasterConfigTypeDef sMasterConfig = {0};//配置TRGO的结构体

  //配置timer6结构体
  htim6.Instance = TIM6;
  htim6.Init.Prescaler = 0;
  htim6.Init.CounterMode = TIM_COUNTERMODE_UP;
  htim6.Init.Period = 16000-1;
  htim6.Init.AutoReloadPreload = TIM_AUTORELOAD_PRELOAD_ENABLE;

  //调用HAL_TIM_Base_Init初始化timer6
  if (HAL_TIM_Base_Init(&htim6) != HAL_OK)
  {
    Error_Handler();
  }

  //配置TRGO结构体
  sMasterConfig.MasterOutputTrigger = TIM_TRGO_RESET;
  sMasterConfig.MasterSlaveMode = TIM_MASTERSLAVEMODE_DISABLE;

//初始化TRGO
  if (HAL_TIMEx_MasterConfigSynchronization(&htim6, &sMasterConfig) != HAL_OK)
  {
    Error_Handler();
  }
}
```
### HAL_TIM_Base_Init

- 在HAL_TIM_Base_Init里，先判断了定时器状态是否为RESET,随后unlocked，这里的锁应该是为多任务操作定时器准备的
```c
  if (htim->State == HAL_TIM_STATE_RESET)
  {
	  htim->Lock = HAL_UNLOCKED;
	  
	  HAL_TIM_Base_MspInit(htim);
	  .......
```
#### HAL_TIM_Base_MspInit
##### #### NVIC_Init与#### RCC_Enable
- 随后调用MspInit开启时钟，初始化NVIC。NVIC的配置有些复杂不展开
```c
  __HAL_RCC_TIM6_CLK_ENABLE();//
    /* TIM6 interrupt Init */
    HAL_NVIC_SetPriority(TIM6_DAC_IRQn, 0, 0);
    HAL_NVIC_EnableIRQ(TIM6_DAC_IRQn);
```

```c


#define __HAL_RCC_TIM6_CLK_ENABLE()   do { \
                                           __IO uint32_t tmpreg = 0x00U; \
                                           SET_BIT(RCC->APB1ENR, RCC_APB1ENR_TIM6EN);\
                                           /* Delay after an RCC peripheral clock enabling */ \
                                           tmpreg = READ_BIT(RCC->APB1ENR, RCC_APB1ENR_TIM6EN);\
                                           UNUSED(tmpreg); \
                                         } while(0U)
```
- `__HAL_RCC_TIM6_CLK_ENABLE`是一个宏，做了使能timer6的en,然后读取。
```c
#define SET_BIT(REG, BIT)     ((REG) |= (BIT))

#define RCC                 ((RCC_TypeDef *) RCC_BASE)

#define RCC_APB1ENR_TIM6EN_Pos             (4U)
#define RCC_APB1ENR_TIM6EN_Msk             (0x1UL << RCC_APB1ENR_TIM6EN_Pos)    /*!< 0x00000010 */
#define RCC_APB1ENR_TIM6EN                 RCC_APB1ENR_TIM6EN_Msk

SET_BIT(RCC->APB1ENR, RCC_APB1ENR_TIM6EN);
#define RCC                 ((RCC_TypeDef *) RCC_BASE)
```
- 下面解析一下SET_BIT干了什么，RCC是一个结构体指针，指向RCC_BASE，其中的成员APB1ENR被拿来和RCC_APB1ENR_TIM6EN做或运算，其实就是和1<<4做或运算
![[Pasted image 20260825130452.png]]

- 由手册中RCC_APB1ENR可见，该或运算就是把寄存器第4位TIM6EN置1，即开启tim6的时钟
![[Pasted image 20260825131210.png]]

![[Pasted image 20260825131646.png]]

#### Set TIM state
```c
  /* Set the TIM state */
  htim->State = HAL_TIM_STATE_BUSY;
  /* Init the base time for the Output Compare */
  TIM_Base_SetConfig(htim->Instance,  &htim->Init);

```

这里先把定时器状态设为Busy后开始调用TIM_Base_SetConfig
```c
TIM_Base_SetConfig(htim->Instance, &htim->Init);
```
传入了TIM寄存器结构体，TIM初始化结构体
```c
 if (IS_TIM_COUNTER_MODE_SELECT_INSTANCE(TIMx))
  {
    /* Select the Counter Mode */
    tmpcr1 &= ~(TIM_CR1_DIR | TIM_CR1_CMS);
    tmpcr1 |= Structure->CounterMode;
  }

  if (IS_TIM_CLOCK_DIVISION_INSTANCE(TIMx))
  {
    /* Set the clock division */
    tmpcr1 &= ~TIM_CR1_CKD;
    tmpcr1 |= (uint32_t)Structure->ClockDivision;
  }

  /* Set the auto-reload preload */
  MODIFY_REG(tmpcr1, TIM_CR1_ARPE, Structure->AutoReloadPreload);

  /* Set the Autoreload value */
  TIMx->ARR = (uint32_t)Structure->Period ;

  /* Set the Prescaler value */
  TIMx->PSC = Structure->Prescaler;

  if (IS_TIM_REPETITION_COUNTER_INSTANCE(TIMx))
  {
    /* Set the Repetition Counter value */
    TIMx->RCR = Structure->RepetitionCounter;
  }
```
这里就在配置定时器了，往定时器结构体对应的寄存器写入对应值。其中if里面的语句是在判断该定时器是否有对应功能,有才往下配置免得配错不该配的寄存器

```c
  /* Initialize the DMA burst operation state */
  htim->DMABurstState = HAL_DMA_BURST_STATE_READY;//把DMA软件层面置ready，我们没有配置
  /* Initialize the TIM channels state */
  TIM_CHANNEL_STATE_SET_ALL(htim, HAL_TIM_CHANNEL_STATE_READY);
  TIM_CHANNEL_N_STATE_SET_ALL(htim, HAL_TIM_CHANNEL_STATE_READY);
```
`TIM_CHANNEL_STATE_SET_ALL`应该是在把主通道
软件层面置位ready，其中多了N的TIM_CHANNEL_N_STATE_SET_ALL函数是把互补通道软件层面的数组置ready。互补通道只有TIM1,TIM8有。所以这里只是统一的软件层面的置ready无硬件意义，也就是没有配置对应寄存器。
```c
#define TIM_CHANNEL_STATE_SET_ALL(__HANDLE__,  __CHANNEL_STATE__)\

  do {\

    (__HANDLE__)->ChannelState[0]  = (__CHANNEL_STATE__);\

    (__HANDLE__)->ChannelState[1]  = (__CHANNEL_STATE__);\

    (__HANDLE__)->ChannelState[2]  = (__CHANNEL_STATE__);\

    (__HANDLE__)->ChannelState[3]  = (__CHANNEL_STATE__);\

  } while(0)
```
最后把定时器从Busy改为Ready，就可以返回HAL_OK了，HAL_TIM_Base_Init就完成了。
```c
  /* Initialize the TIM state*/
  htim->State = HAL_TIM_STATE_READY;
  return HAL_OK;
```
### TRGO初始化
- 随后Msp_Init结束返回，准备TRGO_Init
- ```c
  sMasterConfig.MasterOutputTrigger = TIM_TRGO_RESET;
  sMasterConfig.MasterSlaveMode = TIM_MASTERSLAVEMODE_DISABLE;

  if (HAL_TIMEx_MasterConfigSynchronization(&htim6, &sMasterConfig) != HAL_OK)
  {
    Error_Handler();
  }
  ```
接着结束了定时器的初始化,记得调用HAL_TIM_Base_Start_IT，这样才能接受到中断回调
```c
  if (HAL_TIM_Base_Start_IT(&htim6) != HAL_OK)
  {
    Error_Handler();
  }
```

## 总结
......

[^HSE/HSI]:  HSE代表高速外部时钟(High Speed External),HSI代表高速内部时钟(High Speed Internal)
[^Prescaler]:  Prescaler代表预分频器，即时钟输出=输入源/预分频器值
[^NVIC]: 用于中断分组，设置抢占优先级和响应优先级。有多个中断时要注意