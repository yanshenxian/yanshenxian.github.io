+++
title = "为 gitalk 代理 github api 请求"
slug = "gitalk-proxy-github-api-request"

[taxonomies]
categories = ["2020"]
tags = ["原创", "Zola"]

[extra]
original_statement = true

deprecated = true
+++

<deprecated-exclude>

由于 `gitalk` 使用 `oauth app` 来实现评论功能，其权限粒度过大，**授权 `Token` 具备用户所有的公共仓库的读写权限**，如果不小心泄露或者有人作恶，可能造成严重危害。

我看网上有人说搭建博客使用一个单独的账号来创建评论 `issues` 仓库以避免泄露危害，这种说法是错误的，`Token` 泄露带来的危害是对评论用户来讲的，并不是博客主。

要解决这个问题，暂时有两个方法
1. 评论用户授权一个对自己无危害的账号来评论，另外可以在 [settings/applications](https://github.com/settings/applications) 里面取消自己大号原来的授权。

2. 让网站废弃 `gitalk` 评论系统，可以用 [utterances](https://github.com/utterance/utterances) 替代，这是一个 `github app`，可以做到细粒度权限控制，只需要授权托管评论的仓库的 `issues` 相关权限即可。

</deprecated-exclude>
<!-- more -->
<deprecated-exclude>

一个博主完全可以通过恶意脚本或者本文所说的反代功能来收集评论用户的 `Token`。关于安全讨论的参考链接如下：

- [建议大家弃用 Gitalk 和 Gitment 等权限过高的 Github OAuth App](https://www.v2ex.com/t/535608)  
- [differences-between-github-apps-and-oauth-apps](https://docs.github.com/en/developers/apps/differences-between-github-apps-and-oauth-apps)

</deprecated-exclude>

---

博客当前使用了 [gitalk](https://github.com/gitalk/gitalk) 来评论，但是由于国内网络问题，访问 `api.github.com` 经常被墙，所以考虑加一层代理小优化一下。

gitalk 原生并不支持 api 代理，google 搜索也没有发现现成的方案，只能自己瞎鼓捣了。

## 修改 gitalk 源码 

改动涉及 `src/gitalk.jsx`、`src/util.js`、`typings/index.d.ts` 这三个文件，已提交 [相关 PR](https://github.com/gitalk/gitalk/pull/401)


使用 `npm run build` 构建 js 文件，构建结果位于 `dist/gitalk.min.js`

最后配置你自己的代理地址就行了
```js
var gitalk = new Gitalk({
    // 其他配置省略
    ...
    proxyGithubApi: "https://proxy-github-api.{{USERNAME}}.workers.dev",
    ...
})
```

## 搭建代理服务器

如果是在自己的服务器上用搭建反代服务器，有一些地方需要注意
1. `header` 中的 `authorization` 必须传递下去

2. `gitalk` 评论请求用到了原始响应里的接口地址，必须进行反代域名替换

本着白嫖的心思，直接用 [cloudflare worker](https://workers.cloudflare.com/)，每天支持10万次免费请求。

完整代码如下所示

{{ gist(gistId="yanshenxian/c720ecdafe160653f80a6fbd116916c5") }}

上面的脚本参考了 [在特殊地区科学使用 Disqus 评论系统](https://blog.ichr.me/post/use-disqus-conveniently/)
