# 处境解 Skill 包 v2（dbskill 思路重构版）

## 相比 v1 的改动

参考 [dbskill](https://github.com/dontbesilent2025/dbskill) 的架构思路，做了四层改动：

1. **内容原子化**：所有内容不再直接写死在 SKILL.md 里，而是拆成结构化的 `references/atoms.jsonl`，每条带 `type`、`confidence`、`topic` 等标签，SKILL.md 只保留路由逻辑和对话原则，按需引用 atoms。
2. **单 skill 独立**：每个 skill 自带自己的 `references/atoms.jsonl`，不依赖其他 skill 才能运作——单独装一个 skill 也能完整工作。
3. **主入口 + 消解预检**：新增 `chujingjie` 作为主入口，先判断用户的困惑值不值得走完整诊断（伪问题/单点认知偏差/信息不足 vs 真正需要诊断），再决定调用哪个子 skill，且不预设固定顺序，每一步做完根据实际结论决定下一步。
4. **原来的"四步"拆成四个独立 skill**：`chujingjie-nengliquan`（能力圈）/ `chujingjie-mubiao`（目标）/ `chujingjie-guihua`（规划）/ `chujingjie-zhixing`（执行），可以被路由调用，也可以被用户的具体表达直接单独命中，不强制走完四步。

## 目录结构

```
skills/
  chujingjie/                    # 主入口：消解 + 路由，不产出诊断内容
  chujingjie-nengliquan/         # 能力圈（原第一步）
    SKILL.md
    references/atoms.jsonl
  chujingjie-mubiao/             # 目标（原第二步）
    SKILL.md
    references/atoms.jsonl
  chujingjie-guihua/             # 规划（原第三步）
    SKILL.md
    references/atoms.jsonl
  chujingjie-zhixing/            # 执行（原第四步）
    SKILL.md
    references/atoms.jsonl
  chujingjie-zhongda-juece/      # 重大决策工具箱（原本就独立，这次改为引用atoms）
    SKILL.md
    references/atoms.jsonl
  chujingjie-renzhi-pianwu/      # 认知偏差清单（原本就独立，这次改为引用atoms）
    SKILL.md
    references/atoms.jsonl
state/
  README.md                      # 跨会话状态记录的机制说明和已知限制
```

## atoms.jsonl 字段说明

两种类型的条目：

**编号问答类**（来自原有的问题库）：
```json
{"id": "...", "skill": "...", "type": "principle", "confidence": "high", "topic": "...",
 "question": "追问角度", "core_insight": "要传达的认知", "misconception": "常见误区",
 "plain_talk": "口语化表达", "action": "可执行的小动作", "quote": "金句"}
```

**自由补充类**（来自往期自媒体文案提炼的思维工具，比如逆向倒推法、杠铃策略）：
```json
{"id": "...", "skill": "...", "type": "principle", "confidence": "high", "topic": "...",
 "title": "工具名", "content": "完整说明"}
```

`confidence` 目前全部标为 `high`（因为是从已经提炼过的内容转换而来）。以后如果直接从原始文案/推文批量生成 atoms，应该按实际置信度区分 `high`/`medium`/`low`，方便后续按置信度筛选。

## 已知未完成事项（诚实列出，不是全部做完了）

- **消解判断（chujingjie 的第一步）没有实测**：写的是规则，但"1-2个问题判断出属于哪一类"的实际效果，需要用真实对话测试，不同底层模型（尤其是较弱的模型）的判断准确率可能有明显差异，建议正式使用前先跑几个真实案例测试。
- **豆包侧的落地方式还没确认**：这次重构默认目标环境是支持多文件 skill 结构的环境（比如 Claude）。如果要在豆包上复现同样的效果，需要先确认豆包智能体后台是否支持类似"按需加载知识库"的机制，而不是直接把六个 SKILL.md 全部塞进一个 system prompt——那样等于走回了旧豆包版的老路，之前提的 token 膨胀和路由脆弱问题会原样出现。这一步我没有能力代你确认，需要你去看豆包的产品文档或实测。
- **atoms.jsonl 目前是从已经写好的中文内容自动解析出来的**，不是从原始素材重新原子化提炼的，所以 `confidence` 字段目前区分度不大，只是先把结构立起来。
