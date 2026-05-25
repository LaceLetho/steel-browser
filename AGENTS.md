# AGENTS.md — Steel-Browser Fork 维护指南

> 本文档供 AI agent 和开发者参考，描述本 fork 仓库的分支策略和日常维护流程。

## 仓库关系

| 远程 | URL | 角色 |
|------|-----|------|
| `upstream` | `https://github.com/steel-dev/steel-browser.git` | **官方仓库**，只读，只拉取 |
| `origin` | `https://github.com/LaceLetho/steel-browser.git` | **本 fork**，可读写，存放 patch |

## 分支策略

```
upstream/main (官方最新)
     │
     │  git merge upstream/main (仅在 main 分支执行)
     ▼
  origin/main  ─── 永远与 upstream/main 一字不差，禁止直接修改
     │
     │  git rebase main (定期，在 railway 分支执行)
     ▼
  origin/railway ─── 你的自用魔改分支，含 Railway 部署所需的 patch
     │
     │  GitHub Actions 自动 docker build & push
     ▼
  ghcr.io/LaceLetho/steel-browser-railway:latest
```

## 三个分支的职责

| 分支 | 用途 | 允许的操作 | 禁止的操作 |
|------|------|-----------|-----------|
| `main` | 镜像官方 `upstream/main` | `git merge upstream/main`（仅快进合并） | **禁止直接 commit、禁止 cherry-pick、禁止 merge 其他分支** |
| `fix/cdp-proxy-changeorigin` | 给官方提 PR 的临时分支 | PR 合并后删除 | — |
| `railway` | 自用魔改分支，用于 Railway 部署 | `git rebase main`、`git cherry-pick`、`git commit` | **禁止 merge main（会产生 merge commit）** |

## 当前 Patch 清单

railway 分支相对于 main 只包含一个 patch：

| Commit | 文件 | 改动 | 原因 |
|--------|------|------|------|
| `b285892` fix: add changeOrigin to CDP WebSocket proxy | `api/src/services/cdp/cdp.service.ts` | 给 CDP WebSocket 代理添加 `changeOrigin: true` | 修复 Railway 内网环境下 CDP 连接失败的问题 |

## 日常同步官方更新

当官方 `upstream` 有新提交时，按以下步骤同步：

```bash
# Step 1: 拉取官方最新代码
git checkout main
git fetch upstream
git merge upstream/main          # 一定是 fast-forward，如果失败说明 main 被污染了

# Step 2: 推送到 origin
git push origin main

# Step 3: 将 patch 重新 rebase 到最新 main 上
git checkout railway
git rebase main

# 如果 rebase 有冲突：
#   1. 手动解决冲突文件
#   2. git add <冲突文件>
#   3. git rebase --continue
#   4. 重复直到 rebase 完成

# Step 4: 强制推送（因为 rebase 改变了历史）
git push origin railway --force
```

> ⚠️ **重要**：`railway` 分支使用 rebase 而非 merge 来保持线性历史，因此每次同步后必须 **force push**。

## 添加新的 Patch

如果你需要添加更多修改，直接在 `railway` 分支上 commit：

```bash
git checkout railway
# 修改代码...
git add <files>
git commit -m "fix: 描述你的改动"
git push origin railway
```

如果你有一个新的临时分支想要合并到 railway：

```bash
git checkout railway
git cherry-pick <commit-hash>    # 推荐：精准移植
# 或者
git rebase --onto railway <new-branch>~1 <new-branch>  # 如果只有一个 commit
```

## Docker 镜像打包

### 自动构建（推荐）

在 fork 仓库中配置 GitHub Actions workflow（`.github/workflows/build-railway.yml`），每次 push `railway` 分支时自动构建并推送镜像到 GHCR。

### 本地构建

```bash
git checkout railway

# 构建一体化镜像（API + UI + Chrome 全在一个容器内）
docker build -t ghcr.io/laceletho/steel-browser-railway:latest .

# 推送
docker push ghcr.io/laceletho/steel-browser-railway:latest
```

### 本地构建拆分镜像

```bash
# API 镜像
docker build -f ./api/Dockerfile -t ghcr.io/laceletho/steel-browser-api-railway:latest .

# UI 镜像
docker build -f ./ui/Dockerfile -t ghcr.io/laceletho/steel-browser-ui-railway:latest .
```

## 快速参考：Git 操作速查

```bash
# 查看当前分支和远程状态
git remote -v
git branch -vv

# 查看 railway 分支领先 main 多少
git log main..railway --oneline

# 查看 main 和 upstream/main 的差异
git log main..upstream/main --oneline

# 放弃 railway 上的 rebase 操作
git rebase --abort

# 查看 patch 的完整 diff
git diff main..railway
```

## 注意事项

1. **永远不要在 `main` 分支上修改代码**。`main` 的唯一作用就是保持跟 `upstream/main` 一致。
2. **`fix/cdp-proxy-changeorigin` 分支**是给官方提 PR 用的，PR 被合并后可以删除这个分支。
3. **遇到 `git merge upstream/main` 不是 fast-forward** 时，说明 `main` 分支被意外修改了。此时需要 `git reset --hard upstream/main` 强制对齐，然后 `git push origin main --force`。
4. Railway 部署应使用 `ghcr.io/laceletho/steel-browser-railway:latest` 镜像而非官方镜像。
