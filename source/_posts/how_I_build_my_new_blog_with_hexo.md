---
title: 如何用Hexo新建一个blog
date: 2016-12-29 10:41:43
tags: hexo
---
一年多前，我看到一个很好看的 `Jekyll` 主题[leonids](https://github.com/renyuanz/leonids)（[leonids预览](https://renyuanz.github.io/leonids/)）。于是用它来建了第一个Github-Pages的博客。虽然他视觉元素很棒，但那时候的我还hold不住Jekyll。没实现`只要写好MD格式的post并提交，其他都不用管`的目标，所以写了一篇博文之后就晾着了。
16年底我又回来了，缘起于一个清新的 `Hexo` 主题[Next](https://github.com/iissnan/hexo-theme-next)（[Next预览](http://notes.iissnan.com/)）。这次实现了需要的目标嘿嘿。
> - 用 Hexo 建站，使用你要的主题
- 启用 Github-Pages ，并使用 Travis-CI 自动更新

<!-- more -->

## Hexo建站

安装 Hexo 并建站，下面列出概要，具体参考[Hexo中文文档](https://hexo.io/zh-cn/docs/setup.html)
```bash
# 安装 hexo
npm install hexo-cli -g
# 开始建站
hexo init my_blog
# 新建一个 post
hexo new "my_post"
# 清除缓存
hexo clean
# 打开 hexo 服务器
hexo s
```

## 使用主题

Hexo 的主题，是一种可插拔的设计，主题文件在 `/themes/` 目录下，可以在 `_config.yml` 文件中配置
这里使用 Next 主题，安装使用参考 [Next主题中文文档](http://theme-next.iissnan.com/getting-started.html)，
这里有两个可能产生问题的点。

### 下载主题

这一步文档中推荐使用 `git clone`，劝大家不要这样做。因为你clone下来是一个完整的仓库，寄宿在你的blog仓库下，如果不用 `git submodule` 来管理的话，后面部署到Github-Pages的时候会遇到 `Page build failed: themes/next was not properly initialized with a .gitmodules file` 的问题。
所以正确的姿势是，`git submodule add https://github.com/iissnan/hexo-theme-next themes/next`。
当然，我是不会说，其实我是下载一个release版本压缩包，然后解压到themes/next目录的，因为网速太慢了（记得删除包内的.git文件夹

### 设置菜单

文档中提到，除了主页和归档页，其他都要手动创建。用过 Jekyll 的人，可能会以为要写一些DSL，来创建 分类页、标签页 这样的页面了。
但是，其实这里也是万分简单，next已经帮你搞定了。
```bash
# 如果你需要创建 标签页
hexo new page "tags"
# 分类页也是一样创建
hexo new page "categories"
```
细心的你会发现，这里多了一个 `page` 参数，这条命令不会在 `_posts` 目录下创建md文件，而是在 `source` 目录下，创建 `tags/index.md`。而到浏览器一看，基本可用了诶！

## 启用 Github-Pages

这里推荐先扫一扫文档 [Hexo部署](https://hexo.io/zh-cn/docs/deployment.html)。
其部署原理是，`hexo g` 命令在 `public` 目录下生成整站静态文件，然后 `hexo d` 命令把 `public` 目录下所有文件复制到 `.deploy_git` 文件夹下（自身也是一个repo），并根据 `_config.yml` 文件中的配置，提交到相应的远程仓库和分支上去。最终Github-Pages 伺服的，其实是产生的静态文件——这些逻辑，我们将会在后面使用 Travis-CI 时用到
如果你在 Github 上用的仓库名是 `<username>.github.io` ，默认就会开启 Github-Pages，可以去项目的 `settings/options` 下面查看状态。这个情况下，github认为你这是个人的 Github-Pages，将会从 `master` 分支 build。所以推荐做法是，把你原本的代码，放到另一个分支上去（我自己放在 `code` 分支上）；产生的静态文件，放在 master 分支。

## 使用 Travis-CI 自动更新

还记得我们的目标吗？只在 `code` 分支提交新 post，后续的 `hexo g`、push 到 master 分支上这样的小事，都不想操心。
这时候迎面向我们走来的，是 Travis-CI 方阵。但热闹是`Next`作者的，我什么也没有。
看看他的这篇文章，[使用 Travis CI 自动更新 GitHub Pages](http://notes.iissnan.com/2016/publishing-github-pages-with-travis-ci/)，原理是一样的。
（文章有个坑，`GH_TOKEN`的大小写，加密时和配置文件中一定要保持一致）

我们不能直接用 Travis 执行 `hexo g --deploy`，原因是部署的时候，Travis-CI 需要你的信息，来往 Github 提交代码。所以我们只能放弃这条命令，手动做它实际做的事情，最后用上文中提到的认证方式，让 Travis-CI 来提交代码。

> 假设你和我一样，之前没用过 github 的 Travis-CI，要先去 [github integrations](https://github.com/integrations) 登录 Travis-CI，然后在[Travis profile](https://travis-ci.org/profile/)开启支持。

首先在个人设置那里，获取你的 `Personal Access Token`，然后用下面的方法加密
```bash
# 安装 Travis CI 命令行工具
gem install travis
# 加密 Personal Access Token
travis encrypt -r <username>/<repo_name> GH_TOKEN=<Personal Access Token>
```
把加密之后的 `secret`，写入你的配置文件。下面贴一下我自己的 `.travis.yml`

```yml
language: node_js
node_js: stable

# Travis-CI Caching
cache:
  directories:
    - node_modules

# S: Build Lifecycle
install:
  - npm install

before_script:
  - hexo clean

script:
  - hexo generate # 生成静态文件

after_script:
  - rm -rf .deploy_git
  - mkdir .deploy_git
  - cp -R public/* .deploy_git/
  - cd .deploy_git
  - git init
  - git config user.name "daorren"
  - git config user.email "daorenmc@gmail.com"
  - git add .
  - git commit -m "Update by Travis CI"
  - git push --force --quiet "https://${GH_TOKEN}@${GH_REF}" master
# # E: Build LifeCycle

# safelist
branches:
  only:
    - code
env:
 global:
   - GH_REF: github.com/daorren/daorren.github.io.git
   - secure: "XXX"

```
