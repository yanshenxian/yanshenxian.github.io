+++
title = "为 gitalk 代理 github api 请求"
slug = "gitalk-proxy-github-api-request"
date = 2020-08-14

[taxonomies]
categories = ["2020"]
tags = ["原创", "发现"]

[extra]
original_statement = true
+++

博客当前使用了 [gitalk](https://github.com/gitalk/gitalk) 来评论，但是由于国内网络问题，访问 `api.github.com` 经常被墙，所以考虑加一层代理小优化一下。
<!-- more -->

gitalk 原生并不支持 api 代理，google 搜索也没有发现现成的方案，只能自己瞎鼓捣了。

## 修改 gitalk 源码 

改动涉及 `src/gitalk.jsx`、`src/util.js`、`typings/index.d.ts` 这三个文件，对应地址: [commit](https://github.com/yanshenxian/gitalk/commit/16361aaf7b410394203073de3e0d837a29881662)


使用 `npm run build` 构建 js 文件，构建结果位于 `dist/gitalk.min.js`

最后配置你自己的代理地址就行了
```js
var gitalk = new Gitalk({
    // 其他配置省略
    ...
    proxyGithubApi: "https://proxy-github-api.username.workers.dev",
    ...
})
```

## 搭建代理服务器

本着白嫖的心思，想直接用 [cloudflare worker](https://workers.cloudflare.com/)，每天支持10万次免费请求。完整代码如下。

如果是在自己的服务器上用搭建反代服务器，有一些地方需要注意
1. `header` 中的 `authorization` 必须传递下去

2. `gitalk` 评论请求用到了原始响应里的接口地址，必须进行反代域名替换

```js
addEventListener('fetch', event => {
    event.respondWith(proxy(event));
});

// 这里写你的反代域名
const your_proxy_host_name = "proxy-github-api.username.workers.dev";
const proxy_host_name = "api.github.com";

async function proxy(event) {
    const getReqHeader = (key) => event.request.headers.get(key);

    let url = new URL(event.request.url);
    url.hostname = proxy_host_name;

    let parameter = {
        headers: {
            'Host': proxy_host_name,
            'User-Agent': getReqHeader("User-Agent"),
            'Accept': getReqHeader("Accept"),
            "authorization": getReqHeader("authorization"),
            'Accept-Language': getReqHeader("Accept-Language"),
            'Accept-Encoding': getReqHeader("Accept-Encoding")
        }
    };

    if (event.request.headers.has("Referer")) {
        parameter.headers.Referer = getReqHeader("Referer");
    }

    if (event.request.headers.has("Origin")) {
        parameter.headers.Origin = getReqHeader("Origin");
    }

    let response = await fetch(new Request(url, event.request), parameter);
    if (response.status === 200) {
        const { headers } = response;
        const contentType = headers.get('content-type') || '';
        if (contentType.includes('application/json')) {
            // 替换原域名
            const text = await response.text();
            const body = text.replaceAll("://" + proxy_host_name, "://" + your_proxy_host_name);
            return new Response(body, response);
        } else {
            return response;
        }
    }

    return response;
}
```
上面的脚本参考了 [在特殊地区科学使用 Disqus 评论系统](https://blog.ichr.me/post/use-disqus-conveniently/)
