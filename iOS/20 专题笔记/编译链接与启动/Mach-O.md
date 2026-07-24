

## 2026-05-14 23:26 Mach-O 文件结构与加载机制
**来源**: 博客　**confidence**: 0.90

- Mach-O 文件由 Header、Load Commands、Segment/Section 三部分组成。
- Header 描述 CPU 架构和文件类型。
- Load Commands 指导 dyld 加载文件，常见命令如 LC_SEGMENT_64。
- LC_SEGMENT_64 定义段（如 __TEXT、__DATA）在文件和内存中的布局。

### 整理后内容

Mach-O 文件由 Header + Load Commands + Segment/Section 三部分组成。Header 描述 CPU 架构和文件类型，Load Commands 告诉 dyld 如何加载，最常见的 LC_SEGMENT_64 描述一个段（如 __TEXT、__DATA）在文件和内存中的布局。

<details><summary>原文</summary>

Mach-O 文件由 Header + Load Commands + Segment/Section 三部分组成。Header 描述 CPU 架构和文件类型，Load Commands 告诉 dyld 怎么加载，最常见的 LC_SEGMENT_64 描述一个段（如 __TEXT、__DATA）在文件和内存中的布局。

</details>

---
