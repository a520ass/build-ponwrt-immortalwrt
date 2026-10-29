基于 Airoha AN7581 平台。构建源码使用 [1018ji/merge-immortalwrt-ponwrt](https://github.com/1018ji/merge-immortalwrt-ponwrt) 的 `master` 分支。

保留 PON 管理、基础 IPv6、BBR、LuCI 中文界面和 Bootstrap 主题，精简 USB/存储、无线、VPN、网络扩展及调试工具。

![PonWrt 状态页面预览，包含 CPU/PON 温度显示](assets/ponwrt-status-preview-20261023.png)

## 主要调整

| 项目 | 当前配置 |
| --- | --- |
| 镜像 | Initramfs 使用 ZSTD 压缩，保留 SquashFS 根文件系统 |
| 设备功能 | 关闭 USB、存储扩展和无线功能及其依赖 |
| 网络扩展 | 关闭 WireGuard、OpenVPN、透明代理、组播路由及其他高级网络扩展 |
| DNS/DHCP | 保留 DHCP、DHCPv6、nftset；关闭 DNSSEC、权威 DNS、conntrack 和 TFTP 扩展 |
| SSH | 保留 SSH 服务端、SCP/SFTP 和本地端口转发；关闭 SSH 客户端、agent/远程转发和 FIDO 支持 |
| 内核与工具 | 精简 Swap、namespace、部分 cgroup、内核调试和开发工具；关闭 `procd-ujail`，保留 `procd-seccomp` |
| LuCI | 使用 Bootstrap 主题，移除「插件」「诊断」菜单 |
| 状态信息 | 显示 CPU/PON 温度、总 CPU 与每核使用率、PPE 流表已绑定/总条目数；架构信息仅显示 CPU 架构和核心数 |
| 其他 | 最大连接数设为 `30000`，启用 BusyBox `usleep` |

`kmod-mppe`、`kmod-macvlan` 作为现有依赖保留。内核未保留完整容器支持；如需 Docker/LXC 或 `procd-ujail`，需恢复相应配置。完整编译选项见 [`.config`](.config)。

## 编译与下载

1. Fork 本仓库。
2. 在 **Actions → PonWrt Builder → Run workflow** 选择 SSH、Cache 选项并运行。
3. 成功后到 **Releases** 下载 `.itb` 固件及 `sha256sums`、`config.buildinfo`。失败时到当次运行的 **Artifacts** 查看 `build-logs`。

[构建工作流](.github/workflows/ponwrt-builder.yml) 同一时间只运行一个任务，后续任务排队。Releases 以北京时间命名，与 Actions 运行记录均保留最新 7 条。

### SSH

默认选择「否」。选择「是」后，从 **Setup SSH session** 步骤日志复制连接命令，后台编译会继续运行。仅允许触发任务的账号使用其 GitHub 公钥连接，需先在 **Settings → SSH and GPG keys** 配置公钥。源码目录为 `/workdir/openwrt`，调试完成后执行 `exit` 关闭会话。

### Cache

| 选项 | 缓存行为 | 用途 |
| --- | --- | --- |
| 使用缓存（默认） | 恢复并保存 | 日常编译 |
| 只读缓存 | 仅恢复，不保存 | 临时测试 |
| 重新编译 | 仅恢复下载缓存，重新编译并保存其他缓存 | 缓存损坏或需要重建 |
| 不使用缓存 | 不恢复、不保存 | 排查缓存问题 |

缓存分为下载包、工具链和 ccache 三类，每类保留最新一份。启用缓存保存时，编译失败后也会尝试保存 ccache，便于重试。可在 **Actions → Caches** 查看或删除缓存。外部触发（`repository_dispatch`）按「使用缓存」处理。

## DHCP 租约主机名

点击 DHCPv4 租约主机名旁的铅笔设置显示名称，按 MAC 关联，支持汉字、英文字母、数字、普通空格、`-`、`_`，最多 32 个字符；首尾空格自动去除，留空恢复原名。DHCPv6 租约按同一 MAC 复用该名称，不提供单独编辑；MAC 依次取自租约本身、DUID-LLT/DUID-LL、仅含单个 MAC 的静态租约（先匹配 `DUID%IAID` 再匹配 DUID）；无法识别 MAC 或未设置名称时，继续显示原主机名。不修改 DNS 或 DHCP 地址分配。

保存后先暂存，IPv4、IPv6 租约表同步显示新名字。在原生更改窗口中，「关闭」保留暂存，「恢复」撤销待应用修改，「保存并应用」持久保存；应用时可能同时处理其他页面的待应用修改。

工作流在更新 LuCI feed 后、安装 feed 前自动应用 [补丁](patches/feeds/luci)。LuCI 跟随上游更新，补丁冲突时会停止构建，需适配后重试。

本地构建时，在更新 LuCI feed 后、安装 feed 前运行：

```sh
./scripts/apply-feed-patches.sh /path/to/openwrt/feeds/luci
```

已完整应用的补丁会跳过，冲突或部分应用时停止。已有编译产物时，先清理并重新编译 `luci-base`（含中文翻译）和 `luci-mod-status`。
