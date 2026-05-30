# 第 0 章：整体架构概览

本章是 RMDB 学习之旅的起点。先搞清楚"数据库管理系统到底是什么"，然后用鸟瞰视角了解 RMDB 的整体架构，最后追踪一条 SQL 语句在系统内部的完整执行过程。

## 本章目录

| 序号 | 文档 | 内容 |
|------|------|------|
| 01 | [什么是数据库管理系统](./01-what-is-dbms.md) | 前置知识：DBMS 的定义、核心功能、与文件系统的区别 |
| 02 | [RMDB 分层架构](./02-layered-architecture.md) | RMDB 的模块组成、分层设计、各模块职责一览 |
| 03 | [SQL 执行全流程](./03-sql-execution-flow.md) | 一条 SQL 从客户端输入到返回结果的完整旅程 |

## 阅读建议

- 如果你对 DBMS 还完全没有概念，请从 01 开始顺序阅读
- 如果你已经了解 DBMS 基本概念，可以直接跳到 02
- 03 是整个架构的核心串联，建议阅读 02 后再看

## 关键缩写速查

| 缩写 | 全称 | 含义 |
|------|------|------|
| DBMS | Database Management System | 数据库管理系统 |
| RMDB / RM | Relational Model Database | 关系模型数据库（本项目的名称） |
| SQL | Structured Query Language | 结构化查询语言 |
| DDL | Data Definition Language | 数据定义语言（CREATE/DROP/ALTER） |
| DML | Data Manipulation Language | 数据操作语言（SELECT/INSERT/UPDATE/DELETE） |
| AST | Abstract Syntax Tree | 抽象语法树 |
