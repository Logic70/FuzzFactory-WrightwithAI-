# IoT Fuzzing 平台设计评审意见书

**评审人**: Antigravity (Advanced Agentic AI)
**日期**: 2026-01-20
**评审对象**: HLD, LLD (SF-01 ~ SF-06)

---

## 1. 总体架构评审 (Architecture Review)

### 优点
*   **闭环完整**: 从源码分析到硬件控制，再到报告闭环，覆盖了 IoT 安全测试的完整生命周期。
*   **模块化清晰**: 将不同领域的 Fuzz 能力（协议、移动端、源码）拆分为独立 Agent，有利于技术栈隔离 (e.g., Python vs C++ vs Node.js)。
*   **硬件联动**: 引入智能 PDU 和 Monitor 网卡，解决了嵌入式 Fuzz 中最大的痛点——设备状态监控与恢复。

### 挑战与风险 (Challenges & Risks)

#### 1.1 "Agent 爆炸" 与运维复杂性
*   **问题**: 目前设计了 Protocol Agent (Kali), Mobile Agent (Host PC), Source Agent (Linux) 等多种异构节点。
*   **风险**: 
    *   **环境一致性**: 维护不同 OS (Windows/Mac/Kali/Ubuntu) 的依赖极其痛苦。
    *   **部署难度**: 用户需要准备多台物理机或虚拟机，且需直通各种 USB 硬件，部署门槛极高。
*   **建议**: 
    *   尽可能 **Docker 化**。除了必须直通 USB 的 Protocol Agent，其他组件（如 Source Code Fuzz）应严格运行在标准容器中。
    *   对于 Protocol Agent，提供预构建的 **ISO 镜像** 或 **Ansible Playbook**，而非仅提供源码。

#### 1.2 Master 节点的通信瓶颈
*   **问题**: Master 负责 Crash 归约和语料同步。如果 Agent 数量增多，大量的 Crash 上报（尤其是 False Positive）和 Corpus Sync 会阻塞 gRPC 通道。
*   **建议**: 
    *   **边缘计算**: 将 Crash 的初步去重 (Hash计算) 和 语料过滤 (Corpus Distillation) 下放到 Agent 端进行，Master 仅接收“高质量”数据。
    *   **流式存储**: Crash Dump 不要走 gRPC，直接由 Agent 上传至 S3/MinIO，仅将 Metadata 上报 Master。

---

## 2. 模块级深度评审 (Module-Level Review)

### SF-01: 源码分析 (CodeQL)
*   **痛点**: CodeQL 分析非常慢（尤其是包含大量依赖的 C++ 项目）。
*   **意见**: 
    *   不能在每次 Fuzz 任务启动时都跑 CodeQL。应该将 **静态分析 (Static Analysis)** 与 **Fuzz 执行 (Fuzzing Execution)** 解耦。
    *   静态分析作为“预处理”阶段，生成 Harness 代码存入仓库；Fuzz 阶段直接下载 Harness 编译运行。

### SF-02: 协议 Fuzz (Boofuzz)
*   **痛点**: 状态机（State Graph）的编写成本极高。
*   **意见**: 
    *   当前的自动化程度依赖于“人工定义协议类”。这在实际项目中很难落地，因为未知协议太多。
    *   建议增加 **流量学习 (Traffic Learning)** 功能：解析 PCAP 包自动生成初步的 Boofuzz 脚本，再由人工修正状态机，由“从零编写”转变为“从一优化”。

### SF-04/05: 移动端 Fuzz (Android/Harmony)
*   **痛点**: OS 更新极快，工具链易失效。
*   **意见**: 
    *   **Android**: QEMU-User 模拟 Native 库经常因为 syscall 不支持而 Crash。建议重度依赖 **真机 Fuzz**，并重点开发基于 Frida 的 In-Process Fuzzing，这种方式比 ADB 投递稳定性更高。
    *   **HarmonyOS**: 目前鸿蒙工具链（hdc, hypium）尚不成熟，版本变动大。设计中应增加一层 **Adapter 抽象层**，当 SDK 变更时仅需修改 Adapter，无需重写核心逻辑。

### SF-06: 结果闭环 (Deduplication)
*   **痛点**: 仅靠 Stack Hash 去重在 ASLR 或不同编译版本下并不可靠。
*   **意见**: 
    *   **标准化**: 必须在计算 Hash 前对 Stack Frame 进行规范化（Normalize），去除地址偏移，仅保留函数名+相对偏移。
    *   **GDB/LLDB 集成**: 对于高危 Crash，Agent 端应自动调用 GDB 抓取寄存器状态和周边内存，这比单纯的 Stack Trace 有用得多。

---

## 3. 易用性与工程化建议

1.  **硬件自检 (Self-Check) 是刚需**:
    *   在 Task 开始前，必须有一个 `pre-flight check` 阶段。自动检测：WiFi 网卡是否这就绪？PDU 是否可控？目标设备 IP 是否 Ping 通？
    *   否则，用户会因为一根线没插好而浪费一整晚的 Fuzz 时间。

2.  **Web UI 的“可观测性”**:
    *   Fuzzing 是一个黑盒过程。用户非常需要看到 **实时覆盖率 (Coverage)**、**执行速度 (Exec/s)** 和 **最近一次 Crash 的波形**。
    *   不要只展示最终结果，要展示“心电图”。

3.  **插件化架构**:
    *   PDU 驱动、协议解析器、报告模板，都应该是可插拔的 Plugin。避免将特定厂商的代码硬编码在核心库中。

## 4. 总结

该设计文档在 **广度** 上非常出色，覆盖了 IoT 安全的各个角落。但在 **深度落地** 上，最大的敌人是 **"碎片化"** (Fragmented Hardware/OS) 和 **"配置地狱"** (Configuration Hell)。

**核心建议**: 第一阶段先跑通 **"Master + Docker Agent (Source)"** 和 **"Master + One Protocol Agent"** 的最小闭环，不要试图一次性实现所有 OS 的支持。
