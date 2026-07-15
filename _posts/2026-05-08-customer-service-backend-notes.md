---
title: 客服后台开发零散笔记
date: 2026-05-08 00:00:00 +0800
categories: [Backend, SpringBoot]
tags: [springboot, java, dto, jwt, mybatis, mq, aop, ods]
description: Spring Boot 客服后台开发过程中整理的零散概念笔记，覆盖 DTO、TTL、MQ、MyBatis、ODS、Mapper、XML、AOP 与 JWT 登录流程。
media_subpath: /assets/img/posts/2026-05-08-customer-service-backend-notes
---

# 客服后台开发零散笔记

## DTO

**DTO 的全称是 Data Transfer Object（数据传输对象）**

DTO 就是一个专门为“数据传输”定制的 Java 对象。它只包含接口调用方真正需要的字段，用来保护隐私、节省流量、解耦前后端。

## TTL

**TTL 是 Time To Live（存活时间）的缩写**，意思是**一个数据从创建开始，到被自动丢弃或失效为止的时长。**

## MQ

**MQ 是 Message Queue（消息队列）的缩写**，核心概念是**一种"异步通信"的中间件**。

你可以把它理解成一个可靠的、放在程序之间的"信箱系统"。

## MyBatis

**MyBatis 是 Java 世界里专门用来“操作数据库”的工具框架，你可以把它理解为“增强版的 JDBC”。**

它的核心作用是：让你用最简单的方式，把 Java 代码和数据库里的数据联系起来，把你从 JDBC 那些繁琐、重复的代码中解放出来。

## ODS

**ODS** 的全称是 **Operational Data Store**（操作型数据存储），在数据仓库体系中，它是介于业务数据库（DB）和数据仓库（DW）之间的一层，可以通俗地理解为 **"数据贴源层"**。

## Mapper

**Mapper 就是 MyBatis（或 MyBatis-Plus）框架中，专门用来“定义数据库操作”的那一层代码**。

简单来说，它就是 DAO 层（数据访问层）的具体实现形式。

## XML

**XML 是一种用来“存储和传输数据”的文本格式**，它的全称是 **Extensible Markup Language**（可扩展标记语言）。结合你的 Spring Boot 项目，可以这样理解：**XML 是 MyBatis 框架里，另一种写 SQL 语句的方式**（除了你之前看到的 `@Select` 注解）

![MyBatis XML 示意](image-20260508162507418.png)

## AOP

**AOP 是 Aspect-Oriented Programming（面向切面编程）的缩写**。

结合你的 Spring Boot 项目，可以这样理解：**AOP 是一种“在不修改原代码的情况下，给现有方法统一添加通用功能”的技术**。

## JWT

**JWT 的全称是 JSON Web Token**，是一种**紧凑且自包含的、用于在多方之间安全传输信息**的 JSON 对象。
JWT 默认不是加密的。

客服登录与鉴权流程：

```text
1. 客服登录（输入邮箱 + 动态密码）
   ↓
2. 服务器验证成功后，生成一个 JWT（里面包含：客服ID、角色、过期时间）
   ↓
3. 服务器把 JWT 返回给前端（通常在 HTTP 响应的 Body 或 Header 中）
   ↓
4. 前端把 JWT 存起来（通常是 localStorage 或 Cookie）
   ↓
5. 客服后续每次请求（比如查询玩家信息），前端在 HTTP 请求头中带上：
   Authorization: Bearer <JWT>
   ↓
6. 后端拦截器/过滤器接收到请求，解析并验证 JWT：
   - 签名是否正确？（用服务器密钥验证，防止伪造）
   - 是否过期？（JWT 里有过期时间）
   - 如果都合法，从 JWT 中取出客服 ID 和角色，放行请求
```
