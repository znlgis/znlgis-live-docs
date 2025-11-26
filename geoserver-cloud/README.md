# GeoServer Cloud 培训教程

## 课程简介

本培训教程全面介绍 GeoServer Cloud —— 一个基于云原生架构的 GeoServer 微服务分布式版本。通过本教程，您将学习如何部署、配置、管理和扩展 GeoServer Cloud，以满足企业级地理空间数据服务的需求。

## 目标受众

- GIS开发工程师
- 运维工程师 (DevOps)
- 系统架构师
- 对云原生GIS服务感兴趣的技术人员

## 前置知识

- 基础的 GeoServer 使用经验
- Docker 和容器化基础知识
- Linux 系统管理基础
- 了解微服务架构概念（可选但推荐）
- Kubernetes 基础知识（高级章节需要）

## 培训大纲

### 第一章：GeoServer Cloud 概述与架构
- GeoServer 简介与发展历程
- 云原生计算的概念与优势
- GeoServer Cloud 的设计目标与愿景
- 微服务架构详解
- 技术栈介绍（Spring Boot、Spring Cloud）
- 支持的扩展列表

### 第二章：环境准备与安装部署
- 系统要求与依赖软件
- Docker 与 Docker Compose 安装配置
- 获取 GeoServer Cloud Docker 镜像
- 快速启动指南
- 验证安装与基本配置
- 常见安装问题排除

### 第三章：核心服务详解
- 微服务组件概览
- Gateway（网关服务）
- Discovery（服务发现）
- Config（配置服务）
- WMS（Web地图服务）
- WFS（Web要素服务）
- WCS（Web覆盖服务）
- WPS（Web处理服务）
- REST API 服务
- WebUI（Web管理界面）
- GeoWebCache（瓦片缓存服务）

### 第四章：目录与配置管理
- GeoServer Catalog 概念
- 配置后端选择
- PGConfig PostgreSQL 后端
- 数据目录后端
- JDBC Config 后端（已弃用）
- 外部化配置详解
- 配置最佳实践

### 第五章：安全配置与认证
- GeoServer Cloud 安全概述
- 基础认证配置
- LDAP 认证集成
- OAuth2 认证
- GeoServer ACL 访问控制
- JNDI 数据源配置
- 安全最佳实践

### 第六章：Docker Compose 部署实战
- Docker Compose 基础回顾
- 部署架构设计
- pgconfig 后端部署
- datadir 后端部署
- 服务扩缩容
- 监控集成（Prometheus/Grafana）
- 生产环境配置要点

### 第七章：Kubernetes 部署实战
- Kubernetes 基础概念
- Helm Chart 介绍
- 使用 Helm 部署 GeoServer Cloud
- 配置 Ingress 与负载均衡
- 持久化存储配置
- 自动扩缩容（HPA）
- 云平台部署（AWS/Azure/GCP）

### 第八章：运维监控与故障排除
- 日志管理与集中化
- 指标监控
- 健康检查与自愈
- 常见故障排除
- 性能调优
- 备份与恢复
- 升级与版本迁移

### 第九章：开发扩展与定制
- GeoServer Cloud 项目结构
- 开发环境搭建
- 构建说明
- 编码规范
- 创建自定义扩展
- 贡献代码指南
- 集成测试

### 第十章：最佳实践与案例分析
- 架构设计最佳实践
- 性能优化策略
- 高可用部署方案
- 多租户配置
- 典型应用场景分析
- 企业级部署案例
- 未来发展方向

## 章节文件

| 章节 | 文件名 | 描述 |
|------|--------|------|
| 第一章 | [第01章-概述与架构.md](第01章-概述与架构.md) | GeoServer Cloud 概述与架构 |
| 第二章 | [第02章-环境准备与安装部署.md](第02章-环境准备与安装部署.md) | 环境准备与安装部署 |
| 第三章 | [第03章-核心服务详解.md](第03章-核心服务详解.md) | 核心服务详解 |
| 第四章 | [第04章-目录与配置管理.md](第04章-目录与配置管理.md) | 目录与配置管理 |
| 第五章 | [第05章-安全配置与认证.md](第05章-安全配置与认证.md) | 安全配置与认证 |
| 第六章 | [第06章-Docker-Compose部署实战.md](第06章-Docker-Compose部署实战.md) | Docker Compose 部署实战 |
| 第七章 | [第07章-Kubernetes部署实战.md](第07章-Kubernetes部署实战.md) | Kubernetes 部署实战 |
| 第八章 | [第08章-运维监控与故障排除.md](第08章-运维监控与故障排除.md) | 运维监控与故障排除 |
| 第九章 | [第09章-开发扩展与定制.md](第09章-开发扩展与定制.md) | 开发扩展与定制 |
| 第十章 | [第10章-最佳实践与案例分析.md](第10章-最佳实践与案例分析.md) | 最佳实践与案例分析 |

## 参考资源

- [GeoServer Cloud GitHub 仓库](https://github.com/geoserver/geoserver-cloud)
- [GeoServer 官方文档](https://docs.geoserver.org/)
- [Spring Cloud 文档](https://spring.io/projects/spring-cloud)
- [Docker 官方文档](https://docs.docker.com/)
- [Kubernetes 官方文档](https://kubernetes.io/docs/)
- [Helm Chart 仓库](https://github.com/camptocamp/helm-geoserver-cloud)

## 版本信息

- GeoServer Cloud 版本：2.28.1.0
- GeoServer 版本：2.28.1
- 文档更新日期：2024年

## 许可证

本教程内容基于 GeoServer Cloud 官方文档整理编写，遵循 GPLv2 许可证。
