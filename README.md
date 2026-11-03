# LeiSaiMotion
基于 C# WinForms 开发的雷赛运动控制卡仿真软件，仿制官方上位机核心界面与常用功能。实现轴参数设置（速度、加速度、限位）、JOG 点动、定长运动、回零、IO 监控、模块。采用 .NET Framework 4.7.2，分层架构（UI/业务/硬件抽象），使用状态机模拟卡响应，无需实际硬件即可学习和调试运动控制逻辑。代码注释清晰，适合工控上位机初学者或作为二次开发基础模板。不包含任何原厂受版权保护的资源，仅用于个人学习与交流。欢迎提出改进建议。
## 项目结构

```
LeiSaiMotion/
├── Project.sln                  # Visual Studio 解决方案文件
├── README.md
└── Project/                     # 主项目目录
    ├── Project.csproj           # 项目文件（ WinForms, x64）
    ├── Program.cs               # 应用程序入口（Main → Application.Run(new MainForm())）
    │
    ├── Model/                   # 领域模型层 —— 数据结构定义，不含业务逻辑
    │   ├── Axis.cs              # 轴参数：轴号、方向、距离、速度(MinVel/MaxVel)、加速度(Tac/STac)、运动模式(T曲线/S曲线)
    │   ├── CardAxis.cs          # 卡号与轴号换算工具：每卡12轴，全局轴 = 卡索引 × 12 + 本地轴
    │   ├── DMC1C80.cs           # 雷赛 DMC1C80 原生 DLL 的 P/Invoke 封装（所有 API 声明，含错误码枚举）
    │   ├── Home.cs              # 回零参数：轴号、启停速度、加速度、回零模式、原点信号类型
    │   └── IO.cs                # IO 模型：当前绑定的卡号
    │
    ├── DLL/                     # 业务逻辑层 —— 封装对硬件的调用，处理参数校验与返回值
    │   ├── AxisDLL.cs           # 定长运动：设置脉冲模式 → 配置速度曲线(T/S) → 调用 pmove 发送绝对位移
    │   ├── HomeDLL.cs           # 回零运动：配置原点信号 → 设置回零模式 → 配置速度曲线 → 启动回零
    │   └── IODLL.cs             # IO 操作：读写输入/输出点位，含界面索引与 API 索引的转换（显示从0起编，API从1起编）
    │
    ├── UI/                      # 界面层 —— WinForms 窗体
    │   ├── MainForm.cs          # 主界面：卡初始化(板卡/模拟双模式)、卡片选择(高亮)、连接/断开、打开子窗口
    │   ├── MotionForm.cs        # 运动控制：定长运动(绝对位移)、方向诊断(Ctrl+点击)、急停/减速停/位置清零、定时器刷新速度和位置
    │   └── IOForm.cs            # IO 监控：48路输入(PictureBox 指示灯) + 32路输出(Button 点动切换)，100ms 定时刷新，状态缓存跨窗口保持
    │
    ├── Resources/               # 图片资源
    │   ├── product.jpg          # 产品示意图
    │   ├── Card.png / Card1.png # 卡连接/断开状态图标
    │   ├── io.png / io1.png     # IO 指示灯（关/开）
    │   └── Connect.png / 断开连接.png / ...  # 工具栏按钮图标
    │
    └── Properties/              # 项目属性（程序集信息、资源/设置自动生成文件）
```

## 分层架构说明

项目按三层组织，调用方向从上到下单向依赖：

**UI 层**（`UI/`）负责用户交互和界面渲染。窗体直接调用 DLL 层的静态方法执行业务操作，通过 `csdmc1c80` 命名空间引用 P/Invoke 封装的函数读取实时状态（如速度、位置），用 WinForms Timer 驱动定时刷新，不直接操作原生 DLL。

**业务逻辑层**（`DLL/`）封装完整的运动控制流程。以 `AxisDLL.MoveAbsolute()` 为例：先设置脉冲输出模式，再根据 T 曲线或 S 曲线分别配置速度参数，最后调用 `pmove` 发送指令并返回成功/失败。该层与 Model 层的数据模型配合，接收参数对象而非散落的原始值，同时承担界面索引与实际 API 索引的转换（如 IO 点号偏移）。

**硬件抽象层**（`Model/DMC1C80.cs`）通过 `[DllImport]` 声明雷赛原厂 `dmc1c80.dll` 的全部 API 函数，使用 `CallingConvention.StdCall` 和 ANSI 字符集。所有函数声明为 `public static extern`，上层只需 `using csdmc1c80` 即可像调用本地方法一样调用硬件接口。同时定义了 ERR_CODE_DMC 错误码枚举，方便上层判断调用结果。

**Model 层**（`Model/` 其余文件）是纯数据结构，定义轴、回零、IO 等对象的属性，不含任何方法逻辑（`CardAxis` 是例外，作为静态工具类提供卡-轴索引换算）。

## 运行模式

- **硬件模式**：`d1c80_board_init()` 成功返回卡数 > 0，直接驱动真实 DMC1C80 控制卡。
- **模拟模式**：若 DLL 未找到（`DllNotFoundException`）或位数不匹配（`BadImageFormatException`），自动进入模拟模式，显示 4 张虚拟卡，界面可正常操作但无实际硬件输出，适合学习和调试。初始化逻辑位于 `MainForm.init()`。

## 环境要求

- Visual Studio 2017+（打开 `Project.sln` 编译运行）
- 如需硬件模式：安装雷赛 DMC1C80 驱动，并将 `dmc1c80.dll`（SDK 中 x64 版本）放入 exe 所在目录

欢迎提出改进建议。
