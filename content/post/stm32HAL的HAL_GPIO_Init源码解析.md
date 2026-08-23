---
title: "STM32 HAL 的 HAL_GPIO_Init 源码解析"
description: ""
date: 2026-08-23T18:43:21+08:00
draft: false
tags: ["STM32"]
categories: []
---

## 函数原型

```c
void HAL_GPIO_Init(GPIO_TypeDef  *GPIOx, GPIO_InitTypeDef *GPIO_Init)
```

## 函数参数

- `GPIOx`：GPIO 端口，`x` 取 `A`~`G`。
- `GPIO_Init`：`GPIO_InitTypeDef` 结构体指针；成员有 `Pin`、`Mode`、`Pull`、`Speed`，分别表示引脚号、模式、上下拉、引脚翻转速度。

```c
typedef struct
{
  uint32_t Pin;
  uint32_t Mode;
  uint32_t Pull; 
  uint32_t Speed; 
} GPIO_InitTypeDef;
```

## 相关变量

这些变量的具体作用，后面遇到时再说明

```c
  uint32_t position = 0x00u;
  uint32_t ioposition;
  uint32_t iocurrent;
  uint32_t temp;
  uint32_t config = 0x00u;
  __IO uint32_t *configregister;
  uint32_t registeroffset; 
```

## assert_param 断言

接下来是 `assert_param` 断言 ，

```c
assert_param(IS_GPIO_ALL_INSTANCE(GPIOx));
assert_param(IS_GPIO_PIN(GPIO_Init->Pin));
assert_param(IS_GPIO_MODE(GPIO_Init->Mode));
```

```c
#ifdef  USE_FULL_ASSERT
/**
  * @brief  The assert_param macro is used for function's parameters check.
  * @param  expr If expr is false, it calls assert_failed function
  *         which reports the name of the source file and the source
  *         line number of the call that failed.
  *         If expr is true, it returns no value.
  * @retval None
  */
#define assert_param(expr) ((expr) ? (void)0U : assert_failed((uint8_t *)__FILE__, __LINE__))
/* Exported functions ------------------------------------------------------- */
void assert_failed(uint8_t* file, uint32_t line);
#else
#define assert_param(expr) ((void)0U)
#endif /* USE_FULL_ASSERT */
```

这里用 `#ifdef` / `#else` 判断 `USE_FULL_ASSERT` 是否被定义，从而切换 `assert_param` 的两种实现：

- 默认未定义 `USE_FULL_ASSERT`：`assert_param` 是空宏，不做任何检查。
- 取消注释 `USE_FULL_ASSERT` 后：`assert_param` 会检查传入表达式是否成立；不成立时调用 `assert_failed`，并传入 `__FILE__`、`__LINE__`。
- `assert_failed` 需要自己定义，可以用串口打印 `__FILE__`、`__LINE__`，用于定位运行过程中出错的位置。

## while 遍历 Pin

```c
 while (((GPIO_Init->Pin) >> position) != 0x00u)
 {
    ......
    position++;
 }
```

这段 `while` 的逻辑是逐位判断 `GPIO_Init->Pin` 中哪些 Pin 被配置：

1. `position` 从 0 开始，每轮循环加 1，将`GPIO_Init->Pin`右移表示当前正在检查第 `position` 位。
2. `ioposition` 是将 1 左移 `position` 位得到的掩码，只保留第 `position` 位为 1。
3. `iocurrent = GPIO_Init->Pin & ioposition`，做与操作取出 `GPIO_Init->Pin` 在第 `position` 位的值。
4. 如果 `iocurrent == ioposition`，说明该位为 1，也就是对应 Pin 需要配置。
5. `while` 继续循环，直到 `(GPIO_Init->Pin) >> position` 为 0，即不再有更高的有效位需要检查。


### 具体例子
```c
#define GPIO_PIN_0                 ((uint16_t)0x0001)
#define GPIO_PIN_1                 ((uint16_t)0x0002)
#define GPIO_PIN_2                 ((uint16_t)0x0004)
#define GPIO_PIN_3                 ((uint16_t)0x0008)
#define GPIO_PIN_4                 ((uint16_t)0x0010)
#define GPIO_PIN_5                 ((uint16_t)0x0020)
#define GPIO_PIN_6                 ((uint16_t)0x0040)
#define GPIO_PIN_7                 ((uint16_t)0x0080)
#define GPIO_PIN_8                 ((uint16_t)0x0100)
#define GPIO_PIN_9                 ((uint16_t)0x0200)
#define GPIO_PIN_10                ((uint16_t)0x0400)
#define GPIO_PIN_11                ((uint16_t)0x0800)
#define GPIO_PIN_12                ((uint16_t)0x1000)
#define GPIO_PIN_13                ((uint16_t)0x2000)
#define GPIO_PIN_14                ((uint16_t)0x4000)
#define GPIO_PIN_15                ((uint16_t)0x8000)
#define GPIO_PIN_All               ((uint16_t)0xFFFF)
```
上面是GPIO对应PIN的定义

`GPIO_Init->Pin` 可以通过按位或 `|` 一次配置多个引脚，因此取值范围是 `0x0001H` 到 `0xFFFFH`,或者用二进制表示`0000 0000 0000 0001B`到`1111 1111 1111 1111B`。[^H_B]

例如：

```c
GPIO_Init->Pin = GPIO_PIN_0 | GPIO_PIN_3;   // 0x0009H = 0000 0000 0000 1001B
```

这表示配置位 0 和位 3，也就是 `GPIO_PIN_0` 和 `GPIO_PIN_3`。循环内部每次执行：

```c
while (((GPIO_Init->Pin) >> position) != 0x00u)
{
    ioposition = 0x01u << position;
    iocurrent = (GPIO_Init->Pin) & ioposition;

    if (iocurrent == ioposition)
    {
        // 配置当前位对应的 GPIO_PIN_x
    }

    position++;
}
```
下面省去二进制高12位

| 轮次 | `position` | `(Pin >> position)` | `ioposition` | `iocurrent` | 判断结果 |
| ---: | ---: | ---: | ---: | ---: | --- |
| 第 1 轮 | 0 | `1001B` | `0001B` | `0001B` | 相等，配置 `GPIO_PIN_0` |
| 第 2 轮 | 1 | `0100B` | `0010B` | `0000B` | 不等，跳过 |
| 第 3 轮 | 2 | `0010B` | `0100B` | `0000B` | 不等，跳过 |
| 第 4 轮 | 3 | `0001B` | `1000B` | `1000B` | 相等，配置 `GPIO_PIN_3` |
| 第 5 次 | 4 | `0000B` | 不进入循环 | 不进入循环 | `while` 条件为假，退出 |

所以 `1001B` 会配置 `GPIO_PIN_0` 和 `GPIO_PIN_3`



## 为寄存器赋值

> 这里先跳过 `if` 里的内容不讲。

### 根据 GPIO_Init->Mode 为 config 赋值

虽然 `config` 是 `uint32_t`，但赋值时只用了低 4 位。

要解释原因，先看手册里的 GPIO 寄存器 `CRL` 和 `CRH`[^CRL_CRH]。

![STM32 GPIO 寄存器 CRL](/images/stm32_hal_gpio/image-7.png)

![STM32 GPIO 寄存器 CRH](/images/stm32_hal_gpio/image-8.png)

`GPIO_PIN` 有 0\~15 共 16 个引脚，分成低 8 个和高 8 个。

以低 32 位寄存器 `CRL` 为例：每个 GPIO 引脚占用 4 位，其中 `MODE` 和 `CNF` 各占 2 位，因此 `CRL` 共能配置 `32 / 4 = 8` 个引脚，也就是 `GPIO_PIN_0`\~`GPIO_PIN_7`；`CRH` 同理管理 `GPIO_PIN_8`\~`GPIO_PIN_15`。

一次循环里最多配置一个引脚，所以最多只改变一个寄存器里的 4 位。因此 `config` 赋值时只需赋值低 4 位,这也方便了后续的移位配置。

```c
    configregister = (iocurrent < GPIO_PIN_8) ? &GPIOx->CRL : &GPIOx->CRH;
    registeroffset = (iocurrent < GPIO_PIN_8) ? (position << 2u) : ((position - 8u) << 2u);
```

- `configregister` 是 `uint32_t *`，表示接下来配置的是 `CRL` 还是 `CRH`。
- `registeroffset` 是 `uint32_t`，表示要写入的寄存器偏移量。

`position` 代表偏移了多少位取值0~15。`<<2u`就是乘4。小于8就直接乘4，否则-8再乘4。算出偏移量。
 - 原理是每个GPIO占4位。![GPIO 寄存器位分配示意](/images/stm32_hal_gpio/image-9.png)





```c
MODIFY_REG((*configregister), ((GPIO_CRL_MODE0 | GPIO_CRL_CNF0) << registeroffset), (config << registeroffset));
```

```c
#define WRITE_REG(REG, VAL)   ((REG) = (VAL))
#define READ_REG(REG)         ((REG))
#define MODIFY_REG(REG, CLEARMASK, SETMASK)  WRITE_REG((REG), (((READ_REG(REG)) & (~(CLEARMASK))) | (SETMASK)))
```

先看 `MODIFY_REG` 展开后的核心部分：

```c
WRITE_REG((REG), (((READ_REG(REG)) & (~(CLEARMASK))) | (SETMASK)))
```

等价于：

```c
REG = (REG & (~CLEARMASK)) | SETMASK;
```

- `REG & (~CLEARMASK)`：先按 `CLEARMASK` 把要修改的位清零。
- 再与 `SETMASK` 做按位或，把需要置 1 的位写进去。
- 也就是 `&0` 后 `|1`

### 例子

| 项目 |  16 位二进制 |
| --- | --- |
| `REG` | `1000 1010 0000 1011B` |
| `CLEARMASK` | `0000 1111 0000 0000B` |
| `~CLEARMASK` | `1111 0000 1111 1111B` |
| `SETMASK` | `0000 0011 0000 0000B` |
| `REG & (~CLEARMASK)` | `1000 0000 0000 1011B` | 
| `NewREG` | `1000 0011 0000 1011B` |


```c
MODIFY_REG((*configregister), ((GPIO_CRL_MODE0 | GPIO_CRL_CNF0) << registeroffset), 
(config << registeroffset));
```
- 所以这里意思就是configregister决定CRL or CRH , (GPIO_CRL_MODE0 | GPIO_CRL_CNF0) `<<` registeroffset决定配置哪个GPIO的MODE 和 CNF ， config `<<` registeroffset决定配置模式(0000B~1111B)

## 引脚复用(AFIO)

STM32F1 系列中，AFIO 的 `EXTICR` 寄存器决定每个 `EXTI` 线路选择哪个 GPIO 端口。每个 `EXTICR` 为 16 位，每 4 位选择一个端口：

- `EXTICR1`：负责 `0~3`
- `EXTICR2`：负责 `4~7`
- `EXTICR3`：负责 `8~11`
- `EXTICR4`：负责 `12~15`

![AFIO EXTICR 说明图 1](/images/stm32_hal_gpio/afio-image.png)
![AFIO EXTICR 说明图 2](/images/stm32_hal_gpio/afio-image-1.png)
![AFIO EXTICR 说明图 3](/images/stm32_hal_gpio/afio-image-2.png)
![AFIO EXTICR 说明图 4](/images/stm32_hal_gpio/afio-image-3.png)

开启 AFIO 时钟：

```c
/* Enable AFIO Clock */
__HAL_RCC_AFIO_CLK_ENABLE();
```

```c
temp = AFIO->EXTICR[position >> 2u];//position/4  决定是哪个EXTICRx
CLEAR_BIT(temp, (0x0Fu) << (4u * (position & 0x03u)));

#define CLEAR_BIT(REG, BIT)   ((REG) &= ~(BIT))

// position & 0x03u 相当于 position % 4，得到当前 PIN 在 EXTICRx 中的第几个 4 位，再乘 4 得到偏移量。

SET_BIT(temp, (GPIO_GET_INDEX(GPIOx)) << (4u * (position & 0x03u)));
//GPIO_GET_INDEX获取对应GPIO字母后将对应配置写入temp
#define SET_BIT(REG, BIT)     ((REG) |= (BIT))

AFIO->EXTICR[position >> 2u] = temp;//用temp值配置对应的EXTICRx的EXTIx
```

[^CRL_CRH]: `CRL` 用于配置 `GPIO_PIN_0`\~`GPIO_PIN_7`，`CRH` 用于配置 `GPIO_PIN_8`\~`GPIO_PIN_15`。


[^H_B]: `H` 表示十六进制（Hexadecimal），`B` 表示二进制（Binary）。
