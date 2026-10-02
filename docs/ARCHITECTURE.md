# 架构设计 · libcompress

状态：**v1**（随项目演进更新，重大变更追加决策记录）

## 1. 目标与非目标

**目标**
- 学习项目：亲手实现 RLE / Huffman / LZSS / ANS，理解 zstd、DEFLATE 的原理
- 真实工程形态：静态库 + 薄 CLI，公共 API 稳定、可测试、可演进
- 正确性优先于性能，性能优先于压缩率（Phase 4 再谈调优）

**非目标**
- 不追求超越 zstd/gzip 的压缩率或速度
- 不做归档格式（多文件/目录打包不在范围内）
- 不支持加密

## 2. 分层架构

```
┌──────────────────────────────────────────────┐
│ CLI (src/cli)         comp c/d/info 子命令   │
├──────────────────────────────────────────────┤
│ 公共 API (include/compress)  ← 唯一稳定契约  │
├──────────────────────────────────────────────┤
│ Format 容器层 (src/format)                   │
│   帧头读写 · CRC 校验 · codec 注册表分发     │
├──────────────────────────────────────────────┤
│ Codec 算法层 (src/codec)                     │
│   RLE / Huffman / LZSS / ANS  （可插拔）     │
├──────────────────────────────────────────────┤
│ Core 基础原语 (src/core)                     │
│   BitIO · CRC32 · Buffer                     │
└──────────────────────────────────────────────┘
```

**依赖规则（单向向下）**：CLI → API → Format → Codec → Core。
上层可 include 下层，禁止反向；跨层禁止（如 CLI 不得直接 include codec）。

**关键设计决策——格式与算法分离**：
- Format 层只认识 `ICodec` 接口和注册表，不 include 任何具体算法；
- 具体算法在各自编译单元内向注册表自注册（工厂函数）；
- 新增算法 = 新增一个文件 + 一行注册，格式层与 API 层零改动。

## 3. 模块职责

| 模块 | 职责 | 不允许 |
|------|------|--------|
| `cli` | 参数解析、文件 IO、错误信息 humanize、退出码 | 任何压缩逻辑 |
| `api`（include/compress） | 契约类型 + one-shot 编排：帧头 → 分发 → 校验 | 算法实现细节 |
| `format` | 帧头序列化/反序列化、注册表、分发 | 理解具体算法 |
| `codec` | 各算法的 encode/decode 与其私有序列化 | 越权写帧头 |
| `core` | BitIO、CRC32、通用缓冲 | 依赖任何上层 |

## 4. 公共 API 契约（C++20）

```cpp
namespace comp {

using Byte = std::uint8_t;

enum class Codec : uint8_t { RLE, Huffman, RLEHuff, LZSS, Deflate, Ans };
enum class Error {
    Ok, InvalidMagic, UnsupportedVersion, UnsupportedCodec,
    CorruptData, SizeMismatch, CrcMismatch
};

struct Options { Codec codec = Codec::RLEHuff; int level = 3; };

// 一次性接口（Phase 1）
Error compress(std::span<const Byte> in, std::vector<Byte>& out, const Options&);
Error decompress(std::span<const Byte> frame, std::vector<Byte>& out);

// 流式接口（Phase 3 设计并交付）
class Stream;  // ring buffer + push/drain，见 Phase 3 章节
}
```

约定：

1. **错误码，不抛异常**：所有返回 `Error` 的函数 `noexcept`（分配失败允许
   `std::bad_alloc` 传播，属文档化的例外，与 zlib/zstd 的务实做法一致）。
2. **输出缓冲由调用方提供并复用**：实现 MUST 先 `clear()` 再填充，
   便于调用方循环复用、减少分配。
3. **线程安全**：one-shot 函数无共享可变状态，可并发调用；
   Phase 3 的 Stream 对象非线程安全。
4. **`decompress` 信任帧头的 Original Size 为输出硬上限**（防炸弹，
   见 FORMAT.md §3.5）；CLI 层额外提供 `--max-output`（默认 1 GiB）。
5. **API 向后兼容承诺自 v1 冻结起生效**；新增能力用新函数/新枚举值，
   不改既有签名语义。

## 5. Codec 内部契约

```cpp
class ICodec {
public:
    virtual ~ICodec() = default;
    virtual CodecId   id() const = 0;
    virtual Error encode(std::span<const Byte> in, std::vector<Byte>& out, int level) = 0;
    virtual Error decode(std::span<const Byte> in, std::vector<Byte>& out,
                         size_t expected_size) = 0;
};
```

- `expected_size` 来自帧头，codec MUST 将其视为输出硬上限；
- codec 只负责载荷字节 ↔ 原始字节，**不接触帧头、不接触 CRC**；
- 复合算法（如 RLEHuff）在自身 encode 内部组合子算法，对外仍是单个 ICodec。

## 6. 测试策略（与代码同权重的交付物）

| 测试类型 | 内容 |
|----------|------|
| 单元测试（doctest） | BitIO 位序、CRC32 标准向量、各 codec encode/decode |
| 往返属性测试 | 随机数据 × 多种分布（全零/文本/二进制/高熵）× 各 codec，compress→decompress 逐字节相等 |
| Golden 文件 | 提交已知输入的压缩输出，格式被意外改动时测试立即失败 |
| 对抗测试 | 截断、位翻转、伪造帧头、炸弹帧——期望干净错误码，绝不崩溃/挂死 |
| 端到端 | CLI 对真实文件压缩后解压，比对哈希 |

## 7. 目录结构

```
Compression/
├── include/compress/    # 公共 API（对外唯一可见）
├── src/
│   ├── core/            # bitio.cpp/hpp, crc32.cpp/hpp, buffer.hpp
│   ├── codec/           # icodec.hpp, registry.cpp, rle.cpp, huffman.cpp, ...
│   ├── format/          # frame.cpp/hpp（帧头 + 分发）
│   └── cli/             # main.cpp, args.cpp
├── tests/               # doctest 单测 + golden/ + corpus/
├── docs/                # 本文档、FORMAT.md、ROADMAP.md
├── tools/               # benchmark（Phase 2 起）
└── CMakeLists.txt
```

## 8. 决策记录（ADR 摘要）

| # | 决策 | 理由 | 日期 |
|---|------|------|------|
| 1 | 语言 C++20 | `std::span` 进公共契约，现代编译器均已支持 | 2026-10-02 |
| 2 | 错误码风格，库边界不抛异常 | zlib/zstd 同款，性能可预测 | 2026-10-02 |
| 3 | 注册表模式接入 codec | 新算法零改动接入，依赖倒置 | 2026-10-02 |
| 4 | 位序 LSB-first 全局统一 | 与 DEFLATE 一致，Phase 2 可直接对照学习 | 2026-10-02 |
| 5 | 测试框架 doctest | 单头文件、编译快 | 2026-10-02 |
| 6 | 构建 CMake | 生态事实标准 | 2026-10-02 |
| 7 | 命名：libcompress / comp / .cz | 中性好记 | 2026-10-02 |
