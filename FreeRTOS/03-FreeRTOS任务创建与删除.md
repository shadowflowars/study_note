# FreeRTOS任务的创建与删除
## 任务创建和删除的API函数

任务的创建和删除本质就是调用FreeRTOS的API函数
| API函数 | 功能描述 |
| ---- | ---- |
| `xTaskCreate()` | 动态方式创建任务 |
| `xTaskCreateStatic()` | 静态方式创建任务 |
| `vTaskDelete()` | 删除任务 |

1. **动态创建任务**：任务的任务控制块以及任务的栈空间所需的内存，均由 FreeRTOS 从 FreeRTOS 管理的堆中分配 
2. **静态创建任务**：任务的任务控制块以及任务的栈空间所需的内存，需用户分配提供

## 任务创建和删除（动态方法）（掌握）
### 动态创建任务函数
```
BaseType_t xTaskCreate
( 
    TaskFunction_t 	                pxTaskCode,		/* 指向任务函数的指*/				  
    const char * const              pcName, 	    /* 任务名字，最大长度configMAX_TASK_NAME_LEN */
    const configSTACK_DEPTH_TYPE    usStackDepth, 	/* 任务堆栈大小，注意字为单位*/
    void * const 					pvParameters,	/* 传递给任务函数的参数 */
    UBaseType_t 					uxPriority,		/* 任务优先级，范围：0 ~ configMAX_PRIORITIES - 1 */
    TaskHandle_t * const 			pxCreatedTask 	/* 任务句柄，就是任务的任务控制块*/
)

```
| 返回值 | 描述 |
| ---- | ---- |
| `pdPASS` | 任务创建成功 |
| `errCOULD_NOT_ALLOCATE_REQUIRED_MEMORY` | 任务创建失败 |

### 实现动态创建任务流程
用起来只需要三步
1. 将宏configSUPPORT_DYNAMIC_ALLOCATION 配置为 1
2. 定义函数入口参数
3. 编写任务函数

此函数创建的任务会立刻进入就绪态，由任务调度器调度运行

动态创建任务函数内部实现
1. 申请堆栈内存&任务控制块内存
2. TCB结构体成员赋值
3. 添加新任务到就绪列表中
## 任务创建和删除（静态方法）（掌握）
### 静态创建任务函数
```
TaskHandle_t xTaskCreateStatic
(
        TaskFunction_t		    pxTaskCode,			/* 指向任务函数的指针 */
        const char * const		pcName,				/* 任务函数名 */
        const uint32_t			ulStackDepth, 		/* 任务堆栈大小注意字为单位 */
        void * const			pvParameters, 		/* 传递的任务函数参数 */
        UBaseType_t			    uxPriority, 		/* 任务优先级 */
        StackType_t * const		puxStackBuffer, 	/* 任务堆栈，一般为数组，由用户分配 */
        StaticTask_t * const	pxTaskBuffer	    /* 任务控制块指针，由用户分配 */
); 		
```
| 返回值 | 描述 |
| ---- | ---- |
| `NULL` | 用户没有提供相应的内存，任务创建失败 |
| `其他值` | 任务句柄，任务创建成功 |

### 静态创建任务使用流程
用起来只需这五步
1. 需将宏configSUPPORT_STATIC_ALLOCATION 配置为 1 
2. 定义空闲任务&定时器任务的任务堆栈及TCB
3. 实现两个接口函数
4. 定义函数入口参数
5. 编写任务函数

静态创建内部实现
1. TCB结构体成员赋值
2. 添加新任务到就绪列表中

## 任务删除函数
```
void vTaskDelete(TaskHandle_t xTaskToDelete);

/*xTaskToDelete 待删除任务的任务句柄*/
```
用于删除已被创建的任务
被删除的任务将从就绪态任务列表、阻塞态任务列表、挂起态任务列表和事件列表中移除
1. 当传入的参数为NULL，则代表删除任务自身（当前正在运行的任务）
2. 空闲任务会负责释放被删除任务中由系统分配的内存，但是由用户在任务删除前申请的内存， 则需要由用户在任务被删除前提前释放，否则将导致内存泄露 

用起来只需要两步
1. 使用删除任务函数，需将宏INCLUDE_vTaskDelete 配置为 1 
2. 入口参数输入需要删除的任务句柄（NULL代表删除本身）

删除函数内部实现


1. 获取所要删除任务的控制块，通过传入的任务句柄，判断所需要删除哪个任务，NULL代表删除自身

2. 将被删除任务，移除所在列表，将该任务在所在列表中移除，包括：就绪、阻塞、挂起、事件等列表

3. 判断所需要删除的任务，删除任务自身，需先添加到等待删除列表，内存释放将在空闲任务执行，删除其他任务，释放内存，任务数量--

4. 更新下个任务的阻塞时间，更新下一个任务的阻塞超时时间，以防被删除的任务就是下一个阻塞超时的任务 


## 实操
1. 实验目的：学会 xTaskCreate( )  和 vTaskDelete( ) 的使用
2. 实验设计：将设计四个任务：start_task、task1、task2、task3

| 任务名称 | 功能说明 |
| ---- | ---- |
| `start_task` | 用来创建其他的三个任务 |
| `task1` | 实现LED0每500ms闪烁一次 |
| `task2` | 实现LED1每500ms闪烁一次 |
| `task3` | 判断按键KEY0是否按下，按下则删掉task1 |


```
#include "main.h"          // HAL库主头文件
#include "cmsis_os.h"      // CMSIS-RTOS 封装层头文件
#include "usart.h"         // 串口驱动头文件（用于printf输出）
#include "gpio.h"          // GPIO驱动头文件（用于LED和按键操作）

/* USER CODE BEGIN Includes */
#include <stdio.h>         // 标准输入输出库（printf需要）
#include "freertos.h"      // FreeRTOS 核心头文件
/* USER CODE END Includes */

// ==================== start_task 任务配置（静态方式） ====================
#define START_TASK_PRIO 1                    // 任务优先级，数值越小优先级越低
#define START_TASK_STACK_SIZE 128            // 任务栈大小（单位：字，即StackType_t，通常4字节）
TaskHandle_t start_task_handler;             // 任务句柄，用于引用该任务（如删除任务时使用）
StackType_t  start_task_stack[START_TASK_STACK_SIZE]; // 用户自行分配的栈空间（数组）
StaticTask_t start_task_tcb;                 // 用户自行分配的任务控制块（TCB）
void start_task( void * pvParameters );      // 任务函数声明

// ==================== task1 任务配置（LED0闪烁） ====================
#define TASK1_PRIO 2                         // 任务优先级（高于start_task）
#define TASK1_STACK_SIZE 128                 // 任务栈大小
TaskHandle_t task1_handler;                  // 任务句柄
StackType_t  task1_stack[START_TASK_STACK_SIZE]; // 任务栈空间（用户分配）
StaticTask_t task1_tcb;                      // 任务控制块（用户分配）
void task1( void * pvParameters );           // 任务函数声明

// ==================== task2 任务配置（LED1闪烁） ====================
#define TASK2_PRIO 3                         // 任务优先级
#define TASK2_STACK_SIZE 128                 // 任务栈大小
TaskHandle_t task2_handler;                  // 任务句柄
StackType_t  task2_stack[START_TASK_STACK_SIZE]; // 任务栈空间（用户分配）
StaticTask_t task2_tcb;                      // 任务控制块（用户分配）
void task2( void * pvParameters );           // 任务函数声明

// ==================== task3 任务配置（按键检测与任务删除） ====================
#define TASK3_PRIO 4                         // 任务优先级（最高）
#define TASK3_STACK_SIZE 128                 // 任务栈大小
TaskHandle_t task3_handler;                  // 任务句柄
StackType_t  task3_stack[START_TASK_STACK_SIZE]; // 任务栈空间（用户分配）
StaticTask_t task3_tcb;                      // 任务控制块（用户分配）
void task3( void * pvParameters );           // 任务函数声明

// ==================== 主函数 ====================
int main(void)
{
  freertos_demo();          // 初始化并启动FreeRTOS调度器
  while (1){}               // 正常情况下不会执行到这里（调度器启动后不会返回）
}

/* USER CODE BEGIN 4 */
// ==================== FreeRTOS 演示入口函数 ====================
void freertos_demo(void)
{
    // 使用静态方式创建 start_task 任务
    // 参数依次为：任务函数、任务名称、栈大小、任务参数、优先级、栈地址、TCB地址
    start_task_handler =     xTaskCreateStatic( (TaskFunction_t) start_task,
                   (char *        ) "start_task",  // 任务名称（调试用）
                   (uint32_t      ) START_TASK_STACK_SIZE, // 栈大小
                   (void *        ) NULL,           // 不向任务传递参数
                   (UBaseType_t   ) START_TASK_PRIO, // 任务优先级
                   (StackType_t * ) &start_task_stack, // 栈空间首地址
                   (StaticTask_t *) &start_task_tcb ); // TCB首地址

        vTaskStartScheduler(); // 启动FreeRTOS任务调度器（从此处开始，由调度器接管CPU）
    
}
// ==================== start_task 任务函数 ====================
// 功能：一次性创建 task1、task2、task3 三个任务，然后删除自身
void start_task( void * pvParameters )
{
     taskENTER_CRITICAL();  // 进入临界区——禁止任务调度和中断，保证创建过程的原子性

     // 静态创建 task1 —— LED0 闪烁任务
     task1_handler = xTaskCreateStatic( (TaskFunction_t) task1,
                   (char *        ) "task1",
                   (uint32_t      ) TASK1_STACK_SIZE,
                   (void *        ) NULL,
                   (UBaseType_t   ) TASK1_PRIO,
                   (StackType_t * ) &task1_stack,
                   (StaticTask_t *) &task1_tcb );
     // 静态创建 task2 —— LED1 闪烁任务
     task2_handler = xTaskCreateStatic( (TaskFunction_t) task2,
                   (char *        ) "task2",
                   (uint32_t      ) TASK2_STACK_SIZE,
                   (void *        ) NULL,
                   (UBaseType_t   ) TASK2_PRIO,
                   (StackType_t * ) &task2_stack,
                   (StaticTask_t *) &task2_tcb );
     // 静态创建 task3 —— 按键检测与任务删除任务
     task3_handler = xTaskCreateStatic( (TaskFunction_t) task3,
                   (char *        ) "task3",
                   (uint32_t      ) TASK3_STACK_SIZE,
                   (void *        ) NULL,
                   (UBaseType_t   ) TASK3_PRIO,
                   (StackType_t * ) &task3_stack,
                   (StaticTask_t *) &task3_tcb );
                            
   vTaskDelete(NULL);   // 删除自身（start_task），传入NULL表示删除当前运行的任务
                        // 删除后，start_task 的栈和TCB由空闲任务负责回收
                            
     taskEXIT_CRITICAL(); // 退出临界区（实际上此行不会执行，因为上一行已删除自身）
     // 注意：taskEXIT_CRITICAL() 和 taskENTER_CRITICAL() 应成对出现
     // 此处虽然 vTaskDelete 后代码不会执行，但保留是为了代码完整性
}
                            

// ==================== task1 任务函数 ====================
// 功能：每500ms翻转一次LED0，实现LED0闪烁
void task1( void * pvParameters )
{
     while(1)                      // FreeRTOS 任务必须是死循环，不能返回
     {    
            if(HAL_GPIO_ReadPin(GPIOF,LED0_Pin) == GPIO_PIN_RESET) // 读取LED0当前电平
            {
                    HAL_GPIO_WritePin(GPIOF,LED0_Pin,GPIO_PIN_SET);   // 低→高（翻转）
            }
          else
          HAL_GPIO_WritePin(GPIOF,LED0_Pin,GPIO_PIN_RESET);          // 高→低（翻转）
      printf("task1 working \r\n"); // 串口打印任务运行信息
          vTaskDelay(500);          // 阻塞延时500个系统节拍（tick），让出CPU给其他任务
     }
}
// ==================== task2 任务函数 ====================
// 功能：每500ms翻转一次LED1，实现LED1闪烁
void task2( void * pvParameters )
{
    while(1){
            if(HAL_GPIO_ReadPin(GPIOF,LED1_Pin) == GPIO_PIN_RESET) // 读取LED1当前电平
            {
                    HAL_GPIO_WritePin(GPIOF,LED1_Pin,GPIO_PIN_SET);   // 低→高（翻转）
            }
          else
          HAL_GPIO_WritePin(GPIOF,LED1_Pin,GPIO_PIN_RESET);          // 高→低（翻转）
      printf("task2 working \r\n"); // 串口打印任务运行信息
            vTaskDelay(500);        // 阻塞延时500个tick
     }
}
// ==================== task3 任务函数 ====================
// 功能：检测按键KEY1是否按下，按下后删除task1，再删除自身
void task3( void * pvParameters )
{
   while(1)
    {
         if(HAL_GPIO_ReadPin(GPIOE,KEY1_Pin) == GPIO_PIN_RESET) // 判断按键是否按下（低电平有效）
         {
             printf("task1 stop \r\n");        // 串口提示即将删除task1
             vTaskDelete(task1_handler);       // 删除 task1（通过句柄指定要删除的任务）
             vTaskDelay(10);                   // 短暂延时，确保删除操作完成
             vTaskDelete(NULL);                // 删除自身（task3），此后该分支不再执行
         }
         printf("task3 working\r\n");         // 串口打印任务运行信息
         vTaskDelay(500);                      // 阻塞延时500个tick
     }         
      
}
```
