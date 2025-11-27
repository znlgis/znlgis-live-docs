# SOD框架学习与开发教程

> **SOD框架**（SQL-MAP、ORM、Data Controls）是一个拥有超过15年历史的国产开源企业级数据应用开发框架。本教程旨在帮助开发者全面了解、学习和掌握SOD框架，包括框架的设计理念、核心功能、使用方法以及最佳实践。

## 框架简介

SOD框架诞生于2006年，最初名为PDF.NET框架，后发展演变为SOD框架。框架名称来源于其三大核心功能模块的英文首字母缩写：

- **S** - SQL-MAP：基于XML配置的SQL查询和数据访问层映射
- **O** - ORM：对象关系映射，包含独特的OQL查询语言  
- **D** - Data Controls：数据窗体控件，支持WebForm/WinForm/WPF

**框架目标**：实现对数据访问细节的全方位掌控能力！

## 教程目录

### 基础篇

| 章节 | 标题 | 主要内容 |
|------|------|----------|
| [第一章](01-框架概述与设计理念.md) | 框架概述与设计理念 | SOD框架简介、核心设计理念、架构概览、与其他ORM框架对比 |
| [第二章](02-框架架构与核心组件.md) | 框架架构与核心组件 | 分层架构、AdoHelper、EntityBase、OQL、EntityQuery等核心组件详解 |
| [第三章](03-快速入门与环境配置.md) | 快速入门与环境配置 | 开发环境准备、项目创建、NuGet包安装、第一个SOD程序 |

### 核心功能篇

| 章节 | 标题 | 主要内容 |
|------|------|----------|
| [第四章](04-实体类与ORM映射.md) | 实体类与ORM映射 | EntityBase详解、元数据映射、数据类型映射、动态映射技术 |
| [第五章](05-OQL查询语言详解.md) | OQL查询语言详解 | OQL语法、条件表达式、分页查询、关联查询、聚合查询、GOQL |
| [第六章](06-SQL-MAP技术深入.md) | SQL-MAP技术深入 | XML配置、抽象SQL参数、DAL代码生成、SQL-MAP与实体结合 |
| [第七章](07-数据窗体开发.md) | 数据窗体开发 | WebForm数据控件、WinForm数据控件、DataFormHelper、MVVM模式 |

### 高级篇

| 章节 | 标题 | 主要内容 |
|------|------|----------|
| [第八章](08-企业级解决方案.md) | 企业级解决方案 | 内存数据库、事务日志复制、分布式事务、分布式ID、热缓存、分表分库 |
| [第九章](09-高级特性与扩展.md) | 高级特性与扩展 | 多数据库支持、自定义数据提供程序、查询日志、存储过程、性能优化 |
| [第十章](10-实战案例与最佳实践.md) | 实战案例与最佳实践 | 项目架构设计、完整案例、代码组织、最佳实践、常见问题解答 |

## 快速开始

### 1. 安装NuGet包

```bash
# .NET 6.0及以上版本
dotnet add package PWMIS.SOD

# .NET 5及以下版本
dotnet add package PDF.NET.SOD
```

### 2. 配置数据库连接

```xml
<configuration>
  <connectionStrings>
    <add name="local" 
         connectionString="Data Source=.;Initial Catalog=MyDB;Integrated Security=True" 
         providerName="SqlServer"/>
  </connectionStrings>
</configuration>
```

### 3. 定义实体类

```csharp
public class UserEntity : EntityBase
{
    public UserEntity()
    {
        TableName = "TbUser";
        IdentityName = "ID";
        PrimaryKeys.Add("ID");
    }

    public int ID
    {
        get { return getProperty<int>("ID"); }
        set { setProperty("ID", value); }
    }

    public string Name
    {
        get { return getProperty<string>("Name"); }
        set { setProperty("Name", value, 50); }
    }
}
```

### 4. 执行CRUD操作

```csharp
// 查询
UserEntity user = new UserEntity { Name = "张三" };
var oql = OQL.From(user).Select().Where(user.Name).END;
var users = EntityQuery<UserEntity>.QueryList(oql);

// 插入
var newUser = new UserEntity { Name = "李四" };
EntityQuery<UserEntity>.Instance.Insert(newUser);

// 更新
newUser.Name = "王五";
EntityQuery<UserEntity>.Instance.Update(newUser);

// 删除
EntityQuery<UserEntity>.Instance.Delete(newUser);
```

## 框架特点

### 核心优势

✅ **简单易用**：类似SQL的OQL语法，几乎零学习成本  
✅ **动态映射**：运行时可修改元数据，支持分表分库  
✅ **多模式开发**：SQL-MAP、ORM、Data Controls三种模式灵活选择  
✅ **高性能**：精确SQL生成，减少不必要的查询  
✅ **跨数据库**：支持SQL Server、MySQL、Oracle、PostgreSQL、SQLite等  
✅ **国产化支持**：良好支持达梦、人大金仓等国产数据库  
✅ **轻量级**：核心DLL只有几百KB  
✅ **兼容性强**：支持.NET 2.0到.NET 8.0全版本  

### 适用场景

- 对数据操作安全有严格要求的金融行业
- 对数据访问速度有苛刻要求的互联网行业
- 需求常常变化，要求快速开发上线的项目
- 需要长期维护的企业级应用（MIS、ERP、MES等）
- 预算有限的中小团队
- 需要支持国产数据库的国产化项目

## 相关资源

- **官方网站**：http://www.pwmis.com/sqlmap
- **GitHub仓库**：https://github.com/znlgis/sod
- **Gitee仓库**：https://gitee.com/znlgis/sod
- **作者博客**：https://www.cnblogs.com/bluedoctor
- **QQ群**：18215717、154224970
- **官方图书**：《SOD框架企业级应用数据架构实战》

## 版权说明

SOD框架采用LGPL开源协议，可以在商业项目中免费使用。

---

*本教程由深入分析SOD框架源码和官方文档整理而成，如有疑问请参考官方资源或加入QQ群交流。*
