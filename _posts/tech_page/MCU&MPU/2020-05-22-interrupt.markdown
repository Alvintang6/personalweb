---
layout: tech_post
title:  "Interrupt of ARM A7"
date:   2020-05-22 11:51:36 -0300
catalogue: Mcu&Mpu
tags: ARM interrupt bare-metal
description: 
---

* TOC
{:toc}


### 中断程序逻辑


```C
1. 在汇编start程序中初始化中断向量表，定义复位中断 Reset_handler初始化系统，在复位中断的最后跳转入C语言main函数。
2. main函数中初始化中断引角(在irq_table中注册中断函数, 设置并使能该引角中断)，初始其他外设并执行程序大循环。
3. 定义其他中断(如irq中断)，在irq中断中跳转至irq中断处理C语言函数SystemIrqHandler
4. irq中断处理函数中会调用已经注册的中断函数irqTable[intNum].irqHandler();
5. 程序中断返回后会回到主函数main中继续执行
```

一般情况下在汇编中初始化中断向量表，定义不同的中断函数的操作都是相同的。不同场景下主要需要修改的是中断处理函数和主函数中中断引角和外设的初始化。

### GIC 中断控制器

GIC general interrupt controller

关于GIC有两组寄存器GICD(GIC分发器)和GICC(CPU接口寄存器)，
分发器(distributor)用于管理CPU所有中断源，确定每个中断的优先级，管理中断的屏蔽和中断抢占。最终将优先级最高的中断转发到一个或者多个CPU接口

GIC 的基地址保存在CP15协处理器中
对于cp15协处理器中的数据需要使用mrc指令进行读取。  
<B>mrc {cond} p15, \<Opcode_1>, \<Rd>, \<CRn>, \<CRm>, \<Opcode_2>  </B>

其中GIC的基地址在c15 4 c0中
GICC相对于GIC偏移了0x2000, GICC_IAR寄存器相对于GICC的基地址又偏移了0xC
<div align=center><img width = '60%' height ='60%' src ="/blog_photos/MPU/GIC_offset.png"/><p>GIC_offset</p></div>

|地址偏移 | 寄存器名字 | 类型 | 复位值 | 寄存器描述|
|----|---|---|---|----|
|0x000C|GICC_IAR|RO |0x000003FF | 中断确认寄存器|


GICC_IAR : 中断确认寄存器 Interrupt acknowledge register [31:13]:reserved, [12:10]:CPUID, [9:0]:interrupt ID。
中断id用于和xx表匹配相应的中断函数。



### 1. 中断向量表以及复位中断初始化

```c

.text
.align 2         //设置字节对齐
.global _start
_start:

    ldr     pc, =Reset_Handler           /* Reset                  */
    ldr     pc, =Undefined_Handler       /* Undefined instructions */
    ldr     pc, =SVC_Handler             /* Supervisor Call        */
    ldr     pc, =PrefAbort_Handler       /* Prefetch abort         */
    ldr     pc, =DataAbort_Handler       /* Data abort             */
    .word   0                            /* RESERVED               */
    ldr     pc, =IRQ_Handler             /* IRQ interrupt          */
    ldr     pc, =FIQ_Handler             /* FIQ interrupt          */



/**********************第三部分*********************/
Reset_Handler:
    cpsid   i                         /* 全局关闭中断 */


    mrc     p15, 0, r0, c1, c0, 0     /*读取CP15系统控制寄存器   */
    bic     r0,  r0, #(0x1 << 12)     /*  清除第12位（I位）禁用 I Cache  */
    bic     r0,  r0, #(0x1 <<  2)     /*  清除第 2位（C位）禁用 D Cache  */
    bic     r0,  r0, #0x2             /*  清除第 1位（A位）禁止严格对齐   */
    bic     r0,  r0, #(0x1 << 11)     /*  清除第11位（Z位）分支预测   */
    bic     r0,  r0, #0x1             /*  清除第 0位（M位）禁用 MMU   */
    mcr     p15, 0, r0, c1, c0, 0     /*  将修改后的值写回CP15寄存器   */

/**********************第四部分*********************/
    /* 定义IRQ工作模式的栈起始地址 */
    cps     #0x12
    ldr     sp, =IRQ_model_stack_start

    /*定义User工作模式的栈起始地址，与Supervisor相同*/
    cps     #0x1F
    ldr     sp, =SUP_model_stack_start

    /*定义Supervisor工作模式的栈起始地址，与User相同 */
    cps     #0x13
    ldr     sp, =SUP_model_stack_start

/**********************第五部分*******************/
    /*跳转到系统初始化函数，初始化GIC、CACHE-L1、mmu等等*/
    ldr     r2, =SystemInit
    blx     r2

 /*********************第六部分*******************/
    /*开启全局中断*/
    cpsie   i

    /*跳转到到 main 函数执行，*/
    b main
    b .        /*死循环*/
    /*其他中断处理函数与官方相同这里省略，可直接打开源码查看*/

```

其中SystemInit()函数是在C语言文件system_MCIMX6Y2.c中进行实现的。用于初始化GIC控制器`GIC_Init()`;，设置中断向量表的偏移` __set_VBAR((uint32_t)__VECTOR_TABLE);`， 设置系统控制寄存器SCTLR:使能I，D cache，mmu等`__set_SCTLR(sctlr);` 。

```c
void SystemInit(void)
{
  uint32_t sctlr;
  uint32_t actlr;
 #if ((__FPU_PRESENT == 1) && (__FPU_USED == 1))
  uint32_t cpacr;
  uint32_t fpexc;
 #endif

  L1C_InvalidateInstructionCacheAll();
  L1C_InvalidateDataCacheAll();

  actlr = __get_ACTLR();
  actlr = (actlr | ACTLR_SMP_Msk); /* Change to SMP mode before enable DCache */
  __set_ACTLR(actlr);

  sctlr = __get_SCTLR();
  sctlr = (sctlr & ~(SCTLR_V_Msk | /* Use low vector */
                     SCTLR_A_Msk | /* Disable alignment fault checking */
                     SCTLR_M_Msk)) /* Disable MMU */
          | (SCTLR_I_Msk |         /* Enable ICache */
             SCTLR_Z_Msk |         /* Enable Prediction */
             SCTLR_CP15BEN_Msk |   /* Enable CP15 barrier operations */
             SCTLR_C_Msk);         /* Enable DCache */
  __set_SCTLR(sctlr);

  /* Set vector base address */
  GIC_Init();
  __set_VBAR((uint32_t)__VECTOR_TABLE);

  // rgb_led_init();
  // blue_led_on;
 #if ((__FPU_PRESENT == 1) && (__FPU_USED == 1))
  cpacr = __get_CPACR();
  /* Enable NEON and FPU */
  cpacr = (cpacr & ~(CPACR_ASEDIS_Msk | CPACR_D32DIS_Msk)) | (3UL << CPACR_cp10_Pos) | (3UL << CPACR_cp11_Pos);
  __set_CPACR(cpacr);

  fpexc = __get_FPEXC();
  fpexc |= 0x40000000UL; /* Enable NEON and FPU */
  __set_FPEXC(fpexc);
 #endif /* ((__FPU_PRESENT == 1) && (__FPU_USED == 1)) */
}

```
### 2. main主函数初始化

主函数调用`interrupt_button2_init(void)`初始化
#### 2.1 函数初始化中断引角

```c
void interrupt_button2_init(void)
{
    volatile uint32_t *icr;  //用于保存 GPIO-ICR寄存器的地址,与 icrShift 变量配合使用
    uint32_t icrShift;       //引脚号大于16时会用到,

    icrShift = button2_GPIO_PIN;  //保存button2引脚对应的 GPIO 号

    /*添加中断服务函数到  "中断向量表"*/
    SystemInstallIrqHandler(GPIO5_Combined_0_15_IRQn, (system_irq_handler_t)EXAMPLE_GPIO_IRQHandler, NULL);
    GIC_EnableIRQ(GPIO5_Combined_0_15_IRQn);                 //开启中断


    CCM_CCGR1_CG15(0x3);  //开启GPIO5的时钟

    /*设置 按键引脚的PAD属性*/
    IOMUXC_SetPinMux(IOMUXC_SNVS_SNVS_TAMPER1_GPIO5_IO01,0);     
    IOMUXC_SetPinConfig(IOMUXC_SNVS_SNVS_TAMPER1_GPIO5_IO01, button_PAD_CONFIG_DATA); 
    
    /*设置GPIO方向（输入或输出）*/
    GPIO5->IMR &= ~(1 << button2_GPIO_PIN);  //寄存器重置为默认值
    GPIO5->GDIR &= ~(1<<1);                  //设置GPIO5_01为输入模式

    /*设置GPIO引脚中断类型*/
    GPIO5->EDGE_SEL &= ~(1U << button2_GPIO_PIN);//寄存器重置为默认值

    if(button2_GPIO_PIN < 16)
    {
        icr = &(GPIO5->ICR1);       // interrupt configuration register1 save all interrupt trigger mode//
    }
    else
    {
        icr = &(GPIO5->ICR2);
        icrShift -= 16;
    }

    /*按键引脚默认低电平，设置为上升沿触发中断*/
     *icr = (*icr & (~(3U << (2 * icrShift)))) | (2U << (2 * icrShift));

     button2_GPIO->IMR |= (1 << button2_GPIO_PIN);    // IMR--interrupt mask register 1 for enable interrupt/
}
```

其中有几个寄存器需要配置，GPIO_IMR 和GPIO_ICR。  
- GPIO_IMR(interrupt mask register )  
Bit IMR[n] (n=0...31) controls interrupt n as follows:  
0 --- MASKED — Interrupt n is disabled.  
1 --- UNMASKED — Interrupt n is enabled.

- GPIO interrupt configuration register  
00 --- LOW_LEVEL — Interrupt n is low-level sensitive.  
01 --- HIGH_LEVEL — Interrupt n is high-level sensitive.  
10 --- RISING_EDGE — Interrupt n is rising-edge sensitive.  
11 --- FALLING_EDGE — Interrupt n is falling-edge sensitive.  

- GPIO_EDGE_SEL edge select register
 1 --- enable 使能则ICR寄存器无用，上升下降均可使能
 0 --- disable 通过ICR寄存器使能


#### 2.2 向irq中断表中注册中断函数

在2.1中使用了中断注册函数 `SystemInstallIrqHandler(GPIO5_Combined_0_15_IRQn, (system_irq_handler_t)EXAMPLE_GPIO_IRQHandler, NULL);`向irq_table中注册了中断号为`GPIO5_Combined_0_15_IRQn`的中断处理函数`EXAMPLE_GPIO_IRQHandler`。  

下面展示了中断注册函数的实现，这里用到了一个全局定义的irqTabe, IRQn_Type是定义在NXP官方库中的enum枚举类型

```c
void SystemInstallIrqHandler(IRQn_Type irq, system_irq_handler_t handler, void *userParam)
{
  irqTable[irq].irqHandler = handler;
  irqTable[irq].userParam = userParam;
}
```

```c
typedef struct _sys_irq_handle
{
    system_irq_handler_t irqHandler; /**< IRQ handler for specific IRQ */
    void *userParam;                 /**< User param for handler callback */
} sys_irq_handle_t;

static sys_irq_handle_t irqTable[NUMBER_OF_INT_VECTORS];

```

其中`system_irq_handler_t` 是一个typedef修饰的函数指针变量
```c
typedef void (*system_irq_handler_t) (uint32_t giccIar, void *param);
```



```c
void EXAMPLE_GPIO_IRQHandler(void)
{
    /*按键引脚中断服务函数*/
    button2_GPIO->ISR = 1U << button2_GPIO_PIN;  //清除GIIP中断标志位
    if(button_status > 0)
    {
        button_status = 0;
    }
    else
    {
        button_status = 1;
    }
}

```



### 3.1 汇编定义irq中断

{% highlight c linenos %}
IRQ_Handler:
    push    {lr}                         /* Save return address+4                                */
    push    {r0-r3, r12}                 /* Push caller save registers                           */

    MRS     r0, spsr                     /* Save SPRS to allow interrupt reentry                 */
    push    {r0}

    MRC     P15, 4, r1, C15, C0, 0       /* Page 180 Get GIC base address C15 arm maunal */
    ADD     r1, r1, #0x2000              /* r1: GICC base address CPU_interface */
    LDR     r0, [r1, #0xC]               /* Cpu interface add 0xC register GIC_IAR  */
    /** IAr iar interrupt acknowloge register HAVE interrupt id from bit 0-9  */ 
    push    {r0, r1}

    CPS     #0x13                        /* Change to Supervisor mode to allow interrupt reentry */

    push    {lr}                         /* Save Supervisor lr  */
    ldr     r2, =SystemIrqHandler       /*  r0 passes the variable value to systemirq function  */
    blx     r2
                           
    POP     {lr}

    CPS     #0x12                        /* Back to IRQ mode                                     */

    POP     {r0, r1}

    STR     r0, [r1, #0x10]              /* write interupt num to EOIR responding with IAR            */

    POP     {r0}
    MSR     spsr_cxsf, r0

    POP     {r0-r3, r12}
    POP     {lr}
    SUBS    pc, lr, #4                  /* set pc to lr-4  */

{% endhighlight %}

其中 `ldr     r2, =SystemIrqHandler` , `blx    r2` 执行到systemirqhandler函数的跳转。 并且这个c函数可以将R0(当前保存为GICC_IAR)的值传参。

具体的SystemIrqHandler 函数在 system_MCIMX6Y2.c文件中如下:

{% highlight c linenos %}
__attribute__((weak)) void SystemIrqHandler(uint32_t giccIar)
{

/***********************第二部分********************/
  uint32_t intNum = giccIar & 0x3FFUL;     // last 10 bits indicate interrupt numbers

  // rgb_led_init();
  // blue_led_on;
/***********************第三部分********************/
  /* Spurious interrupt ID or Wrong interrupt number */
  if ((intNum == 1023) || (intNum >= NUMBER_OF_INT_VECTORS))
  {
    return;
  }

  irqNesting++;

  // __enable_irq();      /* Support nesting interrupt */

/***********************第四部分********************/
  /* Now call the real irq handler for intNum */
  irqTable[intNum].irqHandler(giccIar, irqTable[intNum].userParam);

  // __disable_irq();
  irqNesting--;
}

{% endhighlight %}

注意第22行中有`irqTable[intNum].irqHandler(giccIar, irqTable[intNum].userParam);` 此语句根据中断id(intNum)调用着一个该中断所对应的中断函数，这个中断函数的首地址写在irqTable中。




### 附录

#### __attribute__((weak)) 

在本文件中定义：

int  __attribute__((weak))  func(......)
{
return 0;
}


将本模块的func转成弱符号类型，如果遇到强符号类型（即外部模块定义了func），那么我们在本模块执行的func将会是外部模块定义的func。类似于c++中的重载

如果外部模块没有定义，那么，将会调用这个弱符号，也就是在本地定义的func

#### typedef 函数指针

下面代码定义了一个指针变量 pFun， pFun = glFun将函数glFun的地址指向pFun。
```c
char (*pFun)(int);   
char glFun(int a){ return;}   
void main()   
{   
    pFun = glFun;   
    (*pFun)(2);   
}  

```


```c
typedef char (*PTRFUN)(int);   
PTRFUN pFun;   
char glFun(int a){ return;}   
void main()   
{   
    pFun = glFun;   
    (*pFun)(2);   
}   

```



