# 实验 1 记录：第 1 组

现场检查时将打开本文件，按其中的链接核对。证据一律以链接或文本形式给出，不要只贴截图。将 `<填写>` 替换为本组的内容。main 受保护之后，修改本文件也须通过 PR。

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
| 仅凭 BUILD_INFO，能否回答该包是什么、在哪台机器上、何时、用哪份代码构建？哪一项信息会出错？ | **四项中「在哪台机器上」「何时」无法回答，「用哪份代码构建」这一项会出错。** `commit` 由 Makefile 从 `git rev-parse HEAD` 取得，衡量的是 HEAD 指向哪个提交，而非实际参与编译的工作区内容：改动未提交时它照旧输出 `7b63b41…`，而二进制已由新代码编出（sha256 由 `f23447d4…` 变为 `501ca18b…`），两次打包的产物完全无法区分。文件内没有机器与时间字段，`vcs.time` 是提交时间而非构建时间，`run_id=local` 则说明没有构建系统在记账。真正没有撒谎的是二进制内嵌的 `vcs.modified=true`。因此必须把**制品内嵌的 vcs 元数据**与**外部构建系统记录的 `run_id` / `build_no`** 合起来判读。<br>**成立条件**：以上针对「同一仓库、改动未提交」的情形；若改动已提交，`commit` 会随之更新并正确指向源码，但仍回答不了「何时、在哪台机器上构建」，也仍无法区分「同一提交构建了两次」的两份产物。 |

## 3. 证据

| 项 | 链接或文本 |
|---|---|
| 规则集里启用的规则 | `Restrict deletions`、`Block force pushes`、`Require a pull request before merging`（**Required approvals = 1**）、`Require status checks to pass`（必需检查 = **`ci`**）。Enforcement status = **Active**，**Bypass list 为空**。 |
| 组长直接 `git push origin main` 被拒的终端输出 | `remote: error: GH013: Repository rule violations found for refs/heads/main.`<br>`remote: - Changes must be made through a pull request.`<br>`remote: - Required status check "ci" is expected.`<br>`To https://github.com/Gui-05/ops-lab1-1.git`<br>` ! [remote rejected] main -> main (push declined due to repository rule violations)` |
| 第一轮：失败的 PR 运行 | https://github.com/Gui-05/ops-lab1-1/actions/runs/39564921751|
| 第一轮：修复后通过的 PR 运行 | https://github.com/Gui-05/ops-lab1-1/actions/runs/39568164080 |
| 第一轮：合入 main 后的运行（含制品） |  |
| 第二轮：找到的缺陷，以及现有测试为何无法发现它 | 当前测试对于工作日的检查只试了 0、3和8，也就是 `if s.Weekday < 1 \|\| s.Weekday > 6` 也可以通过。<br>【建议补一句前景：缺陷是把 `Valid()` 里的 `s.Weekday > 7` 改成 `s.Weekday > 6`，使合法的周日（`Weekday = 7`，注释为「1=周一 … 7=周日」）被判为非法】 |
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

仅凭构建号不能唯一确定是哪次运行，因为 `ci` 和 `release` 的构建号各自从 1 开始计数。v1.0.0 的构建号 `1` 同时对应 `ci` 运行 #1（https://github.com/Gui-05/ops-lab1-1/actions/runs/39374873071）和 `release` 运行 #1（https://github.com/Gui-05/ops-lab1-1/actions/runs/39566016067）；`BUILD_INFO` 中的 `run_id=39566016067` 才能唯一指回发布它的运行。另可佐证：v1.0.1 的构建号为 `3` 而非 `2`，因为 release gate 演练占用了一次 `release` 运行（`runs/39568265272`），说明构建号会被失败或不发布的运行消耗。

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

时间均从 GitHub 页面读取，并注明时区。

第一轮 gate 演练的时间线：

| 事件 | 时间 | 链接 |
|---|---|---|
| 发起 PR | 2026-10-21 15:48 | https://github.com/Gui-05/ops-lab1-1/actions/runs/39564921751 |
| 第一次 `ci` 失败（运行结束） |  2026-10-21 15:48  | https://github.com/Gui-05/ops-lab1-1/actions/runs/39564921751 |
| 推送修复提交 |  2026-10-21 16:28  | https://github.com/Gui-05/ops-lab1-1/actions/runs/39568164080 |
| `ci` 通过（运行结束） | 2026-10-21 16:28 |https://github.com/Gui-05/ops-lab1-1/actions/runs/39568164080|
| 合入 main | 2026-10-21 16:28 | 【待填 ▸ `/commit/<squash后sha>`】 |

提交到发布的间隔，至少记录三个合入的 PR：

| PR | 第一个提交的时间 | 合入时间 | 对应 Release 的发布时间 | 提交到发布的间隔 |
|---|---|---|---|---|
|  第一轮演练 |  2026-10-21 15:48  |  2026-10-21 16:21  | 无（未发布） | — |
|  release 工作流 |  2026-10-21 15:53 |  2026-10-21 15:48  |  2026-10-21 15:59 | 9min |
|  文档 |  2026-10-21 17:43  |  2026-10-21 21：02 |--| -- |

`ci` 工作流的运行时长（从 Actions 页面读取），至少 5 次，全部列出，不要只给出平均值：

| 运行 | 事件（PR / push） | 时长 | 备注 |
|---|---|---|---|
| `runs/39374873071` | push | 45s | 首次运行 `ci.yml`，产出 artifact；提交 `da96bdaa…` |
| `runs/39564921751` | PR |12s | 第一轮：`ci` 失败（含缺陷） |
| `runs/39568164080` | PR | 17s | 第一轮：`ci` 通过（修复后） |
| `/runs/39571263978` | PR| 10s | 第二轮：`ci`通过，设计未被发现的漏洞 |
| `runs/39572784545` | PR | 20s | 第二轮：`ci` 通过（缺陷漏过合并框） |

## 6. 分析问题

每题先列出所用数据（可引用上面的表），再用一段话（4～8 句）写出结论，并说明结论的成立条件。

**问题 1：gate 的收益与成本。** 两轮演练中，gate 拦住了什么，没有拦住什么？用第 5 节的时间线计算：第一轮因失败多花了多少时间，其中等待 `ci` 的时间和人工修改的时间各占多少？一次 `ci` 大约需要多久，在“提交到发布的间隔”中占多大比例？第二轮补充测试之后，还有哪类问题是测试本身无法发现的？

素材：第一轮 gate 拦住了「测试覆盖到的缺陷」——单双周误判同时被 `TestConflicts` 与 `TestConflictEndpoint` 抓出；第二轮中「周日被误判为非法」因测试从未覆盖 `Weekday=7` 而 `ci` 全绿、径直通过合并框，说明 gate 只能拦住「已有断言能够表达的错误」。第二轮补测试后仍无法发现的是**测试未覆盖到的行为偏差**，以及**无法用单元断言表达的性质**，如性能、并发一致性、跨服务时序。等待ci的时间很快，在项目较小和测试集少的前提下（本项目内）只需要十几秒。

**问题 2：PR 中的构建与 main 上的构建。** 在 PR 触发的运行日志中，`github.sha` 与 `pull_request.head.sha` 是否相同？它们能否在 main 的历史中找到？答案与合并方式有关，请写明本组使用的方式。合并之后，main 上这次改动的提交号是哪个？据此说明，为什么待发布的制品只能从 main 上的提交或 tag 构建。

本组采用 **squash** 合并；`pull_request` 事件的 `github.sha` 指向 `refs/pull/N/merge` 的临时合并提交，`pull_request.head.sha` 指向分支上的真实提交，两者不同，且都**不出现在 main 历史中**；squash 会在 main 上生成**全新 SHA**——例如 PR #2 对应 main 上的 `2824a726…`（即 v1.0.0 的提交），与分支上的提交号无关

**问题 3：可重复构建。** 在本地同一个提交上按下表构建，记录哈希值。④、⑤ 需临时去掉 Makefile 中 tar 的 `--mtime=@0`，完成后用 `git checkout -- Makefile` 恢复，不要提交。

本组在 `main` 的提交 `940984f800ac5b7120d17a2c8746771bf0c5e6f5`（`vcs.time=2026-10-21T09:44:52Z`，即 UTC+08:00 的 17:44:52）上构建，`VERSION=1.0.0`、`RUN_ID=1`、`COMMIT` 固定为该提交。

| 构建 | 条件 | 二进制 sha256 | tar.gz sha256 |
|---|---|---|---|
| ① | `make package`，`VERSION`、`BUILD_NO`、`RUN_ID`、`COMMIT` 均指定固定值（`BUILD_NO=1`，工作区干净，`vcs.modified=false`） | `c6b49a1b1cb7531f2dce332ebe5f2f6711aecd6a5c8bc79795d1148333ce6336` | `7509097bb9f53add66388a137f697bc399ff1b67f599c6635c06e5ee1c56ea17` |
| ② | 与 ① 完全相同，再构建一次 | `c6b49a1b1cb7531f2dce332ebe5f2f6711aecd6a5c8bc79795d1148333ce6336`（与 ① 相同） | `7509097bb9f53add66388a137f697bc399ff1b67f599c6635c06e5ee1c56ea17`（与 ① 相同） |
| ③ | 仅将 `BUILD_NO` 换成另一个值（`BUILD_NO=2`） | `58a573f63424284f5ccf29a08c17185ce762bac723f65212e7c47c8b900397a8` | `f4881d4b2ec7689d91e7b33f7a7cf2688d76a3406508e1b91cec843b2ed588b0` |
| ④ | 去掉 `--mtime=@0`，其余同 ①（`sed` 改动 Makefile，工作区变脏，`vcs.modified=true`） | `3f961d2f4c66d2f53eda9d2c42fbe103ca1b5699fa59eaacd1757a75a870b53d` | `6d81d49db1d17f7e87ec9b4a626bfd13025cf8c415133e1a388521ffe917f150` |
| ⑤ | 同 ④，间隔几秒后再构建一次 | `3f961d2f4c66d2f53eda9d2c42fbe103ca1b5699fa59eaacd1757a75a870b53d`（与 ④ 相同） | `7c01b30dc5215f04d3463cf6bf386d1b58d07f942673d741c8d241a18275a250`（与 ④ 不同） |
| Actions | 对 main 上的一次 push 运行执行 Re-run all jobs，比较两次尝试中 package 步骤打印的值 | 【待填 ▸ 预期两次相同：Re-run 不改变 `run_number`/`run_id`，只增加 `run_attempt`】 | 【待填 ▸ 同上】 |

哪些相同、哪些不同，原因分别是什么？④ 的二进制文件与 ① 是否相同？如果不同，请结合任务 0 中看到的 vcs 信息分析。据此说明制品中应当包含什么、不应包含什么。另外，`release` 在同一个提交上再次编译，而没有直接发布 `ci` 产出的主干制品：为什么这样做？这与 L-05 第 11 页的“一次构建、多处部署”是否冲突，代价是什么？

**①与②的二进制和 tar.gz 均完全相同**（`c6b49a1b…` / `7509097b…`），说明在源码树干净、注入参数与归档参数都一致时，构建是确定性的，同一输入可复现同一输出。**③与①不同**，唯一变化是 `BUILD_NO` 由 1 改为 2，它以 `-X main.buildNo=2` 的形式被链接进二进制，因此构建号是产物的一部分，"同一份源码、不同构建号"必然产出不同字节。**④与①也不同**，且变化原因不是 `--mtime`，而是工作区状态：`go version -m` 显示 `vcs.modified` 由 `false` 变为 `true`。**④与⑤之间恰好相反**：二进制完全相同（两次编译用的是同一个脏树、同一组 `-X` 参数），而 tar.gz 不同（去掉 `--mtime=@0` 后 tar 把文件的实际修改时间写进包头，两次构建时刻不同），这说明**二进制内容与归档元数据是两类独立的不确定性来源**。

**④ 的二进制文件与 ① 不同。** 任务 0 已经实测过同一机制：改动一行而不提交时，`BUILD_INFO` 一字未变（`version`、`build_no`、`commit` 全部照旧），二进制 sha256 却由 `f23447d4…` 变为 `501ca18b…`，而 `go version -m` 中 `vcs.modified` 由 `false` 变为 `true`。本次 ④ 完整复现了这一点：`sed` 修改 `Makefile` 后，HEAD 指针没有移动（`vcs.revision` 仍是 `940984f8…`），但 Go 检测到源码树不干净并把 `vcs.modified=true` 写进二进制。因此**工作区是否干净是一个会被漏进产物的隐式输入**——它既不在 `BUILD_INFO` 里，也不在命令行参数里，只能靠 `go version -m` 才能看到。

**制品中应当包含**可追溯且有意注入的标识：`version`、`build_no`、`commit`、`run_id` 与编译器版本，它们让制品能回答"我是哪个版本、第几次构建、哪份源码、哪次运行编的"。**不应当包含**构建时刻的偶然状态：tar 包头中的文件修改时间、gzip 头中的时间与文件名、构建机的绝对路径，以及 VCS 脏标记造成的差异。前者靠 `-X` 在编译期注入，后者靠 Makefile 中的 `--mtime=@0`、`--sort=name`、`--owner=0 --group=0 --numeric-owner`、`gzip -n` 与 `go build -trimpath` 逐项消除——本实验 Makefile 的注释也说明这几项正是为此存在。

**`release` 重新编译的原因**：版本号必须通过 `-X main.version` 在编译期注入二进制，而 `ci` 在主干的产物固定使用 `0.0.0-main`，无法事后把 `1.0.0` 写进去。这与 L-05 第 11 页的"一次构建、多处部署"**是冲突的**：同一份源码被编译了两次，`ci` 的主干制品与 Release 附件不是同一份字节，"被检查的产物"与"被发布的产物"在严格意义上不再是同一个对象。**代价**是：多一次编译开销；两份制品需各自校验和留存；一旦两次构建的环境存在差异（Go 补丁版本、依赖、VCS 状态），两者就会不一致，而"部署的正是通过检查的那一份"这一保证随之失效。

**成立条件**：上述可重复性只在"源码树干净 + 注入参数固定 + 归档参数完整（`--mtime=@0`、`--sort=name`、`gzip -n`）+ `-trimpath` + Go 补丁版本一致"**全部满足**时成立；任一条件被破坏（例如本次 ④ 的脏工作区，或换用不同的 Go 补丁版本），同一提交就会产出不同哈希，"用哈希比对制品来源"这一手段随即失效。

**问题 4：发布与回退。** 假设 v1.0.1 发布后发现缺陷，应当如何处理？为什么不应删除 v1.0.1 的 tag、再用同一版本号重新发布？任务 6 中将服务从 v1.0.1 切换回 v1.0.0 时，使用的是哪个文件，是否需要重新构建？请用两次 `/version` 的输出说明。如果改为从源码重新构建 v1.0.0，得到的文件与原附件是否相同（参考问题 3）？最后，如果有人删除 v1.0.0 的附件，换上一个新文件和对应的新 `.sha256`，`trace.sh` 能否发现？本组在第 4 节记录的值能否发现？可以查阅 GitHub 的 Immutable releases 设置的用途。

所用数据为第 4 节的两个 tag、提交号、`run_id`、附件 sha256 和两次 `/version` 输出。若 v1.0.1 发布后发现缺陷，应通过受保护的 PR 修复并发布新的修订版本，例如 v1.0.2；不应删除 v1.0.1 的 tag 后复用同一版本号，否则同一版本号会对应不同源码和附件，破坏版本、提交、构建和制品之间的唯一对应关系。任务 6 的回退直接使用已下载的 `enroll-conflict_1.0.0_linux_amd64.tar.gz` 及其中的二进制，没有重新构建，`/version` 随即从 v1.0.1 的版本 `1.0.1`、提交 `23901b07…`、构建号 `3` 切换为 v1.0.0 的版本 `1.0.0`、提交 `2824a726…`、构建号 `1`。若从源码重新构建 v1.0.0，只有在源码树、Go 版本、VCS 状态、注入参数和归档规则完全一致时才可能得到相同哈希；问题 3 已实测验证：①与②在条件完全相同时哈希一致，而③改了构建号、④让工作区变脏后哈希即改变，说明这些条件并非总能自动成立。如果有人同时替换 Release 中的 tar.gz 和对应 `.sha256`，当前 `trace.sh` 无法发现，因为它只能验证两者彼此一致、`BUILD_INFO` 中的提交存在且属于 `main`，不能证明该附件与首次发布时相同。本组在第 4 节记录的原 sha256 能提供对照，GitHub Immutable Releases 则可在平台层面阻止已发布附件被替换，并为审计保留更强证据。

## 7. 选做记录

| 选做 | 完成情况与证据 |
|---|---|
| O1 容器镜像 | 未做 |
| O2 缓存对反馈时间的影响 | 未做 |
| O3 迁移到小组项目 | 未做 |

## 8. 成员贡献

每位成员一行，只填写 GitHub 用户名，无需填写姓名；行数应与小组人数一致。

| GitHub 用户名 | 发起并合入的 PR 编号 | 审批过的 PR 编号 |
|---|---|---|
| Gui-05 | #1、#7 | #4 |
| Ryandl-Lew | [#2](https://github.com/Gui-05/ops-lab1-1/pull/2) | 无 |
| jyhholdon | #5、#6 | #2 |
| yueer1128 | #4 | 无 |
| Ther97 | 无 | #1、#7 |

