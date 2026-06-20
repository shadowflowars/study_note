# FreeRTOS时间片调度
1. 实验目的：学会对FreeRTOS 时间片调度使用
2. 实验设计：将设计三个任务：start_task、task1、task2，其中task1和task2优先级相同均为2。
为了使现象明显，将滴答定时器的中断频率设置为50ms中断一次，即一个时间片50ms

| 任务名称 | 功能说明 |
| ---- | ---- |
| `start_task` | 用来创建其他的2个任务 |
| `task1` | 通过串口打印task1的运行次数 |
| `task2` | 通过串口打印task2的运行次数 |

注意：使用时间片调度需把宏 configUSE_TIME_SLICING 和 configUSE_PREEMPTION 置1

将configTICK_RATE_HZ — 系统时钟节拍频率（单位 Hz）。时间片的长度固定为 1 个 tick，即 1 / configTICK_RATE_HZ
  秒。比如设为 1000 则每个时间片为 1ms。设为 20 则为 50ms。






