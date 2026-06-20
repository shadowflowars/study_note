# FreeRTOS的任务挂起与恢复
## 任务的挂起与恢复的API函数（熟悉）
| API函数 | 描述 |
| ---- | ---- |
| `vTaskSuspend()` | 挂起任务 |
| `vTaskResume()` | 恢复被挂起的任务 |
| `xTaskResumeFromISR()` | 在中断中恢复被挂起的任务 |

| 关键词 | 说明 |
| ---- | ---- |
| 挂起 | 挂起任务类似暂停，可恢复；删除任务，无法恢复，类似"人死两清" |
| 恢复 | 恢复被挂起的任务 |
| `"FromISR"` | 带FromISR后缀是在中断函数中专用的API函数 |

**任务挂起函数介绍**

```
void vTaskSuspend(TaskHandle_t xTaskToSuspend) //xTaskToSuspend 待挂起任务的任务句柄
```
此函数用于挂起任务，使用时需将宏 INCLUDE_vTaskSuspend  配置为 1。 无论优先级如何，被挂起的任务都将不再被执行，直到任务被恢复 。当传入的参数为NULL，则代表挂起任务自身（当前正在运行的任务）

**任务恢复函数介绍（任务中恢复）**
```
void vTaskResume(TaskHandle_t xTaskToResume) //xTaskToResume 待恢复任务的任务句柄
```
使用该函数注意宏：INCLUDE_vTaskSuspend必须定义为 1
任务无论被 vTaskSuspend() 挂起多少次，只需在任务中调用  vTakResume() 恢复一次，就可以继续运行。且被恢复的任务会进入就绪态！ 

**任务恢复函数介绍（中断中恢复）**
```
BaseType_t xTaskResumeFromISR(TaskHandle_t xTaskToResume) //xTaskToResume 待恢复任务的任务句柄
```
函数：xTaskResumeFromISR返回值描述如下：

| 返回值 | 描述 |
| ---- | ---- |
| `pdTRUE` | 任务恢复后需要进行任务切换 |
| `pdFALSE` | 任务恢复后不需要进行任务切换 |

使用该函数注意宏：INCLUDE_vTaskSuspend 和 INCLUDE_xTaskResumeFromISR 必须定义为 1该函数专用于中断服务函数中，用于解挂被挂起任务,中断服务程序中要调用freeRTOS的API函数则中断优先级不能高于FreeRTOS所管理的最高优先级

## 任务挂起与恢复实验（掌握）
1. 实验目的：学会 使用FreeRTOS中的任务挂起与恢复相关API函数：vTaskSuspend( )、vTaskResume( )、xTaskResumeFromISR( )
2. 实验设计：将设计四个任务：start_task、task1、task2、task3

| 任务名称 | 功能说明 |
| ---- | ---- |
| `start_task` | 用来创建其他的三个任务 |
| `task1` | 实现LED0每500ms闪烁一次 |
| `task2` | 实现LED1每500ms闪烁一次 |
| `task3` | 判断按键按下逻辑：<br>1. KEY1按下，挂起task1<br>2. KEY2按下，在任务上下文恢复task1<br>3. KEY3按下，外部中断中调用中断专用API恢复task1 |

**代码如下**
```
#include "main.h"          // HAL库主头文件
#include "cmsis_os.h"      // CMSIS-RTOS 封装层头文件
#include "usart.h"         // 串口驱动头文件（用于printf输出）
#include "gpio.h"          // GPIO驱动头文件（用于LED和按键操作）
#include <stdio.h>         // 标准输入输出库（printf需要）
#include "freertos.h"      // FreeRTOS 核心头文件

// ==================== start_task 任务配置（动态方式） ====================
#define START_TASK_PRIO 1                    // 任务优先级，数值越小优先级越低
#define START_TASK_STACK_SIZE 128            // 任务栈大小（单位：字，通常4字节）
TaskHandle_t start_task_handler;             // 任务句柄，用于引用该任务
void start_task( void * pvParameters );      // 任务函数声明

// ==================== task1 任务配置（LED0闪烁，可被挂起/恢复） ====================
#define TASK1_PRIO 2                         // 任务优先级
#define TASK1_STACK_SIZE 128                 // 任务栈大小
TaskHandle_t task1_handler;                  // 任务句柄（挂起/恢复时需要用到）
void task1( void * pvParameters );           // 任务函数声明

// ==================== task2 任务配置（LED1闪烁） ====================
#define TASK2_PRIO 3                         // 任务优先级
#define TASK2_STACK_SIZE 128                 // 任务栈大小
TaskHandle_t task2_handler;                  // 任务句柄
void task2( void * pvParameters );           // 任务函数声明

// ==================== task3 任务配置（按键检测：挂起/恢复task1） ====================
#define TASK3_PRIO 4                         // 任务优先级（最高）
#define TASK3_STACK_SIZE 128                 // 任务栈大小
TaskHandle_t task3_handler;                  // 任务句柄
void task3( void * pvParameters );           // 任务函数声明

// ==================== 系统初始化函数声明 ====================
void SystemClock_Config(void);               // 系统时钟配置（由CubeMX生成）
void MX_FREERTOS_Init(void);                 // FreeRTOS初始化（由CubeMX生成）

// ==================== 主函数 ====================
int main(void)
{
  HAL_Init();                    // HAL库初始化
  SystemClock_Config();          // 配置系统时钟
  MX_GPIO_Init();                // 初始化GPIO（LED和按键引脚）
  MX_USART1_UART_Init();         // 初始化串口1（用于printf输出）
  freertos_demo();               // 创建任务并启动FreeRTOS调度器

  while (1)                      // 正常情况下不会执行到这里（调度器已接管CPU）
  {

  }

}

// ==================== FreeRTOS 演示入口函数 ====================
void freertos_demo(void)
{
     // 使用动态方式创建 start_task 任务（本次实验采用动态方式，与03篇的静态方式对比学习）
     // 参数依次为：任务函数、任务名称、栈大小、任务参数、优先级、任务句柄指针
     // 动态方式：栈空间与TCB由FreeRTOS从堆中自动分配，无需用户提供数组
     xTaskCreate((TaskFunction_t       ) start_task,
                            (char *                ) "start_task",
                            (configSTACK_DEPTH_TYPE) START_TASK_STACK_SIZE,
                            (void *                ) NULL,       // 不向任务传递参数
                            (UBaseType_t           ) START_TASK_PRIO,
                            (TaskHandle_t *        ) &start_task_handler ); // 保存句柄

                            vTaskStartScheduler(); // 启动任务调度器——CPU控制权交给FreeRTOS

}

// ==================== start_task 任务函数 ====================
// 功能：在临界区内一次性创建 task1、task2、task3 三个子任务，完成后删除自身
void start_task( void * pvParameters )
{
     taskENTER_CRITICAL();  // 进入临界区——禁止任务调度，确保子任务全部创建完毕后才开始调度

     // 动态创建 task1 —— LED0 闪烁任务（可被挂起/恢复）
     xTaskCreate((TaskFunction_t       ) task1,
                            (char *                ) "task1",
                            (configSTACK_DEPTH_TYPE) TASK1_STACK_SIZE,
                            (void *                ) NULL,
                            (UBaseType_t           ) TASK1_PRIO,
                            (TaskHandle_t *        ) &task1_handler );   // 保存句柄——后续挂起/恢复必须用到

     // 动态创建 task2 —— LED1 闪烁任务（不受挂起影响，用于对比观察）
     xTaskCreate((TaskFunction_t       ) task2,
                            (char *                ) "task2",
                            (configSTACK_DEPTH_TYPE) TASK2_STACK_SIZE,
                            (void *                ) NULL,
                            (UBaseType_t           ) TASK2_PRIO,
                            (TaskHandle_t *        ) &task2_handler );

     // 动态创建 task3 —— 按键检测任务（负责挂起/恢复 task1）
     xTaskCreate((TaskFunction_t       ) task3,
                            (char *                ) "task3",
                            (configSTACK_DEPTH_TYPE) TASK3_STACK_SIZE,
                            (void *                ) NULL,
                            (UBaseType_t           ) TASK3_PRIO,
                            (TaskHandle_t *        ) &task3_handler );
     taskEXIT_CRITICAL();  // 退出临界区——此时三个子任务可以开始被调度了
   vTaskDelete(NULL);      // 删除自身（start_task 已完成使命，栈和TCB由空闲任务回收）
}
// ==================== task1 任务函数 ====================
// 功能：每500ms翻转一次LED0，可被 task3 通过按键挂起和恢复
//       被挂起后LED0停止闪烁，恢复后继续闪烁
void task1( void * pvParameters )
{
     while(1)                              // FreeRTOS 任务必须是死循环，不能返回
     {
            HAL_GPIO_TogglePin(GPIOF, LED0_Pin);     // 翻转LED0电平（HAL库专用翻转函数，比手写if-else更简洁）
      printf("task1 working \r\n");        // 串口打印任务运行状态，挂起后此信息会停止打印
              vTaskDelay(500);             // 阻塞延时500个系统节拍（tick），让出CPU给其他任务
     }
}
// ==================== task2 任务函数 ====================
// 功能：每500ms翻转一次LED1，始终运行不受影响（用于对照：验证 task1 确实被挂起/恢复了）
void task2( void * pvParameters )
{
    while(1){
            HAL_GPIO_TogglePin(GPIOF, LED1_Pin);     // 翻转LED1电平
      printf("task2 working \r\n");        // 串口打印——task1被挂起时，只有task2和task3的信息会持续输出
                    vTaskDelay(500);       // 阻塞延时500个tick
     }
}
// ==================== task3 任务函数 ====================
// 功能：轮询检测三个按键，执行不同的挂起/恢复操作
//       KEY1按下 → 挂起 task1（task1 进入挂起态，停止被调度）
//       KEY2按下 → 在任务上下文中恢复 task1（task1 回到就绪态）
//       KEY3按下 → 由外部中断触发恢复（见 HAL_GPIO_EXTI_Callback）
void task3( void * pvParameters )
{
   while(1)
    {
         printf("task3 working\r\n");      // 串口打印——优先级最高，调度最频繁
         if(HAL_GPIO_ReadPin(GPIOE,KEY1_Pin) == GPIO_PIN_RESET) // KEY1按下（低电平有效）
         {
             printf("task1 stop \r\n");    // 串口提示：即将挂起 task1
             vTaskSuspend(task1_handler);  // 挂起 task1——task1 立即停止运行，进入挂起态
                                           // 关键：无论 task1 优先级多高，被挂起后都不会被调度
                                           // 与vTaskDelete不同：挂起后可恢复，删除则无法恢复
             
         }else if(HAL_GPIO_ReadPin(GPIOE,KEY2_Pin) == GPIO_PIN_RESET){ // KEY2按下
           
             printf("renwu\r\n");          // 串口提示：即将恢复 task1
             vTaskResume(task1_handler);   // 在任务上下文中恢复 task1——task1 回到就绪态
                                           // 关键：无论之前 vTaskSuspend 被调用了多少次
                                           //       只需调用一次 vTaskResume 即可恢复运行
             
         };
         
         vTaskDelay(500);                  // 阻塞延时500个tick
     }
      
}
// ==================== 外部中断回调函数（GPIO EXTI中断） ====================
// 功能：KEY3按下触发外部中断，在中断服务函数中调用 xTaskResumeFromISR() 恢复 task1
//       演示中断中专用的任务恢复API —— 与任务上下文中恢复的关键区别
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin)
{
    BaseType_t xYieldRequired;             // 是否需要触发任务切换的标记变量
  if(GPIO_Pin==KEY3_Pin)                   // 判断中断来源是否为 KEY3
  {		
      xYieldRequired = xTaskResumeFromISR(task1_handler); // 在中断中恢复 task1
                                          // 返回值 pdTRUE  ：被恢复的任务优先级更高，需立即切换
                                          // 返回值 pdFALSE ：无需立即切换，等中断退出后自动调度
  }
        if(xYieldRequired == pdTRUE)       // 如果被恢复的 task1 优先级高于当前被打断的任务
        {
               portYIELD_FROM_ISR(xYieldRequired); // 手动触发一次任务切换
                                                  // 让刚恢复的高优先级 task1 立即获得CPU执行权
                                                  // 这是中断中使用 FreeRTOS API 的标准范式
        }
}

```

## 任务挂起和恢复API函数解析（熟悉）
**任务挂起函数内部实现**
| 步骤 | 流程说明 |
| ---- | ---- |
| 1、获取所要挂起任务的控制块 | 通过传入的任务句柄，判断所需要挂起哪个任务，NULL代表挂起自身 |
| 2、移除所在列表 | 将要挂起的任务从相应的状态列表和事件列表中移除（就绪或阻塞列表） |
| 3、插入挂起任务列表 | 将待挂起任务的任务状态列表插入到挂起态任务列表末尾 |
| 4、判断任务调度器是否运行 | 调度器在运行，更新下一次阻塞时间，防止被挂起任务为下一次阻塞超时任务 |
| 5、判断待挂起任务是否为当前任务 | 1. 如果挂起的是任务自身，且调度器正在运行，需要进行一次任务切换<br>2. 调度器没有运行：若挂起任务总数等于全部任务，当前控制块赋值为NULL；否则寻找下一个最高优先级任务 |

**任务恢复函数内部实现**
| 步骤 | 流程说明 |
| ---- | ---- |
| 1、恢复任务不能是正在运行任务 | 禁止恢复当前处于运行态的任务 |
| 2、判断任务是否在挂起列表中 | 若任务存在于挂起列表：将该任务从挂起列表移除，添加到就绪列表 |
| 3、判断恢复任务优先级 | 对比恢复任务与当前运行任务的优先级，若恢复任务优先级更高，则执行任务切换 |