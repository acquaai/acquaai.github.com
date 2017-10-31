---
title: Use Hexo between multiple computers
date: 2017-10-27 22:51:49
categories: Hexo
comments: false
---
## 方法
> 利用github的branch功能可以在不同电脑上写博客。master分支默认存放hexo生成的静态文件，hexo分支存放hexo源代码，在新电脑上，只需要clone hexo分支即可。hexo分支并不需要存放所有的源文件，**node_modules**目录是npm install命令生成的，**public**目录是hexo g命令生成，**.deploy_git**目录是hexo d命令生成的，可以在**.gitignore**文件中把这三个目录忽略提交。

## 原电脑操作
### 新建hexo分支并切换
``` bash
git checkout -b hexo
Switched to a new branch 'hexo'
git checkout命令加上-b参数表示创建并切换，相当于：
git branch hexo
git checkout hexo
Switched to branch 'hexo'
```

<!-- more -->

### 编辑.gitignore文件
``` bash
➜  acquaai.github.com cat .gitignore
.DS_Store
Thumbs.db
db.json
*.log
node_modules/
public/
.deploy*/%
```

