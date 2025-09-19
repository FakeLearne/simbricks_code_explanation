### 4. PCIe 协议（pcie/proto.h）

- **核心作用**：定义主机与 PCIe 设备模拟器间的通信协议，支持配置空间访问、数据传输、中断等。
- **关键结构与宏**：
  - **初始化消息**：`SimbricksProtoPcieDevIntro`（设备端发送）包含设备信息（如 BAR 配置、厂商 ID、中断向量数等）；`SimbricksProtoPcieHostIntro`（主机端回应）为占位结构。
  - **消息类型**：
    - 设备到主机（D2H）：读请求、写请求、中断（`SIMBRICKS_PROTO_PCIE_D2H_MSG_INTERRUPT`，支持 Legacy/MSI/MSI-X）、操作完成通知等。
    - 主机到设备（H2D）：读请求、写请求、设备控制（`SIMBRICKS_PROTO_PCIE_H2D_MSG_DEVCTRL`，如使能中断）等。
  - **中断类型**：通过 `SIMBRICKS_PROTO_PCIE_INT_*` 区分 Legacy/MSI/MSI-X 中断。