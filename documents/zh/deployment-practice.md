# MyTube 部署汇总

> 面向想自托管 MyTube 的实践记录：三套实例、一套云端存储打通方案，以及踩过的坑。
> 本文档不含任何服务器地址、账号、令牌等个人信息，可直接公开。

## 一、项目简介

MyTube（https://github.com/jeffreyrobeson/MyTube）是一个自托管的视频库应用：

- 下载并收藏网络视频（主要为 YouTube），按作者、合集、标签管理，内置播放器；
- 支持订阅频道自动下载、字幕下载、缩略图与作者头像；
- 后端为 Node.js 服务 + SQLite，内置 yt-dlp 管理（自动下载/升级 yt-dlp 发行版）；
- 前端为 SPA，后端可同源 serving 前端；
- 设置存入数据库 settings 表（值多为 JSON 编码，未保存过的键不存在，用默认值）。

本文基于 **v1.11.20** 实践（2026-11-10 从上游合并更新；此前为 v1.11.19）。

**没有的功能（不要误以为有）：** 无转码、无 HLS/自适应播放、无单次下载的质量选择框、无播放器内切换清晰度。清晰度只由全局「首选视频分辨率」决定。

**v1.11.18 新增：批量整理。** 视频列表支持多选，一次操作为多个视频追加标签或加入一个合集（见第十一节）。

**v1.11.19 新增：无功能变化。** 仅安全依赖补丁（上游 PR #539），见第十二节。

**v1.11.20 新增：下载连接 SSRF 防护与移动端界面调整。** 下载连接新增 SSRF 防护（GHSA-ppj6）；移动端操作表共用滑动过渡与遮罩，车载布局修正（见第十三节）。

## 二、三套实例对照

| | 实例 A：VPS（主力） | 实例 B：助理 VM（第二实例） | 实例 C：Simular Box（第三实例，2026-11-12 起在跑） |
| --- | --- | --- | --- |
| 运行方式 | Docker Compose | 源码构建、Node 直跑（环境无 Docker） | 源码构建、Node 直跑（环境无 Docker、无 systemd） |
| 前端 / 后端端口 | 5556 / 5551 | 后端 5551（同源 serving 前端） | 后端 5551（同源 serving 前端） |
| 视频主存储 | 云端（见第三节） | 本地磁盘 + 单向同步备份到云端（见第四节） | 本地磁盘 + 单向同步备份到云端（见第十四节） |
| 公开访问 | 服务器端口直出 | 经 VPS 的 SSH 反向隧道公开（MyTube 25551、OpenList 25244） | MyTube 经 VPS 的 SSH 反向隧道公开（55551）；OpenList 用另一台的 45244（见第十四节第 3 条） |
| 登录 | 已开启（401 保护 API） | 全新部署后尚未开启，待在设置里开启 | 已开启（401 保护 API） |
| yt-dlp cookies | 使用登录 cookies（该出口 IP 未被标记） | 不用登录 cookies（见第五节） | 使用登录 cookies（见第十四节第 4 条） |

## 三、实例 A：视频直存云端（VPS + OpenList + 天翼云盘）

思路：不把视频存在服务器硬盘，而是把云盘挂载成 MyTube 的视频目录。

1. 在同一台服务器上部署 OpenList（systemd 服务，端口 5244），在其中挂载天翼云盘存储。
2. 用 rclone 把 OpenList 的 WebDAV 挂载到本地目录（需安装 fuse3），挂载参数要点：
   - `--vfs-cache-mode writes`（写回缓存，文件关闭后约数秒开始上传）；
   - 容器以 uid/gid 1000 运行，需 `--uid 1000 --gid 1000 --dir-perms 0775 --file-perms 0664 --allow-other`，并在 `/etc/fuse.conf` 中启用 `user_allow_other`；
   - 缓存上限按需设置（实践中从 2 GB 提到 20 GB，以接受大文件合并时的缓存风险）。
3. 在 `docker-compose.yml` 中把挂载目录绑定到后端容器的视频目录（如 `- /mnt/openlist/tianyi:/app/uploads/videos`），然后**强制重建后端容器**。
4. 关键运维规则：**每次 rclone 重新挂载后，必须强制重建后端容器**，否则容器持有旧的绑定，看到的是空目录。

云端上传语义（非常重要）：

- 写操作先进入 VFS 缓存，关闭文件后异步以**单个 WebDAV PUT**上传；
- **文件在整个 PUT 完成前，在云盘侧完全不可见**，不是“传了一半”；
- 在上传完成前删除文件会取消上传；
- 天翼侧上传速度不稳定，多 GB 文件以小时计是正常的；
- 中间产物（如下载合并前的分流文件）在合并后会被 MyTube 删除，其上传失败日志（context canceled）属于正常现象。

## 四、实例 B：本地存储 + 单向同步备份（VM）

约束：该 VM 是受限容器，内核支持 FUSE 但容器禁止创建 `/dev/fuse`，因此 **rclone mount 在此不可用**，云盘不能作为主视频目录。改用同步备份方案：

1. 同机部署 OpenList（v4.2.8，二进制直接运行，不走官方脚本的 systemd 路线），在其中挂载天翼云盘存储。
   - 注意：全新安装不会自动创建管理员账号，需通过 Web 界面的首次设置向导创建；CLI 的 `admin set` 在无管理员时静默失败，不要用直接写库、改密码哈希的方式硬造账号。
2. 用 rclone **copy**（不是 sync）经 OpenList WebDAV 把本地视频目录同步到云端目录：
   - WebDAV 鉴权用 OpenList 的 admin token 作 Bearer 令牌（Basic 方式无效，匿名无效）；
   - token 每次运行现取现用，不写进配置文件、不记录；
   - `--size-only --min-age 10m`：跳过 10 分钟内未完成的文件（合并中的视频），并排除 `.part`、`.temp.*`、`.f[0-9]*.*`、`.ytdlp*` 等中间文件；
   - 上传在 flock 锁保护下 detached 运行，避免慢速大文件上传被定时任务超时杀掉；
   - **单向备份：本地删除视频不会删除云端副本**。
3. 公开访问：经 SSH 反向隧道（`ssh -N -R`）把本地端口转发到 VPS 的公网端口；看门狗每 30 分钟检查，本地正常但公网不通时重建隧道。
   - 注意：该 VM 直连 VPS 的 SSH 会被出口网关重置，须经 HTTP CONNECT 代理建隧道（`ProxyCommand nc -X connect -x <代理>:3128 %h %p`）；不要绕过代理直连。
   - 注意：该 VM 上 ssh 按系统用户家目录读配置（如 root 则读 `/root/.ssh/config`），看门狗脚本里应写显式 `-o` 参数，不要只依赖配置文件。
   - 注意：服务与隧道启动必须用 `setsid` 分离，否则 worker 退出时进程被一起杀；不要在命令行里含目标端口字面量的 shell 中跑 `pkill -f`，它会杀掉自己（看门狗脚本里用 `pkill -f "0.0.0.0:3555[1]"` 这类括号写法，且只在脚本文件内执行）。

## 五、yt-dlp：证书、Cookies 与 Bot 验证

这三类问题报错都是 `yt-dlp process exited with code 1`，要看 stderr 原文区分：

### 1. 证书问题

现象：`SSL: CERTIFICATE_VERIFY_FAILED … self-signed certificate in certificate chain`。

成因：出口代理对 TLS 做中间人，yt-dlp 自带的 certifi 证书包不信任代理 CA。

根治：把所用 yt-dlp 发行版的 `site-packages/certifi/cacert.pem` 替换为**指向系统证书包的软链**（如 `/etc/ssl/certs/ca-certificates.crt`）。这样代理 CA 轮换时系统包自动更新，软链无需再动。

注意：**每次升级 yt-dlp 后必须重做一次**（新发行版会带一份全新的 certifi 文件，不是软链）。

### 2. 登录 Cookies 的反作用（被标记的出口 IP）

现象：`This video is DRM protected`（每个视频都报）。

成因：在已被 YouTube 标记的出口 IP 上使用登录 cookies，YouTube 只返回 DRM 加密格式。

结论：**在这种 IP 上不要用登录 cookies，匿名下载反而正常**；会员专属、私密、年龄限制视频仍可能无法下载，改走未被标记的实例。

### 3. Bot 验证（2026-10-26 起）

现象：`Sign in to confirm you're not a bot`。

成因：YouTube 对匿名请求做机器人验证，出口 IP 被标记后连公开视频也无法匿名获取。

解法：把 YouTube 播放器客户端切到 android：

- 在 MyTube 设置的 yt-dlp 配置中加一行（去掉注释）：

  ```
  --extractor-args youtube:player_client=android
  ```

- android 客户端目前一般只能拿到 360p 的 mp4 合并格式（format 18），要更高清请走未被标记的实例。
- 不要用登录 cookies 绕过（见上一节，会触发 DRM）。

### 4. 一个本地补丁：取信息调用也要走 android

只改设置不够。MyTube 下载流程分两步：先调一次 yt-dlp 取视频信息（标题、作者），再真正下载文件。取信息那次调用在代码里写死了只带 pot-provider 参数，不带用户的 extractorArgs，因此仍走默认 web 客户端撞上 Bot 验证，任务在取信息阶段就失败（日志里表现为没有 `Video info:` 一行），根本走不到真正下载那步。

补丁（针对 `ytdlpVideo.ts` 的取信息调用）：把用户配置里的 extractorArgs 与 pot-provider 参数合并后传给取信息调用。改动后需重新编译后端并重启服务。注意：**这是偏离上游的本地补丁，上游升级后会被覆盖**，升级后需重新评估是否还需打补丁。2026-11-06 合并 v1.11.19 后验证补丁仍在；2026-11-10 合并 v1.11.20 后复核补丁仍在——上游本次改了该文件的缩略图代理一段（与补丁区域无关）。每次合并上游后都要复核这一处。

### 5. Web 客户端 + 登录 Cookies + JS 运行时（实例 C 的组合）

在出口 IP 未被标记、且浏览器里有 YouTube 登录的情况下，web 客户端配合登录 cookies 是可行的，且能拿到比 android 更全的格式。实例 C（Simular Box）验证过的组合（写在 MyTube 设置的 yt-dlp 配置里，一行一个选项，见第十四节第 4 条）：

  ```
  --extractor-args youtube:player_client=web
  --cookies-from-browser chrome:/home/user/.config/cocoonbox-chrome/Default
  --js-runtimes node
  --remote-components ejs:github
  ```

要点（缺一不可，详见第十四节第 5 条）：

- `--js-runtimes node`：yt-dlp 的 JS 运行时默认只认 deno，机器上没有 deno 时报 `JS runtimes: none`，n 挑战解不出来；Box 上有 node（v24），必须显式启用。
- `--remote-components ejs:github`：EJS 挑战求解脚本默认不启用，必须显式加上。
- `--cookies-from-browser` 必须写完整路径：Box 的 Chrome 用的是 `~/.config/cocoonbox-chrome`，而 yt-dlp 默认去 `~/.config/google-chrome` 找，找不到就报 `could not find chrome cookies database`。
- android 客户端不支持 cookies：配了 cookies 后 android 会被跳过，只剩 storyboard 图片格式，见第十四节第 5 条第 3 项。

## 六、全局设置说明

- **首选视频分辨率**（preferredVideoResolution）：`auto` 为原始/最佳，或指定 2160 / 1440 / 1080 / 720 / 480 / 360；默认 `auto`。
- **严格限制分辨率上限**（preferredVideoResolutionStrict）：默认关闭；开启后分辨率作为硬上限。
- **首选容器**（preferredVideoContainer）：如 `mp4`。
- 以上均为 settings 表键值，只在保存后出现；建议在设置界面里改，不要直接写库。
- yt-dlp 配置文本框支持标准 yt-dlp 配置语法（`#` 开头为注释），见第五节第 3 条的用法。

## 七、保活与体检体系

运行环境会不定时重启，重启会杀掉服务进程与隧道，因此需要：

- **每 30 分钟看门狗**：分别检查本地 MyTube、本地 OpenList、两条公网隧道；服务停了就重启，本地正常但公网不通就重建隧道；成功时不打扰用户，只有连续多次失败才告警。
- **每天一次全面体检**：在看门狗之外检查深层健康——证书软链是否还在、磁盘余量、yt-dlp 是否有新版、同步是否有超过 24 小时未落盘的文件；能自动修的修好，修不好的、需要用户决策的才汇报。
- **天翼落盘监控**：每 30 分钟对照本地与云端清单，新落盘的文件才汇报；在途上传不算完成。

## 八、常见故障速查

| 现象 | 原因 | 处理 |
|---|---|---|
| `CERTIFICATE_VERIFY_FAILED` | 代理 CA 不被 certifi 信任 | 证书软链（第五节第 1 条），升级 yt-dlp 后重做 |
| `Sign in to confirm you're not a bot` | 出口 IP 被标记，匿名请求被拦截 | 切 android 客户端；取信息补丁（第五节第 3、4 条）；或走未被标记的实例 |
| `This video is DRM protected` | 登录 cookies + 被标记 IP | 停用登录 cookies，匿名下载 |
| 公网地址打不开，本地正常 | 隧道掉了 | 重建 SSH 反向隧道；以 VPS 侧监听与本地访问为准判断 |
| 服务地址打不开，本地也不通 | 运行环境重启杀掉了服务 | 看门狗会在 30 分钟内拉起，或手动按已知命令重启 |
| 云端看不到刚下的视频 | 整文件 PUT 未完成 | 等待；多 GB 文件以小时计，属正常 |
| 取信息后迟迟不动 | 取信息调用走了 web 客户端 | 见第五节第 4 条 |
| rclone 同步报 401 Unauthorized | OpenList 首次设置向导未完成，token 被服务端判定无效 | 在 OpenList Web 界面完成首次设置、挂载天翼后再同步；未完成前同步无法工作 |
| SSH 连 VPS 被重置 | 出口网关拦截直连 SSH | 经 HTTP CONNECT 代理（见第四节第 3 条） |
| `n challenge solving failed` / `JS runtimes: none` | yt-dlp 默认只认 deno，机器上无 deno | `--js-runtimes node` + `--remote-components ejs:github`（第十四节第 5 条第 6 项） |
| `Plugin and HTTP server major versions are mismatched` | POT 插件与服务端主版本不一致 | 插件与服务端一起升到同一版本（第十四节第 5 条第 5 项） |
| `Requested format is not available`，且只有 storyboard 格式 | 后端附加的 `--cookies` 文件里的 YouTube cookies 已过期失效 | 把过期的 cookies 文件移走备份，只用浏览器实时 cookies（第十四节第 5 条第 7 项） |
| `MyTube SSRF protection: this media transport is not supported` | 字幕语言 `all` 包含 live_chat，SSRF 校验拦了 `youtube_live_chat` 协议 | 把 `youtube_live_chat`、`youtube_live_chat_replay` 加进 `network-guard.py` 白名单（第十四节第 5 条第 8 项） |
| HTTP Error 403（android 客户端，个别视频） | YouTube 对该视频开了 SABR-only 实验 | 换 web 客户端 + 登录 cookies（第十四节第 5 条第 1 项） |
| `relay to remote OpenList is down` / 中继 14544 返回 000 | 远程 OpenList 的 45244 进程停了（主机本身通） | 去那台机器上重启 OpenList，或等它自动恢复（半小时内）；恢复后中继无需重建（第十四节第 5 条第 9 项） |
| Box 本地 /boxtianyi 挂载失败（天翼登录超时） | Box 连不上 189.cn | 同步不走它，走远程 OpenList，可忽略（第十四节第 3 条） |

## 九、同步状态核查

### 同步备份实例（rclone copy，无挂载）

```bash
# 看同步日志（日志是 UTC 时间，北京时间 +8 小时）
tail -f ~/workspace/mytube-tianyi-sync.log
# 找 Transferring（正在传，带百分比和 ETA）、Copied (new)（刚传完）、Errors（失败）

# 看已落盘清单，在里面的 = 已到云端
cat ~/workspace/mytube-tianyi-reported.txt

# 看有没有在传（只判断进程存不存在，不要把完整 ps 输出贴出来，命令行里带令牌）
ps aux | grep -q "[r]clone copy" && echo "正在传" || echo "没在传"

# 手动触发一次同步检查（自动取令牌、传 backlog、打印新落盘的文件）
bash ~/workspace/mytube-tianyi-sync.sh
```

直接查云端清单需要 OpenList 的 admin 令牌，分两步，令牌切勿外发：

```bash
cd ~/workspace/openlist && ./openlist admin token
~/workspace/bin/rclone ls openlist-vm:musetianyi/mytube --config ~/workspace/rclone-openlist.conf --webdav-bearer-token 'TOKEN'
# 带大小用 lsl；本地和云端逐一比对用：
~/workspace/bin/rclone check /home/hatch/workspace/mytube/backend/uploads/videos openlist-vm:musetianyi/mytube --size-only --config ~/workspace/rclone-openlist.conf --webdav-bearer-token 'TOKEN'
```

### 挂载实例（rclone mount，视频直存云端）

```bash
# 在服务器上执行
rclone lsl openlist:tianyi
grep "upload succeeded" /var/log/rclone-openlist.log
grep ERROR /var/log/rclone-openlist.log
```

### 怎么判断“同步完成”

- 完成：本地视频清单 = 云端清单（名字和大小一致）；最新一次同步日志里没有 Transferring，也没有 rclone 进程在跑。
- 进行中：日志有 Transferring 百分比和 ETA，或 rclone 进程在跑。
- 异常：日志出现 Errors、Failed to copy、403/405，或者某个文件超过 24 小时还没落盘（每日体检会报这个）。

注意：云端是整文件 PUT 完成前完全不可见，传到 99% 时云端也看不到它，不代表丢了，等到 100% 才会出现。

## 十、当前状态（2026-11-12）

- 已更新到上游 v1.11.20（2026-11-10 合并，见第十三节）；本地 android 取信息补丁仍在；
- 实例 B：2026-11-12 核查时在助理 VM 上未找到实际部署（该 VM 上没有 MyTube），本节此前写的实例 B 状态是计划值；当前实际在跑的是实例 C（见第十四节）；
- 实例 C（Simular Box）：MyTube v1.11.20 源码构建，本地 5551 正常；OpenList v4.2.8 本地 5244 正常；公网隧道 MyTube 55551 已通并验证；OpenList 用另一台的 45244（Box 本地 55244 部署不成功，不在用）；MyTube 登录已开启；yt-dlp 配置为 web + 登录 cookies + node + ejs（见第十四节第 4 条）；两个视频已下载并同步备份到天翼（见第十四节第 7 条，核查方法见第九节）；
- 中继 OpenList（45244）时断时续：2026-11-12 02:00 与 12:48 两次拒绝连接，主机本身通，是该服务进程停了；有专人监控、半小时内自动恢复（13:01 已恢复，13:00 定时同步成功）；
- Box 本地 /boxtianyi 挂载仍坏：2026-11-12 12:53 复查，Box 到 189.cn 仍 10 秒超时，日志里天翼 OAuth 登录超时 20 次（最后一次 08:29）；同步不走它，不影响落盘；
- yt-dlp 托管发行版 2026.10.26 已安装（证书软链已做，升级后须重做，见第五节第 1 条）；
- VPS 实例：4 个视频已全部落盘天翼；
- 保活（30 分钟看门狗 + 5 分钟 systemd 定时任务，定时任务被平台重建清除后由看门狗自动重装）与每日体检均已启用。

## 十一、批量整理（v1.11.18 新增）

v1.11.18 加入了批量整理能力，解决大量导入后逐个整理的成本。设计文档见 `documents/designs/issue514-bulk-video-organization-design.md`（Issue #514），接口文档见 `documents/zh/api-endpoints.md` 的「批量整理」一节。

- **入口**：首页全部视频/历史视图、全部标签页（`/tags`）可进入选择模式，多选视频卡片；管理页视频表格沿用常驻复选框。一次操作可追加一个或多个标签，或加入一个合集。
- **新增接口**（均需 CSRF 防护；开启登录时要求管理员，访客及 API Key 不可用）：
  - `POST /api/videos/batch/tags`：`{ videoIds, tags }`，一个事务内为 1–500 个视频追加标签，保留旧标签，按大小写不敏感去重；新标签使用全局目录中的拼写。
  - `POST /api/collections/batch-add`：`{ videoIds, collectionId }` 加入现有合集，或 `{ videoIds, newCollection: { id, name } }` 原子新建并加入；两种目标只能指定一种。
- **约束**：加入合集不移动媒体文件；一次请求整体成功或整体回滚，不返回部分成功；重复追加不产生重复标签或关联。标签删除、标签替换、批量移出合集、批量修改其他字段不在本期范围内。
- **其他接口收紧**：`PUT /api/videos/:id` 现在校验 `subtitles`（对象数组，每项至少有字符串 `path` 或 `filename`）和 `tags`（字符串数组）的类型，非法输入在保存任何字段前返回 `400`。

## 十二、v1.11.19 更新（2026-11-06）

上游 v1.11.19 只有安全依赖补丁，无功能变化（上游 PR #539：修补存在漏洞的依赖，并记录一处 CodeQL 误报）。合并时无冲突，本地 android 取信息补丁（第五节第 4 条）验证仍在；部署后 `/api/system/version` 返回 `currentVersion` 与 `latestVersion` 均为 `1.11.19`，`hasUpdate` 为 `false`。

## 十三、v1.11.20 更新（2026-11-10）

上游 v1.11.20 包含两组改动（上游 PR #541 安全修复、PR #542 界面调整），合并时无冲突，本地 android 取信息补丁（第五节第 4 条）验证仍在；部署后本地 5551/5244 与公网 25551/25244 均为 200，yt-dlp 2026.10.26 正常。

- **下载连接 SSRF 防护（GHSA-ppj6-v966-7mx3）**：新增 `backend/src/utils/securityNetwork.ts`、`backend/src/utils/publicAxios.ts` 与 yt-dlp 发行版 `networkGuard.ts`，对下载连接做 SSRF 防护；同时保留源站启动与可信存储兼容性（中英文 docker-guide 各 +12 行说明）。与本次一并更新的还有 yt-dlp 启动器（`release/launcher.ts`）。
- **移动端界面调整**：新增共用的 `ActionSheet` 组件，移动端操作表统一用滑动过渡与遮罩；视频菜单（VideoKebabMenu）相应调整。
- **车载布局修正**：桌面浏览器忽略已记住的车机覆盖；车机端限制已记住的覆盖范围。

## 十四、实例 C：Simular Box（2026-11-12）

### 1. Simular Box 是什么

Simular Box（https://simular.ai）是一台按量计费的云桌面：浏览器里打开就是一台带图形界面的 Ubuntu 桌面，自带 Chrome 浏览器与终端，可像用本地电脑一样操作。本次用的是免费套餐（1,000 点数，免费版无配置可选）。

- 一台名为 main 的云桌面：Ubuntu 24.04.5 LTS，XFCE 桌面，内核 6.8.0-142-generic，内存 3.8 GB，磁盘 20 GB（约 2.3 GB 已用），用户 `user`，家目录 `/home/user`。
- 系统镜像 v19（2026-11-11 从 v15 升到 v17，修好了 Box 对外访问网页的问题；2026-11-12 已是 v19）。
- **没有 systemd**：所有服务都不能用 systemd 管理，全部以前台终端或 `setsid` 分离的方式手工跑，见第 6 条。
- **空闲会休眠**：Box 空闲一段时间后自动休眠，界面显示 Offline 是正常现象，唤醒后几秒恢复；但休眠/重启会杀掉所有手工跑的进程（MyTube 后端、隧道、POT 服务、cron 守护进程），见第 6 条。
- 网络出口：Shell 环境默认把 HTTP/HTTPS 流量经本地代理 `127.0.0.1:3128`、SOCKS5 经 `127.0.0.1:1080`；apt 也走 3128 代理。对外 HTTPS 到多数网站正常（百度、Google 均可达）。**但有两个例外**：天翼云盘 API（open.e.189.cn）从 Box 和 VPS 都访问不到（DNS 能解析，但连接超时）；另一台 OpenList 的地址 Box 也直连不上（出口拒绝）。因此 Box 上不能直接挂天翼云盘，同步要经过 VPS 中继（见第 3 条）。

### 2. 部署步骤

1. 环境检查：Node v24.21.0、npm 11.19.0、git 2.43.0、Python 3.12.3，GitHub 可达，磁盘有空余。
2. 克隆 MyTube 源码到 `~/mytube`。
3. 安装 ffmpeg 6.1.1（apt，无需密码）。
4. 安装 yt-dlp 2026.10.26（`pip install --user --break-system-packages`，并把 `~/.local/bin` 加进 `~/.bashrc` 的 PATH）；同时装好 bgutil-ytdlp-pot-provider 的 pip 插件（2.0.2）与 `curl_cffi`（注意：旧文档曾写 `curl-ffi`，该名字在 PyPI 上不存在，正确包名是 `curl_cffi`）。
5. `npm run install:all` 与 `npm run build`；后端 `.env` 里 `PORT=5551`。
6. 启动后端：`cd ~/mytube/backend && npm run start`，以前台终端跑（终端关了服务就停）；验证 `http://127.0.0.1:5551/api/system/version` 返回 1.11.20。
7. OpenList v4.2.8：二进制解压到 `~/openlist`，`setsid ./openlist server` 启动，端口 5244，日志 `/tmp/openlist.log`。**全新安装不会自动创建管理员账号，需通过 Web 界面的首次设置向导创建；CLI 的 `admin set` 在无管理员时静默失败，不要用直接写库、改密码哈希的方式硬造账号。**
8. 在 MyTube 设置里开启登录保护（开启后读设置、改设置都要登录会话，未登录返回 401；CSRF 校验是 `X-CSRF-Token` 请求头与 `mytube_csrf` cookie 逐字相等）。

### 3. 公开访问与云端同步

公开访问（SSH 反向隧道）：Box 没有公网 IP，借 VPS 的公网端口公开。先在 Box 上生成一对专用 ed25519 密钥（无口令，注释 `box-mytube-tunnel`，私钥不出 Box），公钥加进 VPS 的 `/root/.ssh/authorized_keys`（用 `ssh-keygen -lf` 按指纹核对；**注意：从终端截图上抄 base64/指纹容易看错 1/I、0/O、8/B、g/q，必须按指纹核对**）。隧道命令：

  ```
  setsid ssh -i ~/.ssh/id_ed25519_mytube -o ServerAliveInterval=30 -o ExitOnForwardFailure=yes -N -R 0.0.0.0:55551:localhost:5551 -R 0.0.0.0:55244:localhost:5244 root@<VPS>
  ```

  VPS 侧 `sshd_config` 须把 `GatewayPorts` 设为 `yes`（默认只绑 127.0.0.1，公网打不开；改完 `sshd -t` 检查并重载 sshd）。公开地址：MyTube `http://<VPS 公网 IP>:55551/`，验证返回 200。Box 的 OpenList 虽经隧道转发到 55244 且能返回 200，但它的天翼挂载是坏的，部署不成功、不在用；实际用的 OpenList 是另一台的 45244（见下面同步一节）。

云端同步（Box → VPS 中继 → 另一台 OpenList）：Box 直连不上天翼 API 和中继 OpenList，于是用一条 SSH 本地转发做中继：

  ```
  setsid ssh -i ~/.ssh/id_ed25519_mytube -o ServerAliveInterval=30 -o ExitOnForwardFailure=yes -N -L 127.0.0.1:14544:<中继 OpenList 地址>:45244 root@<VPS>
  ```

  rclone 配置 `~/rclone-openlist-remote.conf`（remote 名 `openlist-remote`，type webdav，url `http://127.0.0.1:14544/dav`，vendor other；**配置里不写令牌**）。同步脚本 `~/mytube-tianyi-sync.sh` 每 30 分钟由 cron 跑（Box 无 systemd，cron 守护进程需 `sudo cron` 手工启动，重启后要重起）：先检查中继通不通，断了就自动重建一次；每次运行从 `~/openlist-remote-token`（600 权限，唯一存放处）现读 admin token，用 `flock -n` 加锁，跑 `rclone copy ~/mytube/backend/uploads/videos openlist-remote:boxtianyi/mytube --config ~/rclone-openlist-remote.conf --webdav-bearer-token "$TOKEN" --size-only --min-age 10m --exclude "*.part" --exclude ".temp.*" --exclude ".f[0-9]*.*" --exclude ".ytdlp*"`，日志 `~/mytube-tianyi-sync.log`；脚本从不回显令牌。目标是远程 OpenList 上的 `boxtianyi/mytube`（该 OpenList 上另有 `/musetianyi`，不要用错）。

  注意：Box 上的 OpenList 本地挂载 `/boxtianyi`（189CloudPC 驱动）因 189.cn 访问不到而加载失败，驱动每次访问都会重试登录，若网络恢复可能自愈；2026-11-12 12:53 复查时 Box 到 189.cn 仍 10 秒超时，OpenList 日志里天翼 OAuth2 登录超时（`context deadline exceeded`）共 20 次，最后一次 08:29（此后日志无新写入）；DAV 返回 401、界面访客被禁，需登录后才能在界面上核实挂载本身。同步走远程 OpenList，不依赖它，可忽略。

  注意：中继 OpenList（45244）时断时续——2026-11-12 02:00 与 12:48 两次拒绝连接，主机本身通（从 VPS ping 过去 0.78ms，只有一跳；22、80、443 端口都开着），是该服务进程停了；有专人监控、半小时内自动恢复（13:01 已恢复，13:00 定时同步成功）。另有一个已知隐患：同步脚本里重建中继的那行写的是 VPS 自己的 45244（`-L 127.0.0.1:14544:127.0.0.1:45244`），而实际跑着的中继用的是 `<中继 OpenList 地址>:45244`；VPS 本机根本没有 45244 监听，万一现在这个中继死了、由脚本重建，新中继会指到一个没有服务的地方，同步会失败——暂未改，监控会兜底。

### 4. yt-dlp 配置（验证过的组合）

MyTube 设置里的 yt-dlp 配置（标准 yt-dlp 配置语法，一行一个选项，空行与 `#` 注释跳过）：

  ```
  --extractor-args youtube:player_client=web
  --cookies-from-browser chrome:/home/user/.config/cocoonbox-chrome/Default
  --js-runtimes node
  --remote-components ejs:github
  ```

- pot-provider 的 script 参数由 MyTube 代码自动合并，**不要手动加**。
- 该配置是全局的，所有下载都用 web 客户端 + 已登录的 Chrome cookies + node + 远程 ejs 求解器。
- 改配置要登录 MyTube（API 未登录返回 401），走 `PATCH /api/settings`，CSRF 用 `X-CSRF-Token` 请求头 + `mytube_csrf` cookie；改完 GET 读回核对。
- 注意：v1.11.20 的 Web 界面**没有** yt-dlp 配置输入框，配置只能走 API（或直接改库，但不推荐）。

### 5. 部署中遇到的问题（逐个讲）

下面是 2026-11-12 一天内实际踩到并逐个解决的问题，顺序就是排查顺序。它们报错都一样——`yt-dlp process exited with code 1`——必须看 stderr 原文和后端日志才能区分。

#### 问题 1：android 客户端个别视频下载 403

现象：取信息正常（标题、格式都能拿到），下载阶段报 `ERROR: unable to download video data: HTTP Error 403: Forbidden`，同一时间用对照视频复测能下（206）。

成因：YouTube 对该视频开了 SABR-only 实验，只针对 android 客户端；是视频级别的差异，不是配置、版本、网络或 IP 封禁问题（日志里有 `Some android client https formats have been skipped as they are missing a URL`，对应 yt-dlp issue #12482）。

解法：换 web 客户端 + 登录 cookies（见第 4 条）。该实验是按会话的，同一视频换个时间也可能恢复，但不要依赖它。

#### 问题 2：web / ios 客户端没有 cookies 时撞 Bot 验证

现象：取信息阶段就失败，`Sign in to confirm you're not a bot`，连媒体地址都拿不到，比 android 更差。

成因：出口 IP 被标记后，YouTube 对匿名请求做机器人验证；web 与 ios 客户端在匿名时比 android 更容易被拦。

解法：给 web 客户端配登录 cookies（见第 4 条）；单独切播放器客户端解决不了这个问题。

#### 问题 3：android + cookies 是死路

现象：配了 cookies 后 android 被跳过（`Skipping client 'android' since it does not support cookies`），回落只剩 storyboard 图片格式（sb3/sb2/sb1/sb0、mhtml），没有音视频格式，`--get-title` 退出 1。

成因：android 客户端不支持 cookies；yt-dlp 有 cookies 时直接跳过 android。

解法：不要在这个组合上浪费时间；用 web + cookies（见第 4 条）。

#### 问题 4：Box 的 Chrome 里没有 YouTube 登录，cookies 找不到

现象：`--cookies-from-browser chrome` 报 `could not find chrome cookies database`；当时 Cookies 库里 0 条 youtube.com 记录。

成因：Box 的 Chrome 用的是 `--user-data-dir=/home/user/.config/cocoonbox-chrome`，而 yt-dlp 默认去 `~/.config/google-chrome` 找，两边不是同一个目录；且当时确实没在 Box 的 Chrome 里登过 YouTube。

解法：在 Box 的 Chrome 里登录一次 YouTube（登录后该 profile 有 54 条 cookies，其中 20 条是 youtube.com，yt-dlp 能提取 53 条）；同时 `--cookies-from-browser` 写完整路径（见第 4 条）。

#### 问题 5：POT 服务没跑、插件与服务端版本对不上

现象：web + cookies 仍失败，报错 `Error reaching GET http://127.0.0.1:4416/ping` 与 `n challenge solving failed`，只剩 storyboard 格式。

成因分两层：

- POT 服务（bgutil）根本没跑过：它从未被验证启动过。启动方式是在 `~/mytube/backend/bgutil-ytdlp-pot-provider/server` 里 `setsid nohup node build/main.js`，日志 `/tmp/pot-server.log`，约 27 秒后就绪，`ping` 返回 200 即正常；它也是脱离进程，Box 休眠/重启会死。
- 版本对不上：启动后发现 yt-dlp 插件是 2.0.2、服务端是 1.2.2，主版本不一致时插件拒绝工作（`Plugin and HTTP server major versions are mismatched`）。把服务端升到 2.0.2 后警告消失，但必须注意：**只升 server/ 目录不够，plugin/ 目录也要一起升到 2.0.2**——后端启动 yt-dlp 时会把仓库内嵌的插件路径加到 PYTHONPATH 最前面，所以实际加载的是仓库内嵌的 1.2.2 插件，而不是 pip 装的 2.0.2；只升 server 的那次重试仍报版本不一致。两个目录都升到 2.0.2 后警告消失（备份留好：`~/mytube/backend/bgutil-ytdlp-pot-provider.bak-1.2.2`）。

#### 问题 6：n 挑战解不出来（JS 运行时）

现象：版本对齐后仍报 `n challenge solving failed`，只有 storyboard 格式。

成因：yt-dlp 的 `--js-runtimes` 默认只认 deno，Box 上没有 deno（有 node v24 但默认不启用），yt-dlp 报告 `JS runtimes: none`，解不了 n 挑战；EJS 挑战求解脚本（远程组件 `ejs:github`）默认也不启用。

解法：配置里同时加 `--js-runtimes node` 与 `--remote-components ejs:github`（见第 4 条）。加上后手动验证：取标题退出 0，媒体 Range 探测返回 206。

注意：yt-dlp 升级到最新（2026.10.26 已是 PyPI 最新）并不能解决这个问题，别在升级上浪费时间。

#### 问题 7：过期的 cookies 文件让格式消失

现象：配置、插件、POT 都对了，终端复测能看到格式 18，但 MyTube 里重试仍报 `Requested format is not available`，且两次重试的 stderr 逐字相同（不是会话运气）。

成因：后端每次启动 yt-dlp 都会额外附加 `--cookies ~/mytube/backend/data/cookies.txt`（`execute.ts` 的 `ensureCookiesFileIsNormalized`，文件存在且可解析时返回路径）。该文件是早先导出的，其中 YouTube 账号 cookies 已在浏览器里轮换失效；同时加载过期文件与新鲜浏览器 cookies 后，YouTube 只返回 storyboard 图片格式，没有任何可下载格式。对照实验证实：终端成功命令只加 `--cookies data/cookies.txt`，格式 18 立刻消失、只剩 sb0–sb3，并出现警告 `The provided YouTube account cookies are no longer valid. They have likely been rotated in the browser as a security measure.`。

解法：把 `data/cookies.txt` 移走备份（不要直接删），让后端只用 `--cookies-from-browser`。移走后探测立即通过、格式 18 恢复。教训：**不要留一份会过期的 cookies 文件给后端自动加载**；浏览器 cookies 由 yt-dlp 每次现读，不会过期。

#### 问题 8：live_chat 字幕触发 SSRF 校验

现象：探测通过、视频信息拿到、下载正式启动，但在 0% 卡住失败，stderr 只有一行：`MyTube SSRF protection: this media transport is not supported, use HTTP(S)/HLS/DASH`。

成因：被拦的不是视频，是字幕。后端下载参数含 `--write-subs --sub-langs all --convert-subs vtt`，`all` 会把 `live_chat` 也当作一种字幕语言（该视频有直播聊天回放）；yt-dlp 先下字幕，调用 `youtube_live_chat` 下载器取聊天回放；SSRF 校验（`backend/scripts/network-guard.py` 的 `checked_download`，白名单在第 118 行）按下载的 `protocol` 字段（`youtube_live_chat`）判定，而不是按 URL 的 https scheme，因此拒绝，整个进程退出 1，视频根本没开始下。视频本身（格式 18，protocol https）在白名单内，不会被拦。

解法：把 `youtube_live_chat` 与 `youtube_live_chat_replay` 加进 `network-guard.py` 第 118 行的 `supported` 元组（改前备份原文件；该 `.py` 每次 spawn 都会重新加载，改后下次下载即生效，无需重编、无需重启后端）。加完后重试一次通过，视频正常走 http 下载器下载。

注意：另一条路（把字幕语言从 `all` 改成 `all,-live_chat`）走不通——`subLangs` 不是设置项，是 `ytdlpConfig.ts` 第 260 行写死的默认值 `"all"`，且用户配置文本里的 subtitle 键会被 `extractUserConfigOptions` 主动剥离，改设置改不了它；要改它得改代码。放宽白名单是有意识的取舍：live_chat 下载实际走的是 youtube.com 的 https 请求。临时绕开是把设置里的 Download subtitles 开关关掉（不再下任何字幕），但会失去所有视频的字幕，不推荐长期用。

#### 问题 9：中继的另一端（远程 OpenList 45244）拒绝连接

现象：同步日志报 `ERROR: relay to remote OpenList is down`；中继 14544 的 curl 返回 000；VPS 上直连 45244 报 `Connection refused`。

成因：不是网络问题，是那台主机上的 OpenList 服务进程停了。主机本身是通的（从 VPS ping 过去 0.78ms，只有一跳；22、80、443 端口都开着），但 45244 端口上没有服务在监听。该服务时断时续（2026-11-12 02:00 与 12:48 两次停过），有专人监控、半小时内自动恢复。

解法：等它自动恢复（半小时内），或去那台机器上把 OpenList 重新启动。从 VPS、Box 都修不了它。恢复后中继无需重建（SSH 转发监听还在，目标一回来就通），下次定时同步照常跑。

### 6. 维护与重启清单（Box 休眠/重启后）

Box 没有 systemd，所有进程在休眠/重启后都会死，需按下面顺序手工重起（全部验证过）：

1. 唤醒 Box（界面 Offline 是正常的，唤醒后几秒恢复）。
2. MyTube 后端：`cd ~/mytube/backend && npm run start`（前台终端跑）。
3. POT 服务：`~/mytube/backend/bgutil-ytdlp-pot-provider/server` 里 `setsid nohup node build/main.js`，日志 `/tmp/pot-server.log`，等 `ping` 返回 200（约 27 秒）。
4. 反向隧道：`setsid ssh -i ~/.ssh/id_ed25519_mytube -o ServerAliveInterval=30 -o ExitOnForwardFailure=yes -N -R 0.0.0.0:55551:localhost:5551 -R 0.0.0.0:55244:localhost:5244 root@<VPS>`，日志 `/tmp/mytube-tunnel.log`（55244 这一路转发还在跑，但 Box 的 OpenList 不在用，见第 3 条）。
5. 同步中继：`setsid ssh -i ~/.ssh/id_ed25519_mytube -o ServerAliveInterval=30 -o ExitOnForwardFailure=yes -N -L 127.0.0.1:14544:<中继 OpenList 地址>:45244 root@<VPS>`，日志 `/tmp/mytube-relay.log`。
6. cron 守护进程：`sudo cron`（每 30 分钟跑同步脚本）。
7. OpenList：`~/openlist` 里 `setsid ./openlist server`。

验证：MyTube 根地址 200、POT `ping` 返回 2.0.2、隧道两端端口通、同步日志里没有报错。

### 7. 当前状态与验证（2026-11-12）

- 两个视频已下载并同步备份到天翼（`boxtianyi/mytube`），云端清单与本地逐字节核对一致：一个 485.52 MiB，一个 366.49 MiB，另有一个 42,321 字节的 `live_chat.json`（白名单修好后直播聊天回放也随视频一起下下来了）。
- 同步是 12:21 手动跑了一次（9 分 5 秒，852.044 MiB、3 个文件），12:30 的定时任务撞上 flock 锁后什么也没做（无重复上传）。
- 13:00 定时同步成功（中继 45244 修复后）：中继 14544 返回 200（0.27 秒），日志 `There was nothing to transfer`，Checks 3/3，用时 2.5 秒；当前没有同步在跑。

### 8. 注意事项

- **不要直接提交调试留下的本地改动**：Box 的仓库里有 14 个未提交修改（bgutil 升 2.0.2 的 13 个文件 + `network-guard.py` 白名单那一行）与 7 个未跟踪条目（两个备份目录、server/ 下新增的 `.gitattributes`、`deno.json`、`deno.lock`、`scripts/`、`network-guard.py.bak-20261112），它们与文档无关，推文档时只提交文档本身。
- OpenList 管理员的存储列表 API 会在 `addition` 字段里返回天翼的凭证 JSON（access_token、refresh_token、密码、用户名），**绝不要打印、记录它的值**。
- OpenList 的 admin token 只存在 Box 上的 `~/openlist-remote-token`（600 权限）里，同步脚本每次运行现读；不要写进配置、日志或文档。
- 从终端截图上抄 base64/指纹容易看错字符（1/I、0/O、8/B、g/q），密钥指纹必须用 `ssh-keygen -lf` 核对。
- MyTube 后端日志只存在跑后端的前台终端里（backend/ 下没有日志文件），查下载失败要去那个终端看。
