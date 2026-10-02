# cz 容器格式规范 · v1

状态：**草案**（Phase 1 结束时冻结 v1）
本文档是 `.cz` 文件格式的**权威定义**。规范与代码冲突时，以本文档为准并修正代码。

关键词 MUST / MUST NOT / SHOULD / MAY 按 RFC 2119 解释。

---

## 1. 概述

cz 是一个**自描述**的压缩容器格式：一帧（frame）= 定长 32 字节帧头 + 变长载荷。
帧头携带原始大小与校验和，使解压可以：

1. 在读载荷之前预知输出上限（防解压炸弹）；
2. 校验"解出来的 = 当初压缩的"；
3. 不依赖任何外部元数据即可分发到正确的解码算法。

设计目标：**简单、可肉眼调试（十六进制查看器友好）、为演进预留空间**。
非目标：多文件归档（目录树打包不在此格式范围内）。

## 2. 帧布局

所有多字节整数一律**小端序（LE）**。比特流内部位序约定见 §6。

```
偏移    大小    字段            类型
0x00    4      Magic           4 字节常量 "CRZ1"
0x04    1      Version         u8
0x05    1      Codec ID        u8
0x06    1      Flags           u8
0x07    1      Reserved        u8
0x08    8      Original Size   u64  原始未压缩数据的字节数
0x10    8      Payload Size    u64  压缩载荷的字节数
0x18    4      CRC32           u32  原始数据的 CRC-32 校验和
0x1C    4      Reserved2       u32
0x20    N      Payload         N = Payload Size 字节的值
```

## 3. 字段语义

### 3.1 Magic

4 字节常量 `"CRZ1"`，即十六进制 `43 52 5A 31`。
解压时 MUST 校验，不匹配报 `Error::InvalidMagic`。

### 3.2 Version

当前值 `1`。解压器 MUST 拒绝大于自身支持版本的帧（`Error::UnsupportedVersion`）。
小版本演进优先使用 Flags 位与保留字段；仅当布局不兼容时才递增 Version。

### 3.3 Codec ID

| 值 | 算法 | 引入 Phase |
|----|------|-----------|
| 0  | RLE | 1 |
| 1  | Huffman（纯熵编码） | 1 |
| 2  | RLEHuff（RLE 预处理后接 Huffman） | 1 |
| 3  | LZSS | 2 |
| 4  | Deflate 风格（LZSS + 双 Huffman） | 2 |
| 5  | LZ + tANS/FSE | 3 |
| 6–255 | 保留 | — |

未识别的 Codec ID MUST 报 `Error::UnsupportedCodec`。

### 3.4 Flags

| 位 | 含义 |
|----|------|
| bit0 | Stored（直存）：载荷即原始数据，未做任何压缩 |
| bit1–bit7 | 保留，写入 MUST 置 0 |

Stored 位置 1 时：
- 载荷大小 MUST 等于原始大小；
- 解压器 MUST 将载荷原样复制为输出，不进入任何 codec 分发；
- 仍 MUST 执行 CRC 校验。

用途：输入为高熵随机数据时，压缩反而膨胀，此时直存保证输出至多比输入多 32 字节帧头。

### 3.5 Original Size（原始大小）

原始未压缩数据的字节总数。它是解压输出的**硬上限**：
解压过程中输出一旦超过该值，MUST 立即终止并报 `Error::SizeMismatch`。
这是防 zip-bomb 的第一道防线（压缩领域最重要的安全课题之一）。

### 3.6 Payload Size（载荷大小）

紧随帧头之后的载荷字节数，用于帧定界。文件总大小 MUST 等于
`32 + Payload Size`。不符报 `Error::CorruptData`。

### 3.7 CRC32

- 算法：CRC-32/ISO-HDLC（即 zlib 同款），反射多项式 `0xEDB88320`，
  初值 `0xFFFFFFFF`，结果异或 `0xFFFFFFFF`。
- 校验对象：**原始未压缩数据**（不是载荷）。语义是"解出来的 = 当初压缩的"，
  而非"字节没坏"。
- 压缩器 MUST 计算并写入；解压器 MUST 在输出完成后校验，不匹配报
  `Error::CrcMismatch`。

### 3.8 Reserved / Reserved2

写入 MUST 置 0；读取 SHOULD 忽略（向前兼容）。未来扩展优先占用保留字段。

## 4. 解压流程（规范性）

```
1. 读 32 字节帧头
2. 校验 Magic                     → InvalidMagic
3. 校验 Version ≤ 自身支持版本     → UnsupportedVersion
4. 查 Codec ID 注册表             → UnsupportedCodec
5. 校验文件剩余字节数 == Payload Size → CorruptData
6. 分配/预留 Original Size 输出空间（上限即帧头声称值）
7. Flags.stored == 1 ? 原样复制 : 调用 codec.decode(载荷, 输出, Original Size)
8. 校验输出字节数 == Original Size  → SizeMismatch
9. 校验 CRC32(输出) == 帧头 CRC32   → CrcMismatch
```

步骤 7–9 任一越界 MUST 立即中止，不得继续写输出。

## 5. 错误码 ↔ 格式违规对照

| Error | 触发条件 |
|-------|---------|
| `InvalidMagic` | Magic ≠ "CRZ1" |
| `UnsupportedVersion` | Version 未知或过新 |
| `UnsupportedCodec` | Codec ID 未注册 |
| `CorruptData` | 载荷长度不符 / codec 内部解码越界 |
| `SizeMismatch` | 实际输出 ≠ 声称的原始大小 |
| `CrcMismatch` | CRC32 校验失败 |

## 6. 位序约定（Phase 0 定义，全格式统一）

- 字节内比特按 **LSB-first** 处理（与 DEFLATE 一致），由 `BitReader/BitWriter` 统一实现；
- 任何 codec 的序列化细节（如 Huffman 码表）写在各自 Phase 的文档章节，但位序 MUST 全局一致。

## 7. 版本演进策略

1. 新增可选能力 → 优先启用一个保留 Flag 位，旧解压器忽略未知 Flag 时必须安全降级或明确报错（v1 起约定：**未知 Flag 置位 → 报 `UnsupportedVersion`**）。
2. 新增算法 → 只增加 Codec ID，布局不动。
3. 布局级变更 → Version 递增，保留字段空间有限，慎用。
