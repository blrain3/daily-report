# 《智能日报生成器》proposal-v1.5 定稿轮评审

## 一、三处验收

1. **截断与入报口径：安全阀修补通过，入报时点仍有残留。** §3.3 与 §2.5 一致，72 小时安全阀只适用于无水位源，已有水位的源须完整扫描水位以来的区间。§3.4 已限定按窗口内记录验收，允许截断范围外缺失，并要求首页标注。但「节后第一个成功运行」仍可能与首次入库归属冲突：若节后首轮为 `partial`，成功源的记录已归入该轮日报，后续首个 `success` 日报不会再次收录。 [§2.5](/root/projects/daily-report/proposal-v1.5.md:156) · [§3.3](/root/projects/daily-report/proposal-v1.5.md:422) · [§3.4](/root/projects/daily-report/proposal-v1.5.md:431)
2. **补渲染：未通过。** §2.6 已规定在采集前扫描历史轮次，取报告日，在该日尚无有效最新文件时，按 `first_seen_report_date` 全量渲染并递增版本。它也规定了不重采、不推进水位、不自动推送、记 `run.note` 和日志、计入告警，以及失败后继续当日流程。不过 §2.6 只扫描 `success`／`partial`，§2.8 却把已有入库记录的 `failed` 轮列为候选。周二入库事务已提交、日报落盘前崩溃时，按 §2.8 的写入顺序，周二可能没有可供扫描的 `run` 记录。周三因此无法发现待补渲染的报告日，周三正文也只读取归属周三的记录。 [§2.6](/root/projects/daily-report/proposal-v1.5.md:193) · [§2.8](/root/projects/daily-report/proposal-v1.5.md:251) · [v1.4 评审中的原始反例](/root/projects/daily-report/gpt-review-proposal-v1.4.md:15)
3. **推送分支：分支修补通过，验收口径仍有冲突。** §2.7 第 2 分支按同推送键查找 `sending`，重发成功后结清旧 `sending`。第 3 分支覆盖仅有 `failed`、仅有 `superseded` 和无记录。正文也明确了 `superseded` 必须伴随同键 `success`，仅有 `superseded` 时按无记录处理并记 WARN。送达保证已写明每次重跑最多重发一次，累计重复送达无上限。§2.7 和 §3.5 仍保留「同一天重跑不会重发内容相同的通道」，缺少同键已有 `success` 的前提。 [§2.7](/root/projects/daily-report/proposal-v1.5.md:214) · [§3.5](/root/projects/daily-report/proposal-v1.5.md:448)

## 二、副作用检查

- **采集与归属（§2.5、§2.6、§2.8、§3.3、§3.4、§4）。** §4 已将窗口下界明确为上次成功推进的 `started_at`，无水位时引用 §2.5。§2.6 仍无条件写周末和节假日记录在首个工作日被采集并入报。§2.8 与 §3.3 仍无条件承诺失败源下一轮覆盖未采区间，但无水位源会重新计算下界，可能截掉前段。 [§2.6](/root/projects/daily-report/proposal-v1.5.md:183) · [§2.8](/root/projects/daily-report/proposal-v1.5.md:258) · [§3.3](/root/projects/daily-report/proposal-v1.5.md:424) · [§4](/root/projects/daily-report/proposal-v1.5.md:512)
- **`failed` 与补渲染（§2.6、§2.8、§3.5）。** `failed` 轮当轮不出报，与后续运行按历史报告日补文件可以并存。实际冲突在于 §2.6 排除了 `failed`，而 §2.8 与 §3.5 又统一要求 `run.status` 在文件落盘后写入，导致无文件的 `failed` 轮缺少明确的落库时机。 [§2.6](/root/projects/daily-report/proposal-v1.5.md:193) · [§2.8](/root/projects/daily-report/proposal-v1.5.md:251) · [§3.5](/root/projects/daily-report/proposal-v1.5.md:447)
- **补渲染、退出码与告警（§2.6、§3.4）。** 补渲染不触发自动推送，因此不会触发 §3.4 的推送失败退出条件。告警仍有缺口：§2.10.3 只在非零退出时发告警，当日报正常完成并以 0 退出时，补渲染失败或待人工补发事项可能只留在日志和备注中。 [§2.6](/root/projects/daily-report/proposal-v1.5.md:197) · [§2.10.3](/root/projects/daily-report/proposal-v1.5.md:341) · [§3.4](/root/projects/daily-report/proposal-v1.5.md:439)
- **推送验收与版本记录（§3.5、§6）。** §3.5 已同步同键 `sending` 和兜底发送分支，但末条仍与重发条件冲突，仅有 `superseded` 时的 WARN 也未纳入验收。§6 的 v1.5 表同时记载「扫描 `success`／`partial`」和「已有记录的 `failed` 也入选」，保留了正文的候选范围矛盾。 [§3.5](/root/projects/daily-report/proposal-v1.5.md:448) · [§6](/root/projects/daily-report/proposal-v1.5.md:574)

## 三、结论与实现待定项

**仍不可冻结。**

1. 补渲染候选与 `run.status` 落库时机互相冲突，周二已入库但落盘前崩溃的日报在周三仍可能无法恢复。
2. 节后首个 `success` 的入报断言和同内容不重发的断言仍缺少各自的适用条件，按首次入库日正确出报或按同键 `sending` 正确重发都可能被验收判为失败。

## 四、打分

**规格成熟度 84/100，低于 85 分冻结线。**