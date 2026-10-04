# Laya Workbench

[中文](README.md) | **English**

A local workbench and one-click installer for decision models, for Windows and Linux, with a Chinese and an English interface.

[Laya](https://github.com/NandhaKishorM/laya) and TypeSafe's Jev are decision models: you give them some material (state) and a few questions, and they return a probability distribution for each question instead of generated text. This project bundles the installation and start-up of local Laya with a web workbench, so the same request can be sent to the local model or to a remote Jev interface.

![Workbench](docs/screenshot.en.png)

## Features

- One-click install: creates a virtual environment and installs PyTorch (matched to your GPU) and `laya[serve]`
- Automatic updates: every start checks for a new Laya version and upgrades to it; if the new version fails to start, it is rolled back
- Web workbench: material as text, JSON object or message list; yes/no, choice and score questions
- Several interfaces: Auto, local Laya, TypeSafe (official), aiask.me (accelerated), and any third-party service that offers `/v1/systemone`
- Keys are added, changed and deleted on the page and stored only on your machine
- Automatic fallback when a remote interface fails; batch requests are split into single calls for remote interfaces
- Chinese and English interface, switchable at any time from the top right of the page
- 8 built-in examples in each language; add your own by dropping JSON files into `examples/`
- Result view, raw response, request JSON, code snippets (curl / Python / JavaScript), history and a debug panel
- The launcher uses only the Python standard library

## Install and start

Download the latest package from the [Releases](../../releases) page.

### Windows

1. Extract `laya-workbench-*.zip` to a permanent location, for example `D:\laya-workbench`
2. Double-click `install.bat` and wait for it to finish
3. Double-click `start.bat`; the browser opens <http://127.0.0.1:8090>

### Linux

```bash
tar -xzf laya-workbench-*.tar.gz
cd laya-workbench
bash install.sh
bash start.sh
```

Python 3.10 or newer is required. On Debian / Ubuntu run `sudo apt install python3 python3-venv` first.

On a server without a desktop, open an SSH tunnel from your own computer and then browse to <http://127.0.0.1:8090> locally:

```bash
ssh -L 8090:127.0.0.1:8090 user@server
```

The first start downloads the local model (multilingual is about 650 MB). You can run requests once the page shows "Local Laya ready". Closing the launcher window, or pressing Ctrl+C, stops everything.

### Remote interfaces only

If you do not want the local model on a machine, copy `config.example.json` to `config.json`, set `install_local_laya` and `start_local_laya` to `false`, then run the installer. PyTorch and Laya are skipped and the install takes a few seconds.

## Automatic updates

Every time you run `start.bat` / `start.sh`, two things are checked before the local model starts:

- **The Laya program**: compared with the latest version on PyPI. If there is a newer one, it is installed with pip before the service starts.
- **The model files**: compared with the latest commit on Hugging Face. Laya downloads the newest files itself when it loads the model; the check only tells you in advance whether there is an update.

The outcome is printed in the launcher window, and the page shows a line under the status at the top left, for example "Laya 0.3.26 (up to date), model revision e4e9ddf2".

How the special cases are handled:

- **No network**: the check is skipped and the current version is used; start-up is not affected.
- **The service fails to start after an upgrade**: the previous version is reinstalled and the service is started again. That version is then skipped until a newer one is released.
- **The new version needs a newer PyTorch**: PyTorch is pinned to its current version during the upgrade so that a GPU build is never replaced by a CPU build. In this case the upgrade fails with a message and the current version keeps running; run the installer again to resolve it.
- **A Laya service is already running on port 8000**: the check runs, but nothing is upgraded.

Controlled by `auto_update` in `config.json`: `true` (default, upgrade automatically), `"check"` (check and report only), `false` (no check).

## Language

- **Page**: switch with "中文 / English" at the top right; the choice is remembered. An example you have not edited is swapped for its version in the other language.
- **Installer and launcher window**: follow the system language by default. Set `language` in `config.json` to `zh` or `en` to fix it.
- **Error messages from the service**: follow the language the page is using.

## Interfaces

The "Interface" drop-down at the top left of the page:

| Interface | Where requests go | Key |
|---|---|---|
| Auto (recommended) | Decided by the model name, see below | The key of whichever interface is used |
| Local Laya | `127.0.0.1:8000` on this machine | None |
| TypeSafe (official) | `https://api.typesafe.ai` | A key from the TypeSafe console |
| aiask.me (accelerated) | `https://aiask.me` | An aiask.me key |
| Third-party interface | The address you enter | That service's key; may be left empty |

**How Auto works**

- Model name is `multilingual`, `english`, `typed-decisions`, or empty: local Laya is used
- Any other model name (such as `jev-latest`): the first remote interface in `auto_order` that has a key is used (aiask.me first, then TypeSafe, by default). If it is unreachable, times out or refuses the request (401 / 403 / 404 / 429 / 5xx), the next one is tried

The result view shows which interface answered, and any fallback that happened.

**Managing keys**

"Interface settings" shows only the interface that is selected under "Interface", so the keys never appear side by side:

- No key yet: paste it and click "Add key"
- Key saved: only its last 4 characters are shown; use "Change" or "Delete" (deleting asks for confirmation)
- "Test connection" fetches the model list with the current key, which confirms that the address and key work

Keys are written only to `config.json` on your machine and added to requests by the local launcher; requests sent by the browser never contain them. For TypeSafe and aiask.me the environment variables `TYPESAFE_API_KEY` and `AIASK_API_KEY` work too.

`config.json` is listed in `.gitignore` and is never committed.

**Third-party interfaces**

Pick "+ Add a third-party interface…" in the "Interface" drop-down and enter a name, base URL, key and default model. Any address that serves the `/v1/systemone` format works, for example another gateway, or a Laya service on another machine in your network.

- Enter only the host or path prefix; the workbench appends `/v1/systemone`. A full endpoint URL is accepted too and its ending is removed
- The key may be left empty, for internal services without authentication
- After adding, the name, base URL, default model and key can be changed, and the interface can be deleted
- A third-party interface is used only when selected and is not part of Auto. To include it, add its id from `config.json` (such as `custom-1`) to `auto_order`

**Differences between remote interfaces and local Laya**

- Remote interfaces have no batch endpoint. In batch mode the workbench calls `/v1/systemone` once per item and merges the results; you are billed per item
- `max_len` only applies to local Laya; for remote interfaces the confidence threshold is applied locally by the workbench
- Remote requests follow the system proxy by default; set `proxy` in `config.json` to use a specific one

## Examples

`examples/zh/` and `examples/en/` each hold one set of examples; the page shows the set for the current language. Every `.json` file is a complete request body and becomes one example button:

| File | What it shows |
|---|---|
| `01-ticket-routing.json` | Ticket routing: department, urgency, churn risk |
| `02-content-moderation.json` | Content moderation (batch) |
| `03-rag-relevance.json` | RAG passage relevance |
| `04-llm-output-check.json` | LLM output check |
| `05-email-triage.json` | Email triage (JSON object material) |
| `06-prompt-guard.json` | Prompt guard (message list material) |
| `07-model-routing.json` | Model routing (batch) |
| `08-confidence-gate.json` | Confidence gate |

Add a file in the same format and refresh the page to get another button. `_title` is the button label, `_note` a one-line description, and the remaining fields are the request. Files placed directly in `examples/` (not in a language folder) are shown in both languages.

The files can also be sent straight to local Laya with curl (fields starting with an underscore are ignored):

```bash
curl -s http://127.0.0.1:8000/v1/systemone \
  -H "Content-Type: application/json" \
  --data-binary "@examples/en/01-ticket-routing.json"
```

In Windows PowerShell write `curl.exe`. The two batch examples, which contain `states`, go to `/v1/systemone/batch`.

## Configuration

`config.json` is created from `config.example.json` on the first install or start. Restart after changing it.

| Setting | Meaning |
|---|---|
| `language` | `auto` (follow the system) / `zh` / `en`: language of the installer and launcher window, and the page's default language |
| `ui_host` / `ui_port` | Address and port of the workbench, `127.0.0.1:8090` by default |
| `open_browser` | Open the browser automatically after starting |
| `install_local_laya` | `false` = skip PyTorch and Laya during installation |
| `start_local_laya` | `false` = do not start the local model |
| `laya_host` / `laya_port` | Address of the local Laya service, `127.0.0.1:8000` by default |
| `models` | Models to preload, comma-separated: `multilingual,english,typed-decisions` |
| `default_model` | Model used when the language cannot be detected |
| `device` | `auto` / `cuda` / `cpu` |
| `api_key` | Access key for local Laya, normally empty |
| `auto_update` | `true` = upgrade Laya automatically at every start; `"check"` = check and report only; `false` = no check |
| `providers` | Remote interfaces: names, base URL, key, default model |
| `auto_order` | Order in which Auto tries remote interfaces |
| `proxy` | Proxy for remote interfaces; empty = follow the system |
| `remote_timeout` | Timeout for remote requests, in seconds |
| `hf_endpoint` | Where models are downloaded from. `auto` = hf-mirror.com in a Chinese environment, Hugging Face directly otherwise; a URL or an empty value also works |
| `pip_index` | pip index used by the installer and by automatic updates. `auto` = the Tsinghua mirror in a Chinese environment, the official index otherwise; a URL or an empty value also works |
| `torch_index` | A specific PyTorch index; normally empty |
| `desktop_shortcut` | Create a desktop shortcut when installing on Windows |

> With `ui_host` set to `0.0.0.0`, anyone who can reach the port can send requests with your saved keys. Do this only on a trusted network; otherwise keep `127.0.0.1` and use an SSH tunnel.

## Troubleshooting

**A remote interface returns 401 / 403**
The key is wrong, or it has no access to the selected model. "Test connection" under Interface settings lists the models the key can use.

**The aiask.me address is wrong**
Requests go to `https://aiask.me/v1/systemone` by default. If your address differs, change the base URL (without `/v1/systemone`) under Interface settings and save.

**The page keeps showing "Local Laya is loading the model"**
Look at the launcher window: if the model is downloading, keep waiting; if there is an error, deal with that error.

**A port is in use**
If a Laya service is already running on port 8000, the workbench uses it. If another program holds the port, change `laya_port` in `config.json`.

**There is an NVIDIA GPU but the device shows cpu**
The CPU build of PyTorch was installed. Pick the command for your CUDA version at <https://pytorch.org/get-started/locally/>, reinstall torch with the virtual environment's `python -m pip`, and restart.

**The installation failed half-way**
Fix the problem and run the installer again; finished steps are skipped.

## Layout

```
install.bat / install.sh   install
start.bat / start.sh       start
config.example.json        configuration template
examples/zh/  examples/en/ example requests (Chinese / English)
app/workbench.html         the workbench page (both languages)
app/launcher.py            launcher: starts the local service, serves the page, forwards requests
app/install.py             installation logic
app/wb_lang.py             language selection
```

## Releasing a new version

Push a tag starting with `v`; GitHub Actions builds the packages and creates the release:

```bash
git tag v1.4.0
git push origin v1.4.0
```

## Uninstall

Delete the folder; on Windows also delete the "Laya Workbench" desktop shortcut. Models are cached under `.cache/huggingface` in your home directory and can be deleted as well.

## License

The code in this project is released under the MIT license. The Laya models and the TypeSafe and aiask.me services are subject to their own licenses and terms.
