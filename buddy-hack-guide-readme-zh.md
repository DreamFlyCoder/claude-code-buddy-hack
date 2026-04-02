# Claude Code Buddy

## Buddy 是什么？

Claude Code 里的 **Buddy** 是一个**终端电子宠物**（编程搭子），不是核心开发工具。它本质上是一个**彩蛋功能**，用来在写代码时提供"赛博陪伴"，**完全不影响你的开发、调试、Git 等正经功能**。

## 快速开始

> 说在前面：你可以先直接执行第 5 步看看自己的运气，说不定直接就出好的了呢？

1. 首先确认你的 `claude --version` > 2.1.89，否则执行：
   ```bash
   npm install -g @anthropic-ai/claude-code@latest
   ```

2. 把附录中的 `buddy-hack-guide.md` 文件放到一个 Claude Code 能读到的地方（比如项目根目录的 `.claude/` 下）

3. 在 Claude Code 中输入："参考这个文档帮我改一个 buddy"

4. 输入你想要的 buddy 配置，一路确认到结束，然后 `/exit` 退出

5. 在终端输入 `claude` 启动，在 Claude Code 中输入 `/buddy` 即可

这样你就获得了一个 Claude Code 编码伙伴！

## 工作原理

Buddy 的所有属性（稀有度、物种、眼睛、帽子、闪光、属性值）都是根据你的用户 ID 通过确定性算法实时计算的：

```
seed = accountUuid + "friend-2026-401"
hash = FNV-1a(seed)
rng  = Mulberry32(hash)
buddy = generateBones(rng)
```

只要用户 ID 不变，每次 `/buddy` 生成的宠物永远一样。改变 ID 就能改变宠物。

## 可选属性一览

| 选项 | 可选值 |
|------|--------|
| **物种** | duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk |
| **稀有度** | common, uncommon, rare, epic, **legendary** |
| **眼睛** | `·` `✦` `×` `◉` `@` `°` |
| **帽子** | none, crown, tophat, propeller, halo, wizard, beanie, tinyduck（common 固定 none） |
| **闪光** | 1% 概率，脚本会帮你刷到 |

## 注意事项

- `seed` 后缀 `friend-2026-401` 可能随 CLI 版本更新变化，届时所有人的宠物会重置
- 如果担心有风险，可以随时通过 `cp ~/.claude.json.bak ~/.claude.json` 恢复原始配置
- Claude Code 更新后如果宠物变了，说明 seed 后缀或算法改了，需要重新逆向

## 附录

详细的逆向原理和一键脚本请参考：[buddy-hack-guide.md](buddy-hack-guide.md)
