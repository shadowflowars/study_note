# FreeRTOS临界段代码保护及调度器挂起与恢复
## 临界段代码保护简介（熟悉）
什么是临界段：临界段代码也叫做临界区，是指那些必须完整运行，不能被打断的代码段

适用场合如：

| 分类 | 说明 |
| ---- | ---- |
| 1、外设 | 需严格按照时序初始化的外设：IIC、SPI等等 |
| 2、系统 | 系统自身需求 |
| 3、用户 | 用户需求 |


## 临界段代码保护函数介绍（掌握）

临界区是直接屏蔽了中断，系统任务调度靠中断，ISR也靠中断

FreeRTOS 在进入临界段代码的时候需要关闭中断，当处理完临界段代码以后再打开中断

| 函数 | 描述 |
| ---- | ---- |
| `taskENTER_CRITICAL()` | 任务级进入临界段 |
| `taskEXIT_CRITICAL()` | 任务级退出临界段 |
| `taskENTER_CRITICAL_FROM_ISR()` | 中断级进入临界段 |
| `taskEXIT_CRITICAL_FROM_ISR()` | 中断级退出临界段 |

**特点：**
1. 成对使用
2. 支持嵌套
3. 尽量保持临界段耗时短


## 任务调度器的挂起和恢复（熟悉）

| 函数 | 描述 |
| ---- | ---- |
| `vTaskSuspendAll()` | 挂起任务调度器 |
| `xTaskResumeAll()` | 恢复任务调度器 |

挂起任务调度器， 调用此函数不需要关闭中断
1. 与临界区不一样的是，挂起任务调度器，未关闭中断；
2. 它仅仅是防止了任务之间的资源争夺，中断照样可以直接响应；
3. 挂起调度器的方式，适用于临界区位于任务与任务之间；既不用去延时中断，又可以做到临界区的安全

**代码实列**
```
void start_task( void * pvParameters )
{
	 taskENTER_CRITICAL();  //进入临界 关闭中断
	//vTaskSuspendAll(); //挂起任务调度器，不关闭中断；
	
	 xTaskCreate((TaskFunction_t       ) task1,
							(char *                ) "task1",	
							(configSTACK_DEPTH_TYPE) TASK1_STACK_SIZE,
							(void *                ) NULL,
							(UBaseType_t           ) TASK1_PRIO,
							(TaskHandle_t *        ) &task1_handler );		

	 taskEXIT_CRITICAL(); //退出临界区 				
 //xTaskResumeAll();						
   vTaskDelete(NULL);						
}
```