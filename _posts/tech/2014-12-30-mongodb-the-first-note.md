---
layout:       post
title:        MongoDB 记录一：初识MongoDB
category:     tech
description:  MongoDB 记录一：初识MongoDB
---
一直说把MongoDB这个记录一下，最近一段一直没时间（其实是自己懒啊）
MongoDB 详见 [MongoDB](http://zh.wikipedia.org/wiki/MongoDB) 维基百科，想了解更多，可去[MongoDB 官方网站](https://www.mongodb.org/)细看

要用MongoDB，首先你得去[下载](https://www.mongodb.org/downloads)它，各种系统都有对应下载链接，这里只拿我大Windows做测试，记得一定要下载64位版本的，官方有指示：32位的MongoDB只能支持数据库小于2G。一般喜欢直接下载ZIP压缩文件

下载后你可以参照[MongoDB安装指南](http://docs.mongodb.org/manual/tutorial/install-mongodb-on-windows/)一步步去操作，推荐下载[最新版的MongoDB手册.PDF](http://docs.mongodb.org/master/MongoDB-manual.pdf)，里面有提到你可以[为MongoDB配置成Windows服务](http://docs.mongodb.org/manual/tutorial/install-mongodb-on-windows/#configure-a-windows-service-for-mongodb)

>**这里就以MongoDB文件夹放在C盘根目录下为例，温馨提示：以下命令请以管理员身份运行CMD**

>* 为MongoDB日志文件创建一个特定的目录<br>
>`md c:\mongodb\logs`
>* 为logpath创建一个配置文件<br>
>`echo logpath=c:\mongodb\log\mongo.log > c:\mongodb\mongod.cfg`
>* 顺手把dbpath也加到配置文件里<br>
>`echo dbpath=c:\mongodb\log\mongo.log > c:\mongodb\data`
>* 运行安装<br>
>`c:\mongodb\bin\mongod.exe --config c:\mongodb\mongod.cfg --install`
>* 完了设置服务为按需求自动，默认自动启动<br>
>`sc config MongoDB start= demand`
>* 启动MongoDB服务<br>
>`net start MongoDB`
>* 关闭MongoDB服务<br>
>`net stop MongoDB`
>* 删除MongoDB服务<br>
>`c:\mongodb\bin\mongod.exe --remove`

你也可以[用SC命令手动去为MongoDB创建一个Windows服务](http://docs.mongodb.org/manual/tutorial/install-mongodb-on-windows/#manually-create-a-windows-service-for-mongodb)，可输入 SC ? 查看帮助

服务做好后你就可以用mongo shell 去连接MongoDB愉快地玩耍了，如果这时候恰好你还没有把MongoDB设置为环境变量那是很让你不舒服的，因为每次你要在CMD里输入 `c:\mongodb\bin\mongo.exe` 才可以连接，此刻你可以在Path里面添加 `c:\mongodb\bin;` 这样你就可以在cmd里面直接输入mongo就进入mongo shell，这酸爽…

当然也推荐大家使用图形管理工具去管理MongoDB，我使用的是[MongoVUE](http://www.mongovue.com/)

>**MongoDB 常用命令 懒得搜**

Commands | Descr              
-------- | -----
`show dbs` | 显示数据库名称
`show collections`| 显示当前数据库包含项
`show users`|  显示当前数据库的用户
`show profile`|  显示最近的系统概述（1ms之内）
`show logs`|  显示可以进入的logger名称|
`show  log [name]`|  输出内存中日志最后一段，默认"global"
`use (db_name)`|  使用当前数据库
`db.foo.find()`|  集合foo中的列表对象
`db.foo.find({a:1}) foo where a = 1`|  列表对象
`it`|  最后一行评估的结果，用于之后迭代
`db.help()`|  数据命令的帮助
`db.mycoll.help()`|   集合方法上的帮助
`db.mycoll.drop()`|  删除集合
`DBQuery.shellBatchSize = x`|  设置要显示的数量
`exit`|  退出
