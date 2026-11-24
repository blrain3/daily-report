# 智能日报生成器

从 GitHub 提交、飞书任务与飞书群消息中采集团队当日工作数据，生成结构化日报（Markdown / HTML），在工作日 18:00 推送邮件与飞书群。

**当前状态：设计定稿，待实现。** 仓库目前只含需求规格、产品文档与评审记录，尚无运行代码。

## 目录结构

```
docs/
  spec/       需求规格演进 v1.0 → v1.5（含定稿的 docx / pdf）
  review/     外部评审记录与首轮评审意见
  product/    最终产品文档 v1.0 → v1.1
diagrams/     架构图、执行时序图、推送状态机（PlantUML 源 + PNG）
config/       members.yaml 示例、节假日表骨架
```

## 文档索引

| 文件 | 说明 |
|---|---|
| `docs/spec/proposal-v1.5.md` | 需求规格定稿（21 页）：功能范围、采集窗口与去重、推送幂等、验收标准 |
| `docs/product/最终产品文档-v1.1.md` | 产品文档（32 页）：架构、五表 DDL、核心流程、运行契约、部署与验收计划 |
| `docs/review/` | 9 份评审：`dsh-review-*` 2 份、`gpt-review-*` 7 份，另含首轮 `review-20261116.md` |
| `diagrams/*.puml` | 图源；`plantuml -tpng` 出图 |
| `config/members.yaml.example` | 成员配置示例 |
| `config/holidays.yaml` | 节假日表骨架（2027 年度待填，以国务院办公厅通知为准） |

## 设计要点

- **增量采集、全量渲染**：各数据源独立水位，窗口为 `(该源上次 started_at, 本次 started_at]`；`seen_item` 以 `item_key` 做记录级去重，窗口重叠不重复入库。
- **归属用首次入库日**：`seen_item.first_seen_report_date` 决定记录进入哪一期日报，避免晚间提交与跨日补采漏报。
- **水位与记录同事务**：`seen_item` 插入与 `source_state` 推进在同一 SQLite 事务内提交；崩溃时两者都不提交，下一轮重读由主键拦截。
- **推送 at-least-once**：按 `(report_date, member_set, channel, content_sha256)` 判定，同键已有 `success` 则跳过；结果未知（残留 `sending`）时重发并标注「可能重复」，不承诺去重。
- **补渲染**：`run` 表在采集前写入 `status='running'`，未正常收尾的轮次在下一轮被识别，按报告日补渲染；补渲染不自动推送，进告警。

详细设计见 `docs/product/最终产品文档-v1.1.md`。

## 计划源码结构（实现阶段建立）

```
src/       采集、归属映射、渲染、推送、运行编排
tests/     单元与集成测试
bin/       run_daily.sh 包装脚本
data/      SQLite 数据库（不入版本控制）
reports/   日报版本文件（不入版本控制）
logs/      运行日志（不入版本控制）
```

## 文档与图的重建

```bash
# 出图（在 diagrams/ 内执行）
cd diagrams && plantuml -tpng *.puml

# 文档转 PDF（在仓库根目录执行，保证图路径可解析）
pandoc docs/product/最终产品文档-v1.1.md -o docs/product/最终产品文档-v1.1.docx --resource-path=. --toc --toc-depth=2
soffice --headless --convert-to pdf --outdir docs/product docs/product/最终产品文档-v1.1.docx
```
