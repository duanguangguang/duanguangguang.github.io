---
title: Hexo+Github多地备份
date: 2018-07-29 22:56:48
categories: Hexo
tags:
  - Hexo
---

## Hexo+Github个人博客多地备份

个人博客家里和公司各有一份环境，随时编辑更新，之前是这么做的，最近在新公司电脑上配置hexo环境的时候，发现还是遇到了很多坑，很多细节操作忘记了，现在趁热做下笔记

#### 一、环境安装

- node.js
- git
- hexo: [Hexo](https://hexo.io/zh-cn/docs/index.html)

验证环境安装：

![](hexobackup\hexo03.png)

#### 二、操作步骤

1. 拉backup分支到本地并做一备份（共两份文件），如图：

   ![](hexobackup\hexo06.png)

2. 新建文件夹初始化hexo;

   - `hexo init`
   - `npm install`
   - `hexo -p 5000 server` 本地测试hexo

   初始化完成后如图：

   ![](hexobackup\hexo05.png)

3. 将初始化完成的hexo文件内容覆盖到backup中；

   注意：node_modules在从hexo拷贝到backup的过程中因为文件名太长导致失败，需要使用命令拷贝

   > 切换到需要拷贝得文件得根目录：`cp -a hexo/node_modules/ duanguangguang.github.io/`

4. 将backup备份内容除了node_modules文件覆盖到backup中

这么做的原因是需要更新本地的hexo jar包，因为node_modules文件在.gitgnore忽略掉了，可以看到最终backup分支比刚备份下来得多了node_modules文件，如下图：

![](hexobackup\hexo04.png)

#### 三、插件安装

- 将hexo与git关联起来

  > npm install hexo-deployer-git --save

- 可能会遇到图片无法显示的问题

  > npm install https://github.com/CodeFalling/hexo-asset-image --save

#### 四、ssh配置

将生成的公有密钥添加到github上即可

#### 五、github分支管理策略

1. master分支

   - master分支存储的是博客生成的可以直接在浏览器上显示的静态文件

   - `hexo generate` 生成静态文件（）

   - `hexo deploy ` 将生成的静态文件部署到master上

   - 我们在_config.yml配置了hexo关联的master分支，`hexo deploy` 命令会自动将生成的静态文件推送到master分支，除此之外master分支切勿自己提交内容，切记！！！

   - master分支内容就是backup分支中的public文件内容

   - 正确的master分支内容如下图：

     ![](hexobackup\hexo01.png)

2. backup分支

   - backup分支是对我们的博客文件包括主题配置等做的一个远程备份

   - `git add --all` 将修改的文件添加到git

   - `git commit -m "description"` 文件提交

   - `git push origin backup` 文件推送到远程backup分支

   - 正确的backup分支内容如下图：

     ![](hexobackup\hexo02.png)

   ​