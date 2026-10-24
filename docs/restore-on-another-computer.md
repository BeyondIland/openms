# 在另一台电脑恢复运行环境

Git 保存源码；原始 WZ、生成资源、数据库和本机配置需要另外准备。只想重新开始游戏，可以按 [Quick Start](index.md) 新建环境；需要保留账号、角色和 Studio 内容时，还要迁移数据库。以下是恢复步骤说明，不代表已在新电脑完成启动验证。

## 1. 恢复源码与依赖

在新电脑安装 Git、Bun 和 PostgreSQL，或按 Quick Start 使用 Podman/Compose。先恢复与旧环境相同的源码提交，确认成功后再升级：

```sh
git clone https://github.com/BeyondIland/openms.git
cd openms
git checkout SOURCE_COMMIT
bun install --frozen-lockfile
```

`SOURCE_COMMIT` 替换为备份时记录的提交。尚未提交的源码不会随克隆恢复，需事先另存私有补丁或提交到自己的分支。

## 2. 准备原始与生成资源

- 原始 `Maplestory-Client/`、WZ 文件和资源 ZIP 不随 Git 分发。单独保存已有资源，或按 Quick Start 获取项目所需输入。
- 推荐用相同源码和输入重新生成：`bun extract --assets ../Maplestory-Client`。路径可改为新电脑上实际包含 WZ 的目录，等待 `Extraction succeeded`。
- 也可以私下复制完整的 `client/public/generated/`，包括 `catalog.json` 及其所有依赖文件。记录旧源码提交、原始输入身份和 catalog 的构建身份，使用与其匹配的源码/提取规则；不要混用不同提取结果，也不要绕过身份校验。
- `client/dist/` 是可重新构建的输出；生产站点中的 `site/generated/` 只有 catalog，不能替代完整生成资源。提取缓存可重新建立。

原始和生成资源应单独保存，不要加入公开源码仓库。数据库中的 Studio 内容可能引用特定资源构建，恢复时也要保留对应生成资源。见[输入与来源](inputs.md)、[资源交付](asset-delivery.md)。

## 3. 重建本机配置

从 `.env.server.example`、`.env.client.example`、`.env.studio.example` 分别复制出同名但不含 `.example` 的文件，不覆盖已有配置。也可私下迁移旧配置，但应逐项核对新电脑上的数据库连接、端口和路径。

重点检查 `.env.server` 的 `DATABASE_URL`、`OPENMS_ORIGIN`，`.env.client` 的 `OPENMS_SERVER_URL`，以及 Studio 的服务地址与 `OPENMS_CONTENT_ROOT`。后者默认是相对仓库的 `client/public/generated`；后台和 Studio 必须使用同一份资源。公开 HTTPS 地址改变时，后台允许的浏览器 origin 也要同步。配置加载和优先级见[开发设置](development.md#settings)。真实密码只留在私有配置中。

## 4. 恢复数据库与角色

账号、密码哈希、角色进度、物品、交易，以及 Studio 修订、上传图片和世界发布保存在 PostgreSQL 中。源码和 WZ 都不能恢复这些数据。数据库备份含密码哈希和玩家资料，应保存在受限的私有位置，并通过私密渠道转移。

先在新电脑建立 PostgreSQL 实例及独立的数据库登录角色。旧本机启动器使用 PostgreSQL 17；仓库 Compose 使用其配置的镜像版本。用与源服务兼容的 `pg_dump`，目标服务使用兼容版本，避免向更老的 PostgreSQL 恢复。数据库监听与 `.env.server` 应一致；现有本机流程使用回环地址和端口 `55432`。

**不保留旧数据：** 新建空数据库后，从仓库根目录执行：

```sh
bun run migrate --database-url "postgresql://DB_ROLE:URL_ENCODED_PASSWORD@127.0.0.1:55432/openms"
```

替换全部占位值；URL 中的密码需要 URL 编码。迁移工具只读取显式 `--database-url`，不会自动读取 `.env.server`。它依次执行 `infra/sql/` 根目录下的编号脚本并记录文件哈希；`tables/`、`data/` 是提取用参考输入，不应手工导入 PostgreSQL。

**保留旧账号和角色：** 先让玩家退出并停止游戏前后台写入，保持源 PostgreSQL 运行，再做完整逻辑备份。以下命令假设 PostgreSQL 工具已加入 PATH，`PRIVATE_BACKUP.dump` 是仓库外的私有备份路径；数据库角色和库名也要换成实际值，密码由工具提示输入：

```sh
pg_dump --host=127.0.0.1 --port=55432 --username=SOURCE_DB_ROLE --password --format=custom --file=PRIVATE_BACKUP.dump SOURCE_DATABASE
```

源数据库角色需要能读取全部待备份对象。保存源码提交和生成资源身份，并检查命令成功退出。不要复制运行中的 `pgdata` 目录作为跨电脑备份。

在目标实例中，由数据库管理员先创建应用登录角色和一个空库。若角色已经存在，跳过创建角色并检查其权限：

```sh
createuser --host=127.0.0.1 --port=55432 --username=TARGET_DB_ADMIN --password --login --pwprompt TARGET_DB_ROLE
createdb --host=127.0.0.1 --port=55432 --username=TARGET_DB_ADMIN --password --owner=TARGET_DB_ROLE --template=template0 TARGET_DATABASE
pg_restore --host=127.0.0.1 --port=55432 --username=TARGET_DB_ROLE --password --dbname=TARGET_DATABASE --no-owner --no-privileges --exit-on-error --single-transaction PRIVATE_BACKUP.dump
```

完整备份包含表结构、数据和 `migrations` 历史：**恢复前不要先运行应用迁移建表**，也不要往已有游戏数据的库直接叠加恢复。`pg_dump` 不备份实例级登录角色；上述步骤重建目标角色，并通过 `--no-owner --no-privileges` 避免沿用旧机器的所有者和授权。当前编号迁移不要求额外 PostgreSQL 扩展；如果自行安装过扩展，先在目标实例安装兼容的扩展支持文件，并安排有权限的角色恢复相应扩展。

恢复成功后，将 `.env.server` 指向目标库，使用备份对应源码的迁移历史进行校验，再按需要升级源码并运行同一条 `bun run migrate --database-url "…"` 命令补齐新增迁移。若历史或哈希不匹配，应先核对源码版本，不要删除历史记录或手工改哈希。迁移成功后才启动游戏。详见[数据库迁移](development.md#migrations)。

## 5. 启动与浏览器状态

使用与旧环境一致的运行模式和私有配置。新建本地开发环境可按 Quick Start 分别运行 `bun run server:dev` 和 `bun run client:dev`；面向公网时使用[生产配置](development.md#production)。每次运行 `server:dev` 都会重设 `admin`、`player` 这两个开发账号的密码，使用 `OPENMS_DEV_PASSWORD` 或公开开发默认值，因此它们原有的自定义密码可能改变；其他已保存账号的密码不因此重设。生产启动保留非默认密码，仅替换仍使用公开默认密码的这两个账号。恢复后检查账号登录、角色列表和原有进度，不要假定模板中的示例密码能登录旧账号。

浏览器缓存、部分窗口位置和开发面板设置保存在浏览器本地，按浏览器配置和网址 origin 隔离。换电脑、浏览器配置、域名、端口，或从 `localhost` 换到 `127.0.0.1` 后，这些内容不会随数据库恢复。重新下载资源、调整本地界面并重新登录即可；账号与角色进度以服务器数据库为准。登录 cookie 和内存中的重连状态不应作为迁移材料。见[浏览器会话](browser-session.md)。

## 可选：恢复私人 Windows 一键启动流程

仓库旁边的 `Start-OpenMS*.ps1/.cmd`、`Stop-OpenMS*.ps1/.cmd`、`OPENMS-FRIENDS.txt` 和 `.openms-runtime/` 属于本机辅助环境，不在公开仓库中。可以单独私下备份脚本和必要工具，也可以直接使用 README 的标准安装方式。

这些脚本依赖既有目录布局、便携 Bun、PostgreSQL 安装与集群、生成资源，好友模式还依赖 cloudflared 和 provisioning helper；换电脑后需检查和修改路径，并重新初始化数据库实例后做逻辑恢复，不能保证复制后直接启动。不要把旧 PID、进程归属状态或日志当作新机器的运行状态恢复。

若要继续使用原好友账号，可以私下保留 `.openms-runtime/friends-private/accounts.json`；它含凭据，不能提交 Git 或整份发送给朋友。该文件只保存凭据，不能替代数据库。现有 provisioning helper 会拒绝覆盖不匹配的自定义密码；丢失凭据时按授权流程重设，勿用新随机文件覆盖后假定旧账号已恢复。

好友分享应按随脚本私下保留的 `OPENMS-FRIENDS.txt` 重新启动并验证生产模式、HTTPS 页面及普通玩家登录，分享本次新生成的 HTTPS 地址。旧临时隧道链接不会随迁移恢复。结束分享时先关闭隧道并确认进程归属，再恢复本地模式。
