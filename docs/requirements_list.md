# IoT Fuzzing 平台需求列表 (IR-SF-SR)

本文档根据用户输入的初始需求 (IR)，推导出系统特性 (SF)，并进一步拆解为详细的系统需求 (SR)。

## 1. 初始需求 (Initial Requirements - IR)
根据用户输入的原始需求整理：
*   **IR-01**: 需要针对源码及其污点分析的灰/白盒 Fuzz 能力。
*   **IR-02**: 需要针对端侧设备的通信协议 Fuzz 能力。
*   **IR-03**: 需要针对端侧设备的 API 接口 Fuzz 能力。
*   **IR-04**: 需要针对 Android App 的 Fuzz 能力。
*   **IR-05**: 需要针对 HarmonyOS Next App 的 Fuzz 能力。
*   **IR-06**: (隐含) 需要全流程管理、结果汇总及闭环能力。

---

## 2. 需求拆解 (SF & SR)

### 针对 IR-01: 源码灰/白盒 Fuzz
*   **SF-01: 源码分析与增强引擎 (Source Code Analysis & Enhancement Engine)**
    *   **SR-01-01**: 系统应集成 CodeQL/Joern 工具，支持对 C/C++ 源码进行静态污点分析，识别 Sources 和 Sinks。
    *   **SR-01-02**: 系统应提供 Fuzz Harness 自动生成器，能依据静态分析的函数签名自动生成 LibFuzzer 驱动代码。
    *   **SR-01-03**: 系统构建环境应支持自动注入 ASan (AddressSanitizer) 和 UBSan 编译选项。

### 针对 IR-02: 端侧协议 Fuzz
*   **SF-02: 协议模糊测试代理 (Protocol Fuzzing Agent)**
    *   **SR-02-01**: 系统应支持定义和解析协议格式（如 MQTT, CoAP, Zigbee），并基于状态机生成测试用例。
    *   **SR-02-02**: 系统应支持调用宿主机的无线网卡接口（Monitor模式）发送 WiFi 管理帧/数据帧。
    *   **SR-02-03**: 系统应支持通过 BLE Dongle (如 Ubertooth) 进行低功耗蓝牙协议的嗅探与注入。
    *   **SR-02-04**: 系统应具备通过 UART 串口读取设备 Log 的能力，用于检测 Crash。
    *   **SR-02-05**: 系统应集成智能 PDU 控制接口，在检测到设备无响应时执行物理断电重启。

### 针对 IR-03: 端侧 API Fuzz
*   **SF-03: 服务接口模糊测试引擎 (API Fuzzing Engine)**
    *   **SR-03-01**: 系统应支持解析 OpenAPI/Swagger 规范文件，自动生成 Web 接口测试用例。
    *   **SR-03-02**: 系统应具备基于流量录制 (PCAP/HAR) 的回放与变异 Fuzz 能力，适配私有 API。
    *   **SR-03-03**: 系统应支持针对 HTTP、RPC 接口的参数类型混淆与边界值注入测试。

### 针对 IR-04: Android App Fuzz
*   **SF-04: Android 应用测试套件 (Android Fuzzing Suite)**
    *   **SR-04-01**: 系统应集成 ADB 桥接模块，支持对连接的 Android 设备进行组件枚举 (Activity/Service)。
    *   **SR-04-02**: 系统应具备 Intent Fuzzer 功能，能构造并发送畸形 Intent (空 Action、超长 Extra) 触发 Crash。
    *   **SR-04-03**: 系统应支持提取 APK 中的 `.so` 库，并在 QEMU-User 发行版或 Android 环境下运行 LibFuzzer 测试。

### 针对 IR-05: HarmonyOS Next App Fuzz
*   **SF-05: 鸿蒙原生应用测试套件 (HarmonyOS Next Fuzzing Suite)**
    *   **SR-05-01**: 系统应集成 HDC 工具链，支持连接鸿蒙设备并进行应用生命周期管理。
    *   **SR-05-02**: 系统应具备 ArkTS 接口 Fuzz 能力，支持生成随机 JS 对象测试 UIAbility/ExtensionAbility。
    *   **SR-05-03**: 系统应支持针对鸿蒙 Native 层 (C/C++) 的 N-API 接口进行 Fuzz Harness 生成与测试。

### 针对 IR-06: 全流程管理与闭环
*   **SF-06: 中央控制与报告中心 (Master Control & Reporting Center)**
    *   **SR-06-01**: 系统应提供 Web GUI 供用户创建任务，支持上传固件、安装包或配置目标 IP。
    *   **SR-06-02**: 系统应具备 Crash 自动归约 (Deduplication) 功能，通过堆栈哈希避免重复告警。
    *   **SR-06-03**: 系统应能生成包含详细堆栈、复现步骤的可导出报告 (HTML/PDF)。
    *   **SR-06-04**: 系统应提供对外 Webhook 接口，在发现高危漏洞时触发 CI/CD 流水线阻断或通知。
