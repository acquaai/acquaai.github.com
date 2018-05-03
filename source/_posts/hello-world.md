---
title: Hello World
date: 2017-10-24 23:45:01
categories: Hexo
---
>7年前使用[WordPress](https://wordpress.com)记录了学习Oracle、IBM的不少文章，后来慢慢变懒了，再后来就只有一个.sql文件静静躺在移动硬盘里。为了加深对知识的理解，
so，keep moving forward!

## 配置git
``` bash
➜  ~ git config --global user.name "username"
➜  ~ git config --global user.email "username@users.noreply.github.com"
➜  ~ ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
➜  ~ ssh-agent -s
➜  ~ ssh-add ~/.ssh/id_rsa
➜  ~ pbcopy < ~/.ssh/id_rsa.pub
在GitHub "Settings"左侧菜单中选择"SSH and GPG keys"，再选择右上角的"New SSH Key"，粘贴key。
测试：
➜  ~ ssh -T git@github.com
```
<!-- more -->

## GitHub上创建仓库并clone到本地
``` bash
➜  ~ git clone https://github.com/acquaai/acquaai.github.com.git
➜  ~ cd acquaai.github.com
➜  acquaai.github.com git:(master) git add .
➜  acquaai.github.com git:(master) git commit -m "first commit"
On branch master
Your branch is up-to-date with 'origin/master'.
nothing to commit, working tree clean
➜  acquaai.github.com git:(master) git push origin master
Everything up-to-date
➜  acquaai.github.com git:(master) git checkout -b hexo
Switched to a new branch 'hexo'
➜  acquaai.github.com git:(hexo) git branch -a
* hexo
  master
  remotes/origin/HEAD -> origin/master
  remotes/origin/master
  
➜  acquaai.github.com git:(hexo) git add .
➜  acquaai.github.com git:(hexo) git commit -m "hexo first commit"
On branch hexo
nothing to commit, working tree clean
➜  acquaai.github.com git:(hexo) git push origin hexo
Total 0 (delta 0), reused 0 (delta 0)
To https://github.com/acquaai/acquaai.github.com.git
 * [new branch]      hexo -> hexo
```

## 安装nvm
``` bash
➜  ~ curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.8/install.sh | bash
or
➜  ~ wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.33.8/install.sh | bash
```
### 配置环境变量
``` bash
nvm默认安装在~/.nvm，自动在shell配置文件（oh-my-zsh -> ~/.zshrc）尾增加如下内容：
➜  ~ cat .zshrc
...
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm
[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"  # This loads nvm bash_completion
```

### 查看nvm信息
``` bash
➜  ~ source ~/.zshrc
➜  ~ nvm --version
```

## 安装[Node](https://github.com/creationix/nvm)
``` bash
➜  ~ nvm ls-remote         #查看Node的版本信息
➜  ~ nvm install 8.9.4
```

如果身在墙内，可以使用国内镜像资源

``` bash
➜  ~ NVM_NODEJS_ORG_MIRROR=https://npm.taobao.org/mirrors/node nvm install 8.9.4
```


## 安装hexo
``` bash
➜  ~ cd acquaai.github.com
➜  acquaai.github.com sudo npm install -g hexo-cli
```
### 部署网站
``` bash
➜  acquaai.github.com hexo init  #初始化时要求目录为空，init完成后再CP回来
➜  acquaai.github.com npm install
```
### 生成静态页面
``` bash
➜  acquaai.github.com hexo clean
➜  acquaai.github.com hexo g	#generate
```
### 运行hexo
``` bash
➜  acquaai.github.com hexo s	#server
localhost:4000	#在浏览器中查看效果
```
## 安装发布插件
``` bash
➜  acquaai.github.com git:(hexo) npm install hexo-deployer-git --save
```

### 网站发布
``` bash
➜  acquaai.github.com git:(hexo) vi _config.yml

# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo: https://github.com/acquaai/acquaai.github.com
  branch: [master]

➜  acquaai.github.com hexo d	#deploy

```
## 安装主题
```bash
➜  acquaai.github.com git:(hexo) npm install hexo-renderer-scss --save    #安装主题插件
➜  acquaai.github.com git:(hexo) git clone https://github.com/ahonn/hexo-theme-even themes/even
➜  acquaai.github.com git:(hexo) vi _config.yml	#修改网站配置文件theme

# Extensions
## Plugins: https://hexo.io/plugins/
## Themes: https://hexo.io/themes/
theme: even
```
如果需要配置RSS，安装插件：

``` bash
➜  acquaai.github.com git:(hexo) npm install hexo-generator-feed --save
```

## [hexo-renderer-markdown-it-plus](https://github.com/CHENXCHEN/HEXO-RENDERER-MARKDOWN-IT-PLUS)
``` bash
➜  acquaai.github.com git:(hexo) npm un hexo-renderer-marked --save
➜  acquaai.github.com git:(hexo) npm i hexo-renderer-markdown-it-plus --save
```

## hexo分支提交源码文件
``` bash
➜  acquaai.github.com git:(hexo) cat .npmignore > .gitignore
➜  acquaai.github.com git:(hexo) cat .gitignore
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/%
```

``` bash
➜  acquaai.github.com git:(hexo) hexo clean
➜  acquaai.github.com git:(hexo) hexo g
➜  acquaai.github.com git:(hexo) hexo s
➜  acquaai.github.com git:(hexo) git add .
➜  acquaai.github.com git:(hexo) git commit -m "hexo configured"
➜  acquaai.github.com git:(hexo) git push origin hexo
```

## master分支提交网站静态文件
``` bash
➜  acquaai.github.com git:(hexo) hexo d
```

## .gitignore

设定 Git 提交时忽略的文件和目录。建义将 .gitignore 文件也提交到仓库中，以便其他人克隆仓库后共享此忽略规则。

### local .gitignore

+ 在终端中进入到 /path/ git仓库目录
+ touch .gitignore
+ 增加**[忽略规则](https://github.com/github/gitignore)**到 .gitignore 文件中。

在 .gitignore 文件创建之后增加新的忽略规则(file)，必须先从暂存区中删除此 file，再次使用`git add .`命令时，.gitignore中新增的忽略file才生效。

```bash
➜  git rm --cached FILENAME
```

### global .gitignore

创建全局 .gitignore 文件，用于忽略本机中所有Git仓库的规则列表。在用户home目录中创建`.gitignore_global`文件，并添加规则。

```bash
➜  git config --global core.excludesfile ~/.gitignore_global

➜  cat ~/.gitconfig
...
[core]
	excludesfile = /Users/acqua/.gitignore_global
...

➜  cat ~/.gitignore_global

###MacOS files
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db
```

### Explicit repository excludes

如果不想创建.gitignore文件与他人共享，则可以创建不与仓库一同提交的规则。
Git仓库根目录下`.git/info/exclude`文件中添加的任何规则都不会被提交，且仅忽略本地仓库中的文件。

```bash
➜  cd /git-repo-path/
➜  cat .git/info/exclude

# git ls-files --others --exclude-from=.git/info/exclude
# Lines that start with '#' are comments.
# For a project mostly in C, the following would be a good set of
# exclude patterns (uncomment them if you want to use them):
# *.[oa]
# *~
.DS_Store
```

## 多台电脑上编辑、发布文章
+ 安装、配置git
+ clone网站仓库，并切换到hexo分支
+ 安装nvm、Node
+ 安装hexo
+ npm install
+ 安装所需插件
+ 编写博客
+ 部署hexo d
+ git push origin hexo


