---
title: "云上细粒度访问管理的参考架构"
summary: "本文翻译自 Manav Mital 的文章 A Reference Architecture for Fine-Grained Access Management on the Cloud。"
authors: ["Manav Mital"]
translators: ["张晓辉"]
categories: ["云原生"]
tags: ["Kubernetes", "Security", "OPA"]
date: 2021-05-08T10:30:00+08:00
---

本文翻译自 [A Reference Architecture for Fine-Grained Access Management on the Cloud](https://www.infoq.com/articles/access-management-reference-architecture/)。

# 什么是访问管理？

访问管理是识别用户或一组用户是否应该能够访问给定资源（例如主机、服务或数据库）的过程。例如，对于开发人员来说是否可以使用 SSH 登录生产应用程序服务器，如果可以，那么可以登录多长时间？如果 SRE 在非支持时间尝试访问数据库，他们这样做？如果数据工程师已转移到其他团队，他们是否应该继续访问 ETL 管道的 S3 存储桶？

# 现在如何进行访问管理？

在云上各种基础设施和数据服务激增之前，访问管理是 DevOps 和 Security 团队要解决的相对简单的问题。VPN 和堡垒主机是（现在仍然是）在网络级别封锁所有关键资源的首选机制。用户必须先通过 VPN 服务器进行身份验证，或者登录到堡垒主机，然后才能访问专用网络上的所有资源。

![](https://atbug.oss-cn-hangzhou.aliyuncs.com/2021/04/27/16195301216852.jpg)

当资源是静态的并且它们的数量相对较小时，此方法效果很好。但是，随着越来越多的资源动态地涌入专用网络的各处，VPN / 堡垒主机解决方案变得站不住脚。

具体来说，在三个方面，VPN 和堡垒主机不足以作为一种有效的访问管理机制。

* **它们作用于网络层面**：用户通过 VPN 进行身份验证并获得对专用网络的访问权限后，他们实际上就可以访问其上运行的所有服务。无法根据用户的身份在基础架构或数据服务的粒度上管理访问。
* **凭据是攻击的媒介**：VPN 和堡垒主机都要求用户记住并存储凭据。过期和轮换凭证作为安全策略非常困难，尤其是在涉及大量用户的情况下，凭证因此成为潜在的攻击媒介。
* **不能管理第三方 SaaS 工具**：SaaS 工具（如 Looker、Tableau 和 Periscope Data）需要直接访问数据端点。因此，使用这些工具访问数据的任何人都无法通过使用了相同的机制和凭据的基础设施进行身份验证。

# 云上访问管理的新架构

在本文中，我们将定义新的参考架构，为那些正在寻求简化访问管理云资源（从 SSH 主机、数据库、数据仓库到消息管道和云存储终结点）解决方案的云原生企业。

它解决了 VPN 和堡垒主机无法克服的以下特定挑战：

* 在细粒度的服务级别上进行访问鉴权
* 消除共享凭据和个人帐户管理
* 通过第三方 SaaS 工具控制访问

此外，它为具有敏感数据的组织带来以下商业利益：

* 通过跨所有服务的会话记录和活动监视来满足 FedRamp 和 SOC2 等合规性标准的可审核性
* 基于访问者的身份，通过细粒度的授权策略来限制或清除敏感数据，从而实现隐私和数据治理

该架构建立在以下三个核心原则的基础上，这些原则的实现使 DevOps 和 Security 团队可以在对所有环境进行全面控制的同时，通过简单而一致的体验来提高用户的工作效率。

* 为访问资源的用户建立不可否认的身份
* 使用短期的短暂令牌和证书代替静态凭证和密钥
* 在一处集中所有资源类型的细粒度访问策略

下图显示了参考架构及其组件。

![](https://atbug.oss-cn-hangzhou.aliyuncs.com/2021/04/27/16195323349746.jpg)

上图中的 VPN / 堡垒主机已替换为接入网关（Access Gateway）。接入网关实际上是微服务的集合，负责验证单个用户、基于特定属性授权他们的请求，并最终授予他们访问专用网络中的基础结构和数据服务的权限。

接下来，让我们看一下各个组件，以了解之前概括的核心原理是如何实现的。

## 访问控制器

支持此体系结构的关键见解是将用户身份验证委派给单个服务（访问控制器），而不是将责任分配给用户可能需要访问的服务。这种联合在 SaaS 应用程序世界中很常见。由单一服务负责身份验证，可以简化应用程序所有者的用户配置和接触配置，并加快应用程序开发。

对于实际的身份验证序列，访问控制器本身通常会与身份提供商集成，例如 [Auth0](https://auth0.com/) 或 [Okta](https://www.okta.com/)，因此，可以跨提供者和协议提供有用的抽象。最终，身份提供商以签名的 SAML 声明\JWT 令牌或临时证书的形式保证用户的身份不可否认。这样就无需依赖受信任的子网作为用户身份的代理。与 VPN 允许用户访问网络上的所有服务不同，它还允许将访问策略配置到服务的粒度。

将身份验证委派给身份提供者的另一个好处是，可以使用零信任原则对用户进行身份验证。 具体来说，可以创建身份提供者策略以强制执行以下操作：

* 禁止从信誉不佳的地理位置和 IP 地址访问
* 禁止从已知漏洞的设备（未修补的 OS、较旧的浏览器等）进行访问
* 成功进行 SAML 交换后立即触发 MFA

### 身份验证序列如何工作：

1. 用户首先通过访问控制器进行身份验证，访问控制器又将身份验证委派给身份提供者。
2. 成功登录到身份提供者后，访问控制器将生成一个短暂的临时证书，进行签名并将其返回给用户。或者，它可以代替证书生成令牌。只要证书或令牌有效，就可以将其用于连接到 接入网关管理的任何授权基础设施或数据服务。到期后，必须获取新的证书或令牌。
3. 用户将在步骤（2）中获得的证书传递给他们选择的工具，然后连接到接入网关。根据用户请求访问的服务，基础设施网关或数据网关将首先允许访问控制器验证用户的证书，然后再允许他们访问该服务。因此，访问控制器充当用户与其访问的服务之间的 CA，因此为每个用户提供了不可否认的身份。

## 策略引擎

当访问控制器强制对用户进行身份验证时，策略引擎会对用户的请求强制进行细粒度的授权。它以易于使用的 YAML 语法接受授权规则（查看最后的示例），并根据用户请求和响应对它们进行评估。

开放策略代理（OPA）是一个开源的 CNCF 项目，是策略引擎的一个很好的例子。它可以自己作为微服务运行，也可以用作其他微服务进程空间中的库。OPA 中的策略以称为 Rego 的语言编写。另外，也可以在 Rego 之上轻松构建一个简单的 YAML 界面，以简化政策规范。

具有独立于基础结构和数据服务的安全模型的独立策略引擎的优点如下：

* 可以以与服务和位置无关的方式指定安全策略
    * 例如在所有 SSH 服务器上禁止特权命令
    * 例如强制执行 MFA 检查所有服务（基础设施和数据）
* 策略可以保存在一个地方并进行版本控制
    * 策略可以作为代码签入 GitHub 存储库
    * 每项变更在提交之前都要经过协作审核流程
    * 存在版本历史记录，可以轻松地还原策略更改

基础设施网关和数据网关都依赖于策略引擎，以分别评估用户的基础设施和数据活动。

## 基础设施网关

基础设施网关管理和监控对基础设施服务的访问，例如 SSH 服务器和 Kubernetes 集群。它与策略引擎连接，以确定细化的授权规则，并在用户会话期间对所有基础设施活动强制执行这些规则。 为了实现负载平衡，网关可以包含一组工作节点，可以在 AWS 上部署为自动扩展组，也可以在 Kubernetes 集群上作为副本集运行。

[Hashicorp 边界](https://www.boundaryproject.io/) 是基础设施网关的示例。这是一个开源项目，使开发人员、DevOps 和 SRE 可以使用细粒度的授权来安全地访问基础设施服务（SSH 服务器、Kubernetes 群集），而无需直接访问网络，同时又禁止使用 VPN 或堡垒主机。

基础设施网关支持 SSH 服务器和 Kubernetes 客户端使用的各种连接协议，并提供以下关键功能：

### 会话记录

这涉及复制用户在会话期间执行的每个命令。捕获的命令通常会附加其他信息，例如用户的身份、他们所属的各种身份提供者组、当天的时间、命令的持续时间以及响应的特征（是否成功、是否有错误、是否已读取或写入数据等）。

### 活动监控

监控使会话记录的概念更进一步。除了捕获所有命令和响应，基础设施网关还将安全策略应用于用户的活动。在发生违规的情况下，它可以选择触发警报、阻止有问题的命令及其响应或完全终止用户的会话。

## 数据网关

数据网关管理和监控对数据服务的访问，例如 MySQL、PostgreSQL 和 MongoDB 等托管数据库、AWS RDS 等 DBaaS 端点、Snowflake 和 Bigquery 等数据仓库、AWS S3 等云存储以及 Kafka 和 Kinesis。它与策略引擎连接，以确定细化的授权规则，并在用户会话期间对所有数据活动强制执行这些规则。

与基础设施网关类似，数据网关可以包含一组工作节点，可以在 AWS 上部署为自动扩展组，也可以在 Kubernetes 集群上作为副本集运行。

由于与基础设施服务相比，数据服务的种类更多，因此数据网关通常将支持大量的连接协议和语法。

此类数据网关的示例是 [Cyral](https://cyral.com/)，这是一种轻量级的拦截服务，以边车（sidecar）的方式部署来监控和管理对现代数据终端节点的访问，如 AWS RDS、Snowflake、Bigquery，、AWS S3、Apache Kafka 等。其功能包括：

### 会话记录

这类似于记录基础设施活动，并且涉及用户在会话期间执行的每个命令的副本，并使用丰富的审计信息进行注释。

### 活动监控

同样，这类似于监视基础设施活动。例如，以下策略阻止数据分析人员读取敏感的客户 PII。

### 隐私权执行

与基础设施服务不同，数据服务授予用户对通常位于数据库、数据仓库、云存储和消息管道中的与客户、合作伙伴和竞争对手有关的敏感数据的读写访问权限。 出于隐私原因，对数据网关的一个非常普遍的要求是能够清理（也称为令牌化或屏蔽）PII，例如电子邮件、姓名、社会保险号、信用卡号和地址。

## 那么这种体系结构如何简化访问管理？

让我们看一些常见的访问管理方案，以了解与使用 VPN 和堡垒主机相比，接入网关架构如何提供细粒度的控制。

## 特权活动监控（PAM）


这是一个简单的策略，可以在一个地方监视所有基础设施和数据服务中的特权活动：

* 仅允许属于 Admins 和 SRE 组的个人在 SSH 服务器、Kubernetes 集群和数据库上运行特权命令。
* 虽然可以运行特权命令，但是有一些例外形式的限制。具体来说，以下命令是不允许的：
    * “sudo” 和 “yum” 命令可能无法在任何 SSH 服务器上运行
    * “kubectl delete” 和 “kubectl taint” 命令可能无法在任何 Kubernetes 集群上运行
    * “drop table” 和 “create user” 命令可能无法在任何数据库上运行

![](https://atbug.oss-cn-hangzhou.aliyuncs.com/2021/04/27/16195354276750.jpg)

## 零特权（ZSP）执行

下一个策略显示了一个实施零特权的示例——一种默认情况下没有人可以访问基础设施或数据服务的范例。只有满足一个或多个合格标准，才能获得访问权限：

* 只允许属于支持组的个人访问
* 个人必须 on-call 才能获得访问权限。可以通过检查事件响应服务（例如 PagerDuty）中的时间表来确定通话状态
* 成功通过身份验证后会触发多因子身份验证（MFA）检查
* 他们必须使用 TLS 连接到基础设施或数据服务
* 最后，如果正在访问数据服务，则不允许进行全表扫描（例如，缺少 WHERE 或 LIMIT 子句的 SQL 请求最终将读取整个数据集）。

![](https://atbug.oss-cn-hangzhou.aliyuncs.com/2021/04/27/16195356881012.jpg)

## 隐私和数据保护

最后一条策略显示了涉及数据清理的数据治理示例：

* 如果市场营销人员正在访问 PII（社会保险号（SSN）、信用卡号（CCN）、年龄），先清洗数据然后再返回
* 如果有人正在使用 Looker 或 Tableau 服务访问 PII，同时清洗数据
* 清理规则由 PII 的特定类型定义
    * 对于 SSN，清洗前 5 位数字
    * 对于 CCN，清洗最后 4 位数字
    * 对于年龄，请清洗最后一位数字，即请求者将知道年龄段，但从不知道实际年龄

![](https://atbug.oss-cn-hangzhou.aliyuncs.com/2021/04/27/16195358881245.jpg)

## 概括

我们看到，对于高度动态的云环境，VPN 和堡垒主机不足以作为高效云环境中的有效访问管理机制。一种新的访问管理体系结构，其重点是不可否认的用户身份，短暂的证书或令牌以及集中的细粒度授权引擎，可有效解决 VPN 和堡垒主机无法解决的难题。除了为访问关键基础设施和数据服务的用户提供全面的安全性之外，该体系结构还可以帮助组织实现其审核、合规性、隐私和保护目标。

我们还讨论了该架构的参考实现，其中使用了以开发人员为中心的著名开源解决方案，例如 Hashicorp Boundary 和 OPA 以及 Cyral（一种用于现代数据服务的快速且无状态的辅助工具）。 他们一起可以在云上提供细粒度且易于使用的访问管理解决方案。

## 关于作者

![](https://atbug.oss-cn-hangzhou.aliyuncs.com/2021/04/27/16195361264391.jpg)

**Manav Mital** 是 Cyral 的联合创始人兼首席执行官，Cyral 是首个为数据云提供可见性、访问控制和保护的云原生安全服务。Cyral 成立于 2018 年，与各种组织合作 - 从云原生初创企业到财富 500 强企业，因为它们采用 DevOps 文化和云技术来管理和分析数据。 Manav 拥有 UCLA 的计算机科学硕士学位和坎普尔的印度理工学院的计算机科学学士学位。

## 关于译者

**Addo Zhang** 云原生从业人员，爱好各种代码。更多翻译：
* [分布式系统在 Kubernetes 上的进化](https://mp.weixin.qq.com/s/beRHn9l2K4eiS8M1IevcRA)
* [2021 年及未来的云原生预测](https://mp.weixin.qq.com/s/V6lO9sT_6hJVled9sOI4IA)
* [应用架构：为什么要随着市场演进](https://mp.weixin.qq.com/s/mw9LhDPiTyooUAXAoKHwTA)
