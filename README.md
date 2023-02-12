# 欢迎来到飙戈的博客
## 概况

这是基于 Hexo + Github Pages 搭建的个人博客。包含的功能如下：

| 插件                         | 作用             | 版本  |
| ---------------------------- | ---------------- | ----- |
| Hexo                         | 博客框架         | 6.3.0 |
| Next                         | 主题             | 7.8.0 |
| hexo-deployer-git            | 部署插件         | 3.0.0 |
| hexo-generator-searchdb      | 本地搜索插件     | 1.4.1 |
| hexo-generator-sitemap       | 谷歌站点地图插件 | 3.0.1 |
| hexo-generator-baidu-sitemap | 百度站点地图插件 | 0.1.9 |

基于 Next 主题下添加的插件有：

| 插件     | 作用                  | 版本 |
| -------- | --------------------- | ---- |
| busuanzi | 访问简单统计+页面展示 |      |
| baidu    | 访问后台统计          |      |
| valine   | 评论系统              |      |

## 使用

1、克隆到本地 hexo 目录

```shell
$ mkdir hexo
$ cd hexo
$ git clone git@github.com:mcbilla/mcbilla.github.io.git
```

2、初始化仓库，在 hexo 目录下执行

```shell
$ npm install
```

3、运行，在在 hexo 目录下执行

```shell
$ hexo server
```

4、部署远程服务器

```
$ npm run deploy
```