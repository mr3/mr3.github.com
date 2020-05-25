---
layout:       post
title:        使用 OutputCache 时设置 Cookie 疑惑小结
category:     tech
description:  使用 OutputCache 时设置 Cookie 疑惑小结
---

工作中用到了输出缓存 [OutputCache][1]，项目 发现有一个 Action 加了 MVC 的 [OutputCache] 特性，输出缓存总是无法生效，
而其他 Action 则是正常的。排查下来发现是这个 Action 有操作 Repsonse.Cookie。

疑问为什么 Cookie 会对 OutputCache 造成影响呢？ 在 [stackoverflow ][2] 上看到了一个答案

> You try to cache this on server side, and at the same time you try to set the cookie on the client - this is not working together.

觉得这个解释很合理，你一方面尝试在服务器缓存，同时又在客户端设置Cookie，两者不可同时起作用。

好奇 OutputCache 在设置了 Cookie 时是怎么处理的？ Google 后在 blogspot 上看到[一篇文章][3]提到了这一点，如下图

![OutputCache OnLeave](/img/posts/tech/OutputCache_OnLeave.png)

在 OutputCacheModule 的 OnLeave 事件中 有对 Response.Cookies 数量的判断，
判断集合有数据的话 就不对 Cache 进行操作了，为了进一步验证，
又去扒了扒 [OutputCacheModule][4] 的源码，在 1109 行 看到下面的代码

```csharp
// MSRC 11855 (DevDiv 297240 / 362405) - We should suppress output caching for responses which contain non-shareable cookies.
// We already disable the HTTP.SYS and IIS user mode cache when *any* response cookie is present (see IIS7WorkerRequest.SendUnknownResponseHeader)
        if (response.ContainsNonShareableCookies()) {
#if DBG
            reason = "Non-shareable response cookies were present.";
#endif
            break;
        }
```

从代码和注释可以看出，抑制包含 non-shareable  cookies 的输出缓存，
在任何响应中存在 Cookie 就会禁止 HTTP.SYS 和 IIS 用户模式缓存。


接着看 [ContainsNonShareableCookies][5] 这个方法的具体实现

```csharp
// returns TRUE iff there is at least one response cookie marked Shareable = false
internal bool ContainsNonShareableCookies() {
    if (_cookies != null) {
        for (int i = 0; i < _cookies.Count; i++) {
            if (!_cookies[i].Shareable) {
                return true;
            }
        }
    }
    return false;
}
```

同样从代码和注释可以看出，只要响应包含 Cookie，且有一个 Cookie 的 Shareable 属性为 false，就返回 true。


结合以上两段代码可以得出结论：只要响应中有一个 cookie 的 Shareable 属性为 false，输出缓存就会失效。

这个 cookie 的 [Shareable][6] 属性在 .NET 4.0 没有找到设置的地方， 后来去 MSDN 看版本信息说是 .NET 4.5后可用，
并且 MSDN 对它的解释为确定 cookie 是否允许参与输出缓存，后面去实测确实如此。

### 总结

* .NET 4.0 如果你在使用 OutputCache 时，响应中包含 Cookie，则导致 OutputCache 无效
* .NET 4.5 如果你在使用 OutputCache 时，响应中包含 Cookie，且所有 Cookie 的 Shareable 属性 都设为 true，则 OutputCache 有效，反之无效。

[1]:https://msdn.microsoft.com/zh-cn/library/system.web.mvc.outputcacheattribute(v=vs.100).aspx
[2]:http://stackoverflow.com/questions/9411180/asp-net-outputcache-and-cookies?answertab=votes#tab-top
[3]:http://androidyou.blogspot.com/2012/10/aspnet-output-cache-not-working-and.html
[4]:http://referencesource.microsoft.com/#System.Web/OutputCacheModule.cs,1109
[5]:http://referencesource.microsoft.com/#System.Web/HttpResponse.cs,890
[6]:https://msdn.microsoft.com/zh-cn/library/system.web.httpcookie.shareable(v=vs.110).aspx
