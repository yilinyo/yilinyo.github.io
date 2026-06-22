---
title: nanobot的安全机制 
tag: agent
date: 2026-06-16 15:03:00
---
# nanobot 安全机制与 Shell 沙箱学习笔记

  

本文整理 nanobot 当前代码中的安全机制，重点说明：聊天权限如何控制、`restrictToWorkspace` 如何限制文件访问、Shell 命令执行前经过哪些检查、`bwrap` 沙箱如何包装并启动命令，以及这些机制能够保护什么、不能保护什么。

  

![展示 nanobot Agent 在可读写 Workspace 中受控执行，同时隔离宿主机配置密钥、SSH 私钥和系统目录的安全机制封面图](https://files.seeusercontent.com/2026/06/15/8uuJ/01-cover.webp)

  

## 一、总体设计

  

nanobot 的安全机制不是一个单独的开关，而是由多层边界共同组成：

  

```text

Channel 发送者校验

↓

聊天 workspace scope

↓

Tool 参数 Schema 校验

↓

文件路径边界检查

↓

Shell 命令文本检查

↓

bwrap 文件系统隔离

↓

网络 SSRF 防护

```

  

这些机制解决的问题不同：

  

- Channel 权限控制“谁可以向 Agent 发消息”。

- Workspace scope 控制“当前聊天以哪个项目目录作为边界”。

- Tool Schema 控制“模型传入的工具参数是否合法”。

- 文件工具限制控制“文件路径是否位于允许目录”。

- Shell guard 控制“命令文本中是否包含明显危险操作”。

- `bwrap` 控制“命令在操作系统层面能够看到哪些文件”。

- SSRF 防护控制“工具是否可以访问内网和云元数据地址”。

  

需要先明确一个核心区别：

  

```text

restrictToWorkspace

= nanobot 应用层路径检查

  

bwrap

= Linux 内核提供的文件系统视图隔离

```

  

前者依赖 nanobot 正确识别路径和命令，后者在进程真正访问文件时生效。

  

![展示 nanobot 从 Channel 身份校验、Workspace Scope、Tool Schema、Shell Guard、bwrap 到 SSRF 防护的多层安全架构图](https://files.seeusercontent.com/2026/06/15/J8eg/02-security-architecture.webp)

  

## 二、Channel 访问控制

  

### 1. `allowFrom`

  

Channel 在把消息发布到 `MessageBus` 前，会先检查发送者：

  

```python

def is_allowed(self, sender_id: str) -> bool:

if "*" in allow_list:

return True

if str(sender_id) in allow_list:

return True

if is_approved(self.name, str(sender_id)):

return True

return False

```

  

规则如下：

  

```text

allowFrom 包含 "*"

→ 允许所有发送者

  

allowFrom 包含当前 sender_id

→ 允许该发送者

  

发送者已经通过 pairing 批准

→ 允许该发送者

  

以上都不满足

→ 拒绝

```

  

`allowFrom` 使用精确字符串匹配，不会把发送者 ID 当作正则表达式或子串。

  

### 2. Pairing

  

未授权用户在私聊中发送消息时，Channel 可以生成临时配对码：

  

```text

未授权 DM

↓

generate_code()

↓

返回 ABCD-EFGH 形式的随机码

↓

Owner 执行 /pairing approve <code>

↓

发送者加入 approved 列表

```

  

配对码使用：

  

```python

secrets.choice(...)

```

  

生成，默认有效期为 600 秒，即 10 分钟。

  

配对信息持久化在：

  

```text

~/.nanobot/pairing.json

```

  

Channel 权限相关代码主要位于：

  

```text

channels/base.py

pairing/store.py

```

  

## 三、聊天 Workspace Scope

  

### 1. 两种访问模式

  

WebUI 聊天可以保存自己的 workspace scope：

  

```json

{

"workspace_scope": {

"project_path": "/path/to/project",

"access_mode": "restricted"

}

}

```

  

当前支持两种实际访问模式：

  

```text

restricted

= 当前聊天限制在 project_path

  

full

= 当前聊天不启用 workspace 路径限制

```

  

WebUI 展示的 Default Access 表示继承全局配置：

  

```json

{

"tools": {

"restrictToWorkspace": true

}

}

```

  

如果全局配置为 `true`，Default Access 最终得到 `restricted`；如果全局配置为 `false`，Default Access 最终得到 `full`。

  

### 2. Scope 会覆盖全局配置

  

每轮 Agent 执行前，系统会把当前 scope 绑定到 `ContextVar`：

  

```python

workspace_token = bind_workspace_scope(effective_scope)

```

  

工具调用时再读取当前 scope：

  

```python

access = current_tool_workspace(...)

```

  

因此 WebUI 会话中保存的：

  

```json

"access_mode": "full"

```

  

可以覆盖全局：

  

```json

"restrictToWorkspace": true

```

  

这也是修改全局配置并重启后，旧聊天仍可能访问 workspace 外部的原因。旧聊天的 scope 保存在 Session metadata 中，不会因为重启自动删除。

  

主要实现位于：

  

```text

security/workspace_access.py

webui/workspaces.py

channels/websocket.py

agent/loop.py

```

  

## 四、`restrictToWorkspace`

  

### 1. 配置定义

  

配置定义在 `config/schema.py`：

  

```python

class ToolsConfig(Base):

restrict_to_workspace: bool = False

```

  

默认值是 `False`，需要显式开启：

  

```json

{

"tools": {

"restrictToWorkspace": true

}

}

```

  

JSON 中使用 camelCase，Pydantic 加载后对应 Python 字段：

  

```python

config.tools.restrict_to_workspace

```

  

### 2. 配置什么时候进入 AgentLoop

  

Gateway 启动时：

  

```text

nanobot gateway

↓

_load_runtime_config()

↓

_run_gateway()

↓

AgentLoop.from_config()

↓

AgentLoop.__init__()

```

  

`AgentLoop.from_config()` 把配置传入构造函数：

  

```python

return cls(

...

restrict_to_workspace=config.tools.restrict_to_workspace,

tools_config=config.tools,

)

```

  

`AgentLoop.__init__()` 随后创建工具注册表并加载工具：

  

```python

self.tools = ToolRegistry()

self._register_default_tools()

```

  

所以修改 `restrictToWorkspace`、`exec.enable` 或 `exec.sandbox` 后，通常需要重启 Gateway，使新的 `AgentLoop` 和工具实例重新创建。

  

## 五、文件工具如何限制路径

  

### 1. 创建文件工具

  

文件工具创建时会判断是否需要限制 workspace：

  

```python

restrict = (

ctx.config.restrict_to_workspace

or ctx.config.exec.sandbox

)

  

allowed_dir = Path(ctx.workspace) if restrict else None

```

  

这表示：

  

- 显式开启 `restrictToWorkspace` 时限制文件工具。

- 只要配置了 Shell sandbox，也自动限制文件工具。

  

### 2. 路径解析流程

  

文件工具调用 `_resolve()`：

  

```python

return resolve_workspace_path(

path,

access.project_path,

access.allowed_root,

self._extra_allowed_dirs,

)

```

  

最终进入 `resolve_allowed_path()`：

  

```python

resolved = resolve_path(path, workspace, strict=False)

  

if not is_path_allowed(resolved, roots):

raise WorkspaceBoundaryError(...)

```

  

完整流程可以表示为：

  

```text

用户传入 path

↓

展开 ~

↓

相对路径拼接 workspace

↓

Path.resolve()

↓

消除 .. 和符号链接

↓

检查最终路径是否位于允许根目录

```

  

### 3. 路径包含判断

  

核心判断使用：

  

```python

resolved_path.relative_to(resolved_root)

```

  

如果成功，说明路径等于根目录或位于根目录下；如果抛出 `ValueError`，说明路径越界。

  

例如：

  

```text

workspace = /home/user/project

  

/home/user/project/src/main.py

→ 允许

  

/home/user/project/../secret.txt

→ resolve 后成为 /home/user/secret.txt

→ 拒绝

  

/home/user/project/link

→ 如果 link 指向 /etc

→ resolve 后位于 /etc

→ 拒绝

```

  

### 4. 额外允许目录

  

文件工具除了 workspace，还可能允许读取：

  

- 上传媒体目录。

- 内置 skills 目录。

- 工具显式配置的其他允许目录。

  

这些额外目录不代表整个宿主机都开放，只是加入允许根目录集合。

  

主要代码位于：

  

```text

agent/tools/filesystem.py

agent/tools/path_utils.py

security/workspace_policy.py

security/workspace_access.py

```

  

## 六、Shell 工具的安全检查

  

### 1. ExecTool 配置

  

`ExecToolConfig` 定义：

  

```python

class ExecToolConfig(Base):

enable: bool = True

timeout: int = 60

path_append: str = ""

sandbox: str = ""

allowed_env_keys: list[str] = []

allow_patterns: list[str] = []

deny_patterns: list[str] = []

```

  

其中：

  

- `enable`：是否注册 `exec` 工具。

- `timeout`：默认命令超时。

- `sandbox`：沙箱后端名称，目前只有 `bwrap`。

- `allowed_env_keys`：额外允许传入子进程的环境变量。

- `allow_patterns`：命令允许规则。

- `deny_patterns`：命令拒绝规则。

  

完全禁用 Shell：

  

```json

{

"tools": {

"exec": {

"enable": false

}

}

}

```

  

此时 `ToolLoader` 会执行：

  

```python

if not tool_cls.enabled(ctx):

continue

```

  

所以 `ExecTool` 不会进入 `ToolRegistry`，LLM 也看不到 `exec` 工具定义。

  

### 2. 命令执行入口

  

LLM 调用 `exec` 后，执行链是：

  

```text

AgentRunner

↓

ToolRegistry.prepare_call()

↓

参数转换和 Schema 校验

↓

ExecTool.execute()

↓

ExecTool._prepare_command()

↓

ExecTool._spawn()

```

  

`_prepare_command()` 负责安全检查和命令包装，`_spawn()` 负责真正创建子进程。

  

### 3. `cwd`

  

`cwd` 是 Current Working Directory，即子进程的初始工作目录：

  

```python

cwd = working_dir or workspace_root or os.getcwd()

```

  

优先级为：

  

```text

工具调用传入 working_dir

↓

当前 workspace

↓

nanobot 进程自己的工作目录

```

  

创建子进程时：

  

```python

await asyncio.create_subprocess_exec(

*args,

cwd=cwd,

env=env,

)

```

  

需要注意：

  

```text

cwd 只是起始目录，不是安全边界。

```

  

没有沙箱时，即使进程从 workspace 启动，也可以执行：

  

```bash

cd /etc

cat passwd

```

  

### 4. `working_dir` 越界检查

  

开启 workspace 限制后，`_prepare_command()` 会检查：

  

```python

requested = Path(cwd).expanduser().resolve()

resolved_root = Path(workspace_root).expanduser().resolve()

  

if not is_path_within(requested, resolved_root):

return "Error: working_dir is outside the configured workspace"

```

  

因此模型不能直接传入：

  

```json

{

"working_dir": "/etc"

}

```

  

但这只限制启动目录，不代表命令运行后绝对不能切换目录。

  

### 5. 危险命令过滤

  

`ExecTool` 内置了一组 deny patterns，用于拦截明显危险命令：

  

```text

rm -r / rm -rf

del /f

rmdir /s

mkfs

diskpart

dd if=

写入 /dev/sd*

shutdown / reboot / poweroff

fork bomb

```

  

另外还会保护 nanobot 内部的：

  

```text

history.jsonl

.dream_cursor

```

  

避免 Shell 直接重定向、`tee`、`cp`、`mv`、`dd` 或 `sed -i` 破坏这些文件。

  

### 6. Shell 路径检查

  

开启 workspace 限制后，`_guard_command()` 会：

  

1. 拒绝命令文本中的 `../` 和 `..\`。

2. 用正则提取显式绝对路径。

3. 对每个路径执行 `resolve()`。

4. 检查路径是否位于当前工作目录或媒体目录。

  

例如：

  

```bash

cat /etc/passwd

```

  

可以提取出 `/etc/passwd` 并拒绝。

  

但以下形式可能没有显式绝对路径：

  

```bash

cd $HOME

cat "$HOME/.ssh/id_rsa"

sh -c 'cd "$HOME" && pwd'

```

  

这说明 Shell guard 属于 best-effort 文本检查，不是严格的操作系统边界。

  

### 7. 环境变量最小化

  

Unix 子进程默认只继承少量环境变量：

  

```python

env = {

"HOME": home,

"LANG": ...,

"TERM": ...,

"PYTHONUNBUFFERED": "1",

}

```

  

API Key 等变量默认不会直接传给子进程。只有配置在：

  

```json

"allowedEnvKeys": ["SOME_KEY"]

```

  

中的变量才会额外传入。

  

不过当前默认使用 login shell：

  

```python

bash -l -c command

```

  

login shell 可能读取用户 profile，因此实际环境还需要结合宿主机 Shell 配置理解。

  

### 8. 超时与输出限制

  

模型单次传入的 timeout 最大为 600 秒：

  

```python

return min(timeout, self._MAX_TIMEOUT)

```

  

默认超时为 60 秒。

  

命令输出默认限制约 10,000 字符，过长时保留开头和结尾，中间替换为截断提示。

  

## 七、Tool 参数 Schema

  

### 1. `@tool_parameters`

  

`ExecTool` 使用：

  

```python

@tool_parameters(

tool_parameters_schema(

command=StringSchema(...),

timeout=IntegerSchema(...),

...

)

)

class ExecTool(Tool):

...

```

  

这个装饰器给 `ExecTool` 动态注入 `parameters` 属性：

  

```python

@property

def parameters(self: Any) -> dict[str, Any]:

return deepcopy(frozen)

  

cls.parameters = parameters

```

  

等价于在 `ExecTool` 中手写：

  

```python

@property

def parameters(self):

return exec_parameter_schema

```

  

### 2. 为什么需要 `parameters`

  

`Tool` 抽象基类要求每个工具提供：

  

```text

name

description

parameters

execute()

```

  

`parameters` 用于生成发给 LLM 的 Function Calling Schema：

  

```python

{

"type": "function",

"function": {

"name": self.name,

"description": self.description,

"parameters": self.parameters,

},

}

```

  

它也用于执行前：

  

```text

cast_params()

= 根据 Schema 做安全类型转换

  

validate_params()

= 检查类型、必填项、最小值和最大值

```

  

### 3. `deepcopy`

  

Schema 保存和返回时都使用 `deepcopy`：

  

```python

frozen = deepcopy(schema)

return deepcopy(frozen)

```

  

第一层复制防止调用者在装饰完成后修改原始 `schema`。

  

第二层复制防止读取者修改返回值后污染类内部保存的 Schema。

  

## 八、工具什么时候加载

  

### 1. 主 Agent

  

`AgentLoop.__init__()` 中：

  

```python

self.tools = ToolRegistry()

self._register_default_tools()

```

  

`_register_default_tools()` 创建 `ToolContext`，然后调用：

  

```python

loader = ToolLoader()

registered = loader.load(ctx, self.tools)

```

  

所以主 Agent 的工具通常在 Gateway 启动时加载一次，后续聊天复用同一个注册表和工具实例。

  

### 2. ToolLoader

  

`ToolLoader` 会扫描：

  

```text

agent/tools/*.py

```

  

寻找所有可发现的 `Tool` 子类，然后执行：

  

```python

if not tool_cls.enabled(ctx):

continue

  

tool = tool_cls.create(ctx)

registry.register(tool)

```

  

`sandbox.py` 位于跳过列表中，因为它只是 `ExecTool` 的辅助模块，不是可以直接给 LLM 调用的 Tool。

  

### 3. Subagent

  

每个 Subagent 启动任务时会创建独立注册表：

  

```python

registry = ToolRegistry()

ToolLoader().load(ctx, registry, scope="subagent")

```

  

主 Agent 的注册表通常一直存在到 Gateway 退出；Subagent 注册表在任务完成并失去引用后可以被垃圾回收。

  

## 九、ToolRegistry 如何使用工具

  

注册时：

  

```python

def register(self, tool: Tool) -> None:

self._tools[tool.name] = tool

```

  

内存结构类似：

  

```python

{

"exec": ExecTool(...),

"read_file": ReadFileTool(...),

"write_file": WriteFileTool(...),

"web_fetch": WebFetchTool(...),

}

```

  

执行时：

  

```python

tool, params, error = self.prepare_call(name, params)

```

  

`prepare_call()` 通过：

  

```python

tool = self._tools.get(name)

```

  

取得具体工具实例。

  

虽然类型标注是：

  

```python

Tool | None

```

  

运行时返回的是具体子类，例如：

  

```text

name == "exec"

→ ExecTool 实例

  

name == "read_file"

→ ReadFileTool 实例

```

  

随后：

  

```python

result = await tool.execute(**params)

```

  

Python 根据实际对象类型调用对应子类的 `execute()`。

  

## 十、bwrap 沙箱配置

  

Linux 环境可以配置：

  

```json

{

"tools": {

"restrictToWorkspace": true,

"exec": {

"enable": true,

"sandbox": "bwrap"

}

}

}

```

  

当前沙箱后端注册表只有：

  

```python

_BACKENDS = {

"bwrap": _bwrap,

}

```

  

因此：

  

- Linux 安装 Bubblewrap 后可以使用。

- macOS 没有 Linux Namespace，不能直接运行。

- Windows 代码会记录不支持警告并按当前逻辑不包装命令。

  

在 macOS 上需要严格隔离时，更合适的方案是：

  

- 在 Docker Desktop 的 Linux VM 中运行 nanobot。

- 使用 Linux 虚拟机。

- 不需要 Shell 时关闭 `exec`。

  

## 十一、沙箱命令如何生成

  

### 1. 没有 Sandbox

  

假设原始命令：

  

```bash

pytest -v

```

  

没有沙箱时：

  

```python

command = "pytest -v"

cwd = "/home/user/.nanobot/workspace"

```

  

`_spawn()` 最终大致启动：

  

```bash

bash -l -c 'pytest -v'

```

  

进程关系：

  

```text

nanobot

└── bash -l -c "pytest -v"

└── pytest

```

  

这个进程直接拥有 nanobot 系统用户的文件和网络权限。

  

### 2. 使用 Sandbox

  

设置：

  

```json

"sandbox": "bwrap"

```

  

后，`_prepare_command()` 执行：

  

```python

command = wrap_command(

self.sandbox,

command,

workspace,

cwd,

)

```

  

`wrap_command()` 根据名称取得后端：

  

```python

if backend := _BACKENDS.get(sandbox):

return backend(command, workspace, cwd)

```

  

最终进入：

  

```python

_bwrap(command, workspace, cwd)

```

  

原始命令会变成类似：

  

```bash

bwrap \

--new-session \

--die-with-parent \

--ro-bind /usr /usr \

--ro-bind-try /bin /bin \

--ro-bind-try /lib /lib \

--proc /proc \

--dev /dev \

--tmpfs /tmp \

--tmpfs /home/user/.nanobot \

--dir /home/user/.nanobot/workspace \

--bind /home/user/.nanobot/workspace /home/user/.nanobot/workspace \

--ro-bind-try /home/user/.nanobot/media /home/user/.nanobot/media \

--chdir /home/user/.nanobot/workspace \

-- sh -c 'pytest -v'

```

  

随后外层 `_spawn()` 再执行包装后的命令：

  

```bash

bash -l -c 'bwrap ... -- sh -c "pytest -v"'

```

  

进程关系：

  

```text

nanobot

└── bash

└── bwrap

└── sh -c "pytest -v"

└── pytest

```

  

每次 `exec` 调用都会创建新的临时沙箱。命令结束后，该沙箱进程随之结束，不存在一个长期运行的共享 bwrap 容器。

  

![展示原始 Shell 命令经过 ToolRegistry、参数校验、Shell Guard 和 bwrap 包装后在临时沙箱内执行的完整流程图](https://files.seeusercontent.com/2026/06/15/3iQw/03-bwrap-flow.webp)

  

## 十二、`_bwrap()` 做了什么

  

### 1. 规范化路径

  

```python

ws = Path(workspace).resolve()

media = get_media_dir().resolve()

```

  

将 workspace 和媒体目录解析成规范绝对路径。

  

### 2. 计算沙箱工作目录

  

```python

try:

sandbox_cwd = str(ws / Path(cwd).resolve().relative_to(ws))

except ValueError:

sandbox_cwd = str(ws)

```

  

如果 `cwd` 在 workspace 内，则保留对应目录；如果位于外部，则回退到 workspace 根目录。

  

### 3. 系统目录只读挂载

  

必需目录：

  

```python

required = ["/usr"]

```

  

可选目录：

  

```python

[

"/bin",

"/lib",

"/lib64",

"/etc/alternatives",

"/etc/ssl/certs",

"/etc/resolv.conf",

"/etc/ld.so.cache",

]

```

  

它们通过：

  

```bash

--ro-bind

--ro-bind-try

```

  

挂载为只读，使沙箱内命令可以使用系统程序、动态链接库、证书和 DNS 配置，但不能修改这些文件。

  

### 4. 创建运行时目录

  

```bash

--proc /proc

--dev /dev

--tmpfs /tmp

```

  

作用：

  

- 创建沙箱内的 `/proc`。

- 创建受限 `/dev`。

- 使用独立、临时的 `/tmp`。

  

### 5. 隐藏配置目录

  

假设：

  

```text

workspace = /home/user/.nanobot/workspace

```

  

那么：

  

```python

ws.parent

```

  

是：

  

```text

/home/user/.nanobot

```

  

代码执行：

  

```bash

--tmpfs /home/user/.nanobot

```

  

用空的临时文件系统遮盖真实 `.nanobot` 目录，因此：

  

```text

config.json

pairing.json

其他同级运行数据

```

  

默认不会出现在沙箱视图中。

  

### 6. 重新暴露 workspace

  

父目录被遮盖后，代码先创建 workspace 挂载点：

  

```bash

--dir /home/user/.nanobot/workspace

```

  

再把真实 workspace 读写挂载回来：

  

```bash

--bind \

/home/user/.nanobot/workspace \

/home/user/.nanobot/workspace

```

  

所以沙箱内仍然存在相同路径：

  

```text

/home/user/.nanobot/workspace

```

  

它不是 workspace 的副本，而是宿主机真实目录的 bind mount。

  

### 7. 媒体目录只读

  

媒体目录通过：

  

```bash

--ro-bind-try media media

```

  

挂载，因此命令可以读取上传附件，但不能修改附件目录。

  

### 8. 执行原始命令

  

最后：

  

```bash

--chdir <sandbox_cwd>

-- sh -c <command>

```

  

表示在沙箱内切换到工作目录，再由 `sh` 执行原始命令。

  

`shlex.join(args)` 负责安全拼接参数和引号，返回完整命令字符串。

  

## 十三、沙箱中的 Workspace

  

### 1. 沙箱中是否存在 Workspace

  

存在，而且通常与宿主机路径相同：

  

```text

宿主机：

/home/user/.nanobot/workspace

  

沙箱：

/home/user/.nanobot/workspace

```

  

原因是：

  

```bash

--bind 宿主路径 沙箱路径

```

  

源路径和目标路径使用同一个字符串。

  

### 2. 修改是否影响真实文件

  

会影响。

  

因为 `--bind` 是读写绑定挂载，不是复制：

  

```text

沙箱创建文件

→ 宿主 workspace 出现文件

  

沙箱修改文件

→ 宿主文件被修改

  

沙箱删除文件

→ 宿主文件被删除

```

  

因此 bwrap 的安全目标不是保护 workspace，而是：

  

```text

允许 Agent 正常修改 workspace，

同时尽量隐藏和保护 workspace 外的宿主机文件。

```

  

如果 workspace 自身含有 `.env`、私钥或其他秘密，沙箱内命令仍然可以读取它们。

  

## 十四、bwrap 可以隔离什么

  

当前实现主要提供文件系统视图隔离。

  

可以保护：

  

- `~/.nanobot/config.json` 等 workspace 同级文件。

- 没有挂载进沙箱的用户私人文件。

- `~/.ssh` 等宿主机敏感目录。

- `/usr`、`/bin`、`/lib` 等系统目录的写入。

- 宿主机真实 `/tmp`。

- 部分设备文件。

- 父进程退出后的遗留子进程。

  

可以正常使用：

  

- workspace，读写。

- media，默认只读。

- 系统命令和动态库，只读。

- DNS 和证书相关文件，只读。

  

## 十五、bwrap 不能隔离什么

  

### 1. Workspace 内容

  

workspace 是读写挂载，沙箱不能防止：

  

- 删除项目文件。

- 修改源代码。

- 覆盖构建配置。

- 读取 workspace 内的秘密。

  

### 2. 网络

  

当前 `_bwrap()` 没有使用：

  

```bash

--unshare-net

```

  

因此沙箱内进程仍然使用宿主网络能力。

  

虽然 `ExecTool` 会检查命令文本中的内网 URL，但这仍是应用层检查，不能等同于网络 Namespace 隔离。

  

### 3. CPU、内存和磁盘资源

  

当前实现没有 cgroup 或资源配额，因此不能严格限制：

  

- CPU 使用量。

- 内存使用量。

- 写入 workspace 的磁盘量。

- 创建的进程数量。

  

命令超时可以终止运行过久的命令，但不等于完整资源治理。

  

### 4. 内核边界

  

bwrap 不是虚拟机，沙箱进程仍与宿主机共享 Linux 内核。它不能提供虚拟机级别的内核隔离。

  

![对比 bwrap 对宿主机文件的保护、Workspace 读写挂载的实际影响，以及网络和资源配额等未覆盖边界](https://files.seeusercontent.com/2026/06/15/o5cD/04-boundary-comparison.webp)

  

## 十六、网络 SSRF 防护

  

### 1. 阻止的地址

  

`security/network.py` 默认阻止：

  

```text

0.0.0.0/8

10.0.0.0/8

100.64.0.0/10

127.0.0.0/8

169.254.0.0/16

172.16.0.0/12

192.168.0.0/16

::1/128

fc00::/7

fe80::/10

```

  

包括：

  

- IPv4 私网。

- IPv6 本地地址。

- loopback。

- link-local。

- 云元数据常用地址。

- CGNAT 地址。

  

### 2. URL 校验流程

  

```text

解析 URL

↓

只允许 http / https

↓

解析 hostname

↓

DNS 查询所有地址

↓

任意地址命中 blocked networks

↓

拒绝请求

```

  

IPv6 映射的 IPv4 地址，例如：

  

```text

::ffff:127.0.0.1

```

  

会先规范化为 IPv4，避免绕过 blocklist。

  

### 3. 重定向

  

Web fetch 不会直接自动跟随无限重定向，而是每次检查下一个目标：

  

```text

请求 URL

↓

收到 3xx

↓

解析 Location

↓

再次执行 SSRF 校验

↓

最多 5 次

```

  

因此公开 URL 重定向到 `127.0.0.1` 或云元数据地址时也会被拒绝。

  

### 4. SSRF whitelist

  

配置可以显式放行 CIDR：

  

```json

{

"tools": {

"ssrfWhitelist": [

"100.64.0.0/10"

]

}

}

```

  

这适用于明确需要访问 Tailscale 等网络的场景，但会扩大网络访问边界。

  

## 十七、WebSocket 与服务暴露

  

### 1. 默认只监听本机

  

API、Gateway 和 WebSocket 默认 host 都是：

  

```text

127.0.0.1

```

  

这使服务默认不会直接暴露到局域网或公网。

  

### 2. WebSocket Token

  

WebSocket 支持：

  

- 静态 Token。

- 有过期时间的临时 Token。

- Token issue secret。

- `hmac.compare_digest()` 常量时间比较。

- TLS 证书配置。

- 最大消息大小限制。

  

当 host 设置为：

  

```text

0.0.0.0

::

```

  

配置校验要求必须提供 Token 或 Token issue secret，避免无认证绑定所有网络接口。

  

### 3. OpenAI 兼容 API

  

OpenAI 兼容 API 当前主要依赖：

  

```text

默认绑定 127.0.0.1

```

  

提供边界。代码中没有看到与 WebSocket 同等级别的内置 Bearer Token 中间件，因此不应把它未经反向代理认证直接暴露到公网。

  

## 十八、Session 数据保护

  

Session key 会经过安全文件名转换：

  

```python

safe_filename(key.replace(":", "_"))

```

  

保存 Session 时采用：

  

```text

写入 .jsonl.tmp

↓

可选 flush + fsync

↓

os.replace(tmp, 正式文件)

↓

可选 fsync 父目录

```

  

这种原子替换主要保护：

  

- 进程崩溃时不会留下半个正式文件。

- 正常关闭时可以将缓存写入持久存储。

- FUSE、NFS 等写回缓存环境中减少最近数据丢失。

  

它解决的是持久化完整性，不是文件内容加密。聊天历史和配置仍然是本地明文文件，需要依靠操作系统权限保护。

  

## 十九、Gateway 中安全配置的生命周期

  

Gateway 启动流程：

  

```text

nanobot gateway

↓

Typer 调用 gateway()

↓

加载 config.json

↓

_run_gateway()

↓

创建 MessageBus / Provider / SessionManager / Cron

↓

AgentLoop.from_config()

↓

AgentLoop.__init__()

↓

注册 Tool

↓

创建 ChannelManager

↓

asyncio.run(run())

```

  

长期任务：

  

```python

await asyncio.gather(

agent.run(),

channels.start_all(),

health_server,

)

```

  

所以主 Agent 的 `ToolRegistry` 和其中的 Tool 实例通常在整个 Gateway 进程期间一直存在。

  

普通聊天消息不会重新读取 `exec.sandbox` 并重建 `ExecTool`。配置更新后需要重新创建 AgentLoop，通常就是重启 Gateway。

  

## 二十、常见问题

  

### 1. 开启 `restrictToWorkspace` 后为什么还能访问外部目录？

  

常见原因有两个。

  

第一，当前 WebUI 会话保存了：

  

```json

"access_mode": "full"

```

  

它会覆盖全局 workspace 限制。

  

第二，Shell 检查是命令文本分析，复杂 Shell 展开可能绕过显式路径提取。

  

### 2. `restrictToWorkspace` 能代替沙箱吗？

  

不能。

  

它对文件工具的路径控制比较直接，但 Shell 具有变量展开、子 Shell、命令替换等复杂语法，应用层正则无法覆盖所有形式。

  

### 3. 配置了 `sandbox` 后是否还需要 `restrictToWorkspace`？

  

建议同时开启。

  

当前文件工具在检测到 `exec.sandbox` 时会自动限制 workspace，但显式配置：

  

```json

"restrictToWorkspace": true

```

  

能更清楚地表达安全策略，也能保证没有通过 Shell 的其他文件工具继续受到限制。

  

### 4. 沙箱是否保护项目文件？

  

不保护。

  

workspace 是读写 bind mount。沙箱的目标是保护宿主机其他目录，不是保护项目目录不被 Agent 修改。

  

### 5. 沙箱是否完全断网？

  

不会。

  

当前没有 `--unshare-net`，沙箱仍可访问网络。

  

### 6. macOS 可以使用 bwrap 吗？

  

不能直接使用。

  

Bubblewrap 依赖 Linux Namespace。macOS 可以通过 Docker Desktop 或 Linux 虚拟机间接运行 Linux 环境中的 bwrap。

  

### 7. 最严格的 Shell 策略是什么？

  

如果不需要命令执行，直接关闭：

  

```json

{

"tools": {

"exec": {

"enable": false

}

}

}

```

  

这会让 `ToolLoader` 不注册 `ExecTool`，从根源上移除 Shell 攻击面。

  

## 二十一、推荐配置

  

### 1. Linux 开发环境

  

```json

{

"tools": {

"restrictToWorkspace": true,

"exec": {

"enable": true,

"sandbox": "bwrap"

}

}

}

```

  

并保证：

  

- 使用非 root 用户运行。

- workspace 中不保存长期密钥。

- 配置文件权限为 `0600`。

- 聊天使用 Default Access。

  

### 2. macOS 本机

  

如果需要 Shell：

  

```json

{

"tools": {

"restrictToWorkspace": true,

"exec": {

"enable": true,

"sandbox": ""

}

}

}

```

  

此时只有应用层限制，不应把它视为严格沙箱。

  

需要更强隔离时，应在 Docker 或 Linux VM 中运行。

  

如果不需要 Shell：

  

```json

{

"tools": {

"restrictToWorkspace": true,

"exec": {

"enable": false

}

}

}

```

  

## 二十二、源码阅读入口

  

建议按下面顺序阅读。

  

### 1. 配置与 Gateway

  

```text

config/schema.py

ToolsConfig

ExecToolConfig

  

cli/commands.py

gateway()

_load_runtime_config()

_run_gateway()

```

  

### 2. Tool 加载与调用

  

```text

agent/loop.py

AgentLoop.__init__()

_register_default_tools()

  

agent/tools/loader.py

ToolLoader.discover()

ToolLoader.load()

  

agent/tools/registry.py

register()

prepare_call()

execute()

  

agent/tools/base.py

Tool

tool_parameters()

to_schema()

validate_params()

```

  

### 3. Workspace 边界

  

```text

security/workspace_access.py

WorkspaceScope

current_tool_workspace()

  

security/workspace_policy.py

resolve_allowed_path()

is_path_within()

  

agent/tools/filesystem.py

_FsTool.create()

_FsTool._resolve()

```

  

### 4. Shell 与 Sandbox

  

```text

agent/tools/shell.py

ExecTool.create()

execute()

_prepare_command()

_guard_command()

_spawn()

  

agent/tools/sandbox.py

wrap_command()

_bwrap()

```

  

### 5. 网络与 Channel

  

```text

security/network.py

validate_url_target()

contains_internal_url()

  

agent/tools/web.py

_get_with_safe_redirects()

  

channels/base.py

is_allowed()

_handle_message()

  

pairing/store.py

generate_code()

approve_code()

```

  

## 二十三、一句话总结

  

nanobot 当前的安全模型可以概括为：

  

```text

Channel allowFrom 和 pairing 控制谁能访问；

workspace scope 和路径解析限制文件工具；

Shell guard 拦截明显危险命令；

bwrap 在 Linux 内核层隐藏 workspace 外的文件系统；

SSRF 校验阻止工具访问内网和云元数据；

但 workspace 本身、网络和资源消耗仍需要额外保护。

```