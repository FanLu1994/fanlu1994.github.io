---
title: Hexo博客自动部署
date: 2023-06-19 21:56:14
tags: 
- CI 
- CD 
- linux 
categories: CI 
---
## 简单介绍
Hexo博客是一个非常好用的静态博客，之前也部署过一阵，后来换了halo，但是遇到了服务器崩溃，丢了一些数据，现在又换回了[Hexo](https://hexo.io/zh-cn/docs/).


## hexo的部署
### github pages
github提供了免费的静态网页托管服务，可以将github repo作为静态网站的部署源码。
同时将repo名设置为yourname.github.io就可以通过这个域名访问到自己的静态网站。

### 以前
之前使用hexo的时候，只能在本地通过`hexo g`命令生成静态资源，然后将静态资源提交到yourname.github.io这个repo中来完成部署。

### 现在
> https://hexo.io/zh-cn/docs/github-pages

得益于github action功能的强大，hexo提供了pages.yml，有了这个action工作流，就可以将整个博客hexo项目源代码上传到项目中，避免了源码和生成的静态网站需要分开放的麻烦。
```yml
name: Pages  
  
on:  
  push:  
    branches:  
      - main # default branch  
  
jobs:  
  pages:  
    runs-on: ubuntu-latest  
    permissions:  
      contents: write  
    steps:  
      - uses: actions/checkout@v2  
      - name: Use Node.js 16.x  
        uses: actions/setup-node@v2  
        with:  
          node-version: "16"  
      - name: Cache NPM dependencies  
        uses: actions/cache@v2  
        with:  
          path: node_modules  
          key: ${{ runner.OS }}-npm-cache  
          restore-keys: |  
            ${{ runner.OS }}-npm-cache  
      - name: Install Dependencies  
        run: npm install  
      - name: Build  
        run: npm run build  
      - name: Deploy  
        uses: peaceiris/actions-gh-pages@v3  
        with:  
          github_token: ${{ secrets.GITHUB_TOKEN }}  
          publish_dir: ./public
```

首先，配置文件中定义了一个工作流（workflow）名为"Pages"。这个工作流将在推送（push）事件发生时触发执行，但仅限于"main"分支。

工作流中包含一个作业（job）名为"pages"，它将在最新的Ubuntu操作系统上运行。权限设置为`contents: write`表示此作业需要写入文件的权限。

该作业由多个步骤（steps）组成：

1. `actions/checkout@v2`：使用这个步骤来检出代码库的最新版本到工作目录中，以便后续步骤可以访问代码。
   
2. `actions/setup-node@v2`：这个步骤用于设置Node.js的环境，指定了要使用的Node.js版本为16.x。
   
3. `actions/cache@v2`：使用此步骤来缓存NPM依赖项，以便在后续构建过程中可以快速恢复。缓存路径为node_modules，缓存键（key）使用了与操作系统相关的唯一标识符。
   
4. `npm install`：这个步骤执行npm install命令，安装项目所需的依赖项。
   
5. `npm run build`：此步骤运行npm run build命令，用于构建静态网站。
   
6. `peaceiris/actions-gh-pages@v3`：最后一个步骤使用了一个名为"peaceiris/actions-gh-pages"的第三方GitHub Action。这个Action用于将构建后的静态网站发布到GitHub Pages。它使用了`${{ secrets.GITHUB_TOKEN }}`作为GitHub API的访问令牌，并指定了发布目录为"./public"。
   

通过这个配置文件，当推送到"main"分支时，GitHub Actions将自动执行这个工作流，检出代码，安装依赖，构建网站，并将构建后的内容发布到GitHub Pages上。这样就实现了自动化的静态网站部署过程。

部署完成后，静态网站就在gh-pages这个分支。


## 在自己的服务器部署hexo
因为有自己的云服务器，有自己的域名，都很便宜，闲着也是闲着，也没有做个网站的想法，就用来部署自己的博客好了。
1. 安装nginx
2. 拉取博客代码：
	`https://github.com/FanLu1994/fanlu1994.github.io.git`
3. 构建博客静态网站
	`hexo g`
4. 配置nginx
```conf
server {
        listen       443 ssl http2;
        listen       [::]:443 ssl http2;
        server_name  www.fanlu.top;
        root         /root/Blog/public;

		location / {
			root /home/xiamu/code/Blog/public;
		}
}
```
其中 /home/xiamu/code/Blog/public是生成后博客的静态代码地址
5. 脚本部署
```shell
#!/bin/bash

cd ~/code/Blog
git reset --hard HEAD

max_retries=10
retry_count=0
success=false
# 毕竟腾讯云在国内，访问github失灵时不灵，所以这里多尝试几次
while [ $retry_count -lt $max_retries ] && [ "$success" = false ]
do
    echo "Attempting to pull Git code (Attempt: $((retry_count+1)))"
    git pull

    if [ $? -eq 0 ]; then
        success=true
        echo "Git pull successful"
    else
        retry_count=$((retry_count+1))
        echo "Git pull failed. Retrying..."
    fi
done

if [ "$success" = false ]; then
    echo "Git pull failed after $max_retries attempts"
    exit 1
fi

hexo g

```


## 自动部署
为了实现每次提交后在自己的服务器自动部署更新博客，我使用了开源的定时任务管理工具[gocron](https://github.com/ouqiang/gocron)，加上github actions来实现
### gocron
使用Go语言开发的轻量级定时任务集中调度和管理系统, 用于替代Linux-crontab；
任务的实现基于[cron库](https://github.com/robfig/cron)，这个库的介绍可以看这篇文章[# Go 每日一库之 cron](https://darjun.github.io/2020/06/25/godailylib/cron/)
搭建gocorn
1. 首先要有一个mysql数据库
2. 然后使用docker启动gocorn管理系统
`docker run --name gocron --link mysql:db -p 5920:5920 -d ouqg/gocron`
3. 打开ip:5920配置数据库
4. 部署一个node （gocron支持多节点任务管理，这里我们用一个本地节点就行了）
在这里下载节点可执行文件  https://github.com/ouqiang/gocron/releases
放到服务器上执行即可。
5. 到ip:5920节点管理上增加节点。

现在一个gocron任务管理系统就搭建好了。
然后来新增一个定时部署博客的任务：
`0 0 0 * * * `表示每天零点执行任务；
![https://blog-1258032198.cos.ap-shanghai.myqcloud.com/Pasted%20image%2020230619214516.png](https://blog-1258032198.cos.ap-shanghai.myqcloud.com/Pasted%20image%2020230619214516.png)


### github Action

GitHub Actions是GitHub提供的一项持续集成（CI）和持续部署（CD）服务。它允许开发者在GitHub上配置和执行自定义的自动化工作流，以响应不同的事件触发器，例如代码推送、问题创建、拉取请求等。
1. 自动化工作流：GitHub Actions允许用户创建自定义的工作流程，其中包含一个或多个步骤。每个步骤可以执行各种操作，例如检出代码、运行命令、构建和测试应用程序、部署到服务器等。这些工作流可以自动执行，减少了手动执行重复任务的工作量。
   
2. 事件触发器：GitHub Actions可以响应各种事件触发器，例如推送到代码仓库、创建拉取请求、问题评论等。用户可以配置工作流在特定事件发生时自动触发，以执行相应的操作。这使得工作流可以根据代码库的状态和活动自动进行响应。
   
3. 社区和市场：GitHub Actions拥有一个丰富的社区生态系统和市场，其中包含许多由GitHub和开发者社区提供的预定义工作流程和操作。用户可以利用这些现成的工作流和操作，加快构建和部署流程的设置。
   
4. 无限扩展性：GitHub Actions提供了强大的扩展性，允许开发者根据自己的需求编写自定义的操作和工作流程。这使得用户可以根据特定的项目要求和工作流程进行灵活配置。
   
5. 集成和协作：GitHub Actions与GitHub平台紧密集成，可以轻松与其他GitHub功能（如拉取请求、问题和代码审查）进行协作。用户可以在工作流中使用GitHub API和第三方集成，实现更复杂的自动化任务和工作流程。

### 自动化具体实现

由于gocron本身并不提供外部调用的api，可以直接拉起任务，但是可以通过http请求，先登录、再启动任务即可。
登录需要用户名和密码，这些不能放到仓库代码中，好在github提供了secret可以配置一些密码变量。

1. repo中配置gocron登录用户名密码
	- repo-Settings-Security-Secrets and variables-Actions  添加action可以用的密码变量；

2. 编写github Action
	- 在 项目根路径/.github/workflows下面新建一个deploy.yml
```yml
name: Deploy On My server
on:
  push:
    branches:
      - main  # 更改为你要触发的分支


jobs:
  login:
    runs-on: ubuntu-latest  # 可以根据需要更改操作系统

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Login and Fetch Token
        run: |
          # 在这里编写登录请求的代码，使用适当的语言和库发送POST请求，获取并提取出令牌
          # 将令牌存储在一个变量中

          # 示例（使用cURL发送请求）：
          token=$(curl -X POST -d 'username= ${{ secrets.username }}&password= ${{ secrets.password }}'  http://gocron地址:5920/api/user/login | jq -r '.data.token')

          echo "Token: $token"

          # 将令牌存储为一个GitHub Actions的环境变量，以便在后续的步骤中使用
          echo "TOKEN=$token" >> $GITHUB_ENV
      
      - name: Start Deploy Task
        run: |
          # 在这里编写发送下一个请求的代码，使用适当的语言和库
          # 将上一步获取的令牌放置在请求标头中

          # 示例（使用cURL发送请求）：
          token=$TOKEN  # 获取上一步中存储的令牌

          curl -H "Auth-Token: $token"  http://gocron地址:5920/api/task/run/1

```

这样就ok啦！