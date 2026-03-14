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
*编译器 wrapper: 上次项目的 `LangChain/build/ohos-cc.sh` / `ohos-cxx.sh` 可直接复用*
