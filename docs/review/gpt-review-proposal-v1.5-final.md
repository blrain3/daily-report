# 《智能日报生成器》proposal-v1.5 收尾复核

## 两条阻塞复核

1. **补渲染与落库时机：周二崩溃路径已打通，`failed` 收尾仍有冲突。** 周二先独立提交 `running` 行，随后提交采集数据。如果首次出报落盘前崩溃，周三会在采集前找到周二的 `running` 行。周二没有有效的最新版本文件时，会按 `first_seen_report_date` 读取周二已入库的记录并补渲染。上一轮指出的「没有 `run` 行可供扫描」已解决。但 2.8 和 3.5 仍要求所有终态在日报落盘后写入，而 `failed` 按定义不出报，正常失败轮次的终态写入时点仍未闭合。 [2.6](/root/projects/daily-report/proposal-v1.5.md:193) · [2.8 状态语义](/root/projects/daily-report/proposal-v1.5.md:251) · [2.8 落库顺序](/root/projects/daily-report/proposal-v1.5.md:266) · [3.5](/root/projects/daily-report/proposal-v1.5.md:448)
2. **推送验收适用条件：部分解决。** 3.4 已引用正确的跳过分支，但「当轮」仍可能被理解为整轮判定。2.7 按通道独立判定，SMTP 命中跳过时，飞书仍可能需要发送。3.5 新增的同键 `success` 前提与 2.7 第一分支一致，但「无 `success` 记录时按 2.7 正常发送」仍把第二分支的 `sending` 重发与第三分支的正常发送混在一起。2.7 自身也保留了无条件的「同一天重跑不会重发内容相同的通道」，与同键 `sending` 的重发规则冲突。 [2.7](/root/projects/daily-report/proposal-v1.5.md:214) · [3.4](/root/projects/daily-report/proposal-v1.5.md:438) · [3.5](/root/projects/daily-report/proposal-v1.5.md:449)

## 状态取值一致性

2.8 字段表和语义段均定义 `running` 加 `success`、`partial`、`failed` 三个终态，3.5 的取值口径一致。第 4 节没有 `run.status` 词条。现行条款中的「三档」均指终态，第 6 节的「三档 `run.status`」属于 v1.3 历史记录。未发现取值矛盾。 [2.8](/root/projects/daily-report/proposal-v1.5.md:241) · [3.5](/root/projects/daily-report/proposal-v1.5.md:448) · [第 4 节](/root/projects/daily-report/proposal-v1.5.md:506) · [第 6 节](/root/projects/daily-report/proposal-v1.5.md:538)

## 结论

**仍不可冻结**

1. `failed` 轮不出报，与终态必须在日报落盘之后写入的规则冲突。
2. 推送验收仍存在整轮与按通道判定的歧义，同键 `sending` 重发与正常发送的口径仍有冲突。

## 打分

**规格成熟度 88/100**（按本次限定范围评分）。