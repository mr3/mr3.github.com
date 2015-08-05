---
layout:       post
title:        unable-to-edid-code-in-visual-studio-2013
category:     tech
description:  VisualStudio 2013 不能编辑代码
---
今天遇到了个小事儿，VisualStudio 2013 无法编辑编码，之前也遇到过类似的情况，使用经典做法：关闭 VS 重新打开之后就好了。可是，可是今天反复使用之前有效的方法，却没有生效，这就很让人讨厌了。

于是 Google 之，Keywords: visual studio 2013 cannot edit code, 果然在<a href="http://stackoverflow.com/" title="Stack Overflow is a question and answer site for professional and enthusiast programmers. It's 100% free. " target="_blank" >StackOverflow</a>看到了其他程序猿也提出了相同的疑问。详情请猛戳：<a href="http://stackoverflow.com/questions/25178283/cannot-edit-checked-out-file-tfs-in-visual-studio-2013" target="_blank" >Cannot edit checked out file (TFS) in Visual Studio 2013</a>

原因：使用了<a href="https://www.jetbrains.com/resharper/" target="_blank" >Resharper</a> 这个强大的工具，在解决方案包含过多项目，且开发机器又不太给力时，Resharper生成的解决方案缓存过大导致此情况的发生。看到这里，看了下解决方案，52个项目，想应该确实是这个问题了。

解决方法：清理Resharper生成的解决方案缓存。使用 Win+E 或者 Win+R 命令，拷贝以下路径，打开文件夹，彻底删除文件即可。

` %userprofile%\AppData\Local\JetBrains\ReSharper\v8.2\SolutionCaches`

PS: 
* 这种小问题发生了，不记录一下，下次再出现了，有印象但没记住解决方案还要去查真是有够麻烦。
* 程序猿查问题还是应该使用 Google，最不及使用 Bing 也可以，Baidu 就不要了。原因很明显，搜索上面提到的 Keywords 即可。

如有错误，欢迎指正。
