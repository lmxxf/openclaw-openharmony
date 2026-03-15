# OpenClaw on OpenHarmony 6.0 (RK3568, aarch64)

在 OpenHarmony 开发板上跑 Node.js 22 + OpenClaw（264k star AI Agent 框架）+ DeepSeek 大模型对话。全链路已验证打通。

```
OH RK3568 → Node.js 22.22.1 (V8 12.4 + OpenSSL 3.5.5)
    → OpenClaw 2026.3.14 Gateway
    → Control UI (网页聊天界面)
    → DeepSeek API (openai-completions)
    → AI 对话 ✅
```

## 快速部署（已有编译产物）

仓库里有两个编译产物，直接推板子就能用，不需要重新编译：

| 文件 | 大小 | 内容 |
|:---|:---|:---|
| `build/node-ohos` | 93MB | Node.js 22.22.1 交叉编译二进制 |
| `build/openclaw-deploy.tar.gz` | 67MB | OpenClaw 构建产物 + Control UI + 运行时依赖（精简后） |

### 前提

- 宿主机能通过 `hdc` 连接到 RK3568 开发板
- 板子上有 `/data/local/tmp/`（14G+ 可用空间）

### 步骤

**1. 推 node 二进制到板子**

```bash
hdc file send build/node-ohos /data/local/tmp/node
hdc shell 'chmod +x /data/local/tmp/node'
```

**2. 推 OpenClaw 部署包到板子**

```bash
hdc file send build/openclaw-deploy.tar.gz /data/local/tmp/openclaw-deploy.tar.gz
hdc shell 'mkdir -p /data/local/tmp/openclaw && cd /data/local/tmp/openclaw && tar xzf /data/local/tmp/openclaw-deploy.tar.gz && rm /data/local/tmp/openclaw-deploy.tar.gz'
```

**3. 配置 DeepSeek API**

```bash
# 设模型
hdc shell 'HOME=/data/local/tmp LD_LIBRARY_PATH=/system/lib64 /data/local/tmp/node /data/local/tmp/openclaw/openclaw.mjs config set gateway.mode local'
hdc shell 'HOME=/data/local/tmp LD_LIBRARY_PATH=/system/lib64 /data/local/tmp/node /data/local/tmp/openclaw/openclaw.mjs config set gateway.controlUi.dangerouslyAllowHostHeaderOriginFallback true'
hdc shell 'HOME=/data/local/tmp LD_LIBRARY_PATH=/system/lib64 /data/local/tmp/node /data/local/tmp/openclaw/openclaw.mjs models set custom/deepseek-chat'
```

然后手动写 config（`hdc shell` 进板子编辑 `/data/local/tmp/.openclaw/openclaw.json`），确保 `models.providers` 部分如下：

```json
{
  "models": {
    "providers": {
      "custom": {
        "baseUrl": "https://api.deepseek.com/v1",
        "apiKey": "你的 DeepSeek API Key",
        "models": [
          {
            "id": "deepseek-chat",
            "name": "DeepSeek Chat",
            "api": "openai-completions",
            "contextWindow": 65536,
            "maxTokens": 8192
          }
        ]
      }
    }
  }
}
```

> **关键**：`api` 必须设为 `"openai-completions"`。OpenClaw 默认用 OpenAI Responses API，DeepSeek 不支持，会返回 404。

**6. 启动 Gateway**

```bash
hdc shell 'HOME=/data/local/tmp LD_LIBRARY_PATH=/system/lib64 /data/local/tmp/node /data/local/tmp/openclaw/openclaw.mjs gateway run --bind lan --port 18800 --force'
```

**7. 端口转发 + 打开网页**

```bash
hdc fport tcp:18800 tcp:18800
```

浏览器打开（token 从 `~/.openclaw/openclaw.json` 的 `gateway.auth.token` 字段获取）：

```
http://localhost:18800/#token=你的gateway-token
```

> 必须通过 `localhost` 访问——浏览器要求 secure context（HTTPS 或 localhost）才能使用 Control UI 的设备标识功能。

**8. 验证**

```bash
# CLI 验证
hdc shell 'LD_LIBRARY_PATH=/system/lib64 /data/local/tmp/node /data/local/tmp/openclaw/openclaw.mjs --version'
# 输出：OpenClaw 2026.3.14 (8db6fcc)

# 模型验证
hdc shell 'HOME=/data/local/tmp LD_LIBRARY_PATH=/system/lib64 /data/local/tmp/node /data/local/tmp/openclaw/openclaw.mjs models list'
# 输出：custom/deepseek-chat  text  64k  no  yes  default

# 网页验证：在 Control UI 发消息，DeepSeek 回复
```

### 板子上的目录结构

```
/data/local/tmp/
├── node                           # 93MB Node.js 22.22.1
├── openclaw/
│   ├── openclaw.mjs               # 入口
│   ├── package.json
│   ├── dist/                      # 66MB 构建产物（JS）
│   │   └── control-ui/            # Gateway Web 界面
│   ├── docs/reference/templates/  # Agent 模板文件
│   └── node_modules/              # 378MB 运行时依赖（精简后）
└── .openclaw/
    └── openclaw.json              # 配置文件
```

---

## 从头编译 Node.js

适用场景：没有编译产物，或想改 Node.js 版本。

### 前提

- OpenHarmony 源码树（需要其中的 Clang 15 工具链和 musl sysroot）
- Linux 宿主机（WSL2 / Ubuntu）
- Python 3.8+（Node.js 的 configure 脚本是 Python 写的）
- 宿主机 gcc/g++（编译 host 工具用）

### 环境变量

```bash
# 按你的实际路径改
OH_SRC=/home/lmxxf/oh6/source
OH_CLANG=$OH_SRC/prebuilts/clang/ohos/linux-x86_64/llvm/bin
OH_SYSROOT=$OH_SRC/prebuilts/ohos-sdk/linux/20/native/sysroot
PROJ=/home/lmxxf/work/openclaw-openharmony
```

### Step 0: 编译器 Wrapper

```bash
mkdir -p $PROJ/scripts

cat > $PROJ/scripts/ohos-cc.sh << EOF
#!/bin/bash
exec $OH_CLANG/clang --target=aarch64-unknown-linux-ohos --sysroot=$OH_SYSROOT "\$@"
EOF

cat > $PROJ/scripts/ohos-cxx.sh << EOF
#!/bin/bash
exec $OH_CLANG/clang++ --target=aarch64-unknown-linux-ohos --sysroot=$OH_SYSROOT "\$@"
EOF

chmod +x $PROJ/scripts/ohos-cc.sh $PROJ/scripts/ohos-cxx.sh
```

### Step 1: 下载 Node.js 源码

```bash
cd $PROJ/sources
wget https://nodejs.org/dist/v22.22.1/node-v22.22.1.tar.gz
tar xzf node-v22.22.1.tar.gz
cd node-v22.22.1
```

### Step 2: 打 Patch（2 处）

Node.js 22 官方已支持 `--dest-os=openharmony`（configure.py 里 `openharmony` 是 valid_os），大部分不需要 hack。但 OH 的 Clang 15 有两个兼容性问题需要修：

**Patch 1：C++ 标准升级到 C++20**

Node.js 22 的 ncrypto 模块用了 C++20 特性（`auto` 函数参数、`operator<=>`），但 gyp 配置还是 C++17。

```bash
# common.gypi 第 514 行
sed -i "s/'-std=gnu++17'/'-std=gnu++20'/" common.gypi
```

**Patch 2：zlib ARM CRC32 指令**

zlib.gyp 里 `zlib_arm_crc32` target 只在非 clang 时加 `-march=armv8-a+aes+crc`。Clang 15 需要这个标志才能编译 ARM CRC32 intrinsics，否则会 `CannotYetSelect` 崩溃。

```bash
# deps/zlib/zlib.gyp 第 72 行
sed -i "s/'OS!=\"win\" and clang==0'/'OS!=\"win\"'/" deps/zlib/zlib.gyp
```

### Step 3: Configure

```bash
cd $PROJ/sources/node-v22.22.1

CC_host=gcc CXX_host=g++ \
CC=$PROJ/scripts/ohos-cc.sh \
CXX=$PROJ/scripts/ohos-cxx.sh \
AR=$OH_CLANG/llvm-ar \
RANLIB=$OH_CLANG/llvm-ranlib \
python3 configure.py \
  --dest-cpu=arm64 \
  --dest-os=openharmony \
  --cross-compiling \
  --prefix=$PROJ/build/node-ohos
```

应该看到 `INFO: configure completed successfully`。

> **注意**：必须在 Node.js 源码目录下执行。`CC_host`/`CXX_host` 指向宿主机编译器（编 mksnapshot 等 host 工具用），`CC`/`CXX` 指向交叉编译器。

### Step 4: Make

```bash
CC_host=gcc CXX_host=g++ \
CC=$PROJ/scripts/ohos-cc.sh \
CXX=$PROJ/scripts/ohos-cxx.sh \
AR=$OH_CLANG/llvm-ar \
RANLIB=$OH_CLANG/llvm-ranlib \
make -j$(nproc)
```

编译时间约 30-60 分钟（取决于 CPU 核数和内存）。V8 的 `code-stub-assembler.cc` 和 `accessor-assembler.cc` 单文件编译吃 ~1GB 内存，8 核并行需要 8GB+ 内存。

### Step 5: Strip 和复制

```bash
mkdir -p $PROJ/build
cp out/Release/node $PROJ/build/node-ohos
$OH_CLANG/llvm-strip $PROJ/build/node-ohos

ls -lh $PROJ/build/node-ohos
# 约 93MB
file $PROJ/build/node-ohos
# ELF 64-bit LSB pie executable, ARM aarch64, dynamically linked, interpreter /lib/ld-musl-aarch64.so.1, stripped
```

### Step 6: 推板子验证

```bash
hdc file send $PROJ/build/node-ohos /data/local/tmp/node
hdc shell 'chmod +x /data/local/tmp/node && LD_LIBRARY_PATH=/system/lib64 /data/local/tmp/node --version'
# 输出：v22.22.1
```

---

## 从头构建 OpenClaw 部署包

适用场景：想更新 OpenClaw 版本，或自定义依赖。

### 前提

- 宿主机有 Node.js 22+ 和 pnpm 10+
- 已 clone OpenClaw 源码（`sources/openclaw/`）

### Step 1: 安装依赖 + 构建

```bash
cd sources/openclaw
pnpm install                                        # 安装依赖，约 40 秒
OPENCLAW_A2UI_SKIP_MISSING=1 pnpm build:docker      # TS → JS（跳过 Canvas A2UI，板子不需要）
pnpm ui:build                                       # 构建 Control UI 网页界面
```

### Step 2: 创建生产部署包

```bash
# pnpm deploy 提取生产依赖
pnpm deploy --filter openclaw --prod --legacy /tmp/openclaw-deploy

# 删除 x86 native addon（板子是 aarch64 用不了）
cd /tmp/openclaw-deploy/node_modules/.pnpm
rm -rf @node-llama-cpp+linux-x64-* node-llama-cpp@* \
  @napi-rs+canvas-linux-x64-* @img+sharp-libvips-linux-x64* \
  koffi@* typescript@* pdfjs-dist@*
```

### Step 3: 补入 Control UI 和模板文件

```bash
# Control UI（pnpm deploy 不包含 ui:build 产物）
cp -r /path/to/sources/openclaw/dist/control-ui/ /tmp/openclaw-deploy/dist/control-ui/

# Agent 模板文件（发消息时需要）
mkdir -p /tmp/openclaw-deploy/docs/reference/
cp -r /path/to/sources/openclaw/docs/reference/templates/ /tmp/openclaw-deploy/docs/reference/templates/
```

### Step 4: 打包

```bash
cd /tmp/openclaw-deploy
tar czf openclaw-deploy.tar.gz \
  --exclude='*.d.ts' --exclude='*.map' --exclude='*.ts' \
  --exclude='*.md' --exclude='LICENSE*' --exclude='CHANGELOG*' \
  --exclude='README*' --exclude='__tests__' --exclude='test' \
  --exclude='tests' --exclude='.github' \
  openclaw.mjs package.json dist/ docs/ node_modules/
# 产物约 67MB
```

然后按"快速部署"的步骤推到板子。

---

## 踩坑备忘

详细排查过程见 [DevHistory.md](DevHistory.md)。

### Node.js 交叉编译

| 坑 | 根因 | 修法 |
|:---|:---|:---|
| `auto` not allowed in function prototype | ncrypto.h 用 C++20，gyp 设的 C++17 | `common.gypi`: `gnu++17` → `gnu++20` |
| `operator<=>` parse error | C++20 三路比较运算符 | 同上 |
| Clang 15 崩溃 `CannotYetSelect` | zlib ARM CRC32 指令没加 `-march=armv8-a+crc` | `zlib.gyp`: 去掉 `clang==0` 条件 |
| SIGBUS（Signal 7） | 推了 SDK 的 libc++.so 与板子 musl 不兼容 | 用系统自带的 `/system/lib64/libc++.so` |
| hdc 传输文件 MD5 不一致 | USB 线质量差 | 换线，传后验证 MD5 |
| libc++.so 找不到 | OH 的 C++ 运行时在 `/system/lib64/` 不在 `/system/lib/` | `LD_LIBRARY_PATH=/system/lib64` |

### OpenClaw 部署

| 坑 | 根因 | 修法 |
|:---|:---|:---|
| `Missing workspace template: AGENTS.md` | 部署包没打进 docs/reference/templates/ | 补推模板文件 |
| `Control UI requires secure context` | 非 localhost 需要 HTTPS | `hdc fport` 端口转发到 localhost |
| `unauthorized: gateway token missing` | Control UI 需要 auth token | URL hash 传 token：`#token=xxx` |
| `Unknown model: deepseek/deepseek-chat` | `providers` 放在顶层而非 `models.providers` | 正确路径：`models.providers.custom` |
| Config: `expected array, received object` | `models` 字段写成了对象 | `models` 是数组：`[{id, name, ...}]` |
| `404 status code (no body)` | 默认用 `openai-responses` API，DeepSeek 不支持 | 显式指定 `"api": "openai-completions"` |
| Gateway 后台进程被杀 | hdc shell 退出后子进程被 OH 回收 | 保持 hdc shell 前台运行 |

## 版本信息

| 组件 | 版本 |
|:---|:---|
| OpenHarmony | 6.0 |
| 开发板 | RK3568 (aarch64) |
| Node.js | 22.22.1 |
| V8 | 12.4.254.21-node.35 |
| OpenSSL | 3.5.5（Node.js 内置） |
| OH Clang | 15.0.4 |
| OpenClaw | 2026.3.14 |
| DeepSeek Model | deepseek-chat（openai-completions API） |
| `process.platform` | `openharmony` |

## 与 Python 移植项目的关系

上一个项目 [langchain-on-openharmony](https://github.com/lmxxf/langchain-on-openharmony) 移植了 Python 3.11 + LangChain + AgentScope 到同一块板子。两个项目的核心经验互通：

- **OHOS musl dlopen 不继承符号** —— Python .so 和 Node.js .node 扩展都需要显式链接依赖
- **`/data/local/tmp` 走 acquiescence 命名空间** —— 没有 namespace 隔离，LD_LIBRARY_PATH 直接能用
- **编译器 wrapper 脚本** —— 同一套 `ohos-cc.sh` / `ohos-cxx.sh`
