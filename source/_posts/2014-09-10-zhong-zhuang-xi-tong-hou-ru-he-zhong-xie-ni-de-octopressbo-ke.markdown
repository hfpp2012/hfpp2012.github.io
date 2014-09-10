---
layout: post
title: "重装系统后如何重写你的Octopress博客"
date: 2014-09-10 12:43
comments: true
categories: ruby on rails
---

当你重装系统后，你可能需要重新安装开发环境和你的开发工具。如果你有用[Octopress](http://octopress.org/)记录的习惯，这个时候就得从github上clone代码来重建整个octopress博客系统，以方便你还能继续你的写博客生涯。

## 1.clone仓库代码

``` ruby 克隆source分支的代码
# git_repository_url是你的仓库地址
git clone git_repository_url --branch=source
```

<!-- more -->

## 2.安装octopress

``` ruby
bundle
```

直到安装完所有gem就可以了

## 3.添加ssh public key到github上

首先需要生成ssh的public key

``` ruby
ssh-keygen -t rsa -C "your_email@example.com"

# 把输出的内容添加到github settings中SSH keys
cat ~/.ssh/id_rsa.pub
```

## 4.新增一篇blog

``` ruby
bundle exec rake new_post['新增一篇blog']
```

## 5.发布blog

``` ruby
bundle exec rake setup_github_pages
bundle exec rake gen_deploy
```

## 6.保存你的成果

``` ruby
git add .
git commit -m "finished"
git push origin source
```

