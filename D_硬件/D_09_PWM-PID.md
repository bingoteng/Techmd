# PWM-PID-电机基本了解

## 一、基本概念

**PWM**（Pulse Width Modulation，**脉冲宽度调制**）是一种通过调节方波信号中**高电平持续时间占整个周期的比例**（即占空比），来**等效控制电压、功率、亮度、转速**等模拟量输出的技术。

---

## 二、核心参数

| 参数                     | 说明                   | 公式/单位        |
| ------------------------ | ---------------------- | ---------------- |
| **周期（Period）**       | 一个完整脉冲的时间长度 | T = 1/f          |
| **频率（Frequency）**    | 每秒脉冲周期数         | f = 1/T（Hz）    |
| **占空比（Duty Cycle）** | 高电平时间占周期的比例 | D = tₕᵢ/T × 100% |

```
        ┌───────┐     ┌───────┐
高电平   │       │     │       │
        │       │     │       │
低电平  ─┘       └─────┘       └─────► 时间
        ←─tₕᵢ─→
        ←──── T ────→
周期：   一个方波信号的周期(从上升沿到下个上升沿的时间)   
占空比 = tₕᵢ / T × 100%
占空比 0% 		→ 输出 	0V
占空比 50% 	→ 输出 	VCC/2
占空比 100% 	→ 满幅 	VCC
```

---

## 三、典型应用

- **LED 调光**：呼吸灯、LCD屏幕背光
- **电机调速**：风扇转速、小车驱动
- **音频 / 信号**：蜂鸣器音量

```
PWM配置过程
1.确定占空比
2.配置定时器：确定从哪个值开始自加，
3.自加到定时器中断阈值，触发中断，中断处理函数将电平翻转
```

# PID算法

## 一、基本概念

### 1.什么是PID算法？

```c
PID算法是一种闭环控制算法。它就是根据目标值与实际值的误差，通过比例、积分、微分三项计算，自动输出控制量，让实际值快速、平稳地逼近目标值。
其中：
Kp：加快响应，减小稳态误差，但过大可能导致振荡和超调		->快速
Kd：预测误差变化趋势，抑制超调与震荡					 ->平稳
Ki：累积并消除过去的稳态误差							->精确

	"复述：PID算法是一种闭环控制算法，它就是根据目标值与实际值的误差，经比例、积分、微分三项计算自动输出控制量，使实际值快速平稳逼近目标值。其中，比例项KP用以加快响应速度，帮助系统快速接近目标值，但过大会导致震荡和超调；微分项KD则通过预测误差变化趋势，抑制震荡和超调，使系统平稳到达目标值;而积分项KI能累积误差并消除稳态误差，让系统精确停在目标值。
    "实例：液位闭环控制系统常用PID控制。目标是维持水箱液位在设定高度，比如50厘米。PID公式为：Output = KP*(50 - Current_Level) + KI*∫(50 - Current_Level)dt + KD*(d(Current_Level)/dt)。Current_Level由液位传感器实时采集，Output控制进水阀门的开度。当液位低于50厘米，比例项会增大阀门开度加快进水；若液位长期稳定在48厘米，积分项会累积误差，继续调大阀门消除稳态误差；当液位快速接近50厘米时，微分项会关小阀门，防止液位超过目标值后溢出。
    "试凑法步骤：第一步，只保留比例控制，把KI、KD设为0，逐渐增大KP直到系统出现等幅震荡，记下此时的临界KP值，实际使用取它的60%-80%。第二步，加入积分控制，从0开始慢慢增大KI，直到稳态误差消除，且系统没有明显超调，注意KI过大会导致系统不稳定。第三步，最后加入微分控制，同样从0开始增大KD，用来抑制超调和震荡，让系统响应更平稳，KD不宜过大，否则会让系统对噪声敏感。
```

PID根据当前时刻的**误差** `e(t) = 目标值 - 实际测量值`，分别计算三个分量并加权求和，输出控制信号：

| 项            | 全称         | 作用机制                             | 特点                                                         |
| ------------- | ------------ | ------------------------------------ | ------------------------------------------------------------ |
| **P（比例）** | Proportional | 与当前误差成正比：`Kp × e(t)`        | **响应快**，但单独使用会残留**稳态误差**；Kp过大易引起**振荡** |
| **I（积分）** | Integral     | 与误差历史累积成正比：`Ki × ∫e(t)dt` | 消除静态误差，提高精度；积分过强会导致**响应迟缓**、超调或振荡 |
| **D（微分）** | Derivative   | 与误差变化率成正比：`Kd × de(t)/dt`  | 预测误差趋势，**提前抑制超调**，增强稳定性；但对测量噪声敏感 |

### 2.极简的c语言代码

```c
/*
 * C语言实现PID控制器
 *
 * 作者：Joshua Saxby（又名@saxbophone），创建于2016年1月1日
 *
 * 这是我尝试用简洁、易懂的C语言实现PID算法。
 *
 * 许可细节请查看LICENSE文件。
 */

// 防止头文件被重复包含
#ifndef SAXBOPHONE_PID_H
#define SAXBOPHONE_PID_H

#ifdef __cplusplus
extern "C"{
#endif


    typedef struct pid_calibration {
        /*
         * PID_Calibration 结构体
         *
         * 存储PID控制器校准后的参数（Kp、Ki、Kd）。
         * 这些参数用于调节算法，具体取值取决于实际应用场景。
         */
        double kp; // 比例增益
        double ki; // 积分增益
        double kd; // 微分增益
    } PID_Calibration;


    typedef struct pid_state {
        /*
         * PID_State 结构体
         *
         * 存储PID控制器的当前运行状态。
         * 作为PID算法函数的输入，同时函数也会返回更新后的结构体。
         *
         * 注意：output字段由PID算法函数计算赋值，计算过程中不会被读取。
         */
        double actual;        // 实际测量值
        double target;        // 目标设定值
        double time_delta;    // 距离上次采样/计算的时间间隔
        double previous_error;// 上一次的误差值（初始为0）
        double integral;      // 误差积分累加值
        double output;        // PID算法计算出的控制输出量，用于补偿误差
    } PID_State;


    /*
     * PID控制算法实现
     *
     * 传入PID校准参数（比例、积分、微分）和当前控制器状态，
     * 计算并返回PID控制器的新状态，并根据算法设置补偿误差的输出值。
     */
    PID_State pid_iterate(PID_Calibration calibration, PID_State state);


#ifdef __cplusplus
}
#endif

// 头文件结束
#endif
```

```
/*
 * C语言实现PID控制器
 *
 * 作者：Joshua Saxby（又名@saxbophone），创建于2016年1月1日
 *
 * 这是我尝试用简洁、易懂的C语言实现PID算法。
 *
 * 许可细节请查看LICENSE文件。
 */

#include "pid.h"


PID_State pid_iterate(PID_Calibration calibration, PID_State state) {
    // 计算目标值与实际值之间的差值（即误差）
    double error = state.target - state.actual;
    // 计算并更新积分值
    state.integral += (error * state.time_delta);
    // 计算微分值
    double derivative = (error - state.previous_error) / state.time_delta;
    // 根据PID公式计算输出值
    state.output = (
        (calibration.kp * error) + (calibration.ki * state.integral) + (calibration.kd * derivative)
    );
    // 将本次误差保存，作为下次的上一次误差
    state.previous_error = error;
    // 返回计算完成后的状态结构体
    return state;
}
```

