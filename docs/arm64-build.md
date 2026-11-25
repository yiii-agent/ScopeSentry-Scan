# ScopeSentry-Scan arm64 扫描节点构建

官方 `autumn27/scopesentry-scan` 镜像仅支持 amd64,本文档记录扫描端 arm64 的编译与镜像构建。
完整流程(含服务端、compose 编排、部署加固)见 ScopeSentry 仓库的 `docs/arm64-build.md`。

## 快速开始

```bash
# 本脚本需在 ScopeSentry 仓库的编排下使用(编译二进制 + 组装构建上下文)
cd ../ScopeSentry && SERVER_VER=v1.9.4 SCAN_VER=v1.9.1 ./scripts/build-arm64.sh
```

单独编译扫描端二进制(在 arm64 宿主机上):

```bash
docker run --rm -v "$(pwd)":/src -w /src -e CGO_ENABLED=0 golang:1.24 sh -c \
  "go build -trimpath -ldflags '-s -w' \
   -o dist/ScopeSentry-Scan_linux_arm64_v1/ScopeSentry-Scan ./cmd/ScopeSentry/main.go"
```

## 关键说明

### 1. 必须单文件编译
`cmd/ScopeSentry/` 下存在误提交的示例文件(`restructure_cmd.go`、`test_yaegi.go`),
与 `main.go` 同包重复声明 `func main`,整包编译报 `main redeclared`。
上游 goreleaser 按 `main.go` 单文件构建,照做即可,源码无需改动。

### 2. 4 个内置工具必须替换为 arm64
代码按 `/apps/ext/<工具>/<工具>` 的固定路径调用(`internal/global` 的 `ExtDir` 约定),
amd64 二进制在 arm64 上无法运行:

| 工具 | arm64 来源 |
|---|---|
| katana v1.8.0 | projectdiscovery/katana 官方 release(`katana_1.8.0_linux_arm64.zip`) |
| rad 1.0 | chaitin/rad 官方 release(`rad_linux_arm64.zip`) |
| rustscan 2.4.1 | RustScan/RustScan 官方 release(`aarch64-linux-rustscan.zip`) |
| ksubdomain | **源码编译**:`Autumn-27/ksubdomain`(旧版 wgpsec v1.x fork) |

### 3. ksubdomain 不能用 knownsec v0.7
- 本仓库 `modules/subdomainscan/ksubdomain/ksubdomain.go` 的自检逻辑按
  旧版 wgpsec v1.x 命令行编写(`v` 子命令)。
- knownsec v0.7 为重写版(`-verify` flag 语法),不兼容,自检报 `run error`。
- 旧版源码保存在作者 fork **Autumn-27/ksubdomain**,编译需
  `CGO_ENABLED=1` + `libpcap-dev`:

```bash
git clone --depth 1 https://github.com/Autumn-27/ksubdomain
docker run --rm -v "$(pwd)":/src -w /src golang:1.24-bookworm sh -c \
  "apt-get update -qq && apt-get install -y -qq libpcap-dev && \
   CGO_ENABLED=1 go build -trimpath -ldflags '-s -w' -o ksubdomain-arm64 ./cmd/ksubdomain"
```

### 4. go-rod 浏览器(arm64 无在线下载源,需两处配合)
go-rod(httpx 截图/指纹、katana)默认在线下载 chromium,但 arm64 无官方下载源
(playwright CDN 返回 400)。处理分两层:

1. **rod 缓存路径**(镜像构建,见 ScopeSentry 仓库 `scan-build/dockerfile`):
   rod(v0.116.2)只认缓存路径 `~/.cache/rod/browser/chromium-1124/chrome`
   (arm64 对应 playwright revision 1124,二进制名必须是 `chrome`),
   构建时把 apt 安装的 chromium 链接到该路径(兼容旧版目录 chromium-1321438);
2. **ROD_BROWSER_BIN**(部署环境变量):katana 的 rod 启动器通过
   `ROD_BROWSER_BIN=/usr/bin/chromium` 直接使用镜像内 chromium。

### 5. HttpxClose 空指针导致任务结束崩溃(本 fork 已修)
`pkg/utils/httpxrun.go` 的 `HttpxClose()` 未对 `HttpxRun` 判空:当 httpx 初始化
失败(如 arm64 浏览器缺失)时,每次任务结束都会触发 nil pointer panic,
扫描进程被容器运行时反复拉起。本 fork 已加判空:

```go
func HttpxClose() {
	if HttpxRun != nil {
		HttpxRun.Close()
	}
}
```

## 实测结果

- 镜像:`scopesentry-scan-arm64:1.9.1`(基于 python:3.7-slim + chromium + default-jdk)
- 节点注册:`Register Success:node - version 1.9.1`
- ksubdomain 自检通过(`baidu.com => 110.242.74.102` 等 DNS 解析输出正常)
- katana / rad / rustscan 均 arm64 aarch64 二进制,`file` 校验通过
