### 3. 内存协议（mem/proto.h）

- **核心作用**：定义主机与内存模拟器间的通信协议，支持读写操作及完成通知。
- **关键结构与宏**：
  - **初始化消息**：`SimbricksProtoMemMemIntro`（内存端发送）和 `SimbricksProtoMemHostIntro`（主机端发送）用于内存接口握手。
  - **消息类型**：
    - 主机到内存（H2M）：读（`SIMBRICKS_PROTO_MEM_H2M_MSG_READ`）、写（`SIMBRICKS_PROTO_MEM_H2M_MSG_WRITE`）等，包含地址（`addr`）、长度（`len`）、请求 ID（`req_id`）等。
    - 内存到主机（M2H）：读完成（`SIMBRICKS_PROTO_MEM_M2H_MSG_READCOMP`）、写完成（`SIMBRICKS_PROTO_MEM_M2H_MSG_WRITECOMP`）等，携带请求 ID 和数据（读完成时）。
  - **消息校验**：通过 `SIMBRICKS_PROTO_MSG_SZCHECK` 确保消息结构大小为 64 字节（对齐和兼容性要求）。