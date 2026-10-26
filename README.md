# BiliBiliToolPro（自用部署版）

上游 [RayWangQvQ/BiliBiliToolPro](https://github.com/RayWangQvQ/BiliBiliToolPro) 的 fork，
只做两件事：**加一份能跑的 GitHub Actions 工作流**，**给观看任务加一个数量上限**。
其余代码、文档全部保持上游原样。

## 和上游的差别

| 改动 | 文件 | 原因 |
| --- | --- | --- |
| 新增可用的 Actions 工作流 | `.github/workflows/bilibili-daily-task.yml` | 上游把模板归档到 `platforms/gitHubActions/bak/`，用的是 .NET 6 + 已废弃的 `::set-output`，fork 出来直接跑不起来 |
| 新增 `MaxWatchAnchors` 配置项 | `LiveFansMedalTaskOptions.cs`、`LiveFansMedalTaskRunner.cs` | 上游会把所有牌子排进观看队列，87 枚 × 每枚 15 分钟 = 三小时以上，Actions 跑不完。加了上限后按**等级从高到低**取前 N 枚，且只取**正在开播**的房间 |

## 部署

1. Fork 本仓库，在 **Settings → Secrets and variables → Actions** 添加：

   | 名称 | 内容 |
   | --- | --- |
   | `COOKIESTR_1` | `SESSDATA=xxx; bili_jct=yyy; DedeUserID=zzz` |
   | `COOKIESTR_2` | 第二个账号，同上（只用单账号就不加） |
   | `PUSHPLUS_TOKEN` | 可选，PushPlus 推送 token，不填就是不推送 |

2. Actions 里手动跑一次 `B站粉丝牌每日任务` 验证。**定时任务已于 2026-10-26 停掉**，现在只能手动触发，原因见下一节。

## 2026-10-26：停掉定时（只留手动）

这份工作流的内容 **100% 是直播挂机**：每号挂 4 个房间 × 70 分钟心跳，两个账号 matrix 并行，
按 `timeout-minutes: 120` 算，一天要烧掉约 **240 个 job 分钟**。

停掉的理由：

- **收益没被证实**。同账号的另一套实现（`bilibili-medal-optimized/scripts/watch.mjs`）
  实测挂满 15 分钟后 `today_feed` 和 `intimacy` 纹丝不动，`watchLive` 任务涨到 `4/10` 就停住。
  也就是说「心跳被服务端受理」不等于「亲密度到账」。
- **成本极不对称**。同账号的日常任务（点亮 / 权益 / 经验 / 银瓜子）全部跑完只要 2~3 分钟，
  为这一条把流水线拖到 4 小时。
- **风控最敏感**。伪造心跳 + 批量挂房间，是这套里唯一明显偏离「正常用户行为」的动作。

仓库本身保留（C# 工具有它的价值，`MaxWatchAnchors` 这类改动也是唯一在维护的东西），
只是不再每天自动跑。

想恢复：把工作流里注释掉的 `schedule` 放回去（cron 是 UTC）：

```yaml
  schedule:
    - cron: '5 16 * * *'    # 北京时间 00:05
```

注意恢复后 `daily-checkin` 的签到总览不会自动拉起它 —— 那边 `projects.json` 里这条已标 `retired`。

## 常用参数

工作流手动触发时可以覆盖（`workflow_dispatch` 的三个输入项）：

- `tasks` — `LiveFansMedal`（只跑直播间互动）或 `Daily&LiveFansMedal`
- `max_watch` — 最多挂几个直播间，默认 4，`0` = 不限
- `heartbeats` — 每个房间挂几分钟，默认 16

对应环境变量：`Ray_LiveFansMedalTaskConfig__MaxWatchAnchors` / `__HeartBeatNumber`。

## 关于亲密度规则（2026-04 改版）

B 站现在的粉丝牌任务表：

| 任务 | 条件 | 收益 | 每日上限 |
| --- | --- | --- | --- |
| 投喂粉丝灯牌 | 1 次 | +6 | 1 |
| **观看直播** | **满 15 分钟** | **+1** | **10** |
| 发弹幕 | 1 条 | +1 | 10 |
| 投喂礼物 | 按电池 | +1/电池 | — |

意思是：**一枚牌子当天想靠看直播拿满 10 点，需要挂 150 分钟**。挂 2 分钟、10 分钟都不会涨，
因为没到第一个 15 分钟的门槛——不是程序没生效。

发弹幕那条（10 条 = 10 点）比挂机便宜得多，但会在别人直播间留下弹幕，所以工作流里
`EnableDanmaku` 默认是 `false`，想开自己改。

## 已知限制

- 只看得到工作流启动那一刻**正在开播**的主播，后半夜才开播的抓不到
- 单枚牌子一天最多 +10（看直播）+ 6（投喂灯牌），升级是慢活
- 上游本身的功能（投币、大会员权益、直播抽奖等）都没动，需要的话改 `tasks` 参数
