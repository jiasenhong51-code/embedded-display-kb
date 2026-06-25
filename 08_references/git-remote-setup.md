# Git Remote Setup

这个文件记录如何把当前本地知识库上传到 GitHub 或 Gitee。

当前本地仓库信息：

- 路径：`/home/tuanzi/embedded-display-kb`
- 分支：`main`
- 首次提交：`cff60ac`

## 1. 先确认本地 Git 可用

当前环境原本没有系统级 `git`，已经在用户目录下准备了一个本地 Git 包装命令：

```sh
git --version
```

如果你以后想安装系统级 Git，可以执行：

```sh
sudo apt update
sudo apt install git
```

## 2. 配置提交身份

建议把下面两项改成你自己的 GitHub 或 Gitee 用户信息：

```sh
git config --global user.name "your-name"
git config --global user.email "your-email@example.com"
```

如果只想对当前仓库生效，去掉 `--global`：

```sh
cd /home/tuanzi/embedded-display-kb
git config user.name "your-name"
git config user.email "your-email@example.com"
```

当前仓库临时配置为：

```sh
git config user.name
git config user.email
```

## 3. 在 GitHub 或 Gitee 创建空仓库

创建远程仓库时建议：

- 仓库名：`embedded-display-kb`
- 不要勾选自动创建 `README`
- 不要勾选自动创建 `.gitignore`
- 不要勾选自动创建 license

原因是本地仓库已经有首次提交，远程空仓库最容易直接推送。

## 4. 只推送到 GitHub

把 `OWNER` 替换成你的 GitHub 用户名或组织名。

SSH 方式：

```sh
cd /home/tuanzi/embedded-display-kb
git remote add origin git@github.com:OWNER/embedded-display-kb.git
git push -u origin main
```

HTTPS 方式：

```sh
cd /home/tuanzi/embedded-display-kb
git remote add origin https://github.com/OWNER/embedded-display-kb.git
git push -u origin main
```

## 5. 只推送到 Gitee

把 `OWNER` 替换成你的 Gitee 用户名或组织名。

SSH 方式：

```sh
cd /home/tuanzi/embedded-display-kb
git remote add origin git@gitee.com:OWNER/embedded-display-kb.git
git push -u origin main
```

HTTPS 方式：

```sh
cd /home/tuanzi/embedded-display-kb
git remote add origin https://gitee.com/OWNER/embedded-display-kb.git
git push -u origin main
```

## 6. 同时同步到 GitHub 和 Gitee

推荐把 GitHub 作为 `origin`，Gitee 作为 `gitee`：

```sh
cd /home/tuanzi/embedded-display-kb
git remote add origin git@github.com:OWNER/embedded-display-kb.git
git remote add gitee git@gitee.com:OWNER/embedded-display-kb.git
git push -u origin main
git push -u gitee main
```

以后每次更新：

```sh
git status
git add .
git commit -m "Update display notes"
git push origin main
git push gitee main
```

## 7. 常见问题

如果提示 `remote origin already exists`：

```sh
git remote -v
git remote set-url origin NEW_REMOTE_URL
```

如果远程仓库已经有 `README` 导致推送被拒绝，优先考虑新建一个空仓库。也可以拉取并合并：

```sh
git pull --rebase origin main
git push -u origin main
```

如果 Gitee 仓库默认分支是 `master`，但本地是 `main`，建议继续使用 `main`。如果你确实要改成 `master`：

```sh
git branch -M master
git push -u origin master
```

## 8. 参考资料

- [GitHub Docs: Adding locally hosted code to GitHub](https://docs.github.com/en/migrations/importing-source-code/using-the-command-line-to-import-source-code/adding-locally-hosted-code-to-github)
- [Gitee 帮助中心：Git 仓库基础操作](https://help.gitee.com/Git%20%E7%9F%A5%E8%AF%86%E5%A4%A7%E5%85%A8/Git%E4%BB%93%E5%BA%93%E5%9F%BA%E7%A1%80%E6%93%8D%E4%BD%9C)
