# OpenClaw on OpenHarmony 6.0 (RK3568, aarch64)

在 OpenHarmony 开发板上跑 Node.js 22 + OpenClaw（264k star AI Agent 框架）。

```
OH RK3568 → Node.js 22.22.1 (V8 12.4 + OpenSSL 3.5.5)
    → process.platform = "openharmony"
    → crypto / net / https OK
    → OpenClaw（进行中）
```

## 快速部署（已有编译产物）

2 条命令把 Node.js 22 跑在 OH 开发板上。

### 前提

- 宿主机能通过 `hdc` 连接到 RK3568 开发板
- 板子上有 `/data/local/tmp/`（14G+ 可用空间）

### 步骤

**1. 推 node 二进制到板子**

`build/node-ohos`（93MB）是 strip 后的 Node.js 22.22.1 静态二进制（V8 + OpenSSL + libuv + ICU 全部静态链入，只依赖系统 libc.so 和 libc++.so）。

```bash
hdc file send build/node-ohos /data/local/tmp/node
hdc shell 'chmod +x /data/local/tmp/node'
```

**2. 验证**

```bash
hdc shell 'LD_LIBRARY_PATH=/system/lib64 /data/local/tmp/node --version'
# 输出：v22.22.1
```

完整验证脚本（保存为 `test_node.js`，推到板子上跑）：

```javascript
console.log('Platform:', process.platform);
console.log('Arch:', process.arch);
console.log('Version:', process.version);
console.log('V8:', process.versions.v8);
console.log('OpenSSL:', process.versions.openssl);

const crypto = require('crypto');
console.log('crypto OK, random:', crypto.randomBytes(4).toString('hex'));

const net = require('net');
console.log('net OK');

const https = require('https');
console.log('https OK');

console.log('\n=== Node.js on OpenHarmony: ALL OK ===');
```

```bash
hdc file send test_node.js /data/local/tmp/test_node.js
hdc shell 'LD_LIBRARY_PATH=/system/lib64 /data/local/tmp/node /data/local/tmp/test_node.js'
```

预期输出：

```
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

**3. 运行命令模板**

```bash
LD_LIBRARY_PATH=/system/lib64 /data/local/tmp/node your_script.js
```

> **注意**：`LD_LIBRARY_PATH=/system/lib64` 是必须的——板子上 libc++.so 在 `/system/lib64/`，不在默认搜索路径里。

---

## 从头编译

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

然后按"快速部署"的步骤验证。

---

## 踩坑备忘

详细排查过程见 [DevHistory.md](DevHistory.md)。

| 坑 | 根因 | 修法 |
|:---|:---|:---|
| `auto` not allowed in function prototype | ncrypto.h 用 C++20，gyp 设的 C++17 | `common.gypi`: `gnu++17` → `gnu++20` |
| `operator<=>` parse error | C++20 三路比较运算符 | 同上 |
| Clang 15 崩溃 `CannotYetSelect` | zlib ARM CRC32 指令没加 `-march=armv8-a+crc` | `zlib.gyp`: 去掉 `clang==0` 条件 |
| SIGBUS（Signal 7） | 推了 SDK 的 libc++.so 与板子 musl 不兼容 | 用系统自带的 `/system/lib64/libc++.so` |
| hdc 传输文件 MD5 不一致 | USB 线质量差 | 换线，传后验证 MD5 |
| libc++.so 找不到 | OH 的 C++ 运行时在 `/system/lib64/` 不在 `/system/lib/` | `LD_LIBRARY_PATH=/system/lib64` |

## 编译产物

| 产物 | 大小 | 说明 |
|:---|:---|:---|
| `build/node-ohos` | 93MB | Node.js 22.22.1，strip 后 |

动态依赖（均为 OH 系统自带）：

```
libc++.so    → /system/lib64/libc++.so
libc.so      → /system/lib/ld-musl-aarch64.so.1
```

V8、OpenSSL 3.5.5、libuv、zlib、brotli、zstd、sqlite、ICU、nghttp2、simdjson、simdutf 等全部静态链入。

## 版本信息

| 组件 | 版本 |
|:---|:---|
| OpenHarmony | 6.0 |
| 开发板 | RK3568 (aarch64) |
| Node.js | 22.22.1 |
| V8 | 12.4.254.21-node.35 |
| OpenSSL | 3.5.5（Node.js 内置） |
| OH Clang | 15.0.4 |
| `process.platform` | `openharmony` |

## 与 Python 移植项目的关系

上一个项目 [langchain-on-openharmony](https://github.com/lmxxf/langchain-on-openharmony) 移植了 Python 3.11 + LangChain + AgentScope 到同一块板子。两个项目的核心经验互通：

- **OHOS musl dlopen 不继承符号** —— Python .so 和 Node.js .node 扩展都需要显式链接依赖
- **`/data/local/tmp` 走 acquiescence 命名空间** —— 没有 namespace 隔离，LD_LIBRARY_PATH 直接能用
- **编译器 wrapper 脚本** —— 同一套 `ohos-cc.sh` / `ohos-cxx.sh`
