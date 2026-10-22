# crash-app（模拟故障源）

一个有意设计为持续崩溃的应用，容器启动后立即 `exit 1`，
从而进入 `CrashLoopBackOff` 状态，为 SRE Agent 提供一个可修复的真实目标。
