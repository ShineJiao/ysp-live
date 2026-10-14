# ysp-live

央视频全频道直播代理（64 路，含 4K / 8K、高码率央视、省级卫视与 7 天回看），基于上游 ysp-live v9.4 修改，
由 GitHub Actions 构建 `linux/amd64` + `linux/arm64` 镜像发布到 GHCR，适合在软路由（iStoreOS / OpenWrt）、NAS 上用 Docker 运行。

上游原版说明见 [README-DOCKER.md](README-DOCKER.md)（保持原样，便于和后续上游版本对比）。

## 与上游的区别

`ysp-live.py`、`ysp-engine.js` 与上游 v9.4 完全一致，本仓库只提供：

- GitHub Actions 构建的 `linux/amd64` + `linux/arm64` 镜像，设备上无需编译
- 适合软路由的 `docker-compose.yml`：host 网络模式、高码率并发上限 `YSP_LINKS_PER_DEVICE=4`、日志大小上限

背景：央视频对获取高码率链接有风控，被拒时 `live/v1/01` 返回 HTTP 400，或 VDN 返回「内部错误」。目前观察到：

- **按设备累计请求次数**：一个设备约 8～10 次成功请求后被拒，同一频道重复获取也计数（例如 3 个 4K 频道各自定时重新获取，几分钟内就会用完）。换新设备后通常立即恢复。
- **IP 也受影响**：同一出口 IP 下注册和请求过于频繁时，换设备也可能无法恢复，需要降低请求频率或等待一段时间。

上游 v9.4 去掉了 v9.0 的热备设备和按配额主动换设备，改为更保守的策略，以减少请求次数：

- **单设备**：26 路高码率频道共用 1 台设备。`YSP_LINKS_PER_DEVICE` 现在是**同时**获取高码率的频道数上限（上游默认 6，compose 中设为 4），
  超 15 秒没有请求的频道释放名额；名额满时新频道直接走 1080p，不再换设备。
- **只在 HTTP 400 时换设备**：错误信息含 `HTTP 400` 时废弃当前设备、注册新设备并重试一次，5 分钟内最多一次、每天最多 5 次。
  VDN「内部错误」等其他风控错误只会重建会话，仍用同一台设备，期间回退 1080p。
- **不再启动预热**：启动时只注册设备，3 个 4K 频道也是有人打开时才获取。
- **换台防抖**：非 4K 频道打开时先出 1080p，停留 10 秒后才在后台获取高码率，播放器下次刷新列表时自动切换（`YSP_ZAP_DEBOUNCE`）。

注意事项（上游行为，未修改）：

- 4K 链接缓存仍只有 60 秒，持续观看时每分钟重新获取一次。按上面观察到的设备次数限制，连续看 4K 约十分钟后可能被拒；
  若被拒的不是 HTTP 400，或 5 分钟换设备间隔未到，会回退 1080p 一段时间。
- 换设备次数只在内存里计数，重启后清零；重启会复用 `./data` 中的旧设备。重启后若很快出现风控，可删除 `./data/device-state-rs.json` 再启动。
- Node.js 兜底引擎监听 `0.0.0.0:8787`，它的 `/segment?url=` 和主程序的 `/engine_proxy?url=` 都能代理任意地址。只在可信的局域网中使用，不要把 8767 / 8787 端口映射到外网。

## 线路与兜底

频道按以下顺序获取，前一级失败自动落到下一级：

1. 设备协议 4K / 高码率（默认 26 个频道，可用 `YSP_ONLY_4K_DEVICE` 改为只走 3 个真 4K 频道）
2. 1080p JCE 解析
3. bkliveinfo 备用线路
4. 网页版兜底引擎（容器内的 Node.js 进程 `ysp-engine.js`，只在前三级都拿不到播放列表时使用）

## 部署

> 必须运行在**家庭宽带**下。设备注册会被机房 IP 拒绝，不要部署到 VPS / 云服务器。

只需要一个 `docker-compose.yml`，镜像已经构建好，无需在设备上编译。

```bash
mkdir -p /opt/ysp-live-docker && cd /opt/ysp-live-docker
curl -fsSL -o docker-compose.yml https://raw.githubusercontent.com/evanzme/ysp-live/main/docker-compose.yml
docker compose pull && docker compose up -d
```

查看日志，出现 `设备身份就绪，高码率并发池已激活` 即启动完成：

```bash
docker logs -f ysp-live
```

之后日志会停止滚动，直到有人打开频道，这是正常的。

### iStoreOS / OpenWrt 说明

- 通过 SSH（`ssh root@路由器IP`）或后台「终端（TTYD）」执行上面的命令。
- 系统自带 `curl`，不需要安装 `git`。
- 若提示 `docker: 'compose' is not a docker command`，把命令里的 `docker compose` 换成 `docker-compose`。
- 若无法访问 `raw.githubusercontent.com`，可在电脑上下载 `docker-compose.yml` 后用 `scp -O` 传到路由器。

## 使用

在播放器（APTV、TiviMate、OTT Navigator 等）中添加订阅：

```
http://<路由器局域网IP>:8767/all.m3u
```

注意填路由器的局域网 IP，不要用 `localhost` / `127.0.0.1`（那会指向播放设备自己）。

| 地址 | 内容 |
|---|---|
| `/` | 服务状态与入口（纯文本） |
| `/all.m3u` | 64 路聚合订阅，含 EPG 与 7 天回看 |
| `/<频道>.m3u8` | 单个频道，如 `/cctv4k.m3u8`、`/cctv13.m3u8` |
| `/all.m3u?group=卫视` | 只订阅某个分组（`央视` / `卫视`） |
| `/health` | JSON 健康信息：运行时间、设备会话、分片缓存命中率 |
| `/diag` | 诊断信息：设备会话状态、分片缓存与各频道刷新情况 |

### 播放建议

- **关闭 APTV 延迟检测**。订阅已带 `#EXT-X-APTV-LATENCY: FALSE`，APTV 会自动关闭；其他播放器若有类似功能也建议关掉。延迟检测会一次性请求全部频道，首次解析需要排队，后面的频道容易超时，也会白白消耗设备的请求次数。
- **非 4K 频道打开后先播 1080p**，停留约 10 秒后自动切到高码率，中间可能有一次短暂卡顿。不想等可设 `YSP_ZAP_DEBOUNCE=0`，代价是快速换台时每个频道都会请求高码率链接。
- **首次打开 4K 频道约需 3 秒**；需要换设备时可能需要十几秒。
- **4K / 8K 频道码率很高**（CCTV-4K 约 36 Mbps），播放设备网络需稳定在 50 Mbps 以上，并建议开启播放器的硬件解码。
- 只有 CCTV-4K、CCTV-8K、CCTV-16 4K 是真 4K / 8K。其余央视频道的「高码率」是约 12 Mbps 的 1080i50（H.264，SDR），比普通 1080p 源清晰，但分辨率仍是 1080。

## 配置

在 `docker-compose.yml` 的 `environment` 中设置：

| 变量 | 默认值 | 说明 |
|---|---|---|
| `YSP_LINKS_PER_DEVICE` | `4`（compose 中设置；不设置为上游默认 6） | 同时获取高码率的频道数上限，超 15 秒未请求的频道释放名额，名额满时新频道走 1080p；`0` 为不限制 |
| `YSP_ZAP_DEBOUNCE` | `10` | 换台防抖秒数：非 4K 频道先出 1080p，停留超过该秒数才获取高码率；`0` 为关闭 |
| `YSP_ONLY_4K_DEVICE` | 关闭 | 设为 `1` 时只有 CCTV-4K、CCTV-8K、CCTV-16 4K 走设备协议，其余走 1080p |
| `TZ` | `Asia/Shanghai` | 日志时区 |

默认 26 路高码率频道：`cctv1`～`cctv5p`、`cctv7`～`cctv17`、`cctv4k`、`cctv8k`、`cctv164k`、`cgtn`、`cgtnfr`、`cgtnru`、`cgtnar`、`cgtnes`、`cgtndoc`。

修改后执行 `docker compose up -d` 生效。

## 更新

```bash
cd /opt/ysp-live-docker && docker compose pull && docker compose up -d
```

`./data` 中只保存当前设备身份，删除不影响使用，下次启动会注册新设备。

从 v9.0 升级时，`./data` 中的设备身份可以继续使用，无需删除。

## 日志说明

| 日志 | 含义 |
|---|---|
| `[xxx] 换台防抖: 先出 1080p, 停留后自动升码率` | 刚打开的非 4K 频道先播 1080p |
| `[xxx] 高码率后台预取完成, 继续停留将自动升级` | 停留超过防抖时间，已拿到高码率链接 |
| `[设备] 检测到 400 (...)，旧身份已废弃，换新设备身份 (...今日第 N 次)` | 设备被拒，已换新设备并重试 |
| `[设备] 24小时内已自动换新身份 5 次，停止自动更换` | 当天换设备次数用完，高码率请求会一直回退 1080p，直到第二天或重启容器 |
| `[xxx] 高码率播放暂时受限 (...)，平滑回退 1080p (冷却 15s)` | 本次未拿到高码率链接，临时用 1080p 播放，冷却后重试。括号内为 `device link quota reached` 表示并发名额已满 |
| `[xxx] 1080p 地址已过期，已重新拉取`、`空闲超时，已停止后台拉取` | 1080p 线路的正常刷新与闲置回收（剧场频道显示为 720p） |

## 排查

1. 浏览器打开 `http://<路由器IP>:8767/all.m3u`，能看到频道列表说明服务正常。
2. 打开 `/diag`：显示设备是否有效、高码率名额占用（如 `占用=2/4`）、各频道刷新情况，末尾附最近 30 条运行日志。
3. 统计风控与回退：`docker logs ysp-live 2>&1 | grep -E "\[设备\]|平滑回退"`。换设备重试后仍回退 1080p，多半是出口 IP 被风控，可减少高码率频道（如 `YSP_ONLY_4K_DEVICE=1`）或稍后再试。
4. 若路由器开了 OpenClash / PassWall 等代理，确认路由器本机出口是国内 IP：`curl -s https://myip.ipip.net`。

## 构建与同步上游

推送到 `main` 分支或打 `v*` 标签时，[.github/workflows/docker.yml](.github/workflows/docker.yml) 会构建多架构镜像并推送到
`ghcr.io/evanzme/ysp-live`（`latest`、`sha-<提交>`、标签名）。

上游发布新版本时：

1. 用上游的 `ysp-live.py`、`ysp-engine.js`、`README.md`（存为 `README-DOCKER.md`）覆盖本仓库文件，提交「Update to upstream vX.Y」。`Dockerfile` 只参考上游的依赖变化，不直接覆盖。
2. 确认 `python3 -m py_compile ysp-live.py` 通过。
3. 推送后等 Actions 完成，在设备上执行更新命令。
