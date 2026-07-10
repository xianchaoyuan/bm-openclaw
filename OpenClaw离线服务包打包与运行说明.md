# OpenClaw 离线服务包打包与运行说明

本文档说明如何在 Windows 环境下，从 OpenClaw 源码生成一个可随 C++/Qt 客户端一起分发的本地服务包。

最终目标不是制作安装程序，而是得到一个可以通过命令行启动的 `openclaw-service` 目录。C++/Qt 客户端后续只需要启动其中的 `openclaw.cmd`，再用 `QWebEngineView` 加载本地 Gateway 页面即可。

## 快速命令总览

以下是核心命令，详细说明见后续章节。

```powershell
pnpm install --frozen-lockfile
pnpm pack --pack-destination .artifacts/offline
npm.cmd install --omit=dev --no-audit --fund=false
openclaw.cmd gateway --dev --auth none --port 19001
```

## 1. 进入源码目录

在有网络的开发机上进入 OpenClaw 源码目录。

```powershell
cd D:\mycode\openclaw-main
```

如果源码目录不固定，也可以理解为进入实际的 `openclaw-main` 根目录。

```powershell
cd *\openclaw-main
```

## 2. 安装源码构建依赖

这一步只需要在有网络的开发机上执行。

```powershell
pnpm install --frozen-lockfile
```

说明：

- `pnpm install` 用于安装源码构建所需依赖。
- `--frozen-lockfile` 表示严格按照 `pnpm-lock.yaml` 中记录的版本安装，避免不同机器安装出不同版本。
- 这一步是“源码构建阶段”的依赖安装，不是最终服务包的运行步骤。

## 3. 构建并打出 npm 包

执行以下命令生成 OpenClaw 的 npm 包。

```powershell
pnpm pack --pack-destination .artifacts/offline
```

说明：

- 该命令会触发 OpenClaw 的打包流程。
- 输出目录是 `.artifacts/offline`。
- 会生成类似下面这样的文件：

```text
.artifacts\offline\openclaw-2026.6.11.tgz
```

注意：

- `.tgz` 是中间包，不是最终运行目录。
- `.tgz` 中包含运行 OpenClaw 所需的程序文件，但还没有安装最终运行依赖。
- 后续需要先解压该 `.tgz`，再补齐运行依赖。

## 4. 解压 tgz 文件

将上一步生成的 `.tgz` 文件解压。

可以使用 7-Zip 解压，也可以使用 Windows 自带的 `tar` 命令解压。

解压后会得到类似下面的目录：

```text
.artifacts\offline\openclaw-2026.6.11\package
```

其中 `package` 目录就是后续要作为服务主体的 OpenClaw 程序目录。

进入解压后的 `package` 目录：

```powershell
cd D:\mycode\openclaw-main\.artifacts\offline\openclaw-2026.6.11\package
```

## 5. 安装运行时依赖

在解压后的 `package` 目录中执行：

```powershell
npm.cmd install --omit=dev --no-audit --fund=false

```

说明：

- 必须在解压后的 `package` 目录中执行。
- 使用 `npm.cmd`，不要直接使用 `npm`，这样可以避免 PowerShell 拦截 `npm.ps1` 脚本。
- `--omit=dev` 表示只安装运行时依赖，不安装开发依赖。
- `--no-audit` 表示不执行安全审计，减少无关网络请求和输出。
- `--fund=false` 表示不显示赞助提示。

这一步完成后，`package` 目录下应该出现：

```text
package\node_modules

```

之后这个 `package` 目录就可以作为离线运行主体使用。目标电脑不需要再执行 `npm install`。

## 6. 整理最终服务目录

建议最终整理成如下结构：

```text
openclaw-service\
  openclaw.cmd
  runtime\
    node\
      node.exe
    openclaw\
      openclaw.mjs
      package.json
      dist\
      node_modules\
  state\
    openclaw.json
    .env

```

目录说明：

- `runtime\node`：Node.js 运行环境。
- `runtime\openclaw`：OpenClaw 程序主体。
- `state`：OpenClaw 运行状态和配置，每台电脑可以不同。
- `openclaw.cmd`：统一启动入口，供命令行或 C++/Qt 客户端调用。

注意：

- `runtime` 可以随客户端一起打包，通常不需要在运行时修改。
- `state` 是运行时数据目录，不建议把开发机运行后的完整 `state` 复制到其他电脑。
- 正式分发时，`state` 目录建议只保留干净的 `openclaw.json` 和 `.env`。

## 7. 启动 OpenClaw Gateway

在最终的 `openclaw-service` 目录下执行：

```powershell
openclaw.cmd gateway

```

说明：

- `gateway` 表示启动 OpenClaw Gateway 服务。

启动成功后，终端中应该能看到类似信息：

```text
[gateway] http server listening
[gateway] ready

```

此时可以在浏览器或 Qt `QWebEngineView` 中打开：

```text
http://127.0.0.1:19001/

```

## 8. Qt 客户端集成方式

C++/Qt 客户端可以按下面的方式集成：

1. 启动客户端时，通过 `QProcess` 启动 `openclaw.cmd`。
2. 等待 Gateway 服务启动完成。
3. 使用 `QWebEngineView` 加载本地地址。

示例加载地址：

```text
http://127.0.0.1:19001/

```

建议客户端启动 Gateway 时使用固定端口，或者从配置文件读取端口。

## 9. 分发注意事项

正式拷贝到其他电脑时，不建议带走开发机的完整运行状态。

不建议直接分发这些目录或文件：

```text
state\agents
state\identity
state\plugin-skills
state\workspace
state\workspace-attestations
state\openclaw.json.last-good

```

建议只分发：

```text
state\openclaw.json
state\.env

```

这样目标电脑第一次启动时，会自动生成属于当前电脑的运行状态，避免出现工作区路径不一致的问题。

## 10. 常见问题

### 为什么不用 npm ci？

在这个流程中建议使用：

```powershell
npm.cmd install --omit=dev --no-audit --fund=false

```

不建议使用：

```powershell
npm ci --omit=dev

```

原因是 `npm ci` 对锁文件一致性要求更严格，可能因为 OpenClaw 的包结构和运行依赖裁剪方式导致安装失败。

### 为什么要用 npm.cmd？

Windows PowerShell 可能会因为执行策略禁止运行 `npm.ps1`，出现类似错误：

```text
无法加载文件 C:\Program Files\nodejs\npm.ps1，因为在此系统上禁止运行脚本

```

使用 `npm.cmd` 可以绕过这个问题。

### `.tgz` 是最终服务包吗？

不是。

`.tgz` 只是中间产物。最终给 C++/Qt 客户端使用的应该是整理后的 `openclaw-service` 目录。

### 目标电脑还需要联网吗？

如果已经在开发机上完成了依赖安装，并把 `runtime\openclaw\node_modules` 一起拷贝过去，目标电脑启动服务本身不需要再联网安装依赖。

但如果使用 DeepSeek 或其他在线大模型，运行时仍然需要访问对应模型服务。
