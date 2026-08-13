# PEDAL - 汽车驾驶模拟仪表盘 功能与设计文档

> 一个基于纯前端技术的拟物风格汽车驾驶模拟器，通过 Canvas 渲染高仿真仪表盘，配合物理引擎模拟加速、换挡、刹车、滑行等真实驾驶行为，并按车型参数严格校准 0-100 km/h 加速时间与最高车速。

---

## 目录

1. [项目概述](#1-项目概述)
2. [技术架构](#2-技术架构)
3. [功能模块](#3-功能模块)
4. [物理引擎设计](#4-物理引擎设计)
5. [校准系统设计](#5-校准系统设计)
6. [UI/UX 设计](#6-uiux-设计)
7. [Canvas 渲染实现](#7-canvas-渲染实现)
8. [车型数据结构](#8-车型数据结构)
9. [交互与控制](#9-交互与控制)
10. [性能优化](#10-性能优化)
11. [关键约束与规范](#11-关键约束与规范)
12. [已知设计决策](#12-已知设计决策)

---

## 1. 项目概述

### 1.1 项目定位

PEDAL 是一款面向汽车爱好者与驾驶模拟爱好者的纯前端单页应用，目标是：

- **拟物视觉**：通过 Canvas 多层光影渲染，复刻经典豪华车仪表盘的金属质感、玻璃反光、LED 灯珠发光等真实细节
- **物理保真**：基于真实车型参数（百公里加速时间、最高车速、刹车距离、变速箱齿比等）模拟加速、换挡、刹车、滑行过程
- **数据严格匹配**：用户按住油门时，0-100 km/h 加速时间与车型参数严格一致（误差 < 0.02s），最高车速同样匹配

### 1.2 文件结构

```
Pedal/
├── index.html          # 单文件应用（HTML + CSS + JS）
├── favicon*.png        # 多尺寸网站图标
└── DESIGN.md           # 本设计文档
```

单文件架构（约 3000 行）的优势：

- 零构建、零依赖、零打包工具
- 直接在浏览器中打开即可运行
- 便于离线分发与版本管理

### 1.3 目标用户

- 汽车爱好者，希望对比不同车型的加速表现
- 驾驶模拟爱好者，关注仪表盘视觉与物理反馈细节
- 前端开发者，学习 Canvas 拟物渲染与物理仿真

---

## 2. 技术架构

### 2.1 技术栈

| 技术 | 用途 |
|------|------|
| HTML5 | 页面骨架 |
| Tailwind CSS (CDN) | 布局与原子化样式 |
| 原生 CSS | 自定义组件样式、响应式断点、动画 |
| 原生 JavaScript (ES6+) | 全部业务逻辑、物理模拟、Canvas 渲染 |
| Canvas 2D API | 仪表盘、指针、LED 点阵、转速灯珠阵列渲染 |
| Google Fonts | Fraunces 衬线字体（标题）、Inter（正文）、Noto Sans SC（中文）|
| Font Awesome | 图标库 |

### 2.2 整体架构

采用经典的三层 MVC 分层：

```
┌─────────────────────────────────────────────────┐
│            Controller（控制器层）                │
│   事件绑定、键盘/触摸/鼠标统一处理、渲染循环      │
└────────────┬───────────────────────┬────────────┘
             │                       │
┌────────────▼───────────┐ ┌─────────▼──────────┐
│   SimulationEngine     │ │        UI          │
│   （模拟引擎/模型层）   │ │    （视图层）       │
│                        │ │                     │
│ • 物理状态             │ │ • DOM 操作          │
│ • 加速/刹车/滑行        │ │ • Canvas 渲染       │
│ • 换挡状态机           │ │ • 静态层缓存        │
│ • RPM 状态机           │ │ • LED 精灵缓存      │
│ • 校准系统             │ │                     │
└────────────┬───────────┘ └─────────┬──────────┘
             │                       │
             └───────────┬───────────┘
                         │
              ┌──────────▼──────────┐
              │      carData        │
              │   （数据层）         │
              │  26 品牌 120 车型   │
              └─────────────────────┘
```

### 2.3 启动流程

```javascript
Controller.init()
  ├─ UI.cacheElements()           // 缓存 DOM 引用
  ├─ UI.populateBrandSelect()     // 填充品牌下拉
  ├─ UI.renderSpeedometer()       // 初始空状态渲染
  └─ Controller.bindEvents()      // 绑定事件
```

---

## 3. 功能模块

### 3.1 车型选择

- **品牌选择**：22 个品牌（BMW、Mercedes-Benz、Audi、Porsche、Tesla、BYD、NIO、XPeng、Li Auto、Toyota、Honda、Volkswagen、Hyundai、Kia、Lexus、Volvo、MINI、Mazda、Nissan、Ford、Chevrolet、Subaru）
- **车型选择**：每个品牌 2-9 个车型，共 120 款（涵盖 BYD/NIO/XPeng/理想/特斯拉/奔驰/宝马/奥迪/保时捷/丰田/本田/大众/现代/起亚/雷克萨斯/沃尔沃/MINI/马自达/日产/福特/雪佛兰/斯巴鲁/极氪/精灵/吉利/长安）
- **车型信息卡**：显示车型名称、年份、0-100 km/h 加速时间、最高时速
- **详细参数面板**（右侧）：发动机、最大功率、变速箱类型、100-0 刹车距离、动力类型

### 3.2 驾驶控制

- **油门按钮**：按住式，松开自动进入滑行
- **刹车按钮**：按住式，松开自动进入滑行
- **键盘控制**：`W`/`↑` 油门，`S`/`↓` 刹车
- **触摸支持**：移动端固定底部控制栏，适配 `safe-area-inset`

### 3.3 仪表盘显示

#### 速度表
- 270° 表盘（-135° ~ +135°）
- 量程根据车型动态调整（向上取整到 40 的倍数，最小 200 km/h）
- 大刻度间隔 20 km/h，小刻度间隔 10 km/h
- 三层刻度绘制（阴影+主体+高光）模拟凸起金属质感
- 数字标注使用无衬线字体 Helvetica Neue

#### 转速表（仅燃油车）
- 内圈 RPM 弧（0-8000 RPM，红线 7000 RPM）
- 48 颗 LED 灯珠阵列，颜色随转速变化（低暖绿→中暖橙→高红）
- 红线区刻度发红光，警示三角符号
- 怠速标记（800 RPM 处蓝色小三角）
- 电动车隐藏转速表

#### 7段 LED 数字车速
- 3 位数字，5×7 点阵字体
- 琥珀色发光，未点亮段显示暗色凹槽
- 玻璃罩反光、整体辉光叠加

#### 档位指示器
- 单字符 5×7 点阵显示
- 燃油车：`1`-`9`（档位数字）、`N`（空挡）
- 电动车：`D`（前进挡）、`N`（空挡）
- 混动/CVT：`E`（经济模式）、`N`（空挡）

#### 指针
- 经典豪华车风格：细长锥形 + 红色尖端
- 抛光金属杆身渐变 + 中心高光条
- 配重尾端椭圆，转轴中心帽 7 层金属质感

### 3.4 驾驶状态

| 状态 | 触发 | 行为 |
|------|------|------|
| 加速 | 按下油门 | 持续加速至最高车速，自动换挡 |
| 刹车 | 按下刹车 | 减速至 0，含 ABS 低速衰减 + 发动机制动/EV 动能回收 |
| 滑行 | 松开油门/刹车且速度>0 | 自然减速，含发动机制动/动能回收 |
| 怠速回落 | 刹车至 0 但 RPM 高于怠速 | RPM 缓慢回落至怠速 |

---

## 4. 物理引擎设计

### 4.1 核心状态变量

```javascript
SimulationEngine = {
  speed: 0,              // 当前车速 km/h
  rpm: 0,                // 当前 RPM
  targetRpm: 0,          // 目标 RPM（用于平滑过渡）
  currentGearIndex: 0,   // 当前档位索引
  gearBoundaries: null,  // 升档边界速度数组
  isShifting: false,     // 是否换挡中
  shiftTimer: 0,         // 换挡计时器
  shiftDirection: 0,     // +1 升挡 / -1 降挡
  shiftStartRpm: 0,      // 换挡起始 RPM
  shiftTargetRpm: 0,     // 换挡结束目标 RPM
  lastSpeed: 0,          // 上一帧速度（判断加减速趋势）
  accelScale: 1,         // 加速度校准系数
  airDragCoeff: 0.0005,  // 空气阻力系数（校准后）
}
```

### 4.2 加速度计算

加速度公式（[index.html#L993-1029](file:///c:/Users/rette/OneDrive/Zone/Software/Pedal/index.html#L993-1029)）：

```
baseAccelMs2 = (100 / acceleration) × accelScale     // 基础加速度 m/s²
torqueFactor = calculateTorqueFactor(speed)            // 扭矩因子 0-1
torqueCutFactor = isShifting ? 0.60 : 1.0              // 换挡扭矩中断
speedLimiter = smoothstep 反转 (speedRatio 0.85~1.0)  // 软限速
airDrag = airDragCoeff × speedMs²                      // 空气阻力
rollingDrag = 0.3 m/s²                                 // 滚动阻力

drivingForce = max(0, baseAccel × torqueFactor × torqueCut × speedLimiter)
netAccelMs2 = drivingForce - airDrag - rollingDrag     // 允许负值（超速减速）
return netAccelMs2 × 3.6                               // 转 km/h/s
```

**关键设计**：驱动力 `Math.max(0, ...)` 防止负驱动力，但阻力始终扣除，允许车辆超过最高车速后自然减速。

### 4.3 扭矩因子（[index.html#L910-990](file:///c:/Users/rette/OneDrive/Zone/Software/Pedal/index.html#L910-990)）

#### 电动车（isEV）
```
tractionRamp = speed<10 ? smoothstep(speed/10)×0.05+0.95 : 1   // 起步牵引力建立
baseTorque = 1 - speedRatio^1.5                                 // 平扭矩曲线
torqueFactor = baseTorque × tractionRamp
```

#### 燃油车
```
clutchFactor = speed<30 ? smoothstep(speed/30)×0.3+0.7 : 1     // 离合器接合
gearShiftFactor = 1 - Σ(换挡点附近凹陷)                          // 换挡扭矩波动
engineTorque = peaky | linear 曲线                              // 发动机扭矩曲线
torqueFactor = clutchFactor × gearShiftFactor × engineTorque
```

**两种扭矩曲线**：
- `peaky`（涡轮增压）：0.92 起步 → 1.0 平台（至 55% 速度）→ smoothstep 下降
- `linear`（自然吸气）：0.92 起步 → 1.0 平台（至 35% 速度）→ smoothstep 下降

### 4.4 换挡状态机（[index.html#L764-884](file:///c:/Users/rette/OneDrive/Zone/Software/Pedal/index.html#L764-884)）

#### 档位边界计算（[index.html#L648-674](file:///c:/Users/rette/OneDrive/Zone/Software/Pedal/index.html#L648-674)）

基于真实齿比序列（参考 ZF 8HP、Aisin 6AT、Getrag 7DCT 等）：

```
gearMaxSpeed[i] = maxSpeed × (topGearRatio / ratios[i])
shiftPoint[i] = gearMaxSpeed[i] × 0.95          // 95% 红线换挡
shiftPoint[0] = gearMaxSpeed[0] × 0.98          // 1→2 额外提升（避免打滑）
```

#### 升挡过渡
- 时长 0.30s
- RPM 用 ease-out 曲线从高位平滑下降到新档位目标值

#### 降挡过渡
- 时长 0.45s
- **前 40%**：RPM 下降（离合分离，发动机失去负载）
- **后 60%**：RPM 跳升到新档位目标值（低档高转速，模拟转速同步）

#### 滞回与刹车降档（[index.html#L717-758](file:///c:/Users/rette/OneDrive/Zone/Software/Pedal/index.html#L717-758)）
```
hysteresis = maxSpeed × 0.06                    // 降档点比升档点低 6%
brakeBonus = isBraking ? maxSpeed × 0.08 : 0    // 刹车时更积极降档
downshiftPoint = boundaries[gear-1] - hysteresis - brakeBonus
```

#### 换挡扭矩中断
- `SHIFT_TORQUE_CUT = 0.60`：换挡期间驱动力保留 60%
- 模拟 AT 液力变矩器缓冲或 DCT 短暂中断
- 换挡触发帧立即应用（与 `simulate0to100` 一致）

### 4.5 RPM 状态机

#### 稳态 RPM
- 传统 AT/MT：`GEAR_MIN_RPM + ratio × (REDLINE_RPM - GEAR_MIN_RPM)`，档位内线性
- CVT/混动：起步快速升至 4500，之后缓慢升至 6000（模拟 e-CVT 高效区间）
- 电动车：返回 0（不显示转速表）

#### 低速混合（消除跳变）
速度 < 10 km/h 时，目标 RPM 从档位 RPM 平滑过渡到怠速：
```
t = smoothstep(speed / 10)
targetRpm = IDLE_RPM + (gearRpm - IDLE_RPM) × t
```

#### RPM 变化速率
| 状态 | 速率 (rpm/s) | 说明 |
|------|--------------|------|
| 加速 | 8000 | 快速上升 |
| 刹车 | 3500 + 档位因子×600 | 低档更强发动机制动 |
| 滑行 | 2100 + 档位因子×300 | 温和减速 |
| 停车后 | 2000 | 缓慢回落至怠速 |
| 稳态 | 2500 | 缓慢跟随 |

#### 怠速回落循环（[index.html#L1403-1424](file:///c:/Users/rette/OneDrive/Zone/Software/Pedal/index.html#L1403-1424)）
刹车至 0 但 RPM 仍高于怠速时，启动独立循环：
- 调用 `updateRPMAndGear(0, dt)` 推进 RPM 回落
- 任何新操作（加速/刹车/速度>0）介入即退出
- RPM 降至怠速+10 即结束

### 4.6 刹车系统（[index.html#L1200-1237](file:///c:/Users/rette/OneDrive/Zone/Software/Pedal/index.html#L1200-1237)）

```
baseBrakeMs2 = 27.78² / (2 × brakePerformance)   // 由 100-0 刹车距离反推
speedFactor = 低速时 ABS 衰减至 0.7                // speed<20 时 smoothstep
baseDecel = baseBrakeMs2 × 3.6 × speedFactor

engineBrakeDecel = 
  EV: 5.0 × (1 + 1.5 × speedRatio)               // 动能回收，高速更强
  燃油: 3.0 + (shiftCount - gear) × 1.5           // 发动机制动，低档更强

totalDecel = min(baseDecel + engineBrakeDecel, 42) // 抓地力上限 11.7 m/s²
```

### 4.7 滑行系统（[index.html#L1290-1342](file:///c:/Users/rette/OneDrive/Zone/Software/Pedal/index.html#L1290-1342)）

```
totalDecel = NATURAL_DECEL(3.6) + coastAirDrag×3.6

叠加：
  EV: 5.0 × (1 + 1.5 × speedRatio)              // 动能回收
  燃油: (3.0 + 档位因子×1.5) × max(0.3, speed/60) // 发动机制动随速度减弱
```

### 4.8 执行顺序（关键）

加速与刹车均遵循严格顺序，确保换挡触发与扭矩中断在同一帧生效：

```javascript
updateAcceleration(timestamp) {
  // 1. 先更新档位和 RPM（可能触发换挡，设置 isShifting）
  this.updateRPMAndGear(this.speed, deltaTime);
  // 2. 再用更新后的 isShifting 状态计算加速度
  const accel = this.calculateAcceleration(this.speed);
  this.speed = Math.max(0, this.speed + accel * deltaTime);
}
```

---

## 5. 校准系统设计

### 5.1 目标

确保按住油门时：
- 0-100 km/h 加速时间 = 车型参数 `acceleration`（误差 < 0.02s）
- 最高车速 = 车型参数 `maxSpeed`（稳态收敛）

### 5.2 双参数校准

| 参数 | 含义 | 影响维度 |
|------|------|----------|
| `accelScale` | 加速度缩放系数 | 0-100 时间 |
| `airDragCoeff` | 空气阻力系数 | 最高车速 |

### 5.3 四步校准流程（[index.html#L1109-1178](file:///c:/Users/rette/OneDrive/Zone/Software/Pedal/index.html#L1109-1178)）

```
第1步：在 airDrag=0 下二分搜索 accelScale（30 次迭代）
       → 使 0-100 时间匹配（无空气阻力干扰）

第2步：二分搜索 airDragCoeff（0.00005~0.08，40 次迭代）
       → 使最高车速匹配

第3步：重新校准 accelScale（0.70x~1.50x，40 次迭代）
       → 补偿空气阻力对 0-100 的影响

第4步：验证，若误差 > 0.02s 再微调（±5%，50 次迭代）
```

### 5.4 simulate0to100 与实际驾驶的一致性

`simulate0to100`（[index.html#L1035-1105](file:///c:/Users/rette/OneDrive/Zone/Software/Pedal/index.html#L1035-1105)）必须与 `updateAcceleration` 逻辑完全一致：

| 维度 | simulate0to100 | updateAcceleration |
|------|----------------|---------------------|
| 帧率 | dt = 0.0167 (60fps) | requestAnimationFrame ≈ 60fps |
| 执行顺序 | 档位检查 → 扭矩中断 → 加速度 | updateRPMAndGear → calculateAcceleration |
| 换挡触发 | 触发帧立即应用 torqueCut | 同 |
| 扭矩因子 | 共享 calculateTorqueFactor | 同 |
| 软限速 | 共享 smoothstep 公式 | 同 |
| 阻力 | airDrag + rollingDrag | 同 |

**这是确保校准结果在实际驾驶中可复现的核心保障。**

### 5.5 触发时机

```javascript
selectCar(car) {
  // ...
  setTimeout(() => this.calibrateAcceleration(), 0);  // 异步执行避免阻塞 UI
}
```

---

## 6. UI/UX 设计

### 6.1 视觉风格

- **色调**：深色主色 `#0e0e12`，暖色卡片 `rgba(255,255,255,0.04)`，琥珀金强调 `#d4a853`，警示红 `#ff3b30`
- **字体**：标题用 Fraunces 衬线（古典感），正文用 Inter + Noto Sans SC
- **卡片**：圆角 2rem，多层阴影，半透明背景
- **动画**：fade-in（0.5s）、slide-up（0.5s 错峰），按钮 `translateY(4px)` 模拟物理按下

### 6.2 布局

#### 桌面端（≥1024px）
三列网格：`1fr 1.4fr 1fr`
- 左：车型选择 + 当前车型信息
- 中：仪表盘 + 踏板
- 右：详细参数面板

#### 平板（768-1023px）
两列：左面板在上、右面板在下，中央仪表盘跨两行

#### 移动端（<768px）
单列垂直堆叠，底部固定控制栏（适配 `env(safe-area-inset-bottom)`）

### 6.3 踏板设计

按真实汽车踏板造型绘制 SVG（**无圆角矩形背景**）：

#### 刹车踏板（左侧/中间）
- 宽矩形造型，顶部略圆
- 5 条水平纹理线（防滑）
- 按下时变红

#### 油门踏板（右侧）
- 窄长楔形，略微倾斜
- 6 条水平纹理线
- 按下时变深灰

```html
<button class="brake-btn control-btn">
  <svg viewBox="0 0 60 80" fill="currentColor">
    <path d="M 10 8 Q 10 4 14 4 L 46 4 Q 50 4 50 8 L 52 70 ..."/>
    <line x1="18" y1="20" x2="42" y2="20" stroke="rgba(0,0,0,0.2)"/>
    ...
  </svg>
</button>
```

颜色通过 `fill="currentColor"` 继承按钮文本色，按下时切换 `color` 实现。

---

## 7. Canvas 渲染实现

### 7.1 高 DPI 支持

```javascript
const SIZE = 800;
const dpr = window.devicePixelRatio || 1;
canvas.width = SIZE * dpr;       // 内部分辨率
canvas.height = SIZE * dpr;
ctx.setTransform(dpr, 0, 0, dpr, 0, 0);  // 绘图坐标系仍 800x800
```

### 7.2 静态层缓存（[index.html#L1580-1850](file:///c:/Users/rette/OneDrive/Zone/Software/Pedal/index.html#L1580-1850)）

仪表盘静态部分（金属边框、刻度、数字、玻璃罩反光）预渲染到离屏 Canvas，仅在量程或车型类型变化时重绘：

```javascript
if (!this.speedometerStaticCanvas ||
    this._lastStaticMaxSpeed !== maxSpeed ||
    this._lastHasTach !== hasTach) {
  this.drawSpeedometerStatic(maxSpeed);
}
ctx.drawImage(this.speedometerStaticCanvas, 0, 0, SIZE, SIZE);
```

### 7.3 仪表盘静态层（拟物渲染）

按 9 个层次绘制：

1. **最外层投影**：Canvas 阴影 API
2. **金属外圈底色**：径向渐变 `#2a2a2e`→`#0e0e12`
3. **外圈顶部弧光**：线性渐变模拟上方光源
4. **外圈底部暗面**：底部阴影渐变
5. **玫瑰金金属边框环**：7 段线性渐变（`#e8e8ec`→`#606068`），顶部高光弧 + 底部暗线
6. **表盘底色**：径向渐变 `#2a2a30`→`#141418`
7. **表盘顶部微光**：上方弧形渐变
8. **刻度线三层绘制**：阴影（右下偏移）+ 主体（浅灰）+ 高光（左上偏移）
9. **玻璃罩反光**：上方大面积弧形渐变 + 左上角小亮点

### 7.4 指针渲染（[index.html#L2143-2245](file:///c:/Users/rette/OneDrive/Zone/Software/Pedal/index.html#L2143-2245)）

- **轮廓**：配重尾端 → 底部 → 锥形收窄 → 尖端，使用 `quadraticCurveTo` 平滑过渡
- **金属杆身**：7 段线性渐变（深灰→亮银→深灰）模拟抛光金属
- **红色尖端**：覆盖指针前 1/3，7 段红色渐变（`#8a1010`→`#e82828`）
- **中心高光条**：细长白色透明区域
- **配重尾端**：椭圆 + 径向渐变 + 高光点
- **投影**：阴影 API（offsetX=3, offsetY=4, blur=12）

### 7.5 中心帽（[index.html#L2248-2324](file:///c:/Users/rette/OneDrive/Zone/Software/Pedal/index.html#L2248-2324)）

7 层金属质感：
1. 外层阴影圈
2. 深色金属外环（线性渐变）
3. 主凸面（径向渐变，上亮下暗）
4. 同心圆装饰纹（模拟抛光纹理）
5. 顶部弧形高光
6. 内圈凹陷（径向渐变）
7. 中心小点

### 7.6 LED 点阵字体（[index.html#L2327-2481](file:///c:/Users/rette/OneDrive/Zone/Software/Pedal/index.html#L2327-2481)）

5×7 点阵字体，支持字符：`0-9 N D P R E S L`

每个字符为 7 行 × 5 列的二维数组，`1` 表示点亮，`0` 表示暗。

**'D' 字模特殊设计**（左侧直边无缺角，避免与 '0' 混淆）：
```
[1,1,1,1,0]
[1,0,0,0,1]
[1,0,0,0,1]
[1,0,0,0,1]
[1,0,0,0,1]
[1,0,0,0,1]
[1,1,1,1,0]
```

### 7.7 LED 灯珠精灵缓存（[index.html#L2007-2070](file:///c:/Users/rette/OneDrive/Zone/Software/Pedal/index.html#L2007-2070)）

每帧需绘制 100+ 灯珠（速度 3 字符 × 35 点 + 档位 35 点 + 转速 48 颗），逐帧创建径向渐变会严重卡顿。

**优化方案**：预渲染灯珠精灵到离屏 Canvas，按 `颜色|半径|是否点亮` 缓存：

```javascript
preRenderLedSprite(color, dotRadius, isLit) {
  const key = `${color.r},${color.g},${color.b}|${dotRadius}|${isLit?1:0}`;
  if (this._ledSpriteCache[key]) return this._ledSpriteCache[key];
  // 渲染 4 层：外发光晕 + 灯珠主体（径向渐变）+ 内核高亮 + 玻璃罩高光
  // ...
  this._ledSpriteCache[key] = sprite;
  return sprite;
}
```

绘制时通过 `globalAlpha` 控制亮度，`drawImage` 直接贴图，性能远优于逐层绘制。

### 7.8 转速 LED 阵列（[index.html#L2088-2140](file:///c:/Users/rette/OneDrive/Zone/Software/Pedal/index.html#L2088-2140)）

48 颗 LED 沿转速弧排列：
- 颜色随 RPM 变化：`getRpmColor(rpm)` 返回低暖绿/中暖橙/高红
- 三种状态：完全点亮（共享精灵）、部分点亮（最后一颗，按比例发光）、未点亮（暗色凹槽精灵）

### 7.9 渲染循环

```javascript
startRenderLoop() {
  const render = () => {
    UI.updateSpeedometer();
    if (SimulationEngine.speed > 0 || isAccelerating || isBraking) {
      this.renderFrame = requestAnimationFrame(render);
    } else {
      this.renderFrame = null;  // 停车时停止渲染节省 CPU
    }
  };
}
```

**按需渲染**：仅在驾驶状态活跃时运行 rAF 循环，停车后停止。

---

## 8. 车型数据结构

### 8.1 数据格式

```javascript
{
  model: "325Li",              // 车型名称
  year: 2023,                  // 年款
  acceleration: 7.9,           // 0-100 km/h 加速时间（秒）
  maxSpeed: 233,               // 最高车速 km/h
  engine: "2.0L 涡轮增压直四", // 发动机描述
  power: "184马力",            // 最大功率
  brakePerformance: 37,        // 100-0 km/h 刹车距离（米）
  isEV: false,                 // 是否电动车
  shiftCount: 8,               // 档位数（0=EV单速，1=CVT/混动，>1=传统AT/MT）
  shiftIntensity: 0.10,        // 换挡扭矩波动强度 0-1
  torqueCurve: "peaky",        // 扭矩曲线类型："peaky" 涡轮 / "linear" 自吸
  tractionBuildup: 0.6         // 起步牵引力建立系数 0-1
}
```

### 8.2 数据覆盖原则

- 主流品牌全覆盖（含中国品牌 BYD/NIO/XPeng/Li Auto）
- 必须保留奥迪 Q3 和 MINI 全系列
- 精简性能车/跑车，更多保留家用常见车型
- 优先选择加速数据丰富的车型
- 数据来源：汽车之家、懂车帝等权威中文汽车媒体

### 8.3 变速箱类型映射

| shiftCount | 类型 | 示例 |
|------------|------|------|
| 0 | 电动车单速 | Tesla Model 3 |
| 1 | E-CVT/混动 | BYD 秦PLUS DM-i、Toyota 凯美瑞双擎 |
| 5-9 | 传统 AT/MT/DCT | BMW 325Li (8AT)、Audi A4L (7DCT) |

---

## 9. 交互与控制

### 9.1 按钮事件绑定（[index.html#L2872-2881](file:///c:/Users/rette/OneDrive/Zone/Software/Pedal/index.html#L2872-2881)）

统一处理鼠标 + 触摸的按住式按钮：

```javascript
bindHoldButton(el, onStart, onStop) {
  const press = (e) => { e.preventDefault(); el.classList.add('is-pressed'); onStart(); };
  const release = () => { el.classList.remove('is-pressed'); onStop(); };
  el.addEventListener('mousedown', press);
  el.addEventListener('mouseup', release);
  el.addEventListener('mouseleave', release);   // 鼠标移出视为松开
  el.addEventListener('touchstart', press, { passive: false });
  el.addEventListener('touchend', release);
  el.addEventListener('touchcancel', release);  // 触摸中断
}
```

### 9.2 键盘控制

| 按键 | 行为 |
|------|------|
| `W` / `↑` | 油门（按下生效，松开停止） |
| `S` / `↓` | 刹车（按下生效，松开停止，速度为 0 时无效） |

键盘按下时同步给按钮添加 `is-pressed` 类，保持视觉一致。

### 9.3 状态切换

```
加速中 ──松开──→ 滑行 ──速度0──→ 怠速回落 ──RPM到怠速──→ 静止
  │                ↑
  └──按下刹车──→ 刹车 ──松开──→ 滑行
                  │
                  └──速度0──→ 怠速回落
```

任何新操作（加速/刹车）介入都会终止当前循环并切换状态。

---

## 10. 性能优化

### 10.1 静态层缓存

仪表盘静态部分（约 200+ 次绘制调用）预渲染到离屏 Canvas，每帧仅 `drawImage` 一次。

### 10.2 LED 精灵缓存

100+ LED 灯珠预渲染为精灵图，避免每帧创建径向渐变（性能提升约 5-10 倍）。

### 10.3 按需渲染循环

停车后自动停止 rAF 循环，CPU 占用降为 0；仅在加速/刹车/滑行时渲染。

### 10.4 高 DPI 适配

`setTransform(dpr, 0, 0, dpr, 0, 0)` 在 Retina 屏幕上保持清晰，同时避免重复设置 transform。

### 10.5 异步校准

`setTimeout(() => this.calibrateAcceleration(), 0)` 将校准计算推迟到下一个事件循环，避免阻塞 UI 渲染。

---

## 11. 关键约束与规范

### 11.1 物理约束（硬性）

- 0-100 km/h 加速时间必须严格匹配车型数据（误差 < 0.02s）
- 最高车速必须严格匹配车型数据
- 油门在右、刹车在左
- `simulate0to100` 与 `updateAcceleration` 必须逻辑一致（同帧率、同执行顺序、同扭矩中断时机）

### 11.2 UI 约束

- 踏板必须使用拟物 SVG 造型，**禁止**圆角矩形背景框
- 仅保留油门（右）和刹车（中）踏板，忽略离合器
- 所有英文文本与数字使用 Courier New 字体
- 速度表必须显示 3 位 7段 LED 数字车速（红橙色发光）
- **禁止**显示：5 个黑色圆形装饰、黑色矩形信息框、速度表上方红点
- 电动车档位：行驶/加速时显示 `D`，静止显示 `N`
- 混动/CVT：行驶/加速时显示 `E`，静止显示 `N`
- 传统 AT/MT：静止未加速显示 `N`，静止加速显示 `1`，行驶显示档位数字

### 11.3 功能约束

- **不增加**重置按键
- **不在**加速/刹车/滑行状态增加文字反馈
- **不保留** elapsedTime / totalDistance 等无关变量

---

## 12. 已知设计决策

### 12.1 为何 16段 LED 改为 7段？

早期实现使用 16 段 LED（可显示全字母），但在快速速度变化时出现渲染异常（字符状态泄漏、显示错乱）。7 段配合矩形段更稳定，且满足数字 + `N/D/E/P/R/S/L` 字符需求。

### 12.2 为何使用 smoothstep？

加速度曲线、扭矩过渡、软限速、低速 RPM 混合等场景均使用 smoothstep：
```
smoothstep(t) = t² × (3 - 2t)
```
优势：
- 端点导数为 0（C1 连续），无突变
- 单调递增，物理意义明确
- 计算简单，性能良好

### 12.3 为何换挡扭矩中断设为 60%？

`SHIFT_TORQUE_CUT = 0.60`：
- 0% 完全中断：顿挫过于明显，不真实
- 100% 无中断：失去换挡感
- 60% 保留：模拟 AT 液力变矩器的缓冲特性，轻微顿挫但可控

### 12.4 为何 1→2 档换挡点额外提升？

1 档齿比大（如 ZF 8HP 1档 4.71），全油门时轮胎易打滑。真实驾驶中 1→2 换挡点会比其他档位更高（接近红线），故 `SHIFT_1TO2_BONUS = 0.03`，1→2 换挡点为 98% 红线。

### 12.5 为何停车后需要独立怠速回落循环？

`stopBraking()` 在速度为 0 时不会启动任何 rAF 循环，导致 `updateRPMAndGear` 不再被调用，RPM 卡在高位。`updateIdleReturn` 循环解决此问题：以 2000 rpm/s 缓慢回落至怠速，模拟真实发动机停车后曲轴惯性。

### 12.6 为何校准分四步？

单次二分搜索无法同时匹配两个相互影响的参数（`accelScale` 影响加速，`airDrag` 影响最高车速，但两者对 0-100 时间都有影响）。四步法：
1. 无空气阻力下校准 accelScale（隔离变量）
2. 校准 airDragCoeff（匹配最高车速）
3. 重新校准 accelScale（补偿空气阻力对 0-100 的影响）
4. 验证 + 微调（确保最终误差 < 0.02s）

---

## 附录：核心代码位置索引

| 模块 | 行号范围 | 说明 |
|------|----------|------|
| CONSTANTS | L427-480 | 物理与视觉常量 |
| carData | L485-659 | 26 品牌 120 车型数据 |
| SimulationEngine | L613-1470 | 物理引擎 |
| - getGearBoundaries | L648-674 | 档位边界计算 |
| - calculateRPM | L680-712 | RPM 稳态计算 |
| - determineGear | L717-758 | 档位决策（含滞回） |
| - updateRPMAndGear | L764-884 | 换挡状态机 |
| - calculateTorqueFactor | L910-990 | 扭矩因子（EV/燃油） |
| - calculateAcceleration | L993-1029 | 加速度计算 |
| - simulate0to100 | L1035-1105 | 0-100 模拟（校准用） |
| - calibrateAcceleration | L1109-1178 | 四步校准 |
| - calculateBrakeDeceleration | L1200-1237 | 刹车减速度 |
| - updateCoasting | L1290-1342 | 滑行模拟 |
| - updateIdleReturn | L1403-1424 | 怠速回落循环 |
| UI | L1475-2853 | 视图层 |
| - drawSpeedometerStatic | L1580-1850 | 仪表盘静态层 |
| - drawTachometerStatic | L1853-1978 | 转速表静态层 |
| - preRenderLedSprite | L2007-2070 | LED 精灵缓存 |
| - drawLedSegments | L2088-2140 | 转速 LED 阵列 |
| - drawNeedle | L2143-2245 | 指针绘制 |
| - drawCenterCap | L2248-2324 | 中心帽绘制 |
| - DOT_MATRIX_FONT | L2327-2481 | 5×7 点阵字体 |
| - drawDigitalSpeed | L2504-2647 | 7段 LED 数字车速 |
| - drawGearIndicator | L2650-2750 | 档位指示器 |
| Controller | L2858-3053 | 控制器层 |
| - bindHoldButton | L2872-2881 | 按住式按钮绑定 |
| - handleKeyDown/Up | L2996-3029 | 键盘控制 |
| - startRenderLoop | L3032-3043 | 按需渲染循环 |

---

*文档版本：2026-06-30*
*对应代码版本：index.html (3062 行)*
