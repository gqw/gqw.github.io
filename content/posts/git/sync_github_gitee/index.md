---
date: 2021-07-28
title: "同步GitHub pages和 Gitee pages方法"
author: 顾起威 ([@gqw](https://gqw.github.io))
menu:
  sidebar:
    name: 同步GitHub pages和 Gitee pages方法
    identifier: sync_github_gitee
    parent: git
    weight: 10
hero: imgs/github_gitee.jpg
---

## 问题

这两天在整理博客，一直使用GitHub和Gitee的Pages功能作为博客的静态托管站。但是遇到以下几个问题：

1. 每次PUSH都要向两个网站分别提交，操作繁琐
2. 域名问题， 博客使用的Hugo博客生成系统，根域名是在config.yaml配置的，导致最终的域名管理比较混乱。比如配置了github的域名，提交给gitee时忘记改，就会导致某些图片或内容访问不稳定。
3. Gitee发布比较麻烦，需要手动更新提交


## 解决方法

1. 对于多次提交的问题，最初解决方法是使用以下命令：
   ```sh
   $ git remote set-url --add origin git@github.com:gqw/gqw.github.io.git

   $ git remote -v
    origin  git@gitee.com:gqw/gqw.git (fetch)
    origin  git@gitee.com:gqw/gqw.git (push)
    origin  git@github.com:gqw/gqw.github.io.git (push)
   ```
   方法解决。添加过后每次提交都会同时往两个仓库提交。

   但是为了解决第二个问题，即不同域名问题，最终并未使用此方案。

2. 解决不同域名问题，解决思路是通过git hook在提交和上次时在hook脚本里修改Hugo生成好的内容。
   1. 首先准备好远程url
    ```sh
    $ git remote -v
    github_auto_change_domian       git@github.com:gqw/gqw.github.io.git (fetch)
    github_auto_change_domian       git@github.com:gqw/gqw.github.io.git (push)
    origin  git@gitee.com:gqw/gqw.git (fetch)
    origin  git@gitee.com:gqw/gqw.git (push)
    ```
   2. 然后在仓库的.git/hooks目录中添加`commit-msg`文件，内容为：
    ```sh
    commit_msg=$(cat $1)
    echo $commit_msg

    if [[ $commit_msg =~ "replace domain to github" ]]; then
        # 向github上传前会修改仓库内容，并做一次提交记录，
        # 这时不需要重新生成文档
        echo "replace domain to github"
    else
        # 清空生成目录内容
        rm -rfv ./docs/*
        # 重新生成
        hugo
        # 复制一些固定内容
        cp -v ./static/search_engine/* ./docs/
        git add ./
    fi
    exit 0
    ```
    这样每次执行git commit时都会更新站点内容，保证每次提交内容都是最新的。

    3. 在仓库的.git/hooks目录中添加`pre-push`文件，内容为：
    ```sh
    remote="$1"
    url="$2"

    # 如果remote为 github_auto_change_domian 将跳过处理，否则会死循环，
    # 因为下面调用了git push 指令
    if [ "$remote" == "origin" ];then
        # 修改域名
        git ls-tree -r source --name-only | grep -E "docs/.*\.html" | xargs sed -i 's/gqw\.gitee/gqw\.github/g'
        # 提交修改
        git commit -a -s -m "replace domain to github"
        # 上传到giithub
        git push github_auto_change_domian
        echo "upload to github end"
    fi
    exit 0
    ```
    至此上传和域名问题解决。

3. Gitee发布问题是通过网上别人提供的github action解决的。
   在 .github/workflows 目录下添加任意*.yaml文件，内容如下：
   ```yaml
    name: Deploy to Github Pages

    # run when a commit is pushed to "source" branch
    on:
    push:
        branches:
        - source

    jobs:
    deploy:
        runs-on: ubuntu-18.04
        steps:


        - name: Build Gitee Pages
        uses: yanglbme/gitee-pages-action@main
        with:
            # 注意替换为你的 Gitee 用户名
            gitee-username: gqw
            # 注意在 Settings->Secrets 配置 GITEE_PASSWORD
            gitee-password: ${{ secrets.GITEE_PASSWORD }}
            # 注意替换为你的 Gitee 仓库，仓库名严格区分大小写，请准确填写，否则会出错
            gitee-repo: gqw/gqw
            # 要部署的分支，默认是 master，若是其他分支，则需要指定（指定的分支必须存在）
            branch: source
            directory: docs
            https: true
   ```
   此action其实只是通过python模拟登陆然后模拟用户提交更新请求。 具体使用请参考原文说明[yanglbme/gitee-pages-action](https://github.com/yanglbme/gitee-pages-action)

## 总结

好了，遇到的问题现在都处理好了， 这样在每次commit是便会自动更新站点内容，push时会自动处理域名并向两个网站分别push, 最后还自动完成了gitee的站点更新。