# dsh-linux-dispatch-preset

DeepSeek Harness 的 **Linux 专用调度 Agent preset**。

本项目基于 `@eddyskywalker/dsh-chatgpt-subscription` 的 dispatch preset，保留任务分诊、计划审批、子代理调度和统一验收机制，同时将 Code Mode 的提示词与 Shell 调用完整适配到 Linux。

## 特性

- **任务分诊**：按 L0、L1、L2 判断复杂度，选择直接执行、简短计划或治理流程。
- **子代理调度**：支持 Lead-Core、Hybrid、Fan-out 拓扑，明确模型路由、任务边界和验收证据。
- **Code Mode**：以 `code_run` 为主要入口，通过程序内的 `tools.*` 调用其他工具。
- **Linux 专用**：Shell 统一使用 `bash`，采用 POSIX 路径，不包含 PowerShell 或 Windows 工具分支。
- **独立提示词**：关闭上游 Windows 版 `tools:sdk` 文案，使用本包自带的 Linux Code Mode 说明。

## 前置条件

- Linux 与 `bash`
- DeepSeek Harness（当前按 **0.2.2-rc.1** 适配）
- 已安装并启用 `@eddyskywalker/dsh-chatgpt-subscription >= 0.80.0`

本包引用上游插件的四个子入口：

- `codemode`
- `plan-mode`
- `plan-mode-pipeline`
- `tool-result-pruner`

因此上游插件是实际运行依赖。缺少它时，preset 会因模块 specifier 无法解析而挂载失败。

## 安装

先取得源码：

```sh
git clone https://github.com/pasze888/dsh-linux-dispatch-preset.git
```

然后将本地目录加入 DSH profile：

```sh
dsh plugin --profile web add /absolute/path/dsh-linux-dispatch-preset
```

安装后重启 DSH Web。补丁行 ID 为 `linux-dispatch-preset`，可在 Plugins 页面单独启停。

也可以不修改 profile 依赖，直接向 `~/.dsh/profiles/<profile>/cordis.patch.yml` 添加：

```yaml
- insert:
    - id: linux-dispatch-preset
      name: /absolute/path/dsh-linux-dispatch-preset
```

## 使用

1. 重启 DSH Web。
2. 新开会话，在 Agent preset 中选择 **Linux 调度模式**。
3. 直接描述任务；主 Agent 会按复杂度决定执行、计划或调度方式。

已有会话不会自动获得更新后的系统提示词。修改本包后，需要重新挂载 preset 并新开会话验证。

## Linux 适配说明

### Shell

本 preset 只注册 `bash`，不注册 `pwsh`。程序内调用示例：

```js
const result = await tools.bash({
  command: "ls -la",
  workdir: "/home/user/project",
  description: "List current directory contents"
})
return result.stdout.text
```

`command` 和 `description` **都是必填参数**，`workdir` 可选。`stdout` 和 `stderr` 是包含 `text`、`truncated` 字段的对象，不是裸字符串。

### Code Mode 提示词

`code-mode.config.sdkSection` 设置为 `false`，只关闭上游硬编码的 Windows 版说明段，不会关闭 Code Mode、`code_run`、工具发现或执行器。

替代说明位于 persona 的 `## Code Mode (Linux)`，覆盖：

- native 工具白名单与程序内调用规则
- 常用工具参数和返回信封
- `bash` 的完整必填参数
- POSIX 路径与环境继承
- `code_tools_list/search/describe` 发现接口

### 其他兼容调整

- 使用 `@deepseek-ai/dsh-workflow-ptc` 的现行包名。
- 移除旧版 `tool-subagent-list-agents` 行；当前 Harness 从主入口注册 `list_agents`。
- 移除 persona 的 `{{cwd}}` suffix，避免当前 Harness 因变量未注册而渲染失败。

## 文件结构

```text
dsh-linux-dispatch-preset/
├── cordis.patch.yml                    # 实际生效的 bundle patch
├── package.json                        # 包元数据与 bundle 声明
├── presets/linux-dispatch/
│   ├── preset.yml                      # UI 名称、描述和排序（参考）
│   └── agent.cordis.yml                # Agent 配置的人读参考副本
└── README.md
```

**真正生效的是 `cordis.patch.yml`。** 修改 Agent 配置时，必须同步更新参考副本，并确保两份 persona 和 rows 一致。

## 维护注意事项

- Linux Code Mode 说明是上游 `tools:sdk` 的本地替代副本。上游改变工具传输契约时，需要同步更新。
- 首次调用不熟悉的程序内工具前，应通过 `code_tools_describe` 查询 `parameters` 和 `required`，不要猜参数。
- 修改后至少检查：YAML 可解析、两份 rows 一致、`sdkSection: false`、所有 `bash` 示例均带 `description`。
- 安装或升级后，检查 DSH 日志中是否有 preset 注册失败或 specifier 解析失败。

## License

MIT
