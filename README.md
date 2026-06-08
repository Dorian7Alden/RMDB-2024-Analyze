# RMDB 数据库内核源码分析与学习指导

## 项目简介

本项目是一个专注于关系型数据库管理系统（DBMS）内核的学习与源码分析项目。

本项目的核心目标是通过分析与拆解 RMDB（由中国人民大学数据库教学团队开发）的源码，建立一套完整的、循序渐进的数据库内核实现教程（所有产出均存放在 my-analysis 目录下，入口为 [my-analysis/README.md](my-analysis/README.md)）。在本项目中，我利用 AI（基于 [CLAUDE.md](CLAUDE.md) 的设定）作为虚拟学习导师，在了解每个数据库组件实现的过程中，逐步沉淀相关的技术文档与架构分析。

## 声明与致谢（必读）

1. **原生项目来源**：本仓库的底层参考源码来自于 [CSCC-DB-Rucbase-2024](https://github.com/Kosthi/CSCC-DB-Rucbase-2024)，原版为沈阳工业大学 DataDance 团队（参赛成员：吴奕民、张梦圆）参加“全国大学生计算机系统能力大赛-数据库管理系统设计赛”的决赛作品。
2. **版权声明**：原版源码遵循 [木兰宽松许可证，第2版 (Mulan PSL v2)](https://license.coscl.org.cn/MulanPSL2)。本项目仅用作个人学习、源码分析研究和教程编写，尊重原作版权，不代表原作者及相关赞助商的立场。
3. **学术诚信警示**：严禁任何参赛队伍直接复制本仓库的源码或分析文档用于提交相关比赛！ 本仓库是在开源作品与实际代码基础上进行的二次学习记录，保留了相关的代码指纹及防伪标记。因违规使用本仓库内容导致的学术处分、比赛资格取消等后果，由使用者自行承担。

## 项目结构定义

本仓库的目录结构划分明确，旨在区分学习指南、标准参考和动手实践三个核心部分：

- my-analysis 目录体系（见 [my-analysis/README.md](my-analysis/README.md)）：【核心产出】存放所有的源码分析、架构设计与学习指导文档
- src 目录：【标准参考】完整的 RMDB 参考实现源码（学习时的参考代码）
- db2026-x 目录：【动手实践】用来自己重新手写的题目空框架
- [.claude/skills/teach/SKILL.md](.claude/skills/teach/SKILL.md) 及其体系：AI 导师的学习过程控制库（Skills），定义了教学与写作的工作流记录
- [CLAUDE.md](CLAUDE.md)：定义了 AI 在本项目中作为“指导老师”的角色设定与全局约束
- docs 目录：官方提供的实验指导与说明文档
- [CMakeLists.txt](CMakeLists.txt)：RMDB 相关的测试与构建环境

## 学习路线规划

学习指导教程按照 DBMS 标准的分层架构组织，逐步递进：

- [my-analysis/00-architecture-overview/README.md](my-analysis/00-architecture-overview/README.md)：整体架构与 SQL 执行流程总览
- [my-analysis/01-storage-layer/README.md](my-analysis/01-storage-layer/README.md)：存储层（磁盘管理、缓冲池、页面替换策略等）
- [my-analysis/02-record-layer/README.md](my-analysis/02-record-layer/README.md)：记录层（数据页管理、定长/变长记录 CRUD 等）
- [my-analysis/03-index-layer/README.md](my-analysis/03-index-layer/README.md)：索引层（B+ 树、节点分裂与合并、范围查询等）
- [my-analysis/04-system-layer/README.md](my-analysis/04-system-layer/README.md)：系统层（数据库与表管理、目录维护等）
- [my-analysis/05-query-processing/README.md](my-analysis/05-query-processing/README.md)：查询处理层（SQL 解析、查询优化与执行器）
- [my-analysis/06-transaction-concurrency/README.md](my-analysis/06-transaction-concurrency/README.md)：事务与并发控制层（事务管理、锁机制、隔离级别等）
- [my-analysis/07-recovery/README.md](my-analysis/07-recovery/README.md)：恢复层（检查点与日志恢复机制机制）

## 环境与构建参考

运行或调试底层 RMDB 源码需要以下环境支持：
- **操作系统**：Ubuntu 18.04 及以上 (64位)
- **编译器**：GCC 7.1+ (要求完全支持 C++17)
- **构建工具**：CMake 3.16+

## License

本仓库的源代码部分沿用原项目的 [木兰宽松许可证，第2版](https://license.coscl.org.cn/MulanPSL2)。而产生的分析文档与教程仅供学术交流及个人学习，请遵守规范，尊重原创。
