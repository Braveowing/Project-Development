# FR 写作模式参考（摘自 MPA Spec 1.7.2）

以下是从 MPA spec 中提取的真实 FR 片段，展示 1.5 UC 需求如何被拆解为 1.7.2 Function Requirement。

> **信号命名说明**：参考示例中的信号名只要能表达物理含义即可，不要求严格遵循现有系统命名规范。重点是表达清楚：状态怎么切换、哪些条件需要被满足、HMI 怎么交互、异常怎么处理、功能限制条件等。

---

## 模式一：状态转换型

**UC → FR 补充状态机细节**

**1.5 原始需求** (1.5.2.1)：
> The function shall automatically detect MPA-Domain and start Automated-MPA-Learning in the background.

**1.7.2 FR 拆解结果**：
> When MPA ODD has been detected, the system shall initialize learning automatically:
> - SIG_FUNC_STATE transit from NONE → LEARNING_READY
> - HMI module shall judge view display mode based on user setting:
>   - If setting is OFF: send SIG_HMI_POPUP = FUNC_AVAILABLE to display
>   - If setting is ON: auto-switch to learning view
>     - send SIG_VIEW_REQUEST = LEARNING_VIEW to display controller
>     - send SIG_POPUP = LEARNING_READY to display

**拆解要点**：UC 一句话「自动启动学习」→ FR 补充状态机转换、HMI 分支逻辑、信号交互代号。

---

## 模式二：条件检查型

**UC → FR 补充具体检查项**

**1.5 原始需求** (1.5.5.1 暗含)：
> 功能需要检查多种前置条件才能激活

**1.7.2 FR 拆解结果**：
> Initialization shall check function readiness = TRUE only if ALL conditions below are met:
> - SIG_VEHICLE_MODE = OPERATIONAL
> - SIG_VEHICLE_SPEED <= MAX_SPEED_SEARCH
> - SIG_SUSPEND_LEVEL = STANDARD
> - SIG_TRAILER_STATE = NOT_EQUIPPED OR RETRACTED
> - SIG_DRIVING_MODE = STANDARD OR SPORT
> - SIG_SNOWCHAIN = NOT_ACTIVE
> - SIG_SLOPE = INIT OR WITHIN_LIMIT
> - SIG_FUNC_CODED = ACTIVE

**拆解要点**：UC 笼统的「检查条件」→ FR 逐条列出信号代号、期望状态、AND/OR 逻辑关系。

---

## 模式三：数据操作型

**UC → FR 补充容量限制和覆盖策略**

**1.5 原始需求** (1.5.2.2)：
> The function shall provide a functionality to set a new MPA-Park-Position.

**1.7.2 FR 拆解结果**：
> If SIG_DOMAIN_STATUS = NEW_DOMAIN AND
> user confirms set position via HMI AND
> stored_count ≤ MAX_NUM_SLOT (2000, calibratable),
> system shall store target position as MPA-Park-Position.
>
> If stored_count > MAX_NUM_SLOT,
> system shall popup confirmation to overwrite by rule of "Last usage".

**拆解要点**：UC 说「提供设置车位的可能性」→ FR 补充前置条件链、容量限制、超限策略。

---

## 模式四：HMI 交互型

**UC → FR 补充信号链路和调用顺序**

**1.5 原始需求** (1.5.4)：
> The function shall provide a visualization of a map.

**1.7.2 FR 拆解结果**：
> Display unit shall request domain list from HMI-VAL via SIG_MAP_LIST_REQUEST.
> HMI-VAL shall respond via SIG_MAP_LIST_RESPONSE with:
> - list_size
> - total_memory
> - error_code
> Display unit shall then request each domain name via SIG_DOMAIN_NAME_REQUEST(index) for i times [i = list_size].

**拆解要点**：UC 说「提供可视化」→ FR 定义了三方信号交互的方法名、参数、循环调用顺序。信号名表达物理含义即可。

---

## 模式五：异常处理型

**UC 隐含需求 → FR 显式覆盖**

**1.5 原始需求**（隐含于 Learning 过程）：
> （无显式描述，但学习过程中速度超限需要处理）

**1.7.2 FR 拆解结果**：
> If vehicle speed > WARNING_SPEED_LEARNING:
> - send SIG_HMI_POPUP = REDUCE_SPEED to display
>
> If learning abort triggered:
> - current route recording shall NOT resume in this round
> - map data recording shall resume once recording conditions fulfilled again
> - send SIG_VIEW_REQUEST = DEFAULT_VIEW
> - send SIG_LEARNING_VIEW_ACTIVE = FALSE

**拆解要点**：UC 中没有显式提到异常路径 → FR 必须主动识别并覆盖。

---

## 模式六：预期功能表现与系统表现型（新增）

**UC → FR 充实场景、量化功能表现和系统执行逻辑**

**1.5 原始需求** (1.5.6.2)：
> The function shall provide the capability to detect and smooth cross the intersection/speed bump/ramp/sharp curve by adopting but not limited to slow down.

**1.7.2 FR 拆解结果**：

> **FR-06-03: Cross special road elements with speed adaptation**
> 追溯: 1.5.6.2
>
> The system shall detect intersection/speed bump/ramp/sharp curve and adapt speed below 10 kph.
>
> The system shall reduce maneuvering speed to ensure smooth crossing with longitudinal limitation when entering intersection/speed bump/ramp/sharp curve:
>
> **场景 A: 当前速度 ≥ 2.22 m/s**
> - lon_v shall decrease to 2.22 m/s
> - lon_a ≤ 0 m/s² (no acceleration)
> - abs(lon_jerk) ≤ 5 m/s³ in 1s (no uncomfortable brake/accelerate) AND filter bumping
>
> **场景 B: 当前速度 < 2.22 m/s**
> - abs(lon_jerk) ≤ 5 m/s³ in 1s (no uncomfortable brake/accelerate) AND filter bumping
>
> Note: The adaptable speed rate shall be selected so that a comfortable drive over is possible.

**拆解要点**：
- UC 只说「减速通过特殊路面」→ FR 按不同初始速度场景拆分为多条量化规则
- 每个场景明确写出**预期功能表现**（速度降到多少、加速度限制）和**预期系统执行逻辑**（纵向控制约束、舒适度约束）
- 性能指标完全量化：2.22 m/s、0 m/s²、5 m/s³、1s

---

## 信号代号约定参考

信号代号用于表达交互逻辑，只要能表达物理含义即可，不要求严格遵循系统命名规范：

| 物理含义 | 信号代号示例 | 说明 |
|---------|------------|------|
| 功能状态 | SIG_FUNC_STATE | 状态机的当前状态 |
| 功能就绪 | SIG_FUNC_READINESS | TRUE/FALSE |
| 用户请求 | SIG_USER_REQUEST | 用户触发的操作 |
| HMI 弹窗 | SIG_HMI_POPUP | 显示内容/类型 |
| 视图切换 | SIG_VIEW_REQUEST | 请求切换的视图 |
| 车辆速度 | SIG_VEHICLE_SPEED | 当前车速 |
| 档位状态 | SIG_GEAR_POSITION | 当前档位 |
| 刹车状态 | SIG_BRAKE_PEDAL | 踏板状态 |
| 转向状态 | SIG_STEERING_WHEEL | 方向盘状态 |
| 定位状态 | SIG_LOCALIZATION_STATUS | 定位质量 |
| 降级状态 | SIG_DEGRADATION_STATE | 功能降级程度 |
