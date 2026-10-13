# AGENTS.md

Neo 是 Strata 的桌面管理层：窗口外壳、进程监督（**崩了不自动重启**，只记退出码 + 一句诊断，
等用户点「启动」）、状态/日志/聊天/监控页面，加"装模型"
和"接编码 agent"。它只通过上游已有的命令行参数、环境变量和 HTTP 端点与 Strata 说话，**不改上游任何
文件**，自己的状态全在 `neo/state/` 下。引擎是 Strata 的 C++/CUDA/HIP 推理引擎加 Python 服务
（`serve/server.py`，OpenAI/Anthropic 兼容 API + 网页应用）——引擎本体住在 `upstream`
（[Niko1221/Strata](https://github.com/Niko1221/Strata)），不在本仓库。

## 引擎从哪来

- 源码只从 GitHub 的 tag 来，钉在 `neo/engine_ref.json`（`url` / `tag` / `sha` / `zip_sha256`；
  今天 `v0.1.43.1` = `a0116b2782a5b82622358c04a06496485d12b2d6`）。`neo/engine_setup.py` 浅克隆
  那个 tag 到 gitignore 的 `strata/`，或先重算 `archive/refs/tags/<tag>.zip` 的 SHA-256、对上
  钉值才解包。镜像给的是仓库当前的文件，不是钉住的那一版（上游在 `setup.py` 的 `model_source()`
  里 `#908` 写了同样的理由）。
- 源码树叫 `strata/`，**不要叫 `engine/`**：上游的 `engine/` 是引擎*二进制*目录。
- 装 venv、装依赖、装引擎二进制、下模型都不归 Neo 做——全走上游 `setup.py`（`neo/wizard.py` 在
  后台跑 `setup.py --yes --no-start`，`neo/models.py` 用 `ast` 读它的 `MODELS`/`FAMILIES` 表，
  从不 import 或执行它）。

## 两个根

`neo/paths.py::strata_root()` 一条规则算出引擎根：`--root` > `NEO_STRATA_ROOT` > 目录里同时有
`setup.py` 和 `serve/server.py` > `<app>/strata` > 应用目录本身（附一句原因）。
`neo/tests/_bootstrap.py` 把 `REPO`（Neo 仓库根）和 `ENGINE`（同一条规则算出的引擎根）分开，
20 个测试模块全走它——所以"测试全绿"说的就是"测试跑在 Neo 真会用的那个引擎根上"。引擎根里没
引擎时，读上游文件的测试自己跳过并说明原因（引擎的真服务 import 不动时也一样，见
`_bootstrap.server_ready`）。

## 开发

- 测试：`python -m neo.tests.run_all`（按测试类并行，`--jobs` 默认 `min(4, 核数)`，失败的类单独
  重跑一次）。**时间标准（2026-10-13）：跑测试必须报时间，单元与套件的墙上时间都按 30 秒封顶**
  （`neo/tests/run_all.py` 的 `BUDGET_S`，`--timeout` 默认就是它，超预算的单元报 `OK(超预算)` 并让
  整跑判失败）；超过 30 秒要优化，不是把超时调大。实测 2026-10-13 同一台机器：
  `--quick --skip-network` **9.6 s / 435 测试 / 86 单元**；`--skip-network` **17.1 s / 523 测试 /
  101 单元**（最慢的两个单元 5.7 s `test_models.TestCatalog`、5.1 s `test_supervisor.TestCrashStops`；
  删掉退避重启之前最慢的是 `TestCrashRestart` 14.7 s，它杀四次进程、走完整条退避，全跑 18.3 s）。
  开发中跑 `--quick`（17 个不起真引擎进程的模块），部署前跑全的。
- 真克隆那一条（`test_engine_setup.TestRunAndPage`）测的是网络，同一台机器同一天量到 6.8 s 与
  145.3 s：全跑用 `--skip-network` 把它摘出去（30 秒标准就是这么守住的），要看它就单独跑那一类
  并放宽 `--timeout`。
- 引擎的真服务起不动时（引擎根目录的 `.venv` 没装上游的包），用真 `serve/server.py` 的那几条
  自己跳过并说清缺哪个包：`neo/tests/_bootstrap.py::server_ready`（ast 读它的 import，毫秒级，
  不起进程）。实测 2026-10-13：缺 `jinja2` 时那个进程 import 就死，监督器按退避一遍遍重启、
  测试按等待跑满，全跑从 16.3 秒变成 5 分钟——这条跳过规则就是为它加的。（那条按退避重启的链路
  当天就删了，见 `neo/supervisor.py` 开头第 2 条。）
- 监督器的测试拿 `neo/tests/mock_server.py` 当真服务跑（stdlib，答 `/health` 与 `/metrics`，
  形状就是 Neo 读的那些），起进程、就绪、杀进程、停着并给出诊断、点启动再起全是真的；spawn 0.27 s，对上
  `serve/server.py --engine mock` 的 0.27 s。`TestStartAndReady.test_start_to_running` 必须用真的
  `serve/server.py --engine mock`——"上游的就绪行进了日志"是 Neo/上游的接缝，不能让 mock 打包票。
- 测量探针在 `neo/tests/probes/`：`probe_cost.py`（进程启动与就绪的代价）、
  `supervisor_events.py`（监督器事件时间线）。要新数字就在仓库里跑，别在临时目录里量。
- 控制台端点一条条接线：`neo/console_server.py` 的 Handler 里一个 `self.server.X(...)` 调用，
  必须在 `ConsoleServer.start()` 里有对应的 `httpd.X = self.X`，否则请求死在 socketserver 的
  `handle_error` 里，页面只看到一个不动的 fetch（`/neo/engine` 2026-10-13 两条都缺过；
  `neo/tests/test_console.py` 现在盯着这份接线表并跑一次真 HTTP 往返）。
- 页面脚本共用一个全局作用域：`neo/web/` 两个 `<script>` 是一个程序，同名声明会让后一个文件永不
  执行——窗口开得到、每个点击都是死的。`python -m neo.webcheck` 约 1 s 查这个，`build_exe` 打包前
  跑它。
- CLI 打印前先 `wide_stdout()`（共用那一份在 `neo/cli.py`，`deploy` / `engine_setup` / `wizard` /
  `models` / `model_settings` / `connect` / `start` 的 `main()` 都调它）：git 输出按 UTF-8 读会带
  `U+FFFD`，cp936 控制台装不下（2026-10-13 之前 `--ensure` 就崩在这上面）；输出进管道时
  `sys.stdout.encoding` 从 65001 回落到 gbk，中文整片变乱码（同一条命令加 `-X utf8` 就是好的）。
- 命令行的 `--root` 不给就走 `neo/paths.py::resolve_root()`（`strata_root()` 那条规则的命令行版），
  与 `neo/manager.py` 起服务用的根是同一个目录：`wizard` / `models` / `model_settings` / `connect`
  都是。2026-10-13 拆仓库前 `wizard.py` 与 `models.py` 的默认根是硬写的"neo/ 上一层"，拆开后就变成
  没有 `setup.py` 的仓库根（`neo.wizard --check` 因此报"这里不是 Strata 的根目录"，还给出一条指向
  不存在的 `<仓库根>/setup.py` 的命令）；`neo/tests/test_paths.py::TestCliRootDefault` 钉着这条。
- 默认永远是 `127.0.0.1` 且不带 API key；token、局域网地址、写 agent 配置全是 opt-in，
  不点就什么都不写。token 形如 `sk-neo-<32 hex>`（`neo/tokens.py`），「复制 token」只对端地址是
  loopback 的请求回全串，且什么都不写。

## 部署

- 部署树（例如 `D:\Work\Strata`）是上游 Strata 的 git 检出：那边不提交，也不给它的 `origin` 加第二个
  remote，上游更新就是普通 `git pull`。Neo 的文件在那儿全 untracked（`NeoStrata.exe`、`neo/`、
  `NEO-START.bat`）。
- `python -m neo.deploy --to <tree>`（默认 dry-run，`--apply` 才写）。闸门：那边 HEAD 等于 pin、
  跟踪文件干净（`--untracked-files=no`，Neo 的文件按设计就是 untracked）、该树的 `NeoStrata.exe`
  没在跑；只写 Neo 清单里的路径。`--apply` 还会把 Neo 的四条路径追加进那边的 `.git/info/exclude`
  （只追加，绝不写进上游跟踪的 `.gitignore`；dry-run 只报告）。写完就地跑 `webcheck` 与快套件。
- 只同步 `.py` 曾把部署树的页面留在坏掉的 `t` / `applyLang` 那一版（白屏那次），所以换版要换三样：
  `NeoStrata.exe`、`neo/` 源码、`neo/web/` 页面。打包产物（`neo/build/`、`neo/dist/`）不进部署树。
- 打包：`python -m neo.build_exe`（spec `neo/NeoStrata.spec`），构建环境里要有 pywebview；缺构建包
  就拒绝打包并一次把缺的全报出来（含 pywebview——少了它 exe 会静默丢掉窗口，只开一个浏览器页）。
  exe 设计成住在 Strata 根目录里，并以那个目录为根。

## 上游侧的改动不放这里

对 `serve/server.py`、`setup.py` 这些上游文件的改动走 fork（`neoneo-ai/Neo-Strata`，改名后
`strata-upstream`），那儿是上游跟踪与提 PR 的基线。`neo/tests/test_upstream_api.py` 盯着上游的形状：
key 只管推理，监控的只读端点对端地址是 loopback 且 `Host` 对得上的请求放行；Neo 的补丁标记必须
保持不出现。嵌入的监控页读 Strata 一律经过 Neo（`neo/console_server.py` 在 `/strata/` 下代理上游页面
与只读端点：只 GET/HEAD、固定路径菜单、`POST` 405、推理路由不转发、上游不在 502、8 MB 与 5 s 上限，
Neo 自己带 key，页面看不见 key）。

## 文档风格

plain words、带"在什么上量的"的实测数字、没有测量就不写结论。细节见
[docs/NEO_DESKTOP.md](docs/NEO_DESKTOP.md)。
