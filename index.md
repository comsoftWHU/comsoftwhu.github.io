---
title: Home
layout: home
nav_order: 1
---
本网站的目的是方便武汉大学comsoft课题组的知识共享。

# 步骤：
1. 申请一个github账号，提醒项目管理员添加到该项目。
2. git clone下载本项目。
3. 添加或修改相关文件，git add加入本地库。
4. 在本地部署测试，利用本地浏览器预览web。
    - > bundle install && bundle exec jekyll serve
    - http://localhost:4000/ 查看
    - 更多参考 https://github.com/just-the-docs/just-the-docs-template/blob/main/README.md
5. 如果测试通过，利用git push提交到github库，会自动部署到https://comsoftwhu.github.io/。

# 说明：
1. 当前有四个目录，编译、AI编译、Linux内核和AOSP，请将文档组织到相关目录。
2. 文档组织以markdown提交为主，会被自动渲染成html并部署到https://comsoftwhu.github.io/comsoftWHU/；markdown文件中的图片，可以保存到一个文件夹一并push（参考compile/gc.mk, compile/gc-image/)。
3. markdown文件的编写，可参考已有文件，或参考https://just-the-docs.github.io/just-the-docs/。
4. 文件名/文件夹名中不要用中文，否则路径操作比较麻烦。


# CS入门课
## MIT：计算机教育中缺失的一课
- https://missing-semester-cn.github.io/


# 以下忽略：


This is a *bare-minimum* template to create a Jekyll site that uses the [Just the Docs] theme. You can easily set the created site to be published on [GitHub Pages] – the [README] file explains how to do that, along with other details.

If [Jekyll] is installed on your computer, you can also build and preview the created site *locally*. This lets you test changes before committing them, and avoids waiting for GitHub Pages.[^1] And you will be able to deploy your local build to a different platform than GitHub Pages.

More specifically, the created site:

- uses a gem-based approach, i.e. uses a `Gemfile` and loads the `just-the-docs` gem
- uses the [GitHub Pages / Actions workflow] to build and publish the site on GitHub Pages

Other than that, you're free to customize sites that you create with this template, however you like. You can easily change the versions of `just-the-docs` and Jekyll it uses, as well as adding further plugins.

[Browse our documentation][Just the Docs] to learn more about how to use this theme.

To get started with creating a site, just click "[use this template]"!

If you want to maintain your docs in the `docs` directory of an existing project repo, see [Hosting your docs from an existing project repo](https://github.com/just-the-docs/just-the-docs-template/blob/main/README.md#hosting-your-docs-from-an-existing-project-repo) in the template README.

----

[^1]: [It can take up to 10 minutes for changes to your site to publish after you push the changes to GitHub](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll#creating-your-site).

[Just the Docs]: https://just-the-docs.github.io/just-the-docs/
[GitHub Pages]: https://docs.github.com/en/pages
[README]: https://github.com/just-the-docs/just-the-docs-template/blob/main/README.md
[Jekyll]: https://jekyllrb.com
[GitHub Pages / Actions workflow]: https://github.blog/changelog/2022-07-27-github-pages-custom-github-actions-workflows-beta/
[use this template]: https://github.com/just-the-docs/just-the-docs-template/generate