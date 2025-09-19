### 1. 基础协议（base/proto.h、base/if.h、base/generic.h）

- **核心作用**：提供通用通信框架，包括版本控制、同步机制、消息结构、初始化流程等，为上层协议（网络、PCIe、内存）提供基础。
- **关键结构与宏**：
  - **协议版本与类型**：`SIMBRICKS_PROTO_VERSION` 定义协议版本；`SIMBRICKS_PROTO_ID_*` 区分上层协议类型（如网络、PCIe、内存）。
  - **初始化消息**：`SimbricksProtoListenerIntro`（监听端向连接端发送）和 `SimbricksProtoConnecterIntro`（连接端回应）用于建立连接，包含共享内存队列信息、同步标志、上层协议标识等。
  - **消息头部**：`SimbricksProtoBaseMsgHeader` 是所有消息的基础头部，包含时间戳（`timestamp`）、消息所有权及类型（`own_type`，通过掩码区分生产者 / 消费者和消息类型）。
  - **同步与终止消息**：定义了同步（`SIMBRICKS_PROTO_MSG_TYPE_SYNC`）和终止（`SIMBRICKS_PROTO_MSG_TYPE_TERMINATE`）等基础消息类型。
  - **通用接口生成**：`SIMBRICKS_BASEIF_GENERIC` 宏用于为上层协议生成标准化的消息处理函数（如消息 peek、poll、发送、释放等），简化代码复用。