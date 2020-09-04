---
layout: doc
title: 配置中心
---

## 配置中心

大鹏配置中心主要分为三大功能模块：配置管理、发布管理和服务监控。本节主要对配置管理功能模块的使用进行详细的说明。而配置管理模块又分为集群管理、服务配置、白名单和ApiKey四个具体的功能。登录大鹏配置中心之后，点击相应的功能标签，即可进入到相应的功能配置界面。

- 集群管理，如下图所示：

![clusters](https://github.com/dapeng-soa/documents/blob/master/images/depeng-config/config-clusters.png?raw=true)

&ensp;&ensp;在该页面中主要展示的是Zookeeper集群相关信息（集群地址、描述、influxdb地址以及添加和修改时间）。表格中显示的字段以及显示的视图可以根据右上角的相应按钮进行定制。
<br>
&ensp;&ensp;集群管理主要对Zookeeper集群进行增删改查。修改和删除功能只需要点击对应集群中操作一栏的相应按钮即可进行相关操作。查找也仅需在搜索框输入集群名称搜索即可。Zookeeper集群的增加需要点击右上角的新增按钮，会弹出如下页面，正确填写相关信息，点击保存即可。

![addClusters](https://github.com/dapeng-soa/documents/blob/master/images/depeng-config/config-addClusters.png?raw=true)

- 服务配置，配置界面如下图所示：

![serviceConfig](https://github.com/dapeng-soa/documents/blob/master/images/depeng-config/config-service.png?raw=true)

&ensp;&ensp;该页面列举了所有服务的服务名及其状态、添加和修改时间以及可以对服务进行的相关操作。操作主要包括发布、修改、查看发布历史、实时配置、查看配置详情以及对服务配置的删除。增加配置需要点击右上角的新增按钮。
<br>
&ensp;&ensp;在状态栏中显示的是当前服务的配置信息是否已经发布，当点击发布按钮并选择了需要发布的Zookeeper集群后状态变为`已发布`，同时发布按钮隐藏。若修改了服务的配置信息但是没有发布，则状态变为`已审>待发布`。
<br>
&ensp;&ensp;点击实时配置，会弹出如下图所示界面，需要选择Zookeeper集群与之通信，获取Zookeeper上相应服务的配置信息。

![chooseClusters](https://github.com/dapeng-soa/documents/blob/master/images/depeng-config/config-chooseClusters.png?raw=true)

&ensp;&ensp;点击发布历史显示的是如下图所示的服务配置发布历史时间节点：

![serviceHistory](https://github.com/dapeng-soa/documents/blob/master/images/depeng-config/config-serviceHistory.png?raw=true)

&ensp;&ensp;对于服务配置，我们主要关心的是对于服务可以配置哪些信息，如何配置这些信息？点击新增按钮则会弹出如下图所示的界面，就可以进行服务信息的配置。

![addServiceConfig](https://github.com/dapeng-soa/documents/blob/master/images/depeng-config/config-addServiceConfig.png?raw=true)

&ensp;&ensp;从图中可以看出，对于服务的配置需要填写的是服务的全限定名称。配置中心可以对服务配置的信息包括：超时配置、负载均衡、路由配置、限流配置等。对于这些配置信息，dapeng都有一定的配置规则，为了保证用户正确的进行配置，我们还提供了参考的配置例子，只需要点击不同配置下面的问号按钮，即可显示出来。如果想详细的了解不同功能的配置规则，可以查看相应章节进行了解。

- 白名单，其配置界面如下图所示：

![whitelist](https://github.com/dapeng-soa/documents/blob/master/images/depeng-config/config-whitelist.png?raw=true)

&ensp;&ensp;对于白名单的配置十分简单。首先我们只需要选定需要进行配置白名单的Zookeeper集群，界面上会显示在集群已经存在的白名单列表。如果我们想要删除白名单，只需要双击相应的服务名即可。如果想要增加白名单，只需要早右侧的编辑框内填写服务的全限定名，有多个服务则则填写多个服务名，点击添加或者回车即可添加成功。

- ApiKey，其配置界面如下图所说：

![ApiKey](https://github.com/dapeng-soa/documents/blob/master/images/depeng-config/config-apikey.png?raw=true)

&ensp;&ensp;该界面展示需要管理的所有的网关ApiKey，及其密码、超时时间、是否验证超时、状态、所属业务以及IP规则等详细信息。对每个ApiKey可以执行的操作包括修改、下发、禁用以及新增等，每个操作只需要点击相应的按钮即可，下图所示为增加操作的界面：

![addApiKey](https://github.com/dapeng-soa/documents/blob/master/images/depeng-config/config-addApiKey.png?raw=true)

如图所示，需要填写的字段与在网关ApiKey管理界面相同，对于密码、超时时间以及ip规则等字段，我们还给出了示例和说明，点击下方问号按钮即可显示。在正确填写了字段后，点击保存就可以正确添加相应的ApiKey，并对其进行管理。