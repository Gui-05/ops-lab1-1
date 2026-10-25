# 实验 1 记录：第 1 组

本文件记录本组实验数据、证据链接与分析，供现场检查逐项核对。历史本地构建和 Docker 运行输出沿用实验时的记录；PR、Actions、Release 与镜像信息结合 2026-10-25～26 的核查结果修订。尚需实际操作或补充的内容集中列在第 9 节，不以预期结果代替实测。main 受保护之后，本文件的修改也须通过 PR 合入。

## 1. 基本信息

| 项 | 内容 |
|---|---|
| 小组 | 第 1 组 |
| 仓库地址 | <https://github.com/Gui-05/ops-lab1-1> |
| 代码载体 | 统一示例 |
| runner 与语言版本 | `ubuntu-24.04`，Go `1.26.9` |
| 合并方式 | squash |

## 2. 任务 0：本地基线

| 问题 | 观察结果 |
|---|---|
| 第一次 `make package` 得到的 BUILD_INFO（粘贴） | `name=enroll-conflict`<br>`version=1.0.0`<br>`build_no=1`<br>`run_id=local`<br>`commit=7b63b41c13f061439997b0e6e4eaa2af922a7aa1`<br>`go=go1.26.9` |
| 修改一行、不提交，再次打包：BUILD_INFO 是否变化？二进制文件的 sha256 是否变化？（贴出两次的 sha256） | **BUILD_INFO 没有变化**（`diff ~/BUILD_INFO.first $P/BUILD_INFO` 无任何输出）。二进制文件的 **sha256 变了**：<br>第一次 `f23447d406c42af7dd35e23c2d1b2f15484cfb709cc9b4bb1262643694b8528e`<br>第二次 `501ca18b6a49b56b16d3575d9f07cef97e30661248459b4e2400e9e38bfef9cb` |
| 两次 `go version -m` 输出中的 vcs 行有何不同？ | `vcs.revision` 与 `vcs.time` **完全相同**（都是 `7b63b41c13f061439997b0e6e4eaa2af922a7aa1` / `2026-10-12T16:26:07Z`），因为 HEAD 指针没有移动。唯一的差别是 **`vcs.modified` 由 `false` 变为 `true`**，即编译器如实记录了第二次是从脏工作区（存在未提交改动）中编译出来的。 |
| 仅凭 BUILD_INFO，能否回答该包是什么、在哪台机器上、何时、用哪份代码构建？哪一项信息会出错？ | **能回答包名与声明的版本，不能回答构建机器和构建时刻；存在未提交改动时，`commit` 不能完整描述实际源码。** Makefile 从 `git rev-parse HEAD` 取提交号，工作区变脏时 HEAD 不变，故两份包的 BUILD_INFO 相同，**仅凭 BUILD_INFO 无法区分两份产物**；二进制的不同 sha256 则能区分其字节内容。`vcs.time` 是提交时间，不是构建时间；`vcs.modified=true` 能提示存在未提交改动，但不能还原改动内容。CI 中应结合干净检出、制品内的版本信息，以及外部构建系统的 `run_id` / `build_no` 追溯。<br>**成立条件**：以上针对本次同一 HEAD、相同构建参数且存在未提交改动的实验；提交改动后 `commit` 会更新，但仅凭提交号仍无法区分同一提交的不同构建。 |

## 3. 证据

| 项 | 链接或文本 |
|---|---|
| 规则集里启用的规则 | [merge gate 规则集](https://github.com/Gui-05/ops-lab1-1/rules/25456336)：目标为默认分支 `main`，Enforcement status = **Active**；包含 `Restrict deletions`、`Block force pushes`、`Require a pull request before merging`（**Required approvals = 1**）、`Require status checks to pass`（必需检查 = **`ci`**）。本组原操作记录为 Bypass list 为空；当前核查账号无仓库管理选项权限，后台配置的再次确认列于第 9 节。 |
| 组长直接 `git push origin main` 被拒的终端输出 | `remote: error: GH013: Repository rule violations found for refs/heads/main.`<br>`remote: - Changes must be made through a pull request.`<br>`remote: - Required status check "ci" is expected.`<br>`To https://github.com/Gui-05/ops-lab1-1.git`<br>` ! [remote rejected] main -> main (push declined due to repository rule violations)` |
| 第一轮：PR 与失败运行 | [PR #1](https://github.com/Gui-05/ops-lab1-1/pull/1)；[首次运行 39564921751](https://github.com/Gui-05/ops-lab1-1/actions/runs/39564921751) 在 `test` 步骤失败。 |
| 第一轮：中间修复仍未通过 | [运行 39567215320](https://github.com/Gui-05/ops-lab1-1/actions/runs/39567215320) 在 `gofmt` 步骤失败，后续测试被跳过。 |
| 第一轮：修复后通过的 PR 运行 | [运行 39568164080](https://github.com/Gui-05/ops-lab1-1/actions/runs/39568164080)，随后由 Ther97 审批并 squash 合入。 |
| 第一轮：合入 main 后的运行（含制品） | [运行 39568670777](https://github.com/Gui-05/ops-lab1-1/actions/runs/39568670777)，对应 [main 提交 a983aa6](https://github.com/Gui-05/ops-lab1-1/commit/a983aa67451cb7ea56a0727ba011d4d77af2ae43)。 |
| 第二轮：找到的缺陷，以及现有测试为何无法发现它 | [PR #4](https://github.com/Gui-05/ops-lab1-1/pull/4) 将 `Valid()` 的星期上界从 `s.Weekday > 7` 改成 `s.Weekday > 6`，使注释规定合法的周日（`Weekday = 7`）被误判为非法；当时合法性测试只检查了星期 0、3、8，没有覆盖 7，因此原测试仍通过。补充合法周日测试后测试失败，撤销缺陷后通过，最终合入的改动只增加测试。 |
| 第二轮：仅含缺陷时仍然通过的运行 | https://github.com/Gui-05/ops-lab1-1/actions/runs/39571263978 |
| 第二轮：补测试后失败的运行 | https://github.com/Gui-05/ops-lab1-1/actions/runs/39571531357 |
| 第二轮：撤销缺陷后通过的运行 | https://github.com/Gui-05/ops-lab1-1/actions/runs/39571700844 |
| 未合入分支上的 tag 被 release gate 拒绝的运行 | https://github.com/Gui-05/ops-lab1-1/actions/runs/39568265272<br>测试提交不属于 `main` 历史，release gate 退出 1；后续构建、打包及发布步骤被跳过。测试 tag 与 `scratch` 分支已清理，未产生测试 Release。 |
| Release v1.0.0 | https://github.com/Gui-05/ops-lab1-1/releases/tag/v1.0.0 |
| Release v1.0.1 | https://github.com/Gui-05/ops-lab1-1/releases/tag/v1.0.1 |

## 4. 追溯与运行

对 v1.0.0 和 v1.0.1 各运行一次 `scripts/trace.sh`，粘贴输出，并填写下表。

| 项 | v1.0.0 | v1.0.1 |
|---|---|---|
| 版本号 | `1.0.0` | `1.0.1` |
| 构建号 | `1` | `3` |
| 提交号（SHA） | `2824a726c640abef3f95819744e068261a4ee69a` | `23901b079b888af61deb98377703aaaf58dccfc8` |
| 引入该提交的 PR | [#2](https://github.com/Gui-05/ops-lab1-1/pull/2) | [#5](https://github.com/Gui-05/ops-lab1-1/pull/5) |
| 构建它的 Actions 运行（`trace.sh` 给出的链接） | [release run 39566016067](https://github.com/Gui-05/ops-lab1-1/actions/runs/39566016067) | [release run 39573305453](https://github.com/Gui-05/ops-lab1-1/actions/runs/39573305453) |
| 制品 sha256（发布当时，取自 Release 里的 .sha256） | `a366f9c98e9496ed8568eafdeec8511633e45c555c3d527561b2fd15fe88cf07` | `8064cf67b3b96339e5bc696bb8bc52373b50f3df1acdf4b1f5137232bbddee6b` |

仅凭构建号不能唯一确定是哪次运行，因为 `ci` 和 `release` 的构建号各自从 1 开始计数。v1.0.0 的构建号 `1` 同时对应 [ci 运行 #1](https://github.com/Gui-05/ops-lab1-1/actions/runs/39374873071) 和 [release 运行 #1](https://github.com/Gui-05/ops-lab1-1/actions/runs/39566016067)；`BUILD_INFO` 中的 `run_id=39566016067` 才能唯一指回发布它的运行。另可佐证：v1.0.1 的构建号为 `3` 而非 `2`，因为 [release gate 演练](https://github.com/Gui-05/ops-lab1-1/actions/runs/39568265272) 占用了一次 `release` 运行，说明构建号会被失败或不发布的运行消耗。

发布 v1.0.1 之后重新下载 v1.0.0 的附件，计算得到的 sha256：`a366f9c98e9496ed8568eafdeec8511633e45c555c3d527561b2fd15fe88cf07`。与上表中发布当时的值一致。

`trace.sh` 输出（v1.0.0）：

```text
== 1. 校验和
enroll-conflict_1.0.0_linux_amd64.tar.gz: OK
== 2. 制品里的 BUILD_INFO
name=enroll-conflict
version=1.0.0
build_no=1
run_id=39566016067
commit=2824a726c640abef3f95819744e068261a4ee69a
go=go1.26.9
== 3. 提交 2824a726c640abef3f95819744e068261a4ee69a
作者: Asuka <3425658599@qq.com>
时间: 2026-10-21T15:59:30+08:00
标题: ci: add gated tag release workflow (#2)
主干: 该提交在 origin/main 的历史上
指向它的 tag: v1.0.0
版本号: 1.0.0    构建号: 1
提交页面（含关联的 PR）: https://github.com/Gui-05/ops-lab1-1/commit/2824a726c640abef3f95819744e068261a4ee69a
构建它的运行: https://github.com/Gui-05/ops-lab1-1/actions/runs/39566016067
```

`trace.sh` 输出（v1.0.1）：

```text
== 1. 校验和
enroll-conflict_1.0.1_linux_amd64.tar.gz: OK
== 2. 制品里的 BUILD_INFO
name=enroll-conflict
version=1.0.1
build_no=3
run_id=39573305453
commit=23901b079b888af61deb98377703aaaf58dccfc8
go=go1.26.9
== 3. 提交 23901b079b888af61deb98377703aaaf58dccfc8
作者: jyhholdon <2920415597@qq.com>
时间: 2026-10-21T16:55:57+08:00
标题: docs: document CI and release pipelines (#5)
主干: 该提交在 origin/main 的历史上
指向它的 tag: v1.0.1
版本号: 1.0.1    构建号: 3
提交页面（含关联的 PR）: https://github.com/Gui-05/ops-lab1-1/commit/23901b079b888af61deb98377703aaaf58dccfc8
构建它的运行: https://github.com/Gui-05/ops-lab1-1/actions/runs/39573305453
```

任务 6 第 4 步，两次 `/version` 的输出：

```text
运行 v1.0.1 时：
{"version":"1.0.1","commit":"23901b079b888af61deb98377703aaaf58dccfc8","build_no":"3","go":"go1.26.9"}

换回 v1.0.0 后：
{"version":"1.0.0","commit":"2824a726c640abef3f95819744e068261a4ee69a","build_no":"1","go":"go1.26.9"}
```

## 5. 数据

本节时间统一为 **UTC+08:00（Asia/Shanghai）**。PR 创建、合入时间分别取 GitHub API 的 `created_at`、`merged_at`；分支提交时间取相应提交的 `committer.date`，它不等于实际推送时刻；工作流结束时间采用已完成运行的 `updated_at`，作为本报告的结果反馈时间。`ci` job 时长另按 job 的 `started_at` 到 `completed_at` 统计，不与整个工作流的反馈窗口混用。

第一轮 gate 演练的时间线：

| 事件 | 时间 | 链接 |
|---|---|---|
| 首个缺陷提交 | 2026-10-21 15:47:51 | [提交 2b8388b](https://github.com/Gui-05/ops-lab1-1/commit/2b8388b2f1f76f1f23e89dd8e85c9b5a63febbb9) |
| 发起 PR | 2026-10-21 15:51:10 | [PR #1](https://github.com/Gui-05/ops-lab1-1/pull/1) |
| 首次 `ci` 创建 | 2026-10-21 15:51:14 | [运行 39564921751](https://github.com/Gui-05/ops-lab1-1/actions/runs/39564921751) |
| 首次 `ci` 失败反馈（`test` 失败） | 2026-10-21 15:51:30 | [运行 39564921751](https://github.com/Gui-05/ops-lab1-1/actions/runs/39564921751) |
| 第一次修复提交 | 2026-10-21 16:09:03 | [提交 a6ab18a](https://github.com/Gui-05/ops-lab1-1/commit/a6ab18a47b0653af5385d0d45edca5069a61ae54) |
| 中间 `ci` 创建 | 2026-10-21 16:09:57 | [运行 39567215320](https://github.com/Gui-05/ops-lab1-1/actions/runs/39567215320) |
| 中间 `ci` 失败反馈（`gofmt` 失败） | 2026-10-21 16:10:10 | [运行 39567215320](https://github.com/Gui-05/ops-lab1-1/actions/runs/39567215320) |
| 最终修复提交 | 2026-10-21 16:17:23 | [提交 0b03212](https://github.com/Gui-05/ops-lab1-1/commit/0b032126e45b9851816556da84ba68d2e414820f) |
| 最终 PR `ci` 创建 | 2026-10-21 16:17:35 | [运行 39568164080](https://github.com/Gui-05/ops-lab1-1/actions/runs/39568164080) |
| 最终 PR `ci` 成功反馈 | 2026-10-21 16:17:56 | [运行 39568164080](https://github.com/Gui-05/ops-lab1-1/actions/runs/39568164080) |
| Ther97 审批 | 2026-10-21 16:21:11 | [审批记录](https://github.com/Gui-05/ops-lab1-1/pull/1#pullrequestreview-5596847968) |
| 合入 main | 2026-10-21 16:21:37 | [PR #1](https://github.com/Gui-05/ops-lab1-1/pull/1)；[squash 提交 a983aa6](https://github.com/Gui-05/ops-lab1-1/commit/a983aa67451cb7ea56a0727ba011d4d77af2ae43) |
| main `ci` 创建 | 2026-10-21 16:21:39 | [运行 39568670777](https://github.com/Gui-05/ops-lab1-1/actions/runs/39568670777) |
| main `ci` 成功反馈 | 2026-10-21 16:22:02 | [运行 39568670777](https://github.com/Gui-05/ops-lab1-1/actions/runs/39568670777) |

GitHub 的提交对象时间与 PR 的 `merged_at` 可能相差一秒；本表“合入 main”统一使用后者。现有证据没有保留精确的 `git push` 时刻，因此不将提交时间冒充推送时间。

提交到发布的间隔：起点取该 PR 首个分支提交的 `committer.date`，终点取**第一个包含该 PR 改动的 Release** 的 `published_at`。这是本实验口径下的间隔，包含开发、人工响应、审批、CI 和等待发布的时间，不能代表真实团队的 DORA 水平。

| PR | 第一个提交的时间 | 合入时间 | 对应 Release 的发布时间 | 提交到发布的间隔 |
|---|---|---|---|---|
| [#1：第一轮演练](https://github.com/Gui-05/ops-lab1-1/pull/1) | 2026-10-21 15:47:51 | 2026-10-21 16:21:37 | [v1.0.1](https://github.com/Gui-05/ops-lab1-1/releases/tag/v1.0.1)，2026-10-21 16:58:34 | **1 小时 10 分 43 秒** |
| [#2：release 工作流](https://github.com/Gui-05/ops-lab1-1/pull/2) | 2026-10-21 15:46:56 | 2026-10-21 15:59:31 | [v1.0.0](https://github.com/Gui-05/ops-lab1-1/releases/tag/v1.0.0)，2026-10-21 16:00:37 | **13 分 41 秒** |
| [#4：第二轮演练](https://github.com/Gui-05/ops-lab1-1/pull/4) | 2026-10-21 16:34:42 | 2026-10-21 16:54:06 | [v1.0.1](https://github.com/Gui-05/ops-lab1-1/releases/tag/v1.0.1)，2026-10-21 16:58:34 | **23 分 52 秒** |
| [#5：文档小改动](https://github.com/Gui-05/ops-lab1-1/pull/5) | 2026-10-21 16:50:30 | 2026-10-21 16:55:57 | [v1.0.1](https://github.com/Gui-05/ops-lab1-1/releases/tag/v1.0.1)，2026-10-21 16:58:34 | **8 分 04 秒** |
| [#9：O1 容器镜像](https://github.com/Gui-05/ops-lab1-1/pull/9) | 2026-10-26 00:44:03 | 2026-10-26 00:51:09 | [v1.0.2](https://github.com/Gui-05/ops-lab1-1/releases/tag/v1.0.2)，2026-10-26 00:53:47 | **9 分 44 秒** |

`ci` 时长样本如下，逐项列出首次构建、两轮 gate 的所有 PR 检查及相应 main 检查，并补充 O1 合入后的 main 检查。时长为 **`ci` job 的执行时长**，不含 job 开始前的排队；这些是明确选取的 10 次样本，并非仓库全部运行。

| 运行 | 事件（PR / push） | job 时长 | 备注 |
|---|---|---|---|
| [39374873071](https://github.com/Gui-05/ops-lab1-1/actions/runs/39374873071) | push | 45s | 首次 main 构建成功并产出 artifact；提交 `da96bdaa…` |
| [39564921751](https://github.com/Gui-05/ops-lab1-1/actions/runs/39564921751) | PR | 12s | 第一轮：含缺陷，`test` 失败 |
| [39567215320](https://github.com/Gui-05/ops-lab1-1/actions/runs/39567215320) | PR | 10s | 第一轮：中间修复，`gofmt` 失败，测试跳过 |
| [39568164080](https://github.com/Gui-05/ops-lab1-1/actions/runs/39568164080) | PR | 17s | 第一轮：最终修复后通过 |
| [39568670777](https://github.com/Gui-05/ops-lab1-1/actions/runs/39568670777) | push | 19s | 第一轮：合入 main 后通过并产出 artifact |
| [39571263978](https://github.com/Gui-05/ops-lab1-1/actions/runs/39571263978) | PR | 10s | 第二轮：只有缺陷时通过，原测试漏检 |
| [39571531357](https://github.com/Gui-05/ops-lab1-1/actions/runs/39571531357) | PR | 13s | 第二轮：补周日测试后，`test` 失败 |
| [39571700844](https://github.com/Gui-05/ops-lab1-1/actions/runs/39571700844) | PR | 17s | 第二轮：撤销缺陷后通过 |
| [39572784545](https://github.com/Gui-05/ops-lab1-1/actions/runs/39572784545) | push | 20s | 第二轮：修复后合入 main，通过并产出 artifact |
| [40213659759](https://github.com/Gui-05/ops-lab1-1/actions/runs/40213659759) | push | 27s | O1 的 PR #9 合入 main 后通过 |

样本范围为 10～45 秒，中位数为 17 秒；其中混有提前失败的 PR job 与包含打包、上传的 main job，不能将差异全部归因于缓存或 runner。共享 runner 负载、缓存状态、失败退出位置及执行步骤不同都会影响时长；本组没有完成 O2 的有缓存／无缓存对照实验，不据此声称缓存带来多少收益。

## 6. 分析问题

每题列出所用数据，再说明结论、机制与成立条件；尚缺的实测数据明确标注，集中见第 9 节。

**问题 1：gate 的收益与成本。** 两轮演练中，gate 拦住了什么，没有拦住什么？用第 5 节的时间线计算：第一轮因失败多花了多少时间，其中等待 `ci` 的时间和人工修改的时间各占多少？一次 `ci` 大约需要多久，在“提交到发布的间隔”中占多大比例？第二轮补充测试之后，还有哪类问题是测试本身无法发现的？

**所用数据**：第 3 节两轮演练、第 5 节 PR #1 时间线、10 次 job 时长样本及 PR #1 的发布间隔。首次失败反馈到最终成功反馈为 `16:17:56 − 15:51:30 = 26 分 26 秒`，拆分如下：

| 观察到的区间 | 计算 | 耗时 |
|---|---|---|
| 两段失败反馈到下一次修复提交 | `16:09:03 − 15:51:30` 加 `16:17:23 − 16:10:10` | 24 分 46 秒 |
| 两次修复提交到下一次工作流创建 | `16:09:57 − 16:09:03` 加 `16:17:35 − 16:17:23` | 1 分 06 秒 |
| 两次后续工作流的创建到结束反馈 | `16:10:10 − 16:09:57` 加 `16:17:56 − 16:17:35` | 34 秒 |
| 合计 | 上述三个区间之和 | **26 分 26 秒** |

第一轮中，必需检查先拦住了测试可发现的单双周冲突缺陷，随后还拦住了一次格式错误，人工审批则要求另一位成员检查后才能合入。第二轮中，原测试没有覆盖合法周日，缺陷提交的 `ci` 虽然通过，但当时仍缺审批，不能据此说完整 gate 已放行；补测试后失败、撤销缺陷后通过，说明自动门禁的能力取决于断言覆盖范围。以“首次失败反馈到最终成功反馈”为本次返修成本的观察口径，耗时为 26 分 26 秒，其中人工响应与修改相关间隔为 24 分 46 秒，修复提交到下一次工作流创建为 1 分 06 秒，后续两次工作流反馈窗口合计 34 秒。前一间隔可能含离开终端、等待或本地验证，后一间隔也含尚未记录的提交后推送延迟，现有时间戳不能测出纯粹写代码或 runner 排队的时长，也不能证明若无缺陷一定能节省全部 26 分 26 秒。10 次样本的 job 中位数为 17 秒，PR #1 的三次 PR 检查 job 合计 `12 + 10 + 17 = 39` 秒，占其提交到 v1.0.1 发布的 `4243` 秒约 **0.92%**，单次 17 秒约占 **0.40%**，其余时间还包含开发、响应、审批和等待发布。这个比例只描述本组这次实验，混合步骤、缓存及共享 runner 负载都会影响时长，小样本不能代表普遍团队效率。补周日测试以后，其他未覆盖的边界、性能、并发与跨服务行为仍可能出错，需要相应的测试和环境验证，人工审批也不能保证发现所有问题。

**问题 2：PR 中的构建与 main 上的构建。** 在 PR 触发的运行日志中，`github.sha` 与 `pull_request.head.sha` 是否相同？它们能否在 main 的历史中找到？答案与合并方式有关，请写明本组使用的方式。合并之后，main 上这次改动的提交号是哪个？据此说明，为什么待发布的制品只能从 main 上的提交或 tag 构建。

**所用数据**：[PR #2 的 CI 运行上下文日志](https://github.com/Gui-05/ops-lab1-1/actions/runs/39565152690/job/119105541878#step:4:14)、[PR #2](https://github.com/Gui-05/ops-lab1-1/pull/2) 及 [main 上的 squash 提交](https://github.com/Gui-05/ops-lab1-1/commit/2824a726c640abef3f95819744e068261a4ee69a)。日志实值与合入结果如下：

```text
event_name = pull_request
ref        = refs/pull/2/merge
github.sha = bda18b2c9fe9fd5cf2f15f5b49e983d88e4e4ace
head.sha   = 50304afd24bb048b4976658f368d15d647225830

main 上的 squash 提交：2824a726c640abef3f95819744e068261a4ee69a
```

本组 PR #2 的 `github.sha` 与 `pull_request.head.sha` 不同，前者是 GitHub 为 PR 检查生成的临时合并提交，后者是分支顶端提交。检查本组 main 历史，两个 SHA 都不在其中，main 保留的是第三个 SHA `2824a726…`。原因是本组采用 squash，把 PR 的最终改动压成一个新提交，PR 阶段的提交身份与正式主干提交身份不同。这一结论适用于本组这次 squash 且分支提交没有通过其他路径进入 main 的情况，不能推广为所有合并方式下 head 提交都不进入 main。PR 检查验证候选改动与目标分支组合的行为，合入后的 main CI 则再次检查并为实际主干提交打包。因此，本实验正式制品从 main 提交或指向 main 历史中提交的 tag 构建，使用该次 `github.sha` 标识来源，并用 release gate 排除未合入分支上的 tag；任意分支或任意 tag 并不自动具有这一资格。

**问题 3：可重复构建。** 在本地同一个提交上按下表构建，记录哈希值。④、⑤ 需临时去掉 Makefile 中 tar 的 `--mtime=@0`，完成后用 `git checkout -- Makefile` 恢复，不要提交。

**所用数据**：以下①～⑤保留本组已有的本地实验记录，构建提交为 `main` 上的 `940984f800ac5b7120d17a2c8746771bf0c5e6f5`（`vcs.time=2026-10-21T09:44:52Z`，即 UTC+08:00 的 17:44:52），`VERSION=1.0.0`、`RUN_ID=1`、`COMMIT` 固定为该提交。该轮本地构建的 Go 版本原始输出尚未单独附上，不从任务 0 或 Actions 的版本倒推；需补充的记录见第 9 节。

| 构建 | 条件 | 二进制 sha256 | tar.gz sha256 |
|---|---|---|---|
| ① | `make package`，`VERSION`、`BUILD_NO`、`RUN_ID`、`COMMIT` 均指定固定值（`BUILD_NO=1`，工作区干净，`vcs.modified=false`） | `c6b49a1b1cb7531f2dce332ebe5f2f6711aecd6a5c8bc79795d1148333ce6336` | `7509097bb9f53add66388a137f697bc399ff1b67f599c6635c06e5ee1c56ea17` |
| ② | 与 ① 完全相同，再构建一次 | `c6b49a1b1cb7531f2dce332ebe5f2f6711aecd6a5c8bc79795d1148333ce6336`（与 ① 相同） | `7509097bb9f53add66388a137f697bc399ff1b67f599c6635c06e5ee1c56ea17`（与 ① 相同） |
| ③ | 仅将 `BUILD_NO` 换成另一个值（`BUILD_NO=2`） | `58a573f63424284f5ccf29a08c17185ce762bac723f65212e7c47c8b900397a8` | `f4881d4b2ec7689d91e7b33f7a7cf2688d76a3406508e1b91cec843b2ed588b0` |
| ④ | 去掉 `--mtime=@0`，其余同 ①（`sed` 改动 Makefile，工作区变脏，`vcs.modified=true`） | `3f961d2f4c66d2f53eda9d2c42fbe103ca1b5699fa59eaacd1757a75a870b53d` | `6d81d49db1d17f7e87ec9b4a626bfd13025cf8c415133e1a388521ffe917f150` |
| ⑤ | 同 ④，间隔几秒后再构建一次 | `3f961d2f4c66d2f53eda9d2c42fbe103ca1b5699fa59eaacd1757a75a870b53d`（与 ④ 相同） | `7c01b30dc5215f04d3463cf6bf386d1b58d07f942673d741c8d241a18275a250`（与 ④ 不同） |

哪些相同、哪些不同，原因分别是什么？④ 的二进制文件与 ① 是否相同？如果不同，请结合任务 0 中看到的 vcs 信息分析。据此说明制品中应当包含什么、不应包含什么。另外，`release` 在同一个提交上再次编译，而没有直接发布 `ci` 产出的主干制品：为什么这样做？这与 L-05 第 11 页的“一次构建、多处部署”是否冲突，代价是什么？

本地记录中①与②的二进制和 tar.gz 哈希均相同，支持“该次固定输入和环境下可重复构建”的结论，但不能据此保证任意环境都可复现。③只把 `BUILD_NO` 从 1 改为 2，它经 `-X main.buildNo` 写入二进制，并改变 BUILD_INFO，所以二进制和归档都不同。④与①的二进制不同：修改 Makefile 让工作区变脏，已有 `go version -m` 观察记录为 `vcs.modified=false` 变成 `true`，与任务 0 中 HEAD 不变而二进制改变的机制一致，不能把二进制差异归因于 tar 的 `--mtime`。④与⑤的二进制相同而归档不同，则支持未固定的文件 mtime 会影响归档字节这一解释。制品应保留版本、构建号、提交号、运行号与工具链版本，其中 `-X` 只注入前三项到二进制，`run_id` 写入 BUILD_INFO，Go 版本由工具链记录；`--mtime=@0`、排序与固定属主、`gzip -n`、`-trimpath` 分别控制归档元数据和构建路径，不会消除 `vcs.modified`，应使用干净工作区并保留真实 VCS 信息。`release` 为把版本从主干的 `0.0.0-main` 改成 tag 对应版本而再次编译，虽重新运行 fmt、vet、test，发布包仍不是 main CI 产出的同一份字节，这与严格的“一次构建、多处部署”存在取舍，代价是额外编译及再次控制环境与留存校验信息；O1 在 Release 到镜像阶段直接复用二进制，没有第三次编译。比较哈希之前须控制源码与工作区状态、注入参数、Go 版本、目标平台和归档工具及参数，哈希不同只表示字节不同，不能单独证明源码不同或判定为篡改。Actions 两次 attempt 的实际哈希尚缺，当前不对这一项宣称“重跑已验证相同”；补齐后还应核对 `run_number`、`run_id` 与增加的 `run_attempt`，并说明可变 runner 镜像和 Action 主版本引用留下的环境局限。

**问题 4：发布与回退。** 假设 v1.0.1 发布后发现缺陷，应当如何处理？为什么不应删除 v1.0.1 的 tag、再用同一版本号重新发布？任务 6 中将服务从 v1.0.1 切换回 v1.0.0 时，使用的是哪个文件，是否需要重新构建？请用两次 `/version` 的输出说明。如果改为从源码重新构建 v1.0.0，得到的文件与原附件是否相同（参考问题 3）？最后，如果有人删除 v1.0.0 的附件，换上一个新文件和对应的新 `.sha256`，`trace.sh` 能否发现？本组在第 4 节记录的值能否发现？可以查阅 GitHub 的 Immutable releases 设置的用途。

**所用数据**：第 4 节两个版本的提交号、`run_id`、附件 sha256 和两次 `/version` 输出，以及问题 3 的本地构建记录。

若 v1.0.1 发布后发现缺陷，可以先回退到已知可用版本，再通过受保护 PR 修复并发布一个尚未使用的修订号；当前 v1.0.2 已用于 O1，后续修订例如使用 v1.0.3。不应删除原 tag 或复用已发布版本号，否则同一版本号会对应不同源码或附件，已有下载和审计记录将失去稳定含义。任务 6 回退直接解压并运行原 `enroll-conflict_1.0.0_linux_amd64.tar.gz` 的二进制，无需重新构建，`/version` 从 `1.0.1 / 23901b07… / build_no=3` 切换为 `1.0.0 / 2824a726… / build_no=1`。从源码重建只有在源码、Go 版本、工作区状态、平台、注入参数及归档规则等输入一致时才可能复现原哈希，本地①②相同而③④不同的记录说明“相同提交”本身不够。若包与 `.sha256` 被一起替换，而 BUILD_INFO 仍声称一个合法 main 提交，`trace.sh` 的校验和及提交归属检查可能全部通过，它不能证明这份包就是首次发布的原件。在原始记录可信且未被同步改动的条件下，第 4 节保存的原 sha256 可以发现内容变化；Immutable Releases 可在平台层面限制已发布附件及关联 tag 的改动，但本报告不声称本组已经启用。上述直接换二进制的回退适用于本实验的无状态服务，若真实系统涉及不可逆数据迁移、配置或协议兼容问题，还需相应回退方案。

## 7. 选做记录

| 选做 | 完成情况与证据 |
|---|---|
| O1 容器镜像 | 已完成：通过 [PR #9](https://github.com/Gui-05/ops-lab1-1/pull/9) 增加镜像 job；[release #4](https://github.com/Gui-05/ops-lab1-1/actions/runs/40213832658) 发布 v1.0.2 并推送 GHCR，已验证匿名拉取、按 digest 运行和服务接口，详见下文。 |
| O2 缓存对反馈时间的影响 | 未做 |
| O3 迁移到小组项目 | 未做 |

### O1：构建与发布

在 `.github/workflows/release.yml` 中新增 `image` job，以 `needs: release` 等待发布成功。该 job 下载 [v1.0.2 Release](https://github.com/Gui-05/ops-lab1-1/releases/tag/v1.0.2) 的 `enroll-conflict_1.0.2_linux_amd64.tar.gz` 和对应 `.sha256`，执行 `sha256sum -c` 后解压，以解压目录为构建上下文、使用仓库中的 `Dockerfile` 构建镜像。`Dockerfile` 从 `scratch` 开始，只复制已发布的二进制和 `BUILD_INFO`，不重新编译；版本、提交号和仓库地址通过 OCI labels 写入镜像元数据。工作流使用具有 `packages: write` 权限的 `GITHUB_TOKEN` 登录 GHCR，推送镜像，并将镜像摘要写入 Actions Summary。

| 项 | 记录 |
|---|---|
| 实现 PR | [#9：ci: publish container image from release assets](https://github.com/Gui-05/ops-lab1-1/pull/9) |
| 源码提交 | [`d638205747a94b2420590efa52ca16a8521f3739`](https://github.com/Gui-05/ops-lab1-1/commit/d638205747a94b2420590efa52ca16a8521f3739) |
| 发布运行 | [release #4 / run 40213832658](https://github.com/Gui-05/ops-lab1-1/actions/runs/40213832658)，结果为 `success` |
| 发布与镜像 job | [release job](https://github.com/Gui-05/ops-lab1-1/actions/runs/40213832658/job/121230578348) 与 [image job](https://github.com/Gui-05/ops-lab1-1/actions/runs/40213832658/job/121230688981) 均成功；后者的下载校验、镜像构建、推送及 digest 记录步骤均成功 |
| Release | [v1.0.2](https://github.com/Gui-05/ops-lab1-1/releases/tag/v1.0.2) |
| Release 压缩包 SHA-256 | `df99fd274e78e13b8c7eddcd08f1c9f17f5071ae3d86f4047c73f191548798a8`，取自 [GitHub Release 资产 API](https://api.github.com/repos/Gui-05/ops-lab1-1/releases) 的 tar.gz `digest` 字段；与下方镜像 digest 标识的对象不同，本行不冒充一次新的本地下载校验 |
| 镜像包 | [GHCR：ops-lab1-1](https://github.com/users/Gui-05/packages/container/package/ops-lab1-1) |
| 镜像 tag | `ghcr.io/gui-05/ops-lab1-1:v1.0.2` |
| 镜像 digest | `sha256:2d36c1322c4d18a517178a1fdabb5915d197fb7ac237c7c30d57c19c6980abf0` |
| 运行主机 | Windows 上通过 Docker Desktop 运行 Linux 容器；Docker 主机报告 `linux/x86_64` |
| 镜像平台 | `linux/amd64`；`x86_64` 与 `amd64` 指同一种 CPU 架构，本次不需要跨架构模拟 |
| 匿名拉取 | 使用新的临时 Docker 配置目录执行 `pull` 成功；另以不提供用户凭据的 GHCR 匿名访问令牌请求该 digest 的镜像清单，返回 HTTP `200`，响应的 `Docker-Content-Digest` 与上表一致，确认该镜像可匿名读取 |

2026-10-26 的匿名清单核对还确认：响应 `Docker-Content-Digest` 与清单响应体计算出的 SHA-256 均为 `sha256:2d36c1322c4d18a517178a1fdabb5915d197fb7ac237c7c30d57c19c6980abf0`；镜像配置报告 `linux/amd64`，OCI labels 的版本为 `1.0.2`、revision 为 `d638205747a94b2420590efa52ca16a8521f3739`、source 为本组仓库地址。该核对验证公开镜像的当前内容与元数据；下方 Docker 命令与容器输出保留原实验记录，不另行推测其执行时刻。

### O1：按 digest 拉取与运行的实测记录

以下为 Windows PowerShell 中成功执行的命令与输出；`$image` 使用完整摘要引用，没有通过 tag 选择运行版本。

```powershell
$image = "ghcr.io/gui-05/ops-lab1-1@sha256:2d36c1322c4d18a517178a1fdabb5915d197fb7ac237c7c30d57c19c6980abf0"
$anonConfig = Join-Path $env:TEMP ("o1-anon-" + [guid]::NewGuid())
docker --config "$anonConfig" pull "$image"
```

```text
ghcr.io/gui-05/ops-lab1-1@sha256:2d36c1322c4d18a517178a1fdabb5915d197fb7ac237c7c30d57c19c6980abf0: Pulling from gui-05/ops-lab1-1
acc1e7b1dab3: Pull complete
b0f5925e895b: Pull complete
Digest: sha256:2d36c1322c4d18a517178a1fdabb5915d197fb7ac237c7c30d57c19c6980abf0
Status: Downloaded newer image for ghcr.io/gui-05/ops-lab1-1@sha256:2d36c1322c4d18a517178a1fdabb5915d197fb7ac237c7c30d57c19c6980abf0
ghcr.io/gui-05/ops-lab1-1@sha256:2d36c1322c4d18a517178a1fdabb5915d197fb7ac237c7c30d57c19c6980abf0
```

架构检查与容器启动：

```text
PS> docker info --format '{{.OSType}}/{{.Architecture}}'
linux/x86_64
PS> docker image inspect "$image" --format '{{.Os}}/{{.Architecture}}'
linux/amd64
PS> docker run -d --name o1-enroll --platform linux/amd64 -p 127.0.0.1:18080:8080 "$image"
cb268b5a1107656a91eaa31377b97d7b44bbc096ecc4559cb118ff7a14d4c3e7
```

接口验证与运行引用检查：

```text
PS> curl.exe --fail http://127.0.0.1:18080/healthz
ok
PS> curl.exe --fail http://127.0.0.1:18080/version
{"version":"1.0.2","commit":"d638205747a94b2420590efa52ca16a8521f3739","build_no":"4","go":"go1.26.9"}
PS> docker inspect o1-enroll --format '{{.Config.Image}}'
ghcr.io/gui-05/ops-lab1-1@sha256:2d36c1322c4d18a517178a1fdabb5915d197fb7ac237c7c30d57c19c6980abf0
```

`/healthz` 返回 `ok`，`/version` 返回的版本为 `1.0.2`、源码提交为 `d638205747a94b2420590efa52ca16a8521f3739`、构建号为 `4`、Go 版本为 `go1.26.9`，与本次发布的版本、提交及工作流构建号对应。`docker inspect` 显示创建容器时指定的是 `@sha256:…` 引用，证明本次按摘要运行；即使 tag 后续改变指向，该引用也不会随之指向另一份内容。验证结束后已停止并删除本次容器：

```text
PS> docker stop o1-enroll
o1-enroll
PS> docker rm o1-enroll
o1-enroll
```

### O1：思考题

**为什么发布 job 创建的 Release 没有触发另一个工作流？** 当前 `release` job 使用仓库自动提供的 `GITHUB_TOKEN` 调用 `gh release create`。GitHub 为防止工作流递归触发，不会因该 token 创建 Release 所产生的 `release` 事件自动启动另一个工作流。因此，在本实验使用内置 token 的方案中，将 `image` job 放在同一个工作流内，通过 `needs: release` 明确执行顺序；若发布 gate、测试或发布步骤失败，镜像 job 会跳过。这不是平台禁止拆成多个工作流，而是当前认证和触发方式的约束，参见 [GitHub 官方说明](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow)。

**为什么需要实际验证匿名拉取，并使用 digest 运行？** GHCR 的新包默认私有，关联仓库及继承访问权限并不能替代匿名访问验证，应在包设置中确认可见性；本次拉取结果与匿名镜像清单请求共同确认了该镜像可匿名读取。镜像 tag 是可变引用，而 digest 根据镜像内容计算，用它拉取和运行能够固定本次实际使用的镜像。镜像 digest、Git 提交 SHA 和 Release 压缩包的 SHA-256 标识的是不同对象，需要分别记录。本实现复用 Release 已编译的二进制进行封装，在 Release 到容器这一阶段没有重新编译，便于将运行结果追溯到发布制品和源码提交。

## 8. 成员贡献


| GitHub 用户名 | 发起并合入的 PR 编号 | 审批过的 PR 编号 |
|---|---|---|
| Gui-05 | [#1](https://github.com/Gui-05/ops-lab1-1/pull/1)、[#7](https://github.com/Gui-05/ops-lab1-1/pull/7)、[#8](https://github.com/Gui-05/ops-lab1-1/pull/8) | [#4](https://github.com/Gui-05/ops-lab1-1/pull/4)、[#5](https://github.com/Gui-05/ops-lab1-1/pull/5)、[#6](https://github.com/Gui-05/ops-lab1-1/pull/6)、[#9](https://github.com/Gui-05/ops-lab1-1/pull/9) |
| Ryandl-Lew | [#2](https://github.com/Gui-05/ops-lab1-1/pull/2) | 无 |
| jyhholdon | [#5](https://github.com/Gui-05/ops-lab1-1/pull/5)、[#6](https://github.com/Gui-05/ops-lab1-1/pull/6) | [#2](https://github.com/Gui-05/ops-lab1-1/pull/2) |
| yueer1128 | [#4](https://github.com/Gui-05/ops-lab1-1/pull/4) | 无 |
| Ther97 | [#9](https://github.com/Gui-05/ops-lab1-1/pull/9) | [#1](https://github.com/Gui-05/ops-lab1-1/pull/1)、[#7](https://github.com/Gui-05/ops-lab1-1/pull/7)、[#8](https://github.com/Gui-05/ops-lab1-1/pull/8) |


