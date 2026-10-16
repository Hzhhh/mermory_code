# Lessons（错误模式库，防止重犯）

> 规则：每次犯错后追加一条；会话开始时先读相关项目段落。

## AML 打榜项目（aml-system）

### 2026-09-24 战略节奏

9. **战略优先级由用户把舵，不要用工程倒排绑架战略节奏**
   - 事故：我把"Full 两次配额的 30 天间隔数学"推成 9.30 首提死线并抢跑部署线；用户纠正：官方 Full 只给分数无 badcase，首提平庸版本的校准价值有限，方案没拉满前不烧配额；部署链路风险用 Smoke（免费 30 次）管理即可。
   - 规则：正式资源（配额/费用/提交窗口）只服务"最强版本"；链路验证找零成本工具；死线数学只在方案成熟后用来选提交时点，不倒逼开发节奏。

### 2026-09-24 口径事故

10. **可选组件的静默降级必须变成响亮失败（跨机路径默认值是重灾区）**
    - 事故：dense.py 的 AML_DENSE_MODEL 默认值是本地 Windows 路径 `E:\models\...`，同步到 198 后被 sentence-transformers 当非法 repo id → dense 静默禁用 → **LME/pmem 的 v03/v2 四组跑批全是 BM25 单路**（日志里一行 `[dense] disabled` 才暴露）；BEAM e2 系列（orchestrator.log 无此错误）dense 正常。后果：LME 47.8 的"dense+注入v2"构成重新归因为"纯 BM25+注入v2"，dense 在 LME/pmem 的增益从未真实测过。
    - 规则：①组件失败≠跳过，跑批脚本必须前置"组件生效断言"（`available()` 不过就终止）；②代码里的路径默认值禁用机器特定绝对路径，用 HF id 等可移植值，机器路径一律环境变量显式注入；③每组跑批的产出验证要看日志里的 disabled/降级行，不只看退出码。

### 2026-09-28 组件验证

11. **注入类组件的预算检查必须在真实数据规模下测试（smoke 短数据会漏静默弃卡）**
    - 事故：时间线卡用 4 条短消息 smoke 全过；真实 LME 数据平均 1000 chars/条，30 条渲染 30K chars 顶爆 12K 预算 → 卡被静默丢弃（全量 build 0 卡），sanity 闸门（grep 卡出现率）才拦住。
    - 规则：①涉及长度/预算/截断的组件，smoke 必须用**真实长度分布**的数据（从 eval_input 抽真实条目）；②build 产物上做触发率 sanity（grep 组件标记 >N 才放行到 answer 阶段）已成本组件验证标准闸门；③静默降级（预算内丢弃/条件不满足返回 None）必须配可观测的触发统计。

12. **验证脚本重跑前必须清理旧 answers/judged（同日第二次踩）**
    - 事故 A（timeline）：两轮跑批逐题型完全一致，发现 answer 断点续跑把 500 题全跳过。事故 B（conflict）：旧 245 行 answers 残留，新 40 题全在 done set 里零追加，judge 报 ID mismatch。
    - 规则：所有验证脚本在 answer 前无条件 `rm -f` 本轮的 answers/judged 产物；断点续跑只服务于"中断恢复"，重跑语义下它是静默作弊。

### 2026-09-29 Full 三连败根因

13. **服务端"全 200 零错误"时瓶颈在系统之外；对照实验是最强诊断证据**
    - 事故：Full 三连 ADD_RUNTIME_ERROR。前两次归因到服务端性能（惰性 conflict、锁队列修复都真实有效但非根因）；第三次铁证：6157 add 全 200（0-8ms）、76 search 全 200、零 5xx/零重启/零 OOM，官方恰在 10m04s 停止发请求。用户指出同服务器同链路的 aml-coding 容器（8001）官方正在正常跑 3 小时+——对照组直接证明"服务器没坏、链路没断"，差异在官方对文本赛道用高并发批量灌库（实测 ~5 连接 10.2 req/s），对跨境链路 RTT(190ms)+抖动(P99 1.2s) 极敏感，官方客户端超时重试耗尽即判死。
    - 规则：①部署类失败先看"服务端视角是否无辜"（请求日志+5xx+重启+OOM 四件套），无辜则瓶颈在链路/对端编排，别再改业务代码；②找同环境对照组（本例：另一赛道的容器），运行差异即根因方向；③跨境部署的评测服务，提交前先从公网测 RTT/P99 抖动（scripts/wan_bench.py），P99>5×P50 视为危险；④BBR+TCP 缓冲调优是跨境服务器标配（P99 减半），但物理 RTT 只能靠换 region。
    - 衍生教训：断点续跑若官方全量重放历史 add，重放本身会耗尽阶段窗口 → 死循环（第三次 10 分钟 = 重放 5820 + 仅新推 337）；request_id 幂等只防重复写入，防不了重放耗时。换 endpoint 前必须向官方确认旧任务续跑是否仍有效。

### 2026-09-21 阶段 4 开发

1. **验证脚本的断言子串必须与生成格式逐字核对**
   - 事故：统计注入分布时搜 `'[LATEST VALUE RECORD]'`（带闭合括号），实际生成文本是 `'[LATEST VALUE RECORD | ...'`——误判"40/40 全部注入失败"，差点推翻正确的实现去"修 bug"，实际实现一直是好的。
   - 规则：出现"功能全失败"类负结果时，**先用生成侧的最小样本肉眼核对验证器的匹配串**，再怀疑实现。

2. **停词表/黑名单改动前，先过一遍目标域的高频查询词**
   - 事故：把 target/goal/deadline/week 放进 STOP（原意是防 marker 词混入主题），恰好杀掉 BEAM knowledge_update 题的核心主题锚点（"weekly word **count target**"），导致 topic 匹配失败、LATEST 不注入。
   - 规则：改任何词表后，跑一遍"目标题型的典型 query vs 主题词"重叠检查。

3. **Edit 工具改代码时，old/new 的上下文行要完整保留**
   - 事故：改 `_render` 时 new_string 漏掉 `q_words = set(topic_words(q))` 一行，下一轮调试才发现补回。
   - 规则：每次 Edit 后立即跑受影响路径的最小回归（本项目保留 assert 式 smoke 三连：LATEST/CONFLICT/开关）。

4. **pip 后台安装必须看真实输出，不能信 exit code**
   - 事故：`pip install torch==2.6.0 --index-url .../pytorch-wheels/cpu/` 报 "No matching distribution" 但经 `| tail` 管道后 exit 0，误以为装成功。
   - 规则：后台安装命令结束后必须 `pip show <pkg>` 或 `python -c "import ..."` 实证；Windows torch 直接用 pypi 主镜像（阿里/清华），不要用 pytorch-wheels 专用镜像。

5. **新模块文件名想清楚再 Write**
   - 事故：把 update_detector 的实现整包写进了 `addons/__init__.py`，再用 mv 纠正。
   - 规则：Write 前确认目标路径是模块文件还是包标记。

6. **脚本必须做目录锚定**（重复犯）
   - 事故：bash `cd scripts` 后工作目录漂移，相对路径 `outputs/...` 失效。
   - 规则：所有 scripts/ 下脚本用 `ROOT = dirname(dirname(abspath(__file__)))` 锚定仓库根，路径统一 `os.path.join(ROOT, ...)`。

7. **指标口径要与参照系对齐再下结论**
   - 事故：最初用 gold answer 词覆盖率当 EvRecall，LoCoMo 算出 0.482 与"全量返回"矛盾；实际 InvMem 的 0.9276 用的是标注证据句口径，换成 evidence_messages 后得 0.997。
   - 规则：引用外部锚点数字前，确认分子分母口径一致；同一指标两种口径（gold 词/标注证据）要在脚本里显式命名区分。

8. **全量 pipeline 跑批前：3 题连通快测 + 启动后 3 分钟产出验证（两道闸门）**
   - 事故 A：BEAM e2a 首轮 400 题 answers+judged 全部 "URL missing protocol" 白跑——后台命令没 source .env，api_config 环境变量全空；错误请求即时返回，2 小时"跑完"是假象。
   - 事故 B：修正配置后 3 题快测又全 500，排查半天（服务健康、payload 复刻全通），重跑同命令 3/3 过——服务瞬时抖动。
   - 事故 C：chunk 粒度 context 132K tokens 超 qwen3 128K 窗口 → 服务端 500（message 67K 安全）。
   - 规则：① 任何全量跑批前用 `head -3` 快测且逐条检查 generated_answer 非空；② 后台命令统一 `set -a; source .env; set +a` 前缀；③ 启动 3 分钟后 wc -l + 抽验有效性；④ 500 先重试一次再定性（瞬时 vs 持续）；⑤ 新粒度配置先算 context token 量级是否超模型窗口。

9. **answers/judged 全 0 或全空 = 配置/服务事故，不是模型真实得分**
   - 规则：看到分数异常（整组 0、整组同错误串）先查 error 字段，不要往方案问题上归因。

10. **复用历史跑通的命令前缀时，环境变量一个都不能丢**
    - 事故：重启 A/B 时只带了 `PIPELINE_CONCURRENCY=8`，漏了 `PIPELINE_JSON_MODE=0`——judge 阶段 400 题全 500（内部服务不支持 response_format，answers 不用 json mode 所以成功，极具迷惑性）。
    - 规则：pipeline 类命令的"历史成功前缀"（.env source + PIPELINE_JSON_MODE + PYTHONIOENCODING + CONCURRENCY）作为整体模板复用，改一个参数其他原样保留；judge 阶段失败先查 json mode。

11. **数据集 × 粒度适配表：换粒度前先算 context token 预算**
    - 事故：LME v0.3 首跑用默认 chunk 粒度，context 膨胀 4 倍（119K→497K 字符≈124K tokens 逼近 qwen3 窗口），全线 -21.2 崩盘——EvRecall 0.959 反而更高，纯呈现层灾难；BEAM chunk 同理（132K tokens 超窗直接 500）。
    - 适配表（top_k=100 前提）：LoCoMo 库小（~40 chunk 全量 92K chars）chunk✅；LME/BEAM 库大必须 message（119K/270K chars）；正式提交还要按 gpt-4o-mini 的 128K 窗口复核。
    - 规则：新数据集 build 后先看 [stats] avg context 行再跑 answer；换粒度=换变量，A/B 对照必须同粒度。

12. **消融与变量的纪律**：v0.3 首轮 LME 翻车根因是"特性开关全开 + 粒度也换了"双变量；LoCoMo 的 cat1/cat3 跌幅同理无法归因到单一特性。规则：跨数据集迁移配置时，粒度保持与该数据集基线一致，特性开关单变量递增。
