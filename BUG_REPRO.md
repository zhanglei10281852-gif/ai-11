# Bug Reproduction

## 包的性质

当前 test_model_fix 保存的是被测模型修复后的结果源码，不是初始含 Bug 源码。要复现原始缺陷，必须检出下面固定的 parent SHA；不要在当前修复结果源码上期待重新出现修复前失败。生成系统使用的可信验证补丁和完整验证日志仅在本地留存，不提交到结果分支。

## 问题现象

上海源数据区每天 06:00 切换业务日，单日上限是 1。05:30 已经有一条运行，06:30 再提交的新计划本应进入下一业务日，却仍收到 `daily run limit reached`。排查期间代码保持不变，生产代码、测试和配置均维持原样。请把排期时间换算出的窗口、计数查询参数和最终接口结果对应起来，确认额度实际按哪个日历计算。

## 含 Bug 版本

- 仓库：zhanglei10281852-gif/ai-11
- 仓库地址：https://github.com/zhanglei10281852-gif/ai-11.git
- parent SHA：9c568908c667771cc4d32830351966653b6b49fe

## 复现步骤

```bash
git clone -- https://github.com/zhanglei10281852-gif/ai-11.git bug-repro
cd bug-repro
git checkout --detach 9c568908c667771cc4d32830351966653b6b49fe
go test ./internal/service -run ^TestDailyLimitUsesSourceZoneCutoffNotWorkspaceCalendar$ -count=1
```

## 双架构完整错误信息

### linux/amd64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/service -run ^TestDailyLimitUsesSourceZoneCutoffNotWorkspaceCalendar$ -count=1
--- FAIL: TestDailyLimitUsesSourceZoneCutoffNotWorkspaceCalendar (0.79s)
    annotation_repo_behavior_test.go:63: plan in the next source-zone business day: data_zone conflict: daily run limit reached
FAIL
FAIL	github.com/zhanglei10281852-gif/ai/internal/service	0.815s
FAIL

```

stderr：

```text
(empty)
```

### linux/arm64

- 容器内复现预期退出码：1
- 容器内复现实际退出码：1

stdout：

```text
$ go test ./internal/service -run ^TestDailyLimitUsesSourceZoneCutoffNotWorkspaceCalendar$ -count=1
--- FAIL: TestDailyLimitUsesSourceZoneCutoffNotWorkspaceCalendar (1.56s)
    annotation_repo_behavior_test.go:63: plan in the next source-zone business day: data_zone conflict: daily run limit reached
FAIL
FAIL	github.com/zhanglei10281852-gif/ai/internal/service	1.817s
FAIL

```

stderr：

```text
(empty)
```

## 通过条件

诊断必须定位 internal/domain/workspace.go 的 Workspace.BusinessDayWindow 及 internal/service/inference.go 的 InferenceService.PlanInferenceRun，证明额度查询错误使用工作空间自然日窗口而非包含源数据区 CutoffHour 的业务日窗口；需列出 05:30 与 06:30 所属窗口、COUNT 查询边界和最终限额错误。定向用例复现该链路，代码、测试和配置均不改动。
