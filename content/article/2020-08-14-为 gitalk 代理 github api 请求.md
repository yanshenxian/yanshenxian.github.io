+++
title = "为 gitalk 代理 github api 请求"
slug = "gitalk-proxy-github-api-request"
date = 2020-08-14

[taxonomies]
categories = ["2020"]
tags = ["原创", "Zola"]

[extra]
original_statement = true
+++

博客当前使用了 [gitalk](https://github.com/gitalk/gitalk) 来评论，但是由于国内网络问题，访问 `api.github.com` 经常被墙，所以考虑加一层代理小优化一下。
<!-- more -->

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

完整代码如下

{{ gist(url="https://gist.github.com/yanshenxian/c720ecdafe160653f80a6fbd116916c5") }}

上面的脚本参考了 [在特殊地区科学使用 Disqus 评论系统](https://blog.ichr.me/post/use-disqus-conveniently/)
