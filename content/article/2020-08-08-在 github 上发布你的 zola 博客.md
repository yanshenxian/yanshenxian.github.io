+++
title = "在 github上发布你的 zola 博客"
slug = "build-and-deploy-zola-on-github-pages"
date = 2020-08-07

[taxonomies]
categories = ["2020"]
tags = ["原创", "Zola", "wsl"]

[extra]
original_statement = true

+++

本地开发环境 `wsl2`，选取的博客框架 `Zola`，托管服务 `github pages`，自动构建 `github actions`，另外自定义域名还需要一个 DNS 服务商，这里我用的是 `cloudflare`，顺便白嫖 CDN。

`Rust` 是个很酷的语言。
<!-- more -->

## 配置 Wsl2 开发环境

因为刚从 Mac OS 切到 Win10 开发环境，也是第一次尝试 `wsl2`。虽然这没有 Mac OS 或者 Linux 好用，但是也极大的增强了 win10 开发的体验。

1. 首先根据 [官网文档](https://docs.microsoft.com/zh-cn/windows/wsl/install-win10) 开启 `wsl2`

2. 运行 `wsl` 命令启动环境，这个时候你就可以看到一个简洁的 linux 终端了，**wsl2 的文件和端口是和宿主机共享的。** 比如你 c 盘上的文件会映射到 `/mnt/c/` 这个路径，你在 wsl 里面启动了一个 web 服务监听端口 8080，你就可以在宿主机上直接使用 127.0.0.1:8080 访问它。

3. 然后像在 Linux 上一样配置开发环境，修改国内镜像、安装 git、zsh、oh-my-zsh等等操作，你可能还会在宿主机上安装 VS Code + `Remote - WSL` 插件，这个用来管理 wsl 里的项目很方便，它的终端默认就是 wsl 的终端。另外也可以换个漂亮点的终端替换掉 cmd 或者 powershell，`windows Terminal` 就是个不错的选择。

4. 花了一堆时间终于搞得有模有样了，你可能还会碰到一些问题，它终究不是一个完整的 linux 。。。比如 `ping` 命令无法使用，搜索发现需要 `sudo chmod u+s` 下，再比如域名解析会莫名其妙的出现问题，解决方法是修改 `resolv.conf` 文件 (必须先在 `wsl.conf` 里设置 generateResolvConf 参数否则重启无效)
```zsh
λ cat /etc/wsl.conf
[network]
generateResolvConf = false

λ cat /etc/resolv.conf
nameserver 8.8.8.8
```

## Wsl2 中安装 Zola

因为在安装的过程中，我遇到了几个坑，所以这里单独提出来说下。

首先看下 [Zola 文档](https://www.getzola.org/documentation/getting-started/installation/)，官网没有提供 Ubuntu/Debian 系的源，但是提供了一个 [Snapcraft](https://snapcraft.io/install/zola/ubuntu) 的方式安装。

在 `wsl2` 里用 snap 就会出现一些奇怪的问题，你 `sudo apt install snapd` 之后并没法直接使用 snap，当你尝试 `sudo snap install zola --edge` 你会得到一个错误信息
>  error: cannot communicate with server: Post http://localhost/v2/snaps/zola: dial unix /run/snapd.socket: connect: no such file or directory...    

因为 wsl 终究不是一个完整的 linux 系统，所有有些命令比如 systemctl 没法使用。错误解决可以查看这个 [issue](https://github.com/microsoft/WSL/issues/5126#issuecomment-653715201)

好了 就算你尝试用上面 issue 的方法暂时解决了 snap 安装的命令，当你接下来准备在本地构建运行的时候又会出现 `No port available` 的错误。Snap 安装的 Zola 不知怎的没法监听端口。

所以我放弃了 Snap 的方式，但是我又不想源码编译安装。幸好 Zola 的 github 里提供了 linux 版本的 Release 包。就直接拿过来用吧。。

## 本地构建 Zola 项目

为了方便开发，我们把项目放在 wsl 子系统里面，之后在宿主机上用 VSCode 进行开发。

1. 初始化 Zola 项目
```zsh
λ zola init my-blog
Welcome to Zola!
Please answer a few questions to get started quickly.
Any choices made can be changed by modifying the `config.toml` file later.
> What is the URL of your site? (https://example.com): https://blog.yanshenxian.xyz
> Do you want to enable Sass compilation? [Y/n]: Y
> Do you want to enable syntax highlighting? [y/N]: Y
> Do you want to build a search index of the content? [y/N]: N

Done! Your site was created in "/home/user/my-blog"

Get started by moving into the directory and using the built-in server: `zola serve`
Visit https://www.getzola.org for the full documentation.

λ cd my-blog && ls
config.toml  content  sass  static  templates  themes
```
基础的架子已经生成好了，为了了解 `config.toml` 配置文件，你需要阅读下 [Zola 文档](https://www.getzola.org/documentation/getting-started/configuration/)，en 可以不求甚解...

2. 选择一个你喜欢的主题并配置它，Zola 提供的 [主题列表](https://www.getzola.org/themes/) 有限，但是对我来说够用了。。我选的是 [even](https://www.getzola.org/themes/even/)，基于我自己的喜好，我小小的改动了一些

- 参考 [simple-dev-blog](https://www.getzola.org/themes/simple-dev-blog/) 把头部显示的站点名称改成了头像
- 参考 [anpu](https://www.getzola.org/themes/anpu/) 加了个 footer
- 修改了默认的锚点符号 见 `templates/anchor-link.html`
- favicon.ico 和 sideout.min.js 脚本本地化，见 `static/` 目录，主题里面用的 cdn 访问速度太慢了
- 修改文章路径以 `article` (sub section) 子目录开头，还可以根据年/月来创建子文件夹，但是子目录的 `_index.md` 必须包含 `transparent = true` 的配置
- 集成 [gitalk](https://github.com/gitalk/gitalk) 评论，配置在 `config.extra.enable_comment`，并且文章单独可以配置是否开启评论 `page.extra.enable_comment`
- 所有外链新标签打开 `config.extra.enable_target_blank`
- ...

你可以从我的 github 博客仓库 拷贝我的配置和修改后的主题，`even` 主题在 `themes/even/content` 里自带了一些示例文章，可以参考下文章编写规则（简单说就是一些 metadata 头部 + markdown 内容）

3. 本地运行下看看效果
```zsh
λ zola serve
Building site...
-> Creating 3 pages (1 orphan), 0 sections, and processing 0 images
Done in 54ms.

Listening for changes in /home/user/my-blog/{content, config.toml, static, templates, themes, sass}
Press Ctrl+C to stop

Web server is available at http://127.0.0.1:1025
```
为了避免没有文章的尴尬，可以先 `cp -r themes/even/content content/` 构建部署下主题自带的测试文章

然后用宿主机访问 http://127.0.0.1:1025 就能看到搭建好的博客网站了

## 创建 && 发布

你可能需要先看下相关的文档
- [Github Page 文档](https://docs.github.com/cn/github/working-with-github-pages/about-github-pages)
- [Zola Deploy 文档](https://github.com/shalzz/zola-deploy-action/)

1. 创建一个命名为 `<user>.github.io` 的仓库，例如 [yanshenxian.github.io](https://github.com/yanshenxian/yanshenxian.github.io)  

2. 创建 `github actions` 脚本，具体见 [ci.yml](https://github.com/yanshenxian/yanshenxian.github.io/blob/master/.github/workflows/ci.yml)

3. 申请一个 `access token`，申请方法见上面的 `Zola Deploy 文档`，并在仓库的 `Settings->Secrets` 里设置环境变量，变量名称和 ci 脚本中的变量名称保持一致 `${{ secrets.TOKEN }}` 

4. 创建 `static/CNAME` 文件，里面写上你要自定义的域名，具体见 [CNAME](https://github.com/yanshenxian/yanshenxian.github.io/blob/master/static/CNAME)，ci 构建的时候会自动把 `static` 里的文件拷贝到 `gh-pages` 分支的根目录，如果没有 `CNAME` 这个文件，每次脚本构建之后会清除之前设置的域名重定向。

5. 将本地项目 push 到仓库的 `master` 分支，如果没问题，可以在 `commit id` 旁边看见构建的标志，脚本会自动构建出一个 `gh-pages` 分支，里面生成了所有的静态文件

6. 在仓库的 `Settings->Github pages` 确认相关配置，**page build 的 source branch 是 gh-pages**，如下图所示
![github-pages-settings](/image/2020/08/github-pages-settings.bmp)

7. 在 [Cloudflare](https://dash.cloudflare.com/) 中设置 DNS CNAME 以及设置 CF 的 CDN （比起 Github 经常被墙... cf 好点），如下图所示
![github-pages-cname-setting](/image/2020/08/github-pages-cname-setting.bmp)

**如果 github pages settings 中设置了强制 ssl，那么 cf 中 ssl 也必须设置为 Full，要不然会造成无限重定向。**

8. 现在你可以用你的自定义域名访问你的博客了，Enjoy~

由于我是搭建完之后再写这篇文章的，有些步骤可能没那么明确了，比如我因为有一些踩过的坑，所以调整了些步骤，有些组件也不是必要的，比如 `wsl2`、`cloudflare`

但是总体操作应该可以顺下来。多看官方文档也会有很大的帮助。