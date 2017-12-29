---
layout: post
title: "Review of AWS re:invent"
description: ""
category: articles
tags: [AWS]
---
11月27日，我有幸参加了AWS re:invent大会。以下是我关注的一些AWS服务及个人看法，供参考。
## Amazon EKS
Amazon Elastic Container Service for Kubernetes (Amazon EKS) 是一项托管服务，借助该服务，您可以轻松在 AWS 上运行 Kubernetes，而无需安装和操作您自己的 Kubernetes 群集。借助 Amazon EKS, AWS 可以为您管理升级和高可用性。Amazon EKS 可在三个可用区中运行三个 Kubernetes 主节点，从而确保了高可用性。Amazon EKS 可以自动检测和替换运行状况不佳的主节点，并对主节点进行自动版本升级和修补。

### 优势
- 完全托管和高可用
- 和其他AWS服务高度集成 
- 自动升级和patch
- 和社区工具完全兼容

### 工作原理
![](/images/15129744814539.jpg)

### 常见案例
![](/images/15129745074877.jpg)

### 观感
借助EKS，我们可在云上实现完全托管的Kubernetes集群，而无需人工部署和维护。但从目前的文档来看，还无法和本地环境hybrid。

## AWS Fargate
AWS Fargate 是一项适用于 Amazon ECS 和 EKS* 的技术，让您无需管理服务器或群集即可运行容器。使用 AWS Fargate，您不必再预置、配置和扩展虚拟机群集即可运行容器。这样一来，您就无需再选择服务器类型、确定扩展群集的时间和优化群集打包。AWS Fargate 让您省去了考虑服务器和群集以及与之交互的麻烦。使用 Fargate，您可以专注于设计和构建应用程序，而不是管理运行应用程序的基础设施。
### 优势
- 无需管理集群
- 无缝扩展
- 与ECS和EKS集成

### 工作原理
![](/images/15129750528074.jpg)

### 观感
AWS Fargate实际就是一种部署和管理容器的方式，它使得用户无需关心底层容器的基础设施集群，而只需关心容器和应用即可。目前Fargate只设计支持ECS和EKS，这使得它的应用场景较窄。
## Amazon Aurora 
Amazon Aurora 是一种为云打造并且兼容 **MySQL** 和 **PostgreSQL** 的关系数据库，既具有高端商用数据库的性能和可用性，又具有开源数据库的简单性和成本效益。

Aurora 的速度最高可以达到标准 MySQL 数据库的五倍、标准 PostgreSQL 数据库的三倍。它可以实现商用数据库的安全性、可用性和可靠性，而成本只有商用数据库的 1/10。Aurora 由 Amazon Relational Database Service (RDS) 完全托管，而 RDS 可以自动执行各种耗时的管理任务，例如硬件预置以及数据库设置、修补和备份。

Aurora 采用一种分布式、有容错能力并且可以自我修复的存储系统，这一系统可以把每个数据库实例扩展到最高 64TB。Aurora 具备高性能和高可用性，支持最多 15 个低延迟读取副本、时间点恢复、持续备份到 Amazon S3，还支持跨三个可用区复制。

### 优势
- 高性能和高可扩展性
- 高可用性和高可持久性
- 高度安全
- 兼容MySQL和PostgreSQL
- 完全托管
- 迁移支持

### 观感
Amazon Aurora是一种为云打造的RDS服务，可兼容MySQL和PostgreSQL，并且性能是它们的3-5倍。

## DynamoDB Global Tables
Amazon DynamoDB global tables 可提供全托管的支持多region、多主的数据库集群，而无需自行实现数据复制方案。当创建一个global表时，用户可以指定表可用的region，DynamoDB可在这些地区间自动创建相同的表，并同步数据。
因为AWS服务绝大多数是和region关联的，所以这实际上是DynamoDB官方的跨region解决方案。目前支持的region包括：

* US East (N. Virginia)
* US East (Ohio)
* US West (Oregon)
* EU (Ireland)
* EU (Frankfurt)

我记得Andy Jassy说后续China Region在DynamoDB Global Tables会提供全球直连，这块还需和AWS进一步确认。

## AWS S3 As Data Lake
在会上，AWS明确了Amazon S3是实现Data Lakes的最好方式，并且给出了建立Data Lake的最佳实践。它的优势在于：

- 高可用、高可扩展和高持久性
- 安全、兼容和审计
- 对象级控制
- 商业级洞悉分析
- 多数据源导入
- 支持PB级的数据分析

配合Amazon Glue，可实现完整解决方案的Amazon S3 Data Lake：

![](/images/15129813924127.jpg)

![](/images/15129784028843.jpg)

在获取数据方面，Amazon S3后续支持使用标准SQL：

- 可在对象中select和retrieve数据
- 加速应用获取数据子集的性能
- 提升4倍的数据访问速度

### 观感
AWS在会上打造了以Amazon S3为存储、AWS Glue为数据目录和ETL服务的Data Lake标准，给出了完整的大数据解决方案的设计思路和具体实现，实现了大数据的闭环。

## Amazon Sagemaker
Amazon SageMaker 是一项完全托管的服务，使开发人员和数据科学家能够快速轻松地以任何规模构建、训练和部署机器学习模型。Amazon SageMaker 消除了通常会阻碍开发人员使用机器学习的所有障碍。
### 工作原理
![](/images/15129793085726.jpg)

### 优势
- 使用机器学习快速部署到生产中
- 选择任意框架或算法
- 一键式训练和部署
- 可与现有工作流集成
- 可访问经过训练的模型

## 总结
此次AWS re:invent，Amazon发布了很多重量级的应用，如Amazon EKS、DynamoDB Global Tables、S3 Select、Amazon SageMaker、AWS DeepLens等。在我看来，这些应用正在逐渐填补AWS在各个领域的空白，并最终使用AWS Lambda连接起来。AWS正努力建立一个个服务闭环，并应用到各个业务领域，并在此过程中正反馈给闭环网络。

个人认为，未来AWS的重心必会放在PaaS层，并且提供的PaaS服务一定是既独立又相互配合的，即模块化服务。这些服务可通过Amazon Lambda（接口），按照任意业务场景将AWS服务连接起来。
