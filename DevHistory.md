# OpenClaw on OpenHarmony 6.0 — 开发记录

## 2026-03-15 第一次讨论：可行性分析

### 目标

把 OpenClaw（TypeScript/Node.js AI Agent 框架，264k star）跑在 OpenHarmony 6.0（RK3568，aarch64）上。

GitHub: https://github.com/openclaw/openclaw

### 背景

之前已完成 LangChain + AgentScope（Python 生态）在 OH 上的移植（见 `/home/lmxxf/work/langchain-on-openharmony/DevHistory.md`）。这次是 Node.js 生态。

### 技术栈关系

```
Linux 内核（syscall）
    ↑
musl libc（OH 定制版，target triple: aarch64-unknown-linux-ohos）
    ↑
Node.js 22.16.0+（V8 + libuv + OpenSSL，C/C++）  ← 要交叉编译
    ↑
OpenClaw（TypeScript → JS，pnpm monorepo）
    ↑
native addon（.node 文件，逐个交叉编译）
```

### OH 工具链（已确认，上次项目验证过）

| 组件 | 路径 | 状态 |
|------|------|------|
| 交叉编译器 | `prebuilts/ohos-sdk/linux/20/native/llvm/bin/aarch64-unknown-linux-ohos-clang` | Clang 15 |
| musl sysroot | `prebuilts/ohos-sdk/linux/20/native/sysroot/` | libc.a/so, crt*.o 全套 |
| 宿主机 Python | `prebuilts/python/linux-x86/3.11.4/` | x86，编译时用 |

**OH target triple**: `aarch64-unknown-linux-ohos`

### OpenClaw 项目结构

| 项 | 值 |
|---|---|
| 版本 | 2026.3.14 |
| Node.js 要求 | ≥22.16.0（启动时强制检查） |
| 构建系统 | pnpm 10.23.0 monorepo + tsdown |
| 总依赖包 | ~1688 个（pnpm-lock.yaml） |
| 入口 | `openclaw.mjs` |
| 工作区 | `packages/`（2 个）、`extensions/`（36+ 插件）、`apps/`（移动端，不管） |

### 与 Python 移植的对比

| | Python（上次） | Node.js（这次） |
|---|---|---|
| 运行时 | CPython 3.11（C，50 万行） | Node.js 22（C++，V8 几百万行） |
| 核心引擎 | CPython 解释器 | V8 JIT 编译器 |
| 构建系统 | autoconf（./configure && make） | **GN + ninja**（V8 自带），Node.js 外层有 ./configure |
| 扩展机制 | .so（dlopen） | .node（也是 dlopen） |
| OHOS musl dlopen 问题 | 已踩过：.so 必须显式链 libpython | **同理：.node 必须显式链依赖** |
| 编译难度 | 中 | 高 |

### native addon 依赖分析

OpenClaw 有 11 个需要交叉编译的 native addon：

**必须编的：**

| 包 | 语言 | 作用 | 说明 |
|---|---|---|---|
| **sharp** | C++（libvips 绑定） | 图片处理 | 依赖链最深：libvips → libpng/libjpeg/libwebp/zlib |
| **@lydell/node-pty** | C/C++ | 终端/PTY 模拟 | 平台相关 syscall（ioctl/fork/exec） |
| **koffi** | C/C++ | FFI（外部函数接口） | 系统集成用 |
| **authenticate-pam** | C | PAM 认证 | 需确认 OH 有无 PAM |

**Rust/NAPI 类（按需）：**

| 包 | 语言 | 作用 | 说明 |
|---|---|---|---|
| **@napi-rs/canvas** | Rust/NAPI | Canvas 图形渲染 | 可选 peer dependency |
| **@matrix-org/matrix-sdk-crypto-nodejs** | Rust/NAPI | Matrix E2E 加密 | 只有用 Matrix 才需要 |

**可选/可绕过的：**

| 包 | 语言 | 作用 | 说明 |
|---|---|---|---|
| **node-llama-cpp** | C++（llama.cpp） | 本地 LLM 推理 | 可选 peer dependency |
| **protobufjs** | 可纯 JS | Protocol Buffers | 检查是否用 native 变体 |
| **esbuild** | Go | 构建工具 | 仅 build 时需要，不上板子 |

**纯 JS fallback 的：**

| 包 | 说明 |
|---|---|
| @discordjs/opus | 有 opusscript 纯 JS fallback |
| @whiskeysockets/baileys | 核心纯 JS，音频可选 |

### sharp 依赖链（最头疼的）

```
sharp
  ↓
libvips（C，图片处理核心）
  ↓
├── libpng
├── libjpeg-turbo
├── libwebp
├── libexif
├── zlib（已有，上次编过）
├── librsvg（可选，SVG）
└── glib（libvips 依赖 glib！这是个大坑）
```

### 交叉编译 Node.js 的关键点

1. **Node.js 官方支持交叉编译**：`./configure --dest-cpu=arm64 --cross-compiling`
2. **但不认 OHOS**：需要 patch `configure` 和 V8 的 GN 配置，让它认 `aarch64-unknown-linux-ohos`
3. **V8 构建系统**：GN + ninja，不是 autoconf，配置方式不同
4. **OpenSSL**：Node.js 22 内置 OpenSSL 3.x 源码（`deps/openssl/`），不需要外部 OpenSSL
5. **libuv**：Node.js 内置（`deps/uv/`），纯 C，平台适配层——可能需要 patch OHOS 特有的 syscall
6. **ICU**：Node.js 内置（`deps/icu-small/`），国际化支持，C/C++

### OHOS musl 已知问题（上次项目踩过）

| 问题 | 影响 | 解法 |
|---|---|---|
| dlopen 不继承进程符号 | .node 扩展必须显式链接依赖 | 编译时 `-l` 显式指定 |
| 动态链接器路径 `/system/lib/ld-musl-aarch64.so.1` | 不是标准 `/lib/` | 静态链接或 patchelf |
| `ld-musl-namespace` 命名空间隔离 | `/data/local/tmp` 走 acquiescence，不受限 | 不影响 |
| 部分 POSIX 函数缺失（如 `preadv2`、`gettext`） | 编译报错 | patch 掉或 `#undef` |
| 没有 CA 证书包 | HTTPS 不通 | 推 `cacert.pem` 上去 |

### 最小验证路径

1. **交叉编译 Node.js 22** → 得到能在 OH 上跑的 `node` 二进制
2. `node -e "console.log('hello')"` → 验证 V8 能跑
3. `node -e "require('https')"` → 验证 OpenSSL/TLS
4. `node -e "require('net').createServer()"` → 验证 libuv 网络
5. 装一个最小 npm 包 → 验证模块加载
6. 编 sharp → 验证 native addon 交叉编译链
7. `npx openclaw` → 最终目标

### 下一步

- [ ] 下载 Node.js 22 源码
- [ ] 研究 Node.js 交叉编译配置（`./configure` 参数 + V8 GN 配置）
- [ ] 交叉编译 Node.js 22 for `aarch64-unknown-linux-ohos`
- [ ] 板子上验证 `node` 二进制能跑
- [ ] 验证 HTTPS/TLS
- [ ] 交叉编译 native addon（sharp 优先）
- [ ] 装 OpenClaw 依赖，跑通

### 文件结构

```
/home/lmxxf/work/openclaw-openharmony/
├── sources/
│   └── openclaw/          # git clone --depth 1（原版源码）
├── build/                 # 编译产物（待创建）
├── scripts/               # 编译脚本（待创建）
└── DevHistory.md          # 本文件
```

*OH 工程路径: `/home/lmxxf/oh6/source`*
*上次项目: `/home/lmxxf/work/langchain-on-openharmony/`（Python 移植，可复用 OpenSSL/zlib 经验）*
*编译器 wrapper: `scripts/ohos-cc.sh` / `scripts/ohos-cxx.sh`*

---

## 2026-03-15 实战：Node.js 22 交叉编译

### 编译环境

| 组件 | 版本/路径 |
|------|-----------|
| Node.js 源码 | 22.22.1（`sources/node-v22.22.1/`） |
| OH Clang | 15.0.4（`prebuilts/clang/ohos/linux-x86_64/llvm/`） |
| CC wrapper | `scripts/ohos-cc.sh`（`--target=aarch64-unknown-linux-ohos --sysroot=...`） |
| CXX wrapper | `scripts/ohos-cxx.sh` |

### 关键发现

**Node.js 官方已支持 OpenHarmony 作为 `--dest-os`！** configure.py 第 49 行：`'openharmony'` 是 valid_os 之一。代码里大量 `OS=="openharmony"` 条件分支，不需要 hack。

### configure 命令

```bash
CC_host=gcc CXX_host=g++ \
CC=scripts/ohos-cc.sh \
CXX=scripts/ohos-cxx.sh \
AR=<llvm>/llvm-ar \
RANLIB=<llvm>/llvm-ranlib \
python3 configure.py \
  --dest-cpu=arm64 \
  --dest-os=openharmony \
  --cross-compiling \
  --prefix=build/node-ohos
```

### 踩坑记录

| 坑 | 原因 | 解法 |
|---|---|---|
| `auto` not allowed in function prototype | ncrypto.h 用了 C++20 `auto` 函数参数，但 common.gypi 设的 `-std=gnu++17` | 改 common.gypi 第 514 行：`gnu++17` → `gnu++20` |
| `operator<=>` 三路比较运算符 | ncrypto.cc 用了 C++20 `<=>` | 同上，改为 C++20 一并解决 |
| Clang 15 崩溃 `CannotYetSelect` 编译 zlib `crc32_simd.c` | zlib.gyp 里 `zlib_arm_crc32` target 只在 `clang==0`（非 clang）时加 `-march=armv8-a+aes+crc`，clang 编译时缺 CRC32 指令支持 | 改 zlib.gyp 第 72 行条件：`OS!="win" and clang==0` → `OS!="win"`（clang 也加 `-march`） |

### 编译产物

| 产物 | 大小 | 说明 |
|------|------|------|
| `out/Release/node` | 117MB（未 strip） | ELF 64-bit ARM aarch64，动态链接 |
| `build/node-ohos` | **93MB**（strip 后） | 部署用 |

### 动态依赖

```
libc++.so   — OH 系统自带
libc.so     — OH 系统自带（musl）
```

**V8、OpenSSL 3.x、libuv、zlib、brotli、zstd、sqlite、ICU、nghttp2 全部静态链入。** 只依赖 OH 系统库。

### interpreter 路径问题

二进制 interpreter 是 `/lib/ld-musl-aarch64.so.1`，但 OH 板子上动态链接器在 `/system/lib/ld-musl-aarch64.so.1`。

**解法选项：**
1. 板子上创建软链：`ln -s /system/lib/ld-musl-aarch64.so.1 /lib/ld-musl-aarch64.so.1`
2. patchelf 改 interpreter 路径
3. 直接用 `LD_LIBRARY_PATH` + 绝对路径调用（上次 Python 的经验）

### patch 记录

改了两个文件（相对于 Node.js 22.22.1 原版）：

1. **`common.gypi` 第 514 行**：`-std=gnu++17` → `-std=gnu++20`
2. **`deps/zlib/zlib.gyp` 第 72 行**：`'OS!="win" and clang==0'` → `'OS!="win"'`

### 板子验证通过 ✅

```
$ LD_LIBRARY_PATH=/system/lib64 /data/local/tmp/node /data/local/tmp/test_node.js

Platform: openharmony
Arch: arm64
Version: v22.22.1
V8: 12.4.254.21-node.35
OpenSSL: 3.5.5
crypto OK, random: 03c77e1c
net OK
https OK

=== Node.js on OpenHarmony: ALL OK ===
```

**`process.platform` = `openharmony`** ——Node.js 官方 OH 支持，不是 hack。

### 踩坑：libc++.so 路径

| 问题 | 原因 | 解法 |
|---|---|---|
| SIGBUS（Signal 7） | 从 SDK 推的 libc++.so 与板子 musl 不兼容 | 用板子系统自带的 `/system/lib64/libc++.so` |
| 上次 Python 二进制也 SIGBUS | 板子上 `/data/local/tmp/lib/libc++.so` 被本次推的坏文件覆盖 | 删掉，用系统 lib64 |
| hdc 第一次传输 MD5 不一致 | USB 线质量问题 | 换线，验证 MD5 |

**运行命令模板：**
```bash
LD_LIBRARY_PATH=/system/lib64 /data/local/tmp/node xxx.js
```

### 下一步

- [x] Node.js 22.22.1 交叉编译成功
- [x] 推板子验证 `node --version` ✅
- [x] 验证 crypto / net / https ✅
- [ ] 交叉编译 native addon（sharp 等）
- [ ] 部署 OpenClaw

### 文件结构

```
/home/lmxxf/work/openclaw-openharmony/
├── sources/
│   ├── openclaw/                    # git submodule（原版源码）
│   ├── node-v22.22.1/               # Node.js 源码（含 patch）
│   └── node-v22.22.1.tar.gz         # 源码包
├── build/
│   └── node-ohos                    # 93MB strip 后的 node 二进制
├── scripts/
│   ├── ohos-cc.sh                   # C 编译器 wrapper
│   └── ohos-cxx.sh                  # C++ 编译器 wrapper
└── DevHistory.md
```
