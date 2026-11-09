# Passwall 专用精简构建（Xray 26.11.9）

`193-optional-passwall-lite.patch` 增加 `xray_lite` 构建标签。
默认仍编译完整 Xray；在 menuconfig 的 xray-core 选项下开启
`Build minimal VLESS/REALITY client for Passwall`，对应：

```text
CONFIG_PACKAGE_xray-core=y
CONFIG_PACKAGE_xray-core_INCLUDE_LITE=y
```

修改后重新编译 xray-core 即可，包安装路径仍是 `/usr/bin/xray`。
无需改 Passwall 的配置生成代码或启动脚本。这里验证的是用户提供的配置组合；
切换到被裁掉的协议/功能时，须使用完整版。

## 保留范围

- SOCKS5 TCP/UDP 入站，保留同端口 HTTP 回退；tunnel/dokodemo-door 入站。
- TCP REDIRECT、UDP TPROXY、获取原始目标地址、SO_MARK=255，以及原有 sockopt。
- VLESS 客户端及 VLESS 反向代理、RAW/TCP、TLS、REALITY、Chrome 指纹、xtls-rprx-vision、UDP/XUDP。
- freedom/direct、blackhole/block；原有路由规则、日志、policy、环境变量和版本约束。
- 原有 JSON 配置加载/合并、`run -c`、`run -test -c`、`run -dump`、`version`。
- ARM64 的 190/191/192 EIP93 补丁及运行时检测、硬件失败后的软件回退。

不注册其他代理协议或传输（VMess、Trojan、Shadowsocks、WireGuard、Hysteria、
MASQUE、gRPC、XHTTP、WebSocket 等），不提供 VLESS 服务端、API、metrics、
自定义内置 DNS、FakeDNS、观测配置、Lua 路由脚本、geodata 自动更新及额外 CLI 子命令。
按已有裁剪范围排除 YAML/TOML、公共 Lua 绑定、gRPC 凭据适配和 TLS ECH DNS 查询。
SOCKS HTTP 回退与 VLESS 反向代理使用完整上游实现。后续进一步删除功能前须先确认。
系统 DNS 解析仍可用，`env.XRAY_LOCATION_ASSET` 和常规 geodata 路由规则仍保留。
已知被删除的配置项会显式报错；未改写上游的数据转发、Vision 或加密算法。

## 实测体积

2026-11-09，源码包 SHA256：
`d5a47bed1b85e1cdfe3cb2f88a02e2db7144f0f8bee541d20b84fafa8fdf6c8d`。

完整版和精简版使用相同源码、Go 1.27.2、EIP93 overlay 和下列编译参数，
仅后者增加 `-tags=xray_lite`。未使用 UPX。

| Linux ARM64 静态 ELF | 字节 | MiB |
| --- | ---: | ---: |
| 完整版 | 30,081,148 | 28.69 |
| 当前精简版（r3，保留 HTTP 回退与 VLESS 反向代理） | 15,532,156 | 14.81 |
| 相比完整版减少 | 14,548,992 | 13.88（48.37%） |

`go list -deps -tags=xray_lite ./main` 保留 `proxy/http` 和 `app/reverse`，
排除了上述解析器、Lua、gRPC 和 miekg/dns。`go.mod` 仍保留完整版需要的模块条目，
模块声明不代表代码被链接。

当前 ELF 的主要组成（`readelf -SW`）：

| 内容 | MiB |
| --- | ---: |
| 机器码 `.text`，包括 Go runtime、TLS/uTLS/REALITY、Xray 等 | 6.26 |
| Go 函数、文件、PC 定位信息 `.gopclntab` | 5.51 |
| Go 类型和反射信息 `.go.type` | 1.56 |
| 其余常量、数据、对齐及 ELF 结构 | 1.49 |

函数表和类型信息随保留代码增减，并非所有 Go 程序都固定需要这么大；标准 `-s -w`
已经应用，不能再简单删除这些运行时结构。
这是一份已验证的裁剪结果，不是理论最小值。保留完整的 Go、TLS/uTLS、REALITY
和 Xray 转发机制需要一定体积；进一步删除通用机制需要额外实现和回归验证。

ARM64 设备实际内存和吞吐尚未实测。二进制体积变化不能直接换算为运行内存变化。

## 验证记录

- 四个补丁依次在哈希匹配的干净上游源码包上 `patch -p1 --fuzz=0` 成功。
- 从重新打补丁的干净源码编译，交付 ARM64 文件与开发树构建逐字节相同。
- `file` / `readelf` 确认 ARM aarch64、静态链接、已 strip，无 ELF interpreter。
- 用户配置的脱敏样本在 `infra/conf/testdata/lite-passwall.json`。
  完整和精简版生成的 core protobuf 配置逐字节相同（1213 字节），SHA256：
  `da8cfb6dcc8e546db4990404d29e64638272798378b8ba98c5582b18a73ca010`。
- 精简 `infra/conf` 测试通过，覆盖原始目标地址、REDIRECT/TPROXY、mark、UseIP、
  REALITY、环境变量、被裁功能拒绝以及公开 VLESS 无加密检查。
- 完整版配置回归、YAML/TOML 加载回归、精简版网络/IP 路由选定用例通过。
  精简版额外测试覆盖格式拒绝、ECH 拒绝。
  未运行依赖外部 GeoSite/GeoIP 数据文件的用例。
- 本机精简客户端与完整版服务端互通：REALITY/Vision 下 1 MiB TCP 随机数据
  SHA256 一致、1 MiB TLS 1.3 应用流量一致、UDP/XUDP 的 1/512/1400/4096 字节
  回显一致；TCP REDIRECT 和 UDP TPROXY 监听成功。
- 交付的 ARM64 二进制在 QEMU 7.2 中运行 `version` 和原配置的 `run -test`
  成功，且同样通过上述 TCP、TLS 1.3 和 UDP/XUDP 互通测试（服务端仍为本机完整版）。
- ARM64/QEMU 实测 SOCKS 同端口 HTTP CONNECT 回退经 REALITY/Vision 转发 64 KiB 数据成功。
- ARM64/QEMU 实测完整版服务端经 VLESS 反向隧道，到精简客户端 direct 出站的 64 KiB 回显成功。
- SOCKS server.go 与 VLESS outbound.go 在打补丁后与上游逐字节相同。
- 新开关 `CONFIG_PACKAGE_xray-core_INCLUDE_LITE` 通过 Kconfig 解析、启用/禁用构建标签检查；
  `PKG_CONFIG_DEPENDS` 同步使用新名称。
- 没有修改本机防火墙；真实 Passwall 防火墙重定向和 AN7581 EIP93 硬件/IRQ、
  设备吞吐及负载内存仍须上机验证，不能从本机测试推断。

测试命令（在打补丁后的源码目录）：

```sh
go test -tags=xray_lite -short ./infra/conf ./infra/conf/serial ./proxy/socks ./proxy/vless/outbound
go test -tags=xray_lite -short ./app/router \
  -run 'Test(RoutingRule|LocalOSRule|SimpleRouter|IPOnDemand|IPIfNonMatch)'
python3 tools/lite/smoke.py /path/to/xray-lite-amd64 /path/to/xray-full-amd64
# 可选第四个参数为 qemu-aarch64-static，用来执行 ARM64 精简客户端。
```

`smoke.py` 用 Go 编译本地 TLS 1.3 测试端点，可用 `GO=/path/to/go` 指定编译器。
它创建临时密钥和配置，使用本地端口，无需真实服务器账号。

## 独立交叉编译

使用 Go 1.27.2，源码先按序应用本目录全部补丁。
Go toolchain 若位于 GOMODCACHE 下，先复制到其外部目录，Go 禁止 overlay 替换
GOMODCACHE 下的文件。无需修改共享 GOROOT。

```sh
python3 tools/eip93/overlay.py --go go --out "$PWD/.eip93-overlay"
CGO_ENABLED=0 GOOS=linux GOARCH=arm64 GOARM64=v8.0 \
  go build -buildvcs=false -trimpath -tags=xray_lite \
  -overlay="$PWD/.eip93-overlay/overlay.json" \
  -ldflags='-s -w -buildid= -X github.com/xtls/xray-core/core.build=OpenWrt -X github.com/xtls/xray-core/core.version=26.11.9' \
  -o xray-linux-arm64 ./main
```

去掉 `-tags=xray_lite` 即为对比的完整版。
交付文件位于工作区 `bin/xray-lite/26.11.9-r3/`，含原始静态二进制、gzip 和 SHA256SUMS。
在设备上先运行 `./xray-linux-arm64 version`，再用真实配置
`./xray-linux-arm64 run -test -c /tmp/etc/passwall/实际配置.json` 检查。
确认后在 Passwall 的 Xray 可执行文件路径中选择它，或安装至 `/usr/bin/xray`。
