# Laya 工作台

**中文** | [English](README.en.md)

判定模型的本地工作台和一键安装包，支持 Windows 和 Linux，界面有中文和英文两个版本。

[Laya](https://github.com/NandhaKishorM/laya) 和 TypeSafe 的 Jev 都是判定模型：给它一段素材（state）和几道题，它返回每道题的概率分布，而不是生成文本。这个项目把本地 Laya 的安装、启动和一个网页工作台打包在一起，同一份请求可以发给本地模型，也可以发给远程的 Jev 接口。

![工作台界面](docs/screenshot.png)

## 功能

- 一键安装：自动建虚拟环境，按显卡情况安装 PyTorch 和 `laya[serve]`
- 自动更新：每次启动检查 Laya 有没有新版本，有就自动升级，升级后起不来会自动回退
- 网页工作台：素材支持文本、JSON 对象、消息列表；题目支持是非、单选、打分
- 多种接口：自动、本地 Laya、TypeSafe 官方、aiask.me 网络加速，以及任何提供 `/v1/systemone` 的第三方接口
- 密钥在页面上添加、修改、删除，只保存在本机
- 远程接口失败时自动切换，批量请求自动拆成逐条调用
- 中文、英文两种界面，页面右上角随时切换
- 8 个内置示例（中英文各一套），往 `examples/` 里放 JSON 文件即可增加
- 结果、原始响应、请求 JSON、代码片段（curl / Python / JavaScript）、历史记录、调试面板
- 启动器只用 Python 标准库，没有额外依赖

## 安装和启动

从 [Releases](../../releases) 页面下载最新的安装包。

### Windows

1. 解压 `laya-workbench-*.zip` 到一个固定位置，例如 `D:\laya-workbench`
2. 双击 `install.bat`，等它跑完
3. 双击 `start.bat`，浏览器自动打开 <http://127.0.0.1:8090>

### Linux

```bash
tar -xzf laya-workbench-*.tar.gz
cd laya-workbench
bash install.sh
bash start.sh
```

需要 Python 3.10 或更高版本。Debian / Ubuntu 先执行 `sudo apt install python3 python3-venv`。

没有桌面的服务器，在自己的电脑上开一个 SSH 隧道，再用本机浏览器打开 <http://127.0.0.1:8090>：

```bash
ssh -L 8090:127.0.0.1:8090 用户名@服务器地址
```

首次启动要下载本地模型（multilingual 约 650 MB）。页面左上方显示「本地 Laya 已就绪」后就可以运行。关闭启动窗口或按 Ctrl+C 即停止全部服务。

### 只用远程接口

不想在这台机器上装本地模型时，先把 `config.example.json` 复制为 `config.json`，把 `install_local_laya` 和 `start_local_laya` 改成 `false`，再运行安装脚本。这样会跳过 PyTorch 和 Laya，几秒钟装完。

## 自动更新

每次运行 `start.bat` / `start.sh`，启动本地模型之前会先检查两样东西：

- **Laya 程序**：和 PyPI 上的最新版本比较，有新版本就自动用 pip 升级，然后再启动。
- **模型文件**：和 Hugging Face 上的最新提交比较。模型文件由 Laya 在加载时自己下载最新版本，这里只是提前告诉你有没有更新。

检查结果会显示在启动窗口里，页面左上方状态行下面也有一行，例如「Laya 0.3.26（已是最新）　模型版本 e4e9ddf2」。

几种情况的处理：

- **连不上网**：跳过检查，继续用当前版本，不影响启动。
- **升级后服务起不来**：自动装回原来的版本并重新启动，同时记住跳过这个版本，等更新的版本发布再升级。
- **新版本需要更新 PyTorch**：升级时会把 PyTorch 固定在当前版本，避免 GPU 版被换成 CPU 版。遇到这种情况升级会失败并提示，继续使用当前版本；重新运行安装脚本即可处理。
- **8000 端口上已有 Laya 服务在运行**：只检查，不升级。

用 `config.json` 的 `auto_update` 控制：`true`（默认，自动升级）、`"check"`（只检查并提示，不升级）、`false`（不检查）。

## 语言

- **页面**：右上角的「中文 / English」随时切换，选择会记住。切换时，没改动过的示例会一起换成对应语言的版本。
- **安装和启动窗口**：默认跟随系统语言。要固定成某种语言，把 `config.json` 的 `language` 改成 `zh` 或 `en`。
- **服务端返回的报错**：跟随页面当前的语言。

## 接口

页面左上角的「接口」下拉框：

| 接口 | 发到哪里 | 密钥 |
|---|---|---|
| 自动（推荐） | 按模型名决定，规则见下 | 用到哪个接口就需要哪个的密钥 |
| 本地 Laya | 本机 `127.0.0.1:8000` | 不需要 |
| TypeSafe 官方 | `https://api.typesafe.ai` | TypeSafe 控制台的密钥 |
| aiask.me 网络加速 | `https://aiask.me` | aiask.me 的密钥 |
| 第三方接口 | 你填写的地址 | 该服务的密钥，没有可以不填 |

**「自动」的规则**

- 模型名是 `multilingual`、`english`、`typed-decisions`，或者留空：用本地 Laya
- 其他模型名（如 `jev-latest`）：按 `auto_order` 的顺序（默认先 aiask.me，再 TypeSafe 官方）找第一个填了密钥的远程接口；它连不上、超时或被拒绝（401 / 403 / 404 / 429 / 5xx）时，自动换下一个再试

结果区会显示这次实际用的是哪个接口，以及换过接口的经过。

**管理密钥**

在「接口」里选中哪个接口，「接口设置」里就只显示那一个接口的设置，不会混在一起：

- 没有密钥时：粘贴密钥，点「添加密钥」
- 已有密钥时：只显示末 4 位，可以「修改」或「删除」（删除前要再确认一次）
- 「测试连接」会用当前密钥取一次模型列表，用来确认地址和密钥是否可用

密钥只写进本机的 `config.json`，由本机启动器加到请求头上，浏览器发出的请求里没有密钥。TypeSafe 和 aiask.me 也可以用环境变量 `TYPESAFE_API_KEY`、`AIASK_API_KEY`。

`config.json` 已经写进 `.gitignore`，不会被提交。

**第三方接口**

在「接口」下拉框里选「＋ 添加第三方接口…」，填写名称、接口地址、密钥和默认模型。任何按 `/v1/systemone` 格式提供服务的地址都可以，例如别的网关，或者内网里另一台机器上的 Laya 服务。

- 接口地址只填到域名或路径前缀，工作台会自动加上 `/v1/systemone`；把完整地址贴进来也会自动去掉结尾
- 密钥可以不填，适合不需要鉴权的内网服务
- 添加后可以改名称、地址、默认模型和密钥，也可以整个删除
- 第三方接口只在被选中时使用，不参与「自动」。想让它参与，把它在 `config.json` 里的 id（如 `custom-1`）加进 `auto_order`

**远程接口和本地 Laya 的差别**

- 远程接口没有批量端点。选「批量」时工作台逐条调用 `/v1/systemone` 再合并结果，按条计费
- `max_len` 只对本地 Laya 生效；置信度阈值对远程接口由工作台在本地判断
- 访问远程接口默认跟随系统代理，要单独指定就填 `config.json` 的 `proxy`

## 示例

`examples/zh/` 和 `examples/en/` 里各有一套示例，页面按当前语言显示对应的一套。每个 `.json` 文件都是一个完整的请求体，对应一个示例按钮：

| 文件 | 内容 |
|---|---|
| `01-ticket-routing.json` | 工单分流：部门、紧急度、流失风险 |
| `02-content-moderation.json` | 内容审核（批量） |
| `03-rag-relevance.json` | RAG 段落相关性 |
| `04-llm-output-check.json` | LLM 输出校验 |
| `05-email-triage.json` | 邮件分拣（JSON 对象素材） |
| `06-prompt-guard.json` | 提示词护栏（消息列表素材） |
| `07-model-routing.json` | 模型路由（批量） |
| `08-confidence-gate.json` | 置信度门控 |

按同样格式再放一个文件进去，刷新页面就多一个按钮。`_title` 是按钮名，`_note` 是一句说明，其余字段就是请求内容。直接放在 `examples/` 下（不进语言子目录）的文件在两种语言下都会显示。

这些文件也可以直接用 curl 发给本地 Laya（以下划线开头的字段会被忽略）：

```bash
curl -s http://127.0.0.1:8000/v1/systemone \
  -H "Content-Type: application/json" \
  --data-binary "@examples/zh/01-ticket-routing.json"
```

Windows PowerShell 里要写 `curl.exe`。带 `states` 的两个批量示例要换成 `/v1/systemone/batch`。

## 配置

首次安装或启动时，会从 `config.example.json` 生成 `config.json`。修改后重新启动生效。

| 配置项 | 说明 |
|---|---|
| `language` | `auto`（跟随系统）/ `zh` / `en`，决定安装和启动窗口的语言，以及页面的默认语言 |
| `ui_host` / `ui_port` | 工作台监听地址和端口，默认 `127.0.0.1:8090` |
| `open_browser` | 启动后是否自动打开浏览器 |
| `install_local_laya` | `false` = 安装时跳过 PyTorch 和 Laya |
| `start_local_laya` | `false` = 启动时不拉起本地模型 |
| `laya_host` / `laya_port` | 本地 Laya 服务地址，默认 `127.0.0.1:8000` |
| `models` | 启动时预载的模型，逗号分隔：`multilingual,english,typed-decisions` |
| `default_model` | 判断不出语言时用哪个模型 |
| `device` | `auto` / `cuda` / `cpu` |
| `api_key` | 本地 Laya 的访问密钥，一般留空 |
| `auto_update` | `true` = 每次启动自动升级 Laya；`"check"` = 只检查并提示；`false` = 不检查 |
| `providers` | 远程接口：名称、地址、密钥、默认模型 |
| `auto_order` | 「自动」尝试远程接口的顺序 |
| `proxy` | 访问远程接口用的代理，留空 = 跟随系统 |
| `remote_timeout` | 远程请求超时秒数 |
| `hf_endpoint` | 模型下载地址。`auto` = 中文环境用 hf-mirror.com，其他环境直连 Hugging Face；也可以填具体地址或清空 |
| `pip_index` | 安装和自动更新用的 pip 源。`auto` = 中文环境用清华镜像，其他环境用官方源；也可以填具体地址或清空 |
| `torch_index` | 指定 PyTorch 下载源，一般留空 |
| `desktop_shortcut` | Windows 安装时是否创建桌面快捷方式 |

> 把 `ui_host` 改成 `0.0.0.0` 后，能访问这个端口的人都可以用你保存的密钥发请求。请只在可信内网这样做，否则保持 `127.0.0.1` 并用 SSH 隧道。

## 常见问题

**远程接口返回 401 / 403**
密钥不对，或者这个密钥没有所选模型的权限。在「接口设置」里点「测试连接」，能看到这个密钥可用的模型。

**aiask.me 的接口地址不对**
默认按 `https://aiask.me/v1/systemone` 调用。地址不同的话，在「接口设置」里改接口地址（不带 `/v1/systemone`）后保存。

**页面一直显示「本地 Laya 模型加载中」**
看启动窗口：正在下载模型就继续等；有报错就按报错处理。

**端口被占用**
8000 上已有 Laya 服务时工作台会直接使用它。被别的程序占用时，改 `config.json` 的 `laya_port`。

**有 NVIDIA 显卡但显示用的是 cpu**
装到的是 CPU 版 PyTorch。到 <https://pytorch.org/get-started/locally/> 选好 CUDA 版本，用虚拟环境里的 `python -m pip` 重装 torch 后重新启动。

**安装中途失败**
修复后重新运行安装脚本，已完成的步骤会自动跳过。

## 目录结构

```
install.bat / install.sh   安装
start.bat / start.sh       启动
config.example.json        配置模板
examples/zh/  examples/en/ 示例请求（中文 / 英文）
app/workbench.html         工作台页面（含中英文文案）
app/launcher.py            启动器：拉起本地服务、提供页面、转发请求
app/install.py             安装逻辑
app/wb_lang.py             语言选择
```

## 发布新版本

推送一个 `v` 开头的标签，GitHub Actions 会自动打包并创建 Release：

```bash
git tag v1.4.0
git push origin v1.4.0
```

## 卸载

删除整个文件夹；Windows 上再删掉桌面的「Laya 工作台」快捷方式。模型缓存在用户目录的 `.cache/huggingface` 下，不需要时可以一并删除。

## 许可

本项目的代码以 MIT 许可发布。Laya 模型和 TypeSafe、aiask.me 的接口各自遵循其自己的许可和服务条款。
