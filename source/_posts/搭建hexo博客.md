---
title: 搭建hexo博客
date: 2022-03-24 14:46:12
tags: 
  - hexo
categories:
  - usage
---

# win10搭建hexo博客

## 1. win安装node.js

- 1.1 下载到本地并双击安装：https://nodejs.org/en/download/
  ![3-24-1](/images/3-24/1.png)
- 1.2 一路下一步安装，默认安装完成路径为 `C:\Program Files\nodejs`
- 1.3 进入win终端测试`node.js`是否已经安装成功
  ```
  C:\Users\Think>node -v
  v16.14.0

  C:\Users\Think>npm -v
  8.3.1
  ```
- 1.4 配置缓存路径，路径自行选择
  - 1.4.1 参考以下命令配置`node_cache`和`node_global`
  ```
  npm config set prefix "C:\Users\Think\Desktop\ssh-key\node_cache"
  npm config set cache "C:\Users\Think\Desktop\ssh-key\node_global"
  ```
  - 1.4.2 使用以下命令检查是否生效
  ```
  C:\Users\Think>npm config list
  ; "builtin" config from C:\Program Files\nodejs\node_modules\npm\npmrc

  ; prefix = "C:\\Users\\Think\\AppData\\Roaming\\npm" ; overridden by user

  ; "user" config from C:\Users\Think\.npmrc

  cache = "C:\\Users\\Think\\Desktop\\ssh-key\\node_cache"  
  list = ""
  prefix = "C:\\Users\\Think\\Desktop\\ssh-key\\node_global"

; node bin location = C:\Program Files\nodejs\node.exe
; cwd = C:\Users\Think
; HOME = C:\Users\Think
; Run `npm config ls -l` to show all defaults.
  ```
- 1.5 配置环境变量
  - 1.5.1 参考图中方式找到环境变量配置，点击`环境变量`
  ![3-24-2](/images/3-24/2.png)
  - 1.5.2 在`系统变量`下新建`NODE_PATH`，输入`C:\Program Files\nodejs\node_modules`
  ![3-24-3](/images/3-24/3.png)
  - 1.5.3 将`用户变量`下的`Path`添加`C:\Users\Think\Desktop\ssh-key\node_global`
  ![3-24-4](/images/3-24/4.png)
- 1.6 安装完成后，尝试安装`hexo`，检查是否真的安装成功
  ```
  npm install hexo hexo-cli -g
  hexo -v 
  ```
  ![3-24-5](/images/3-24/5.png)
  
## 2. 安装git
- 2.1 参考对应[博客](https://www.cnblogs.com/xueweisuoyong/p/11914045.html)

## 3. 配置 github 仓库
- 3.1 创建仓库的名字必须为 注册名+github.io的形式，如图
  ![3-24-6](/images/3-24/6.png)
- 3.2 通过以下方式生成 github ssh key，`-C` 后加登录github邮箱
  ```
  # ssh-key-gen -t rsa -C "1524204934@qq.com"
  ```
- 3.3 将pub key复制到 github ，并创建新的ssh key
  ![3-24-7](/images/3-24/7.png)
- 3.4 测试是否成功将ssh key绑定
  ```
  # ssh -T git@github.com
  ```

## 4. 生成本地博客内容
- 4.1 在配置好的博客目录执行以下命令
```
# hexo init
# hexo g
```

- 4.2 启动博客
```
# hexo s
```

## 5. 发布博客到 github
- 5.1 修改博客根路径中 `_config.yml`文件以下部分
  其中， type为git， repository链接会在 `5.2` 中告知如何获取， branch 为仓库的分支，当前启用的是ssh 的方式进行推送
```
deploy:
  type: git
  repository: git@github.com:LinYuXiang087241/LinYuXiang087241.github.io
 # repository: https://github.com/LinYuXiang087241/LinYuXiang087241.github.io.git
  branch: main
```
- 5.2 仓库地址有https和ssh，分别对应不同的推送方式
  - 5.2.1 https仓库地址获取方式为
    ![3-24-8](/images/3-24/8.png)
  - 5.2.2 ssh 仓库地址为
    ![3-24-9](/images/3-24/9.png)
	
**<u><font color=red>仅用作经验记录</font></u>**