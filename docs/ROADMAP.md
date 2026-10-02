# 路线图 · Phase 0–4

原则：**每个 Phase 结束 = 可运行产物 + 测试全绿 + 一次 commit**。
永远保持 main 可编译可运行。

---

## Phase 0 · 地基

**交付物**
- [ ] CMake 骨架：`libcompress` 静态库目标 + `comp` CLI 目标 + doctest 测试目标
- [ ] `core/crc32`：标准 CRC-32/ISO-HDLC 实现 + 已知向量测试（"123456789" → 0xCBF43926）
- [ ] `core/bitio`：`BitWriter` / `BitReader`，LSB-first，flush 边界明确
- [ ] CLI 骨架：`comp info <file>` 能打印帧头（先占位）或提示格式未支持
- [ ] 往返测试脚手架（随机数据 × 分布生成器）

**验收**：`cmake --build build && ctest` 全绿；BitIO 写 N bit 读回 N bit 相等。

**学习点**：比特级 IO 为什么是压缩的地基；小端序与位序的区别。

## Phase 1 · 第一个可用的格式（v1 冻结）

**交付物**
- [ ] `codec/rle`：PackBits 风格（字面量段 / 重复段两种令牌）
- [ ] `codec/huffman`：频率统计 → 规范化 Huffman（码长上限 15）→ 编解码
  - 序列化：只存 256 个符号的码长表（256 B，Phase 2 再学 DEFLATE 的表头压缩，形成对比）
- [ ] `codec/rlehuff`：`huffman(rle(data))` 复合
- [ ] `format/frame`：帧头读写、注册表、分发、Flags.stored 直存路径
- [ ] CLI：`comp c <in> [-o out.cz] [-m codec]`、`comp d <in.cz> [-o out]`、`comp info <file.cz>`
- [ ] golden 文件 + 对抗测试（截断/位翻转/伪造头）

**验收**：真实文本文件压缩→解压逐字节相等；`comp info` 正确解析所有帧头字段；炸弹帧（声称巨大 Original Size 的截断帧）返回错误码而非崩溃/OOM。

**学习点**：熵、前缀码、canonical Huffman 为什么只存码长；防炸弹设计。

## Phase 2 · DEFLATE 风格（对标 gzip）

**交付物**
- [ ] `codec/lzss`：32 KiB 滑窗、最小匹配 3、hash chain 匹配查找、惰性匹配
- [ ] `codec/deflate_style`：LZSS 令牌流 → literal 树 + 距离/长度树（双 Huffman）
  - 码长表用"码长即符号"的二级 Huffman 压缩（DEFLATE 的 HCLEN 思想）
- [ ] `tools/benchmark`：压缩率 / 压缩速度 / 解压速度，对比 `gzip -1..-9`
- [ ] corpus 测试：一组多样化真实样本纳入常规测试

**验收**：文本类语料压缩率进入 gzip -6 的 ±15% 区间；benchmark 报告自动生成。

**学习点**：字典压缩与熵编码的分工；为什么 DEFLATE 是 LZ77 + Huffman 而非别的组合。

## Phase 3 · zstd 风格（现代熵编码 + 流式）

**交付物**
- [ ] `codec/ans`：tANS 表构建、序列化、编解码；替换/并列于 Huffman
- [ ] 流式 API `comp::Stream`：push/drain 模型、环形缓冲、流式帧（多帧拼接或块化，届时定型并更新 FORMAT.md）
- [ ] CLI 支持 stdin/stdout 管道

**验收**：大文件（> 内存余量）流式压缩成功；tANS 对比 Huffman 有可测的压缩率提升。

**学习点**：ANS 为什么用一张表逼近算术压缩率；流式状态机与一次性接口的本质差异。

## Phase 4 · 工程化打磨

**交付物**
- [ ] 压缩级别 -1..-9 映射到策略（匹配努力程度 / 直存阈值）
- [ ] fuzz：随机变异字节流持续打击 decoder（简单变异器起步，可选 libFuzzer）
- [ ] CI（GitHub Actions：Windows + Linux，build + test）
- [ ] README + 最终 benchmark 报告

**验收**：CI 全绿；fuzz 数小时无崩溃；项目可写进简历并被追问任何细节。

---

## Commit 节奏

- 每个 Phase 至少一次清晰 commit；大型子步骤（如 Huffman 完成编码）单独 commit
- 信息格式：`phase(N): 简述`，如 `phase(1): canonical huffman encode/decode`
- 格式冻结（Phase 1 结束）时在 FORMAT.md 顶部更新状态为 v1
