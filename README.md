本网站的目的是方便系统级软件方面的知识共享。由武汉大学comsoft课题组主要维护。欢迎加入贡献，欢迎git clone。

# 加入贡献：
1. 申请一个github账号，提醒项目管理员（请联系：qingan@whu.edu.cn），将该账号添加到该项目。
2. >git clone https://github.com/comsoftWHU/comsoftwhu.github.io.git
3. 添加或修改.md文件，git add加入本地库。
    - 如果需要新增子目录dir，来存放dir/filename.md，则需要在新目录dir下额外添加一个index.md文件，作为该子目录的首页。 
        - >mkdir dir && cd dir
        - >touch filename.md && touch index.md
    - 如果需要x.md文件需要引用图片，则在同目录下增加一个x.assets子目录，专门用来存放对应的图片。
    - 为了加快网页加载速度，如非必要，建议不要使用高分辨率图片（比如：QQ截图软件，可以设置为低分辨率）。
    - 文件名/文件夹名中不要用中文，否则不同系统环境下，路径操作比较麻烦。
4. 在本地部署测试，利用本地浏览器预览web。
    - > bundle install && bundle exec jekyll serve
    - 通过 http://localhost:4000/ 查看本地测试结果
    - 更多参考 https://github.com/just-the-docs/just-the-docs-template/blob/main/README.md
5. 如果测试通过
    - > git commit： 提交到本地库
    - > git push：提交到远程github库（提交后，会自动部署到https://comsoftwhu.github.io/提供静态web服务）。

# 说明：
1. 当前有如下子目录，请将文档组织到相关目录。
    - cs-startup：CS系统领域的入门资料。
    - 编译：编译相关的入门资料。
    - AI编译：AI编译项目相关资料，包括AI优化概念、TVM框架等。
    - Linux内核：Linux内核相关资料。
    - AOSP：Android的开源实现AOSP项目相关资料。
    - FH-compile：全同态加密编译。
2. 文档组织以markdown提交为主，会被自动渲染成html并部署到https://comsoftwhu.github.io/comsoftWHU/；markdown文件中的图片，可以保存到一个文件夹一并push（参考compile/gc.mk, compile/gc.assets/)。
3. markdown文件的编写，可参考已有文件，或参考https://just-the-docs.github.io/just-the-docs/。
4. 文件名/文件夹名中不要用中文，否则路径操作比较麻烦。



# just-the-docs-template

This is a *bare-minimum* template to create a [Jekyll] site that:

- uses the [Just the Docs] theme;
- can be built and published on [GitHub Pages];
- can be built and previewed locally, and published on other platforms.

More specifically, the created site:

- uses a gem-based approach, i.e. uses a `Gemfile` and loads the `just-the-docs` gem;
- uses the [GitHub Pages / Actions workflow] to build and publish the site on GitHub Pages.

To get started with creating a site, just click "[use this template]"!

If you want to maintain your docs in the `docs` directory of an existing project repo, see [Hosting your docs from an existing project repo](#hosting-your-docs-from-an-existing-project-repo).

After completing the creation of your new site on GitHub, update it as needed:

## Replace the content of the template pages

Update the following files to your own content:

- `index.md` (your new home page)
- `README.md` (information for those who access your site repo on GitHub)

## Changing the version of the theme and/or Jekyll

Simply edit the relevant line(s) in the `Gemfile`.

## Adding a plugin

The Just the Docs theme automatically includes the [`jekyll-seo-tag`] plugin.

To add an extra plugin, you need to add it in the `Gemfile` *and* in `_config.yml`. For example, to add [`jekyll-default-layout`]:

- Add the following to your site's `Gemfile`:

  ```ruby
  gem "jekyll-default-layout"
  ```

- And add the following to your site's `_config.yml`:

  ```yaml
  plugins:
    - jekyll-default-layout
  ```

Note: If you are using a Jekyll version less than 3.5.0, use the `gems` key instead of `plugins`.

## Publishing your site on GitHub Pages

1.  If your created site is `YOUR-USERNAME/YOUR-SITE-NAME`, update `_config.yml` to:

    ```yaml
    title: YOUR TITLE
    description: YOUR DESCRIPTION
    theme: just-the-docs

    url: https://YOUR-USERNAME.github.io/YOUR-SITE-NAME

    aux_links: # remove if you don't want this link to appear on your pages
      Template Repository: https://github.com/YOUR-USERNAME/YOUR-SITE-NAME
    ```

2.  Push your updated `_config.yml` to your site on GitHub.

3.  In your newly created repo on GitHub:
    - go to the `Settings` tab -> `Pages` -> `Build and deployment`, then select `Source`: `GitHub Actions`.
    - if there were any failed Actions, go to the `Actions` tab and click on `Re-run jobs`.

## Building and previewing your site locally

Assuming [Jekyll] and [Bundler] are installed on your computer:

1.  Change your working directory to the root directory of your site.

2.  Run `bundle install`.

3.  Run `bundle exec jekyll serve` to build your site and preview it at `localhost:4000`.

    The built site is stored in the directory `_site`.

## Publishing your built site on a different platform

Just upload all the files in the directory `_site`.

## Customization

You're free to customize sites that you create with this template, however you like!

[Browse our documentation][Just the Docs] to learn more about how to use this theme.

## Hosting your docs from an existing project repo

You might want to maintain your docs in an existing project repo. Instead of creating a new repo using the [just-the-docs template](https://github.com/just-the-docs/just-the-docs-template), you can copy the template files into your existing repo and configure the template's Github Actions workflow to build from a `docs` directory. You can clone the template to your local machine or download the `.zip` file to access the files.

### Copy the template files

1.  Create a `.github/workflows` directory at your project root if your repo doesn't already have one. Copy the `pages.yml` file into this directory. GitHub Actions searches this directory for workflow files.

2.  Create a `docs` directory at your project root and copy all remaining template files into this directory.

### Modify the GitHub Actions workflow

The GitHub Actions workflow that builds and deploys your site to Github Pages is defined by the `pages.yml` file. You'll need to edit this file to that so that your build and deploy steps look to your `docs` directory, rather than the project root.

1.  Set the default `working-directory` param for the build job.

    ```yaml
    build:
      runs-on: ubuntu-latest
      defaults:
        run:
          working-directory: docs
    ```

2.  Set the `working-directory` param for the Setup Ruby step.

    ```yaml
    - name: Setup Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '3.1'
          bundler-cache: true
          cache-version: 0
          working-directory: '${{ github.workspace }}/docs'
    ```

3.  Set the path param for the Upload artifact step:

    ```yaml
    - name: Upload artifact
        uses: actions/upload-pages-artifact@v1
        with:
          path: "docs/_site/"
    ```

4.  Modify the trigger so that only changes within the `docs` directory start the workflow. Otherwise, every change to your project (even those that don't affect the docs) would trigger a new site build and deploy.

    ```yaml
    on:
      push:
        branches:
          - "main"
        paths:
          - "docs/**"
    ```

## Licensing and Attribution

This repository is licensed under the [MIT License]. You are generally free to reuse or extend upon this code as you see fit; just include the original copy of the license (which is preserved when you "make a template"). While it's not necessary, we'd love to hear from you if you do use this template, and how we can improve it for future use!

The deployment GitHub Actions workflow is heavily based on GitHub's mixed-party [starter workflows]. A copy of their MIT License is available in [actions/starter-workflows].

----

[^1]: [It can take up to 10 minutes for changes to your site to publish after you push the changes to GitHub](https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/creating-a-github-pages-site-with-jekyll#creating-your-site).

[Jekyll]: https://jekyllrb.com
[Just the Docs]: https://just-the-docs.github.io/just-the-docs/
[GitHub Pages]: https://docs.github.com/en/pages
[GitHub Pages / Actions workflow]: https://github.blog/changelog/2022-07-27-github-pages-custom-github-actions-workflows-beta/
[Bundler]: https://bundler.io
[use this template]: https://github.com/just-the-docs/just-the-docs-template/generate
[`jekyll-default-layout`]: https://github.com/benbalter/jekyll-default-layout
[`jekyll-seo-tag`]: https://jekyll.github.io/jekyll-seo-tag
[MIT License]: https://en.wikipedia.org/wiki/MIT_License
[starter workflows]: https://github.com/actions/starter-workflows/blob/main/pages/jekyll.yml
[actions/starter-workflows]: https://github.com/actions/starter-workflows/blob/main/LICENSE
