- 原文：[Designing a DIY PaaS](https://medium.com/@sandipd/designing-a-paas-platform-as-a-service-add1e3519b5)
- 翻译：[GarenRhine](http://ghost.garenfeather.cn/author/garenrhine/)
- 未授权，转载请注明出处


----------


# 设计一个你自己的PaaS（平台即服务）

本文面向想要创建一个自定义PaaS来简化部署工作流程的开发者，这是PaaS架构、实现与部署系列的第一篇  

<!--more-->

## PaaS是什么

平台即服务（Platform as a Service），或者说PaaS，是指基于底层基础设施架构的平台和环境抽象（这里的公有云提供商被称为基础架构即服务，IaaS，提供商），它能够让开发者更轻松的部署应用和服务。

PaaS平台通常都托管在云上，作为由提供商负责维护的共享/专有基础设施池的一部分（例：Heroku，Bluemix）或者一系列归属用户、能够部署在基础设施上的平台工具（例：Pivotal Cloud Foundry，Openstack）。

> 在Hasura，你可以在托管和自托管版本之间选择，两者之间可以无缝迁移

如今一个新的趋势是大规模组织会构建他们自己的PaaS（即自定义的PaaS）因此他们的工具都是量身定制的。而在这篇文章中，为了想要准备构建自己的PaaS工具的开发者们，我们回顾一下PaaS应致力于解决的问题，解决这些问题的潜在方法以及做出任何技术选择时应有的考虑


----------


## PaaS需要解决什么问题

作为基础设施的抽象层，一个PaaS首先要解决以下问题：

 1. 使代码独立于其运行的机器
 2. 在软件部署和扩容上提供协助开发者的工具

支撑一个典型的代码-机器交互工作流程需要的一些抽象层：

![PaaS需要的抽象层](https://cdn-images-1.medium.com/max/600/1*8vuFxeKbDqNjtgXDDokfXQ.png)

 1. 预分配设施
 2. 在设施上安装依赖
 3. 机器上安装代码/二进制文件
 4. 编辑配置文件
 5. 在supervisor监控下运行代码
 6. 更新前端服务器/API网关

让我们看看上面提到的问题和潜在解决方案更多的细节：

### 1）预分配设施

**问题：** 需要创建、安装或配置**大量**如下的实体：

![](https://cdn-images-1.medium.com/max/800/1*NHZCpZNdM1l_xcEVIWLMng.png)

**解决方案：** 以下选项组成了该问题的潜在解决方案：

 1. 使用虚拟化的基础设施
 2. 将 CoreOS 用于：
	 - 完全回滚式更新
	 - 将操作系统与应用程序级别的问题分开
 3. 使用基础设施提供商的API来自动化虚拟机的生命周期：
	- 创建一个虚拟机
	- 更新一个虚拟机
	- 删除一个虚拟机
	- 将一个虚拟机添加到集群
	- 从集群中移除一个虚拟机

### 2）在设施上安装依赖

**问题：**  PaaS需要解决系统和应用两方面的依赖问题

**解决方案：** 简单——使用容器就行（[_Docker_](https://www.docker.com/)_,_ [_Containerd_](https://containerd.io/) _或者_ [_rkt_](https://coreos.com/rkt)）。容器帮你将所有的依赖跟代码一起打包在一个镜像里，而打包他们的任务被转交给开发者。

![](https://cdn-images-1.medium.com/max/800/1*e6k8G58FhyzVc6KDM_LF4Q.png)

### 3）机器上安装代码/二进制

**问题：**  代码频繁更新。每一次更新最新的代码都需要安装到虚拟机上。

**解决方案：** 容器又一次成为了救星！此问题一个优雅的解决方案是为每次代码更新都创建一个新的容器镜像。 由代码仓库每一次更新引发的一个基于git的管道复制代码到虚拟机上并在其上自动化镜像的构建流程，这大大改善了开发者的体验。

![基于`git-push`的持续部署管道](https://cdn-images-1.medium.com/max/800/1*C7MFyQW9bYic7NYVgqnnVg.png)

### 4）编辑配置文件

**问题：**  每次部署或运行环境的改变都需要解决以下配置问题：

- 配置参数和conf文件
- 标识已部署服务地址的IP/端口
- 密码和密钥

解决这些问题的复杂性在集群中变得更糟糕了

**解决方案：** 大部分配置应该是作为容器一部分的。一个能通过内部DNS处理服务发现并提供集群范围配置管理系统的容器编排工具可以将参数注入其中

![管理配置和服务发现的基于Kubernetes的解决方案](https://cdn-images-1.medium.com/max/800/1*DbBsQkoZeXs9H8HD4eCLVw.png)

### 5）在supervisor监控下运行代码

**问题：**  解决以下问题需要supervisor

- 日志，日志的循环使用
- 初始化时重启服务/容器
- 崩溃情况下重启服务/容器

**解决方案：** 到目前为止，管理日志可能需要定制工具，但容器编排工具可以解决后面两个问题。大多数编排者提供管理底层资源状态的原语。

### 6）更新前端服务器/API网关

**问题：**  API网关在每次系统发生变化时都需要重新配置和部署。比如说每次更新都会导致上游IP/端口的变化的部署

**解决方案：** 一个响应式的API网关需要一个集群操作者代理来监控变化，然后重新配置和部署网关。操作者必须依赖支持监控变化的分布式数据库，并在容器新部署时进行更新

![基于分布式key-vaule存储、拥有操作者代理的响应式网关](https://cdn-images-1.medium.com/max/600/1*bfMlVBgq8QeayAONwpllNQ.png)

----------

集成上述解决方案，一个完整的PaaS看起来可能是这样的：

![enter image description here](https://cdn-images-1.medium.com/max/1000/1*ZC1Wrt3k5DveHmxtsmkn_g.png)

话虽如此，PaaS还需要解决一些其他问题，比如拥有标准化的部署方法，确定底层组件的透明度级别（*如果走对开发者友好的路线，还要提供一个安全的方法访问他们*），管理有状态的服务等等。

下一篇文章我们来看一看一个使用Docker和Kubernetes 作为容器和容器编排技术选择实现的示例，并解决上面所列出的问题。

Hasura是一个基于Kubernetes的PaaS和PostgreSQL驱动的BaaS，从这里开始吧：

- [https://hasura.io](https://hasura.io)


----------


> Written with [StackEdit](https://stackedit.io/).
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEzMjE2MzAwNTRdfQ==
-->