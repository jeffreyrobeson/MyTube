# MyTube 部署汇总

> 面向想自托管 MyTube 的实践记录：两套实例、一套云端存储打通方案，以及踩过的坑。
> 本文档不含任何服务器地址、账号、令牌等个人信息，可直接公开。

## 一、项目简介

MyTube（https://github.com/jeffreyrobeson/MyTube）是一个自托管的视频库应用：

- 下载并收藏网络视频（主要为 YouTube），按作者、合集、标签管理，内置播放器；
- 支持订阅频道自动下载、字幕下载、缩略图与作者头像；
- 后端为 Node.js 服务 + SQLite，内置 yt-dlp 管理（自动下载/升级 yt-dlp 发行版）；
- 前端为 SPA，后端可同源 serving 前端；
- 设置存入数据库 settings 表（值多为 JSON 编码，未保存过的键不存在，用默认值）。

本文基于 **v1.11.15** 实践（上游 master 当时为 1.11.14，v1.11.15 取自发行版提交）。

**没有的功能（不要误以为有）：** 无转码、无 HLS/自适应播放、无单次下载的质量选择框、无播放器内切换清晰度。清晰度只由全局「首选视频分辨率」决定。

## 二、两套实例对照

| | 实例 A：VPS（主力） | 实例 B：助理 VM（第二实例） |
|---|---|---|
| 运行方式 | Docker Compose | 源码构建、Node 直跑（环境无 Docker） |
| 前端 / 后端端口 | 5556 / 5551 | 后端 5551（同源 serving 前端） |
| 视频主存储 | 云端（见第三节） | 本地磁盘 + 单向同步备份到云端（见第四节） |
| 公开访问 | 服务器端口直出 | 经 VPS 的 SSH 反向隧道公开 |
| 登录 | 已开启（401 保护 API） | 已开启（401 保护 API） |
| yt-dlp cookies | 使用登录 cookies（该出口 IP 未被标记） | 不用登录 cookies（见第五节） |

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

补丁（针对 `ytdlpVideo.ts` 的取信息调用）：把用户配置里的 extractorArgs 与 pot-provider 参数合并后传给取信息调用。改动后需重新编译后端并重启服务。注意：**这是偏离上游的本地补丁，上游升级后会被覆盖**，升级后需重新评估是否还需打补丁。

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

## 十、当前状态（2026-10-26）

- 两实例均运行中，登录保护均已开启；
- VPS 实例：3 个视频已全部落盘天翼；
- VM 实例：本地视频与天翼备份同步中，新下载按 30 分钟周期自动备份（本地删除不影响云端）；
- VM 实例已应用取信息 android 补丁，望春风歌单视频已验证可下载（360p mp4 单文件）；
- 保活（30 分钟）与每日体检均已启用。
