# 分支维护与上游同步

本分支基于 [tensorfish/openms](https://github.com/tensorfish/openms)，沿用原项目的 [GPL 许可证](../LICENSE)。源码更新与游戏资源、玩家存档分别维护；仓库不提供官方 WZ、提取后的资源、玩家数据库、本地账号或 `.env` 配置。

## 获取本分支

```sh
git clone https://github.com/BeyondIland/openms.git
cd openms
bun install --frozen-lockfile
```

首次运行继续按 [README 的 Quick Start](../README.md#quick-start) 准备本地配置、外部资源和数据库。原 README 的资源下载与提取步骤仍然适用。不要把资源包、`client/public/generated/`、数据库备份、账号密码、运行日志或本地验证报告加入提交。

## 合并上游更新

先将需要保留的本地源码修改提交到当前分支，确认 `git status --short` 没有未提交改动。提交前检查暂存区，避免带入本地配置、凭据或资源。下面的操作从你要继续维护的分支创建更新分支，保留已有提交历史。

第一次同步时添加上游地址；已有 `upstream` 时用 `git remote -v` 检查地址，不要重复添加：

```sh
git remote add upstream https://github.com/tensorfish/openms.git
```

每次更新：

```sh
git fetch upstream
git switch -c codex/sync-upstream
git merge upstream/main
```

如果更新分支名已存在，换一个新的分支名，例如 `codex/sync-upstream-YYYYMMDD`。合并出现冲突时，逐个检查并保留需要的上游变化和本分支修复，暂存解决后的文件，再运行 `git merge --continue`。需要放弃本次未完成的合并时可用 `git merge --abort`。同步不会自动保证断线恢复、切图和攻击等修复与未来上游兼容，冲突解决后仍需验证。

## 验证与运行

依赖或锁文件更新后运行 `bun install --frozen-lockfile`。按改动范围执行格式、静态检查和回归测试，参见[验证范围](validation-method.md#validation-scope)。涉及本分支的联机、切图、攻击或生产构建修改时，可选择对应测试：

```sh
bun test client/test/transport-recovery.test.js client/test/portal-transport.test.js client/test/native-transitions.test.js server/test/attack-input.test.js client/test/online-deployment.test.js
```

数据库迁移 SQL 有更新时，先备份自己的数据库，再使用真实连接地址运行 `bun run migrate --database-url URL`；迁移不会从 `.env.server` 自动读取地址。参见[数据库说明](development.md#server)。普通源码同步不需要清空玩家数据库。

原始资源、提取输入或提取规则变化时，按 Quick Start 重新提取；仅修改联机、切图或界面代码时复用已有 `client/public/generated/`。更新运行代码后重启受影响的后端和前端并刷新浏览器，确认规则和资源版本匹配。验证通过后，将更新分支按自己的评审流程合并回维护分支；发布前再次检查提交中没有本地数据或凭据。
