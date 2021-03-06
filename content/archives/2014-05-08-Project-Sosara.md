+++
categories = ["dev"]
date = "2014-05-08T16:41:24+08:00"
draft = false
slug = "Project-Sosara"
title = "Project Sosara"

+++

#### **最初的想法**
---
 一直想写一个备份系统用于备份服务器上的数据，之前刚来公司的时候写过一个简单的系统，由于时间紧迫，也没有把所有的精力放在这上面，所以第一版的系统比较简单，用shell配合cron实现备份计划任务，所有的备份文件写在shell里面，通过http协议调接口上报服务器信息以及文件备份状态。通过web界面展示服务器状态和文件备份状态。算是一个备份信息汇总，web端只能查看信息。但是这个系统不能满足我们的需求，连我自己这关都过不了。

#### **我所希望的备份系统是这样的：**
---
- 备份任务尽可能与网络稳定性无关，虽然这不可能
- 统一的配置平台，而不是跑到每台机器上去配置要备份的文件以及相关参数
- 备份状态具体到每个文件，每个文件的备份状态都可以查询
- 强大的分组功能，备份配置可以扩大到组，每个组共享一个备份配置，不用配置组内的每台机器了
- 合理的权限配置，接入公司域账号登陆
- 完善的服务器心跳检测以及可靠的备份报警机制
- client 和 server端的高可用性，即使server端停止服务，仍然不影响已配置的数据备份任务
- server端支持分布式部署，server端和client端可靠的断线重连机制
- 启用组备份时，在一定时间范围内的随机时间备份，防止IDC带宽以及服务器接口阻塞
- 对于系统级别的重要文件进行预配置，无需再手动配置
- 提供备份限速设置
- 简洁美观的UI，良好的用户体验 (自适应)
- 简单、方便的client部署方式
- 支持目录备份，支持*等匹配规则批量备份文件


#### **过程总是美好的：**
---
` 2014年2月17日 `，开始着手设计并编写整个系统，技术选型如下：


	* agent端：  采用Golang语言，与server端保持长连接（依赖：Rsync、wget、gcc >? ——）
	* server端： 采用Golang语言，下发配置信息，收集上报的服务器信息
	* web端：    采用PHP语言，Bootstrap前端框架，给管理人员提供信息展示和配置界面
	* 数据库：    采用MongoDB，用于存储配置信息和备份信息


` 2014年3月09日 ` ，经历了20天，系统的雏形终于出来了。除了报警以及一些简单的展示页面没有做，其他的已经成型，接下来就是测试阶段，解决一些bug。

` 2014年3月14日 ` ，系统日趋完善，决定部署在生产线上。




#### **截图：**
---
架构图：
![](/images/2014/1399538840.png)
服务器备份设置：
![](/images/2014/1399538432.png)
文件备份状态：
![](/images/2014/1399538450.png)
备份错误文件列表：
![](/images/2014/1399538463.png)

#### **开源：**

我是一个开源爱好者，虽然这个系统目前没有开源，但是相信之后不久的将来会开源的，只待你长发及腰！