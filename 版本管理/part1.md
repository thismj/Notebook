# Git常用命令

1. git config

   ```bash
   //查看当前目录Git配置信息
   git config --list
   //查看全局Git配置信息
   git config --global --list
   //查看当前目录Git配置的用户邮箱
   git config user.email
   //配置当前目录Git的用户名字
   git config user.name "ThisMJ"
   ```

2. git subtree 

   ```bash
   //父仓库中添加子仓库，
   git subtree add --prefix=_book git@github.com:thismj/Notebook.git gh-pages --squash
   //从源仓库拉取更新
   git subtree push --prefix=_book git@github.com:thismj/Notebook.git gh-pages
   //推送本地修改到源仓库
   git subtree push --prefix=_book git@github.com:thismj/Notebook.git gh-pages
   ```

   