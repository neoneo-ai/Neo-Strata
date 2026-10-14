# Neo - Strata 的桌面管理层

**中文** · [English](README.md)

<p align="center"><b>在你自己的游戏电脑上跑 1250 亿参数的模型——并且用一个窗口管着它。</b><br>
NVIDIA 或 AMD 显卡（12 GB 及以上）· Windows · 免费开源</p>

<p align="center"><a href="https://github.com/Niko1221/Strata/releases/download/v0.1.10/Pagoda.mp4"><img src="https://raw.githubusercontent.com/Niko1221/Strata/main/docs/media/pagoda-preview.webp" width="720" alt="Strata 的模型写出的体素宝塔花园，在浏览器中运行"></a><br>
<sub>体素宝塔花园，一次提示生成，在 RTX 5070 上用 Strata 运行（IQ3_S，128K 上下文）·
<a href="https://github.com/Niko1221/Strata/releases/download/v0.1.10/Pagoda.mp4">完整视频（49 秒）</a></sub></p>

## 你能拿到的速度

这些数字是上游 Strata 自己在两台普通游戏电脑上测的。一个 token 大约相当于 ¾ 个英文单词。
**写回答**：短对话里回复出现的速度——每秒 60 个 token 已经比你的阅读速度还快。
**读提示**：读入你发送内容的速度（这里是一份 32K token 的文档、代码或聊天记录）。

| 规格 | RTX 5070 · 写回答 | RTX 5070 · 读提示 | RX 9070 XT · 写回答 | RX 9070 XT · 读提示 |
|---|---:|---:|---:|---:|
| **Q2_0** | 94 tokens/s | 2,650 tokens/s | 60 tokens/s | 1,160 tokens/s |
| **IQ2_XS** | 79 tokens/s | 2,090 tokens/s | 52 tokens/s | 1,110 tokens/s |
| **IQ3_XXS** | 62 tokens/s | 1,750 tokens/s | — | — |
| **IQ3_S** | 53 tokens/s | 1,620 tokens/s | — | — |
| **Coder** | 55 tokens/s | 2,180 tokens/s | 44 tokens/s | 1,420 tokens/s |

RTX 5070（12 GB）配 Ryzen 5 7600、64 GB 内存；RX 9070 XT（16 GB）配 Ryzen 9 3900X、47 GB 内存。
Q2_0 用的是引擎 0.1.36，其他各行用的是 0.1.26（4K 回答、32K 提示）。显存越大的显卡越快：
RTX 3090（24 GB）写回答应该能达到每秒约 100-140 个 token。同批测试的完整表格见 Strata 的
[DETAILS.md](https://github.com/Niko1221/Strata/blob/main/docs/DETAILS.md#speed-measured)、
[各模型的速度](https://github.com/Niko1221/Strata/blob/main/docs/MODELS.md#how-fast-is-each-size)、
[社区测试结果](https://github.com/Niko1221/Strata/blob/main/docs/COMMUNITY_BENCHMARKS.md)。
Neo 不复测这些数：它按 pin 提供引擎源码（今天 `v0.1.43.1`），页面上显示的是跑着的引擎报的数。

## Neo 加的是什么

窗口套在这台引擎外面：进程监督（**崩了不自动重启**，只记退出码 + 一句诊断，等用户点「启动」）、
状态/日志/聊天/监控页面，外加"装模型"和"接编码 agent"两件事。它只通过上游已有的命令行参数、
环境变量和 HTTP 端点和 Strata 说话，**不改上游任何一个文件**。默认永远是 `127.0.0.1` 且不带
API key；token、局域网地址、agent 配置文件全是 opt-in，不点就什么都不写。

上游是 [Niko1221/Strata](https://github.com/Niko1221/Strata)（本仓库的 remote `upstream`）。引擎
源码只从 GitHub 的 tag 来：钉在 `neo/engine_ref.json` 里（今天 `v0.1.43.1` =
`a0116b2782a5b82622358c04a06496485d12b2d6`）。

> 完整设计文档（含实测数据）：[docs/zh/NEO_DESKTOP.zh-CN.md](docs/zh/NEO_DESKTOP.zh-CN.md)
> （英文版：[docs/NEO_DESKTOP.md](docs/NEO_DESKTOP.md)）。开发约定见 [AGENTS.md](AGENTS.md)。

## 快速上手

### 这台机器已经装好 Strata？把目录给 Neo 就行——什么都不用复制

```bash
python -m neo.engine_setup --set-root D:\Work\Strata   # 记一次（写进 neo/state/launcher.json 的 root）
python -m neo.manager                                   # 开窗口
```

上手就是这两行。之后每条 Neo 命令和窗口用的都是这个目录。只想单次指定：
`python -m neo.manager --root D:\Work\Strata`，或 `set NEO_STRATA_ROOT=D:\Work\Strata`；窗口里就是
接入页"用外置的引擎目录"那一格。Neo 跑一份现成的安装**不往它里面写任何东西**——自己的状态全在
`neo/state/` 下。那棵树的 HEAD 与 pin 不一样时 Neo 只说、不动它的 HEAD，要动得显式 `--align`。
下面那条"部署"不是用现成安装的方法——它是把窗口版 exe 送进那棵树，让别人双击就能用。

### 还没装？Neo 替上游的安装器跑腿

```bash
python -m neo.engine_setup --ensure                     # 按 pin 把引擎源码浅克隆进 strata/（已 gitignore）
python -m neo.wizard --check                            # 这台 PC 的情况 + 该装哪个模型（只读，不装）
python -m neo.wizard --family coder --source auto       # 跑上游 setup.py --yes --no-start
python -m neo.manager                                   # 开窗口
```

引擎要的东西（上游的口径）：12 GB 及以上的 NVIDIA/AMD 显卡、32 GB 内存（64 GB 什么规格都跑得动）、
约 80 GB 空闲磁盘、Windows 10 / 11。venv、依赖、引擎二进制、下模型全是上游 `setup.py` 的事——
Neo 只跑它、只用 ast 读它的模型表。

### Neo 认哪个目录

一条规则，`neo/paths.py::strata_root()`：`--root` > `NEO_STRATA_ROOT` > 记住的引擎目录
（`neo/state/launcher.json` 的 root）> 应用目录本身（Neo 就放在 Strata 根目录里时）> `<app>/strata` >
都没有就回应用目录并说明"这里还没有引擎"，接入页会接手去装。测试走的是同一条规则
（`neo/tests/_bootstrap.py` 里的 `ENGINE`），所以"测试全绿"说的就是"测试跑在 Neo 真会用的那个
引擎根上"。

## 开发（在这个仓库里）

```bash
python -m neo.engine_setup --ensure      # 按 pin 把引擎源码浅克隆进 strata/（已 gitignore）
python -m neo.wizard --check             # 只读：这台 PC + 该装哪个模型
python -m neo.tests.run_all --quick --skip-network
python -m neo.webcheck
```

### 测试

| 跑法 | 实测（2026-10-14，同一台机器） |
|---|---|
| `python -m neo.tests.run_all --quick --skip-network` | 9.6 s，435 测试，86 个并行单元 |
| `python -m neo.tests.run_all --skip-network` | 16.3 s，523 测试，101 个并行单元 |
| `python -m neo.tests.run_all`（含真克隆那一条） | 真克隆那一条走网络——同一台机器同一天量到 6.8 s 与 145.3 s；30 秒标准靠 `--skip-network` 守住 |

`run_all` 把每个模块拆成测试类并行跑（`--jobs`，默认 `min(4, 核数)`），失败的类单独重跑一次，
所以端口/负载引起的偶发失败会和真失败分开报。测量用的探针在 `neo/tests/probes/`
（`probe_cost.py`：进程启动与就绪的代价；`supervisor_events.py`：监督器事件时间线）。

### 页面

`neo/web/` 里两个 `<script>` 共用一个全局作用域，同名声明会让后一个文件永不执行——窗口开得到、
每个点击都是死的。`python -m neo.webcheck` 按加载顺序把两个文件当一个程序解析（约 1 s），
`build_exe` 打包前会先跑它。

## 部署到 Strata 树（可选）

```bash
python -m neo.deploy --to D:\Work\Strata            # 先看要写什么（dry-run）
python -m neo.deploy --to D:\Work\Strata --apply    # 写：只写 neo/** + NEO-START.bat + NeoStrata.exe
python -m neo.build_exe                             # 打窗口版 exe（先跑 webcheck，缺包会一次报全）
```

部署树是上游 Strata 的 git 检出，Neo 的文件在那儿全是 untracked；闸门：那边 HEAD 等于 pin、
跟踪文件干净、没有该树的 `NeoStrata.exe` 在跑。换版要换三样：`NeoStrata.exe`、`neo/` 源码、
`neo/web/` 页面。细节见 [docs/zh/NEO_DESKTOP.zh-CN.md](docs/zh/NEO_DESKTOP.zh-CN.md)。