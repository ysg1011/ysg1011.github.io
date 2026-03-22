阶段 0: 硬件上电 & PBL
芯片上电 / PMIC 供电
         │
         ▼
┌─────────────────────────────────────────────┐
│  PBL (Primary Boot Loader)                  │
│  位于 SoC 内部 ROM，不可修改                 │
│  运行在: Boot ROM (片上 SRAM)                │
│  权限: EL3 (最高特权级)                      │
└─────────────────────────────────────────────┘

PBL 具体步骤
① 芯片 Reset 后，CPU Core 0 从 ROM 固定地址取指
② 初始化最低限度硬件:
   - PLL 锁定（最基础时钟）
   - UART 调试口（最早能出 log 的地方）
   - 电源复位状态检查
③ 选择 Boot Device（eMMC / UFS / SPI-NOR）
   - 读取 BOOT_CONFIG 寄存器决定
④ 从 Boot Device 读取 SBL(XBL) 到片上 SRAM
   - eMMC: 从 RPMB 或 User Area 特定偏移读取
   - UFS: 从 LUN 特定 LBA 读取
⑤ 验证 XBL 完整性:
   - 验证证书链 (PKC / ECDSA)
   - 验证镜像签名 (SHA256 + RSA)
   - 失败 → 停机 / 进入 EDL 模式 (Emergency Download)
⑥ 跳转到 XBL 入口

PBL 崩溃特征
- 无 UART 输出 → 硬件问题 / DDR 未初始化
- EDL 模式 → 短接 test point / 9008 端口
- SEC FAIL → 签名验证失败，分区损坏

--------------------------------------------------------------------------------
阶段 1: XBL (eXtensible Boot Loader)
XBL 分为两部分:

┌──────────────┐    ┌──────────────┐
│  XBL_SECAPP  │    │  XBL_CORE    │
│  安全启动 App │    │  核心引导     │
└──────┬───────┘    └──────┬───────┘
       │                   │
       │    ┌──────────────┘
       │    │
       ▼    ▼
  ┌──────────────┐
  │     ABL      │
  │  (UEFI封装)  │
  └──────────────┘

XBL_SECAPP (安全启动 App)
① 运行在 Secure World (EL3/S-EL3)
② 初始化 TrustZone 环境
③ 验证 XBL_CORE 签名
④ 验证后续所有分区签名链:
   boot → tz → devcfg → keymaster → ...
⑤ 初始化安全存储
⑥ 返回验证结果给 PBL → PBL 跳转 XBL_CORE

XBL_CORE (核心引导)
① DDR 初始化（最关键的硬件初始化之一）
   ├── 读取 DDR SPD 信息（频率/时序/容量）
   ├── 配置 DDR 控制器 (DDRCC)
   ├── 执行 DDR Training:
   │   ├── Write Leveling (WL)
   │   ├── DQ/DQS 训练
   │   ├── CA Training
   │   └── VREF Training
   ├── 配置 DDR 频率 (LPDDR4X/LPDDR5)
   └── DDR 容量检测，建立内存映射

② 时钟初始化
   ├── 配置 GCC (Global Clock Controller)
   ├── 配置主要 PLL:
   │   ├── GPLL0/1 (全局 PLL)
   │   ├── MMPLL (多媒体 PLL)
   │   └── GPU PLL
   └── 配置时钟分频树

③ PMIC 通信初始化
   ├── SPMI 总线初始化
   ├── 读取 PMIC 型号 (PM8150/PMK8550/PMR735D 等)
   ├── 配置电压轨 (LDO/Buck/Switch)
   └── 配置 PMIC GPIO

④ UFS/eMMC 控制器初始化
   ├── UFS PHY 初始化
   ├── 配置 UFS Host Controller
   └── 枚举设备，获取容量信息

⑤ 跳转到 ABL

DDR 初始化失败特征
- 死在 DDR Training → 硬件焊接 / PCB 问题
- DDR 容量错误 → SPD 信息异常
- 无法出 UART log → DDR 未就绪，无法加载后续代码到 DDR

--------------------------------------------------------------------------------
阶段 2: ABL (Android Boot Loader)
ABL 基于 LittleKernel (LK) 或 UEFI 封装

┌─────────────────────────────────────────┐
│  ABL 主要职责:                          │
│  1. 解析分区表                          │
│  2. 加载 boot.img                       │
│  3. 加载固件到 DDR                      │
│  4. 配置内核启动参数                     │
│  5. 跳转到 Kernel                       │
└─────────────────────────────────────────┘

ABL 详细步骤
① 读取分区表
   ├── GPT 分区表 (UFS: Primary + Backup GPT)
   ├── 解析所有分区:
   │   boot / vendor_boot / dtbo / vbmeta
   │   system / vendor / product / odm
   │   tz / modem / rpm / aop / cpucp / adsp / cdsp
   │   ...
   └── 挂载所需分区

② 加载并验证 boot.img (Android 11+ 分 A/B)
   ├── boot.img 结构:
   │   ┌──────────────────┐
   │   │  kernel header   │  (boot image header v4)
   │   │  kernel (Image)  │  (压缩的内核镜像)
   │   │  ramdisk (GKI)   │  (通用内核镜像 ramdisk)
   │   │  dtb             │  (设备树, Android 13+ 可选)
   │   │  bootconfig      │  (boot config, Android 13+)
   │   └──────────────────┘
   ├── 验证 vbmeta 签名 (AVB - Android Verified Boot)
   │   ├── 检查 vbmeta 分区头部
   │   ├── 验证 boot.img hash
   │   └── 验证 vendor_boot.img hash
   └── 解压 kernel 到 DDR

③ 加载 vendor_boot.img (Android 11+)
   ├── vendor ramdisk (厂商驱动/固件)
   ├── DTB overlay (设备树叠加)
   └── vendor bootconfig

④ 加载 dtbo.img (Device Tree Overlay)
   ├── 解析 DTBO 索引表
   ├── 选择对应硬件版本的 DT overlay
   └── 应用 overlay 到 base DT

⑤ 加载 Hypervisor (可选, 新平台)
   ├── 加载 Gunyah hypervisor 镜像
   ├── 配置 Stage-2 页表
   ├── 创建 pVM (Protected VM)
   └── Hypervisor 运行在 EL2

⑥ 加载 TrustZone 镜像
   ├── 加载 tz.img → Secure World (EL3/S-EL1)
   ├── 加载 keymaster → TA (Trusted Application)
   ├── 加载 cmnkeymaster / cmnlib
   └── TZ 初始化完成 → 准备好处理 SMC 调用

⑦ 加载远端子系统固件
   ├── 读取 firmware 目录:
   │   /vendor/firmware_mnt/image/
   │   ├── modem.mbn          → Modem (Hexagon DSP)
   │   ├── aop.mbn            → Always-On Processor
   │   ├── cpucp.elf          → CPU Control Processor
   │   ├── adsp*.mbn          → Audio DSP
   │   ├── cdsp*.mbn          → Compute DSP
   │   ├── slpi*.mbn          → Sensor Low Power Island
   │   └── ...
   ├── 将固件写入 DDR 指定保留区域
   ├── 配置远端处理器内存映射
   └── 配置 PIL (Peripheral Image Loader) 表

⑧ 配置内核启动参数 (cmdline)
   ├── 合并来源:
   │   ├── boot.img header 中的 cmdline
   │   ├── ABL 硬编码的 cmdline
   │   ├── 从 bootconfig 读取 (Android 13+)
   │   └── 动态生成 (DDR 容量、分区 UUID 等)
   ├── 典型参数:
   │   console=ttyMSM0,115200n8
   │   earlycon=msm_geni_serial,0xa90000
   │   androidboot.hardware=qcom
   │   androidboot.baseband=msm
   │   androidboot.slot_suffix=_a
   │   androidboot.verifiedbootstate=green
   │   androidboot.veritymode=enforcing
   │   androidboot.keymaster=1
   │   androidboot.vbmeta.device=...
   │   ...
   └── bootconfig 参数:
       androidboot.hardware.platform=sm8650
       androidboot.fstab.suffix=.qti

⑨ 跳转到 Kernel
   ├── 将控制权交给 kernel Image 入口
   ├── 传入参数:
   │   x0 = DTB 物理地址
   │   x1 = 0 (保留)
   │   x2 = 0 (保留)
   │   x3 = boot protocol 版本
   └── CPU 模式切换: EL2 → EL1 (如果有 Hypervisor)

--------------------------------------------------------------------------------
阶段 3: Linux Kernel 启动
Kernel 入口 (arch/arm6/kernel/head.S)
         │
         ▼
┌─────────────────────────────────────────────┐
│  __primary_switched (CPU 0)                 │
│  1. 计算内核 image 物理/虚拟地址偏移         │
│  2. 创建恒等映射 (identity mapping)          │
│  3. 创建内核页表                             │
│  4. 启用 MMU                                │
│  5. 设置栈指针                              │
│  6. 清除 BSS 段                             │
│  7. 调用 start_kernel()                     │
└─────────────────────────────────────────────┘

start_kernel() 详细初始化顺序
start_kernel()                          // init/main.c
│
├── setup_arch(&command_line)
│   ├── early_fixmap_init()             // 早期页表映射
│   ├── early_ioremap_init()
│   ├── setup_machine_fdt(__fdt_pointer) // 解析 DTB
│   │   ├── of_flat_dt_match_machine()  // 匹配机器类型
│   │   ├── early_init_dt_scan()        // 扫描 DTB 节点
│   │   └── unflatten_device_tree()     // 展开设备树
│   ├── arm64_memblock_init()
│   │   ├── memblock_add()              // 添加可用内存
│   │   ├── memblock_reserve()          // 保留固件/设备内存
│   │   └── memblock_allow_resize()
│   ├── paging_init()                   // 初始化完整页表
│   ├── parse_early_param()             // 解析 early 参数
│   └── qcom_scm_init()                 // QCOM SCM (SMC 调用)
│
├── trap_init()                         // 异常向量表
├── mm_init()
│   ├── page_ext_init()
│   ├── mem_init()                      // buddy 系统初始化
│   ├── kmem_cache_init()               // slab 分配器
│   └── percpu_init()
│
├── sched_init()                        // CFS 调度器初始化
├── early_irq_init()
├── init_IRQ()
│   ├── irqchip_init()                  // 从 DTB 初始化中断控制器
│   │   └── GICv3 初始化 (高通平台)
│   └── 中断向量表设置
│
├── timer_init()                        // 系统定时器
│   ├── clocksource_of_init()           // 从 DTB 初始化时钟源
│   └── clockevents_of_init()           // 时钟事件设备
│
├── time_init()                         // 时间子系统
├── softirq_init()                      // 软中断
├── console_init()                      // 早期控制台
│
├── rest_init()
│   ├── kernel_thread(kernel_init)      // 创建 init 线程 (PID 1)
│   └── kernel_thread(kthreadd)         // 创建 kthreadd (PID 2)
│
└── kernel_init()                       // 内核态 init 线程
    ├── kernel_init_freeable()
    │   ├── do_basic_setup()
    │   │   ├── cpuset_init_early()     // CPU 集合
    │   │   ├── drivers_init()          // 驱动核心初始化
    │   │   │   ├── devtmpfs_init()     // /dev 文件系统
    │   │   │   └── ...
    │   │   │
    │   │   ├── do_initcalls()          // ★ 驱动初始化主流程
    │   │   │   │
    │   │   │   │  执行顺序 (越早优先级越高):
    │   │   │   │
    │   │   │   ├── 0. pure_initcall
    │   │   │   │      └── 无依赖的最基础初始化
    │   │   │   │
    │   │   │   ├── 1. core_initcall
    │   │   │   │      ├── smp_init()           // 多核启动
    │   │   │   │      ├── radix_tree_init()
    │   │   │   │      └── ...
    │   │   │   │
    │   │   │   ├── 2. postcore_initcall
    │   │   │   │      ├── gic_init()           // GIC 中断控制器
    │   │   │   │      ├── of_platform_populate() // DTB → platform_device
    │   │   │   │      ├── iommu_init()         // IOMMU/SMMU
    │   │   │   │      └── ...
    │   │   │   │
    │   │   │   ├── 3. arch_initcall
    │   │   │   │      ├── qcom_scm_abc_init()  // QCOM SCM
    │   │   │   │      ├── qcom_rpmh_init()     // RPMh 电源管理
    │   │   │   │      └── ...
    │   │   │   │
    │   │   │   ├── 4. subsys_initcall
    │   │   │   │      ├── remoteproc_init()    // ★ 子系统固件加载
    │   │   │   │      │   ├── pil_init()       // Peripheral Image Loader
    │   │   │   │      │   ├── 加载 modem       // 基带处理器
    │   │   │   │      │   ├── 加载 adsp        // 音频 DSP
    │   │   │   │      │   ├── 加载 cdsp        // 计算 DSP
    │   │   │   │      │   └── 加载 slpi        // 传感器
    │   │   │   │      ├── platform_bus_init()
    │   │   │   │      └── ...
    │   │   │   │
    │   │   │   ├── 5. fs_initcall
    │   │   │   │      ├── proc_init()          // /proc
    │   │   │   │      ├── sysfs_init()         // /sys
    │   │   │   │      └── ...
    │   │   │   │
    │   │   │   ├── 6. device_initcall          // ★ 大部分驱动在这里
    │   │   │   │      ├── qcom_scm drivers
    │   │   │   │      ├── pinctrl (TLMM)       // 引脚复用
    │   │   │   │      ├── clk (GCC/CC/MCCC)    // 时钟驱动
    │   │   │   │      │   ├── gcc-sdxpinn/
    │   │   │   │      ├── regulator (SPMI)     // PMIC 电压调节
    │   │   │   │      ├── interconnect          // 总线带宽
    │   │   │   │      ├── ufs (UFSHC)          // 存储控制器
    │   │   │   │      ├── i2c/spi bus
    │   │   │   │      ├── iommu (SMMU)         // 内存管理单元
    │   │   │   │      ├── sde (Display)        // 显示控制器
    │   │   │   │      ├── kgsl (GPU)           // Adreno GPU
    │   │   │   │      ├── camss (Camera)       // Camera 子系统
    │   │   │   │      ├── qcom-audio (ASoC)    // 音频
    │   │   │   │      ├── ipa (数据路径加速)
    │   │   │   │      ├── usb (DWC3)
    │   │   │   │      ├── pcie
    │   │   │   │      ├── wlan (cnss2)         // WiFi
    │   │   │   │      ├── bluetooth
    │   │   │   │      ├── nfc
    │   │   │   │      └── ...
    │   │   │   │
    │   │   │   ├── 7. late_initcall
    │   │   │   │      ├── cpufreq (CPUfreq)    // CPU 调频
    │   │   │   │      ├── devfreq (GPU/DDR)    // 设备调频
    │   │   │   │      ├── thermal              // 温控框架
    │   │   │   │      └── ...
    │   │   │   │
    │   │   │   └── 8. console_initcall
    │   │   │          └── 高级控制台初始化
    │   │   │
    │   │   └── ...
    │   │
    │   ├── prepare_namespace()          // 挂载 rootfs
    │   │   ├── mount_root()
    │   │   │   ├── dm_init()            // device-mapper
    │   │   │   ├── 挂载 /dev/dm-0       // super 分区 (system/vendor/product)
    │   │   │   └── 挂载 /dev/dm-1       // userdata
    │   │   └── ...
    │   │
    │   └── ksoftirqd_init()            // 软中断线程
    │
    └── run_init_process("/init")        // exec /init → PID 1 → 进入用户空间

--------------------------------------------------------------------------------
阶段 4: Android Userspace
/init (PID 1)
│
├── early-init 阶段
│   ├── 创建 /dev, /proc, /sys 等挂载点
│   ├── 挂载 tmpfs, proc, sysfs, cgroup
│   ├── start ueventd                   // 设备节点创建
│   └── load_system_props()             // 加载系统属性
│
├── init 阶段
│   ├── setcon(u:object_r:init:s0)      // 设置 SELinux 上下文
│   ├── 安装 SELinux 策略
│   ├── 执行 /vendor/etc/init/*.rc      // 厂商 init 脚本
│   └── 执行 *.rc 中的 on init 触发器
│
├── late-init 阶段
│   │
│   ├── trigger late-init
│   │   │
│   │   ├── class_start early_hal       // 早期 HAL 服务
│   │   │   ├── hwservicemanager        // HIDL/AIDL HAL 注册中心
│   │   │   ├── vndservicemanager       // vendor HAL 注册中心
│   │   │   └── ...
│   │   │
│   │   ├── class_start early           // 早期服务
│   │   │   ├── ueventd                 // 设备节点守护
│   │   │   ├── watchdogd               // 看门狗
│   │   │   └── ...
│   │   │
│   │   ├── class_start main            // ★ 主要服务
│   │   │   ├── servicemanager          // Binder 服务注册中心
│   │   │   ├── hwservicemanager        // HIDL 服务管理
│   │   │   ├── logd                    // 日志守护
│   │   │   ├── surfaceflinger          // ★ 图形合成
│   │   │   ├── audioserver             // 音频服务
│   │   │   ├── cameraserver            // 相机服务
│   │   │   ├── media.codec             // 媒体编解码
│   │   │   ├── drm                     // DRM 服务
│   │   │   ├── keystore2               // 密钥存储
│   │   │   ├── gatekeeperd             // 门禁 (锁屏)
│   │   │   ├── android.hardware.graphics.composer
│   │   │   ├── vendor.camera-provider  // 相机 HAL
│   │   │   ├── vendor.audio-hal        // 音频 HAL
│   │   │   └── ...
│   │   │
│   │   └── class_start core            // ★ 核心服务
│   │       │
│   │       ├── zygote (app_process64)  // ★ Java 进程孵化器
│   │       │   └── ZygoteInit.main()
│   │       │       ├── preload()               // 预加载类/资源
│   │       │       │   ├── preloadClasses()    // 预加载 ~12000 个类
│   │       │       │   ├── preloadResources()  // 预加载 drawable/layout
│   │       │       │   └── preloadSharedLibs() // 预加载 .so
│   │       │       ├── forkSystemServer()      // ★ fork 出 system_server
│   │       │       └── runSelectLoop()         // 等待 fork App 进程
│   │       │
│   │       └── system_server (PID 自动分配)    // ★ Android 核心
│   │           └── SystemServer.java
│   │               │
│   │               ├── startBootstrapServices()
│   │               │   ├── Installer
│   │               │   ├── DeviceIdentifiersPolicyService
│   │               │   ├── ActivityManagerService  ★ (AMS)
│   │               │   ├── PowerManagerService     ★ (PMS)
│   │               │   ├── RecoverySystem
│   │               │   ├── LightsService
│   │               │   ├── DisplayManagerService
│   │               │   ├── PackageManagerService   ★ (PKMS)
│   │               │   ├── UserManagerService
│   │               │   ├── OverlayManagerService
│   │               │   └── SensorService
│   │               │
│   │               ├── startCoreServices()
│   │               │   ├── BatteryService
│   │               │   ├── UsageStatsService
│   │               │   ├── WebViewUpdateService
│   │               │   ├── CacheBinderService
│   │               │   └── ...
│   │               │
│   │               └── startOtherServices()
│   │                   ├── WindowManagerService    ★ (WMS)
│   │                   ├── InputManagerService     ★ (IMS)
│   │                   ├── AlarmManagerService
│   │                   ├── JobSchedulerService
│   │                   ├── NotificationManagerService
│   │                   ├── BluetoothService
│   │                   ├── WifiService
│   │                   ├── ConnectivityService
│   │                   ├── TelephonyRegistry
│   │                   └── ... (200+ services)
│   │
│   └── class_start late              // 延迟服务
│       ├── logcatd
│       └── ...
│
└── Android 系统就绪
    ├── Home Launcher 启动
    ├── Boot Animation 结束
    └── ACTION_BOOT_COMPLETED 广播

--------------------------------------------------------------------------------
各阶段时间线 (典型值)
PBL              │  ~100 ms     │  ROM 代码
XBL/DDR Init     │  ~800 ms     │  DDR training 占大头
ABL/加载固件     │  ~500 ms     │  加载 6-8 个子系统固件
Kernel           │  ~1500 ms    │  驱动 probe 占大头
Userspace        │  ~5000 ms    │  Zygote preload + SystemServer
                 │              │
Total Boot       │  ~8-10 s     │  到 Launcher 可用

--------------------------------------------------------------------------------
稳定性视角的关键阶段
阶段常见崩溃排查手段PBLSEC FAIL / 死机UART log / EDL 模式DDR InitTraining 失败DDR 寄存器 dump / 硬件测量ABL固件加载失败ABL log (UART)TZTA 崩溃 / SMC 超时dmesg | grep qseecomKernel earlyOops/Panic in initcallearlycon UART / ramdumpKernel driverprobe 失败 / IRQ 风暴dmesg / ftraceRemoteproc子系统启动失败cat /sys/kernel/debug/remoteproc/*/statusinit服务启动超时logcat / adb bugreportSystemServerWatchdog 超时 / Binder 死锁am watchdog / binder state
