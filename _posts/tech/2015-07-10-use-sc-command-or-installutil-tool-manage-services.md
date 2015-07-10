---
layout:       post
title:        使用SC命令或InstallUtil工具管理服务
category:     tech
description:  使用SC命令或InstallUtil工具管理服务
---
此篇只为做个记录，方便以后使用。其实平常也比较少用到这些命令，只有你需要将程序安装为 Windows 服务时才会需要。
#### 1. InstallUtil Tool
很早之前做WIndows 服务程序时，习惯性用 InstallUtil 工具去创建，非常方便。在服务设计里面设置好服务运行程序的Account，服务安装程序的StartType，Description，ServiceName就OK了。剩下就只需要找到你安装的Visual Studio Tools文件夹里面 开发人员命令提示，运行并执行
`InstallUtil <yourproject>[^CLSK].exe`

如果你要卸载服务，类似地 重复以上步骤，只需把命令改为
`InstallUtil /u <yourproject>.exe`

这里需要提示一下 如果指定了 /u or /uninstall，它就会卸载服务，否则就安装。
>另外，有时，服务的可执行文件被删除后，该服务可能仍然会出现在注册表中。 这种情况下，请使用命令 [sc delete][1] 从注册表中删除服务的条目。

以上若有任何不清晰可查看[如何：安装和卸载服务][2]
#### 2. SC Command
[SC][3] 是与服务控制器和已安装的服务进行通信的命令行程序。SC 的命令就比较齐全了，我常用的包含`sc create/delete/config/description/query/start/stop/pause/failure ServiceName`

>**简单举例**

>* 创建服务<br>
>`sc create MyService binPath= <yourproject>.exe DisplayName= MyService start= auto`
>* 增加描述<br>
>`sc description MyService "MyService Description"`
>* 启动服务<br>
>`sc start MyService`
>* 删除服务<br>
>`sc delete MyService`
>* 查询服务<br>
>`sc query MyService`

#### Footnotes
[^CLSK]:[命令行语法键][4]

[1]:https://technet.microsoft.com/zh-cn/library/cc742045.aspx
[2]:https://msdn.microsoft.com/zh-cn/library/sd8zc8ha.aspx
[3]:https://technet.microsoft.com/zh-cn/library/cc754599.aspx
[4]:https://technet.microsoft.com/zh-cn/library/cc771080.aspx
