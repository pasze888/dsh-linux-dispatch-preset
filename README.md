# dsh-linux-dispatch-preset

把 `@eddyskywalker/dsh-chatgpt-subscription` 的 **dispatch（调度模式）** preset 复制出来，做成一个
**Linux 专用**的 bundle 包：同样是 Code Mode + 调度纪律，但去掉所有 Windows 分支与 `win_bash` 说法。

## 装它

包声明了 `dsh.bundle.patch: ./cordis.patch.yml`，所以能被当**套件层**装进 profile
（判定只看这个字段，见 `@deepseek-ai/dsh-plugin-manager` 的 `bundleManifest()`）：

```sh
dsh plugin --profile web add /home/lz/dev/dsh-linux-dispatch-preset
# 装完重启 dsh web；补丁插入的行 id 是 linux-dispatch-preset，可在 Plugins 页单独开关
```

不想改 profile 依赖时，直接往 `~/.dsh/profiles/<profile>/cordis.patch.yml` 里插一行绝对路径：

```yaml
- insert:
    - id: linux-dispatch-preset
      name: /home/lz/dev/dsh-linux-dispatch-preset
```

**前置依赖**：本包必须与 `@eddyskywalker/dsh-chatgpt-subscription` 同时安装 —— 四个子路径行
（`codemode` / `plan-mode` / `plan-mode-pipeline` / `tool-result-pruner`）指向它。缺了它们，
preset 会在 mount 时因为 specifier 解析不了而整体失败。

## 与上游 dispatch preset 的差异

| # | 改动 | 原因 |
| --- | --- | --- |
| 1 | persona 白名单句子 `including win_bash` → `including bash` | 本机没有 `win_bash` 这个工具 |
| 2 | 新增 `[Platform: Linux]` 段 | `bash` 的返回信封是 `{ kind, cwd, exitCode, …, stdout: { text, truncated } }`，`stdout` 是对象不是字符串；程序内 `process.env` 为空；bash 子进程 `TMPDIR` 为空；路径是 POSIX |
| 3 | 删 `tool-pwsh` 行，`tool-bash` 去掉 `!!js process.platform === 'win32'` | preset 只服务 Linux，条件表达式没有意义 |
| 4 | 删 `tool-subagent-list-agents` 行 | 本机 `@deepseek-ai/dsh-tool-subagent-control` 没有 `./list-agents` 导出（新 harness 从主入口注册） |
| 5 | workflow 行固定写 `@deepseek-ai/dsh-workflow-ptc` | 上游靠 `preset-sync.ts` 把 0.1.5 时代的老包名改写过来；纯 bundle patch 没有改写者，写老名会挂不上 |
| 6 | 删掉 persona 的 `suffix: Your working directory is {{cwd}}.` | 本机 harness 不注册 `{{cwd}}` 变量，渲染期抛 `unknown prompt variable "{{cwd}}"`；上游靠 `resolveDispatchPreset` 剥掉这一键，纯 patch 没有剥离者 |
| 7 | `code-mode` 设置 `sdkSection: false`，persona 内置完整 Linux Code Mode 说明 | 上游 `tools:sdk` 说明段硬编码 `win_bash`、Windows 路径与环境变量；在 preset 层关闭并替换，无需修改上游源码 |

其余 rows（plan-mode 两个 fork、compaction 组、delegation 组、skills、goals、
tool-web / tool-todo / tool-ask-user、present）与上游逐字一致；persona 与 code-mode config
除上述改动外保持上游内容。

## Linux Code Mode 说明

`code-mode.config.sdkSection: false` 只关闭上游的 Windows 版提示文案，不会关闭 Code Mode、
`code_run`、工具发现或执行器。persona 中的 `## Code Mode (Linux)` 提供替代说明，覆盖 native
白名单、程序内工具调用、`bash` 参数与返回信封、POSIX 路径、环境继承和 discovery API。
其中 `bash` 的 `command` 与 `description` 都是必填参数，所有调用示例必须显式提供两者；
否则程序内调用会以 `invalid arguments: missing required property "description"` 失败。

维护注意：这份说明是上游 `tools:sdk` 的本地替代副本。上游改变工具传输契约时，需要同步
更新 persona；修改上游 `src/host/codemode/rules.ts` 不会影响本 preset。

## 验证

```sh
# 1) YAML 是否合法（本机有 yaml 包）
node -e "const YAML=require('/home/lz/.npm-global/lib/node_modules/@deepseek-ai/dsh/node_modules/yaml'); \
  const d=YAML.parse(require('node:fs').readFileSync('/home/lz/dev/dsh-linux-dispatch-preset/cordis.patch.yml','utf8')); \
  console.log(d[0].insert[0].config.plugins.length, 'rows')"

# 2) 挂没挂上：重启 dsh web 后看日志有无 preset 注册失败 / specifier 解析失败
```
