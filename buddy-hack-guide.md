# Claude Code Buddy (Companion) 逆向重置指南

> 版本：2026-04-01 | CLI 版本 2.1.87 | seed 后缀 `friend-2026-401`

## 快速开始（一键换宠）

### 第一步：选择你想要的 Buddy 配置

| 选项 | 可选值 |
|------|--------|
| **物种** | duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk |
| **稀有度** | common, uncommon, rare, epic, **legendary** |
| **眼睛** | `·`(1) `✦`(2) `×`(3) `◉`(4) `@`(5) `°`(6) |
| **帽子** | none, crown, tophat, propeller, halo, wizard, beanie, tinyduck（common 固定 none） |
| **闪光** | 加 `--shiny` 参数（1% 概率，脚本会帮你刷到） |

### 第二步：运行一键脚本

将下方脚本保存为 `/tmp/buddy-onestep.js`，然后运行即可，**无需手动确认任何步骤**：

```bash
# 示例：legendary 闪光龙，戴皇冠，✦ 眼
node /tmp/buddy-onestep.js legendary dragon --shiny --eye "✦" --hat crown

# 示例：legendary 闪光六角恐龙，螺旋桨帽，@ 眼
node /tmp/buddy-onestep.js legendary axolotl --shiny --eye "@" --hat propeller

# 示例：只要 legendary 龙，不挑细节
node /tmp/buddy-onestep.js legendary dragon
```

脚本执行完会输出：
```
✅ Done! 重启 Claude Code 后输入 /buddy 即可孵化新宠物。
   如需恢复原配置：cp ~/.claude.json.bak ~/.claude.json
```

**然后重启 Claude Code，输入 `/buddy`，结束。**

---

## 一键脚本源码

```javascript
#!/usr/bin/env node
// Claude Code Buddy 一键换宠脚本
// 用法: node buddy-onestep.js <rarity> [species] [--shiny] [--eye "X"] [--hat name]
// 全自动：刷 ID → 备份 → 改配置 → 删旧宠物，无需手动确认

const fs = require("fs");
const path = require("path");
const { randomUUID } = require("crypto");

// ─── 常量（逆向自 cli.js）───
const SEED_SUFFIX = "friend-2026-401";
const SPECIES = ["duck","goose","blob","cat","dragon","octopus","owl","penguin",
                 "turtle","snail","ghost","axolotl","capybara","cactus","robot",
                 "rabbit","mushroom","chonk"];
const EYES = ["·","✦","×","◉","@","°"];
const HATS = ["none","crown","tophat","propeller","halo","wizard","beanie","tinyduck"];
const STATS_KEYS = ["DEBUGGING","PATIENCE","CHAOS","WISDOM","SNARK"];
const RARITY_WEIGHTS = {common:60, uncommon:25, rare:10, epic:4, legendary:1};
const RARITY_ORDER = ["common","uncommon","rare","epic","legendary"];
const RARITY_BASE = {common:5, uncommon:15, rare:25, epic:35, legendary:50};

// ─── 核心算法（逆向自 cli.js）───
function fnvHash(str) {
  let h = 2166136261;
  for (let i = 0; i < str.length; i++) {
    h ^= str.charCodeAt(i);
    h = Math.imul(h, 16777619);
  }
  return h >>> 0;
}

function mulberry32(seed) {
  let s = seed >>> 0;
  return function() {
    s |= 0; s = s + 1831565813 | 0;
    let t = Math.imul(s ^ s >>> 15, 1 | s);
    t = t + Math.imul(t ^ t >>> 7, 61 | t) ^ t;
    return ((t ^ t >>> 14) >>> 0) / 4294967296;
  };
}

function pick(rng, arr) { return arr[Math.floor(rng() * arr.length)]; }

function pickRarity(rng) {
  let r = rng() * 100;
  for (const k of RARITY_ORDER) { r -= RARITY_WEIGHTS[k]; if (r < 0) return k; }
  return "common";
}

function genStats(rng, rarity) {
  const base = RARITY_BASE[rarity];
  const primary = pick(rng, STATS_KEYS);
  let secondary = pick(rng, STATS_KEYS);
  while (secondary === primary) secondary = pick(rng, STATS_KEYS);
  const stats = {};
  for (const k of STATS_KEYS) {
    if (k === primary) stats[k] = Math.min(100, base + 50 + Math.floor(rng() * 30));
    else if (k === secondary) stats[k] = Math.max(1, base - 10 + Math.floor(rng() * 15));
    else stats[k] = base + Math.floor(rng() * 40);
  }
  return stats;
}

function generateBuddy(userId) {
  const rng = mulberry32(fnvHash(userId + SEED_SUFFIX));
  const rarity = pickRarity(rng);
  const species = pick(rng, SPECIES);
  const eye = pick(rng, EYES);
  const hat = rarity === "common" ? "none" : pick(rng, HATS);
  const shiny = rng() < 0.01;
  const stats = genStats(rng, rarity);
  return { rarity, species, eye, hat, shiny, stats };
}

// ─── 解析命令行参数 ───
const args = process.argv.slice(2);
const TARGET_RARITY = args[0] || "legendary";
const TARGET_SPECIES = args[1] && !args[1].startsWith("--") ? args[1] : null;
const WANT_SHINY = args.includes("--shiny");

let TARGET_EYE = null;
let TARGET_HAT = null;
const eyeIdx = args.indexOf("--eye");
if (eyeIdx !== -1 && args[eyeIdx + 1]) TARGET_EYE = args[eyeIdx + 1];
const hatIdx = args.indexOf("--hat");
if (hatIdx !== -1 && args[hatIdx + 1]) TARGET_HAT = args[hatIdx + 1];

// ─── 参数校验 ───
if (!RARITY_ORDER.includes(TARGET_RARITY)) {
  console.error(`❌ 无效稀有度: ${TARGET_RARITY}\n   可选: ${RARITY_ORDER.join(", ")}`);
  process.exit(1);
}
if (TARGET_SPECIES && !SPECIES.includes(TARGET_SPECIES)) {
  console.error(`❌ 无效物种: ${TARGET_SPECIES}\n   可选: ${SPECIES.join(", ")}`);
  process.exit(1);
}
if (TARGET_EYE && !EYES.includes(TARGET_EYE)) {
  console.error(`❌ 无效眼睛: ${TARGET_EYE}\n   可选: ${EYES.join(" ")}`);
  process.exit(1);
}
if (TARGET_HAT && !HATS.includes(TARGET_HAT)) {
  console.error(`❌ 无效帽子: ${TARGET_HAT}\n   可选: ${HATS.join(", ")}`);
  process.exit(1);
}

// ─── Phase 1: 暴力搜索匹配 ID ───
console.log("🔍 搜索匹配的 Buddy ID...");
console.log(`   目标: rarity=${TARGET_RARITY} species=${TARGET_SPECIES || "any"} eye=${TARGET_EYE || "any"} hat=${TARGET_HAT || "any"} shiny=${WANT_SHINY}`);

const MAX_ATTEMPTS = 50_000_000;
let found = [];
const startTime = Date.now();

for (let i = 0; i < MAX_ATTEMPTS; i++) {
  const uid = randomUUID();
  const buddy = generateBuddy(uid);

  if (buddy.rarity !== TARGET_RARITY) continue;
  if (TARGET_SPECIES && buddy.species !== TARGET_SPECIES) continue;
  if (TARGET_EYE && buddy.eye !== TARGET_EYE) continue;
  if (TARGET_HAT && buddy.hat !== TARGET_HAT) continue;
  if (WANT_SHINY && !buddy.shiny) continue;

  const totalStats = Object.values(buddy.stats).reduce((a, b) => a + b, 0);
  found.push({ uid, buddy, totalStats });

  if (found.length >= 20) break;

  if (i > 0 && i % 5_000_000 === 0) {
    console.log(`   ... ${(i / 1e6).toFixed(0)}M checked, ${found.length} found, ${((Date.now() - startTime) / 1000).toFixed(1)}s`);
  }
}

if (found.length === 0) {
  console.error(`❌ 在 ${MAX_ATTEMPTS.toLocaleString()} 次尝试中未找到匹配。尝试放宽条件（去掉 --eye / --hat / --shiny）。`);
  process.exit(1);
}

// 按总属性排序，取最高的
found.sort((a, b) => b.totalStats - a.totalStats);
const best = found[0];

console.log(`\n🎯 找到 ${found.length} 个匹配，选择属性最高的：`);
console.log(`   UUID:    ${best.uid}`);
console.log(`   Species: ${best.buddy.species}`);
console.log(`   Rarity:  ${best.buddy.rarity}`);
console.log(`   Eye:     ${best.buddy.eye}`);
console.log(`   Hat:     ${best.buddy.hat}`);
console.log(`   Shiny:   ${best.buddy.shiny}`);
console.log(`   Stats:   ${JSON.stringify(best.buddy.stats)} (total: ${best.totalStats})`);

// ─── Phase 2: 备份并修改 ~/.claude.json ───
const claudeJsonPath = path.join(process.env.HOME, ".claude.json");

if (!fs.existsSync(claudeJsonPath)) {
  console.error(`❌ 找不到 ${claudeJsonPath}，请先正常启动一次 Claude Code。`);
  process.exit(1);
}

// 备份
const backupPath = claudeJsonPath + ".bak";
fs.copyFileSync(claudeJsonPath, backupPath);
console.log(`\n💾 已备份到 ${backupPath}`);

// 读取并修改
const data = JSON.parse(fs.readFileSync(claudeJsonPath, "utf-8"));

// 修改 accountUuid（优先级最高的 seed 来源）
if (data.oauthAccount) {
  data.oauthAccount.accountUuid = best.uid;
  console.log(`🔧 accountUuid → ${best.uid}`);
} else if (data.userID !== undefined) {
  data.userID = best.uid;
  console.log(`🔧 userID → ${best.uid}`);
} else {
  data.userID = best.uid;
  console.log(`🔧 写入 userID → ${best.uid}`);
}

// 删除已有 companion
if (data.companion) {
  delete data.companion;
  console.log("🗑️  已删除旧 companion");
}

// 写回
fs.writeFileSync(claudeJsonPath, JSON.stringify(data, null, 4), "utf-8");

// ─── Phase 3: 验证 ───
const verify = JSON.parse(fs.readFileSync(claudeJsonPath, "utf-8"));
const verifyId = verify.oauthAccount?.accountUuid ?? verify.userID ?? "anon";
const verifyBuddy = generateBuddy(verifyId);

console.log(`\n🔍 验证结果：`);
console.log(`   seed ID:  ${verifyId}`);
console.log(`   Species:  ${verifyBuddy.species} | Rarity: ${verifyBuddy.rarity} | Eye: ${verifyBuddy.eye} | Hat: ${verifyBuddy.hat} | Shiny: ${verifyBuddy.shiny}`);

if (verifyBuddy.rarity === TARGET_RARITY &&
    (!TARGET_SPECIES || verifyBuddy.species === TARGET_SPECIES) &&
    (!TARGET_EYE || verifyBuddy.eye === TARGET_EYE) &&
    (!TARGET_HAT || verifyBuddy.hat === TARGET_HAT) &&
    (!WANT_SHINY || verifyBuddy.shiny)) {
  console.log(`\n✅ Done! 重启 Claude Code 后输入 /buddy 即可孵化新宠物。`);
  console.log(`   如需恢复原配置：cp ~/.claude.json.bak ~/.claude.json`);
} else {
  console.error(`\n❌ 验证失败！配置可能写入有误，请手动检查 ~/.claude.json`);
  process.exit(1);
}
```

---

## 原理详解

### 核心代码逻辑（逆向自 cli.js）

```
ch1() → oauthAccount?.accountUuid ?? userID ?? "anon"
seed   = ch1() + "friend-2026-401"          // 拼接固定后缀
hash   = FNV-1a(seed)                       // 32位 FNV-1a 哈希
rng    = Mulberry32(hash)                   // Mulberry32 伪随机数生成器
buddy  = generateBones(rng)                 // 确定性生成全部属性
```

**关键点**：
- 只要 `accountUuid`（或 `userID`）不变，每次 `/buddy` 生成的宠物永远一样
- 改变这个 ID 就能改变宠物
- `oauthAccount.accountUuid` 优先级最高，其次是 `userID`，最后 fallback 到 `"anon"`

### 属性生成规则

| 属性 | 来源 |
|------|------|
| **稀有度** | 加权随机：common:60, uncommon:25, rare:10, epic:4, legendary:1 |
| **物种** | 18 种（见上方表格） |
| **眼睛** | 6 种：`·` `✦` `×` `◉` `@` `°` |
| **帽子** | common 固定 none；其余从 8 种中随机 |
| **闪光** | 1% 概率 |
| **属性值** | 5 项（DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK），基础值由稀有度决定 |

**属性基础值**：common=5, uncommon=15, rare=25, epic=35, legendary=50

### 为什么改 accountUuid 就够了

`companion` 字段只存 `name` 和 `personality`（孵化时 AI 生成的文本），真正的骨架属性（rarity/species/eye/hat/shiny/stats）是每次从 ID **实时计算**的。所以：

1. 改 `accountUuid` → 改变 seed → 改变所有骨架属性
2. 删除 `companion` → 下次 `/buddy` 重新孵化（生成新的 name/personality）
3. **不恢复原 accountUuid** → 宠物永久保持

---

## 备选方案：通过 OAUTH_TOKEN 绕过 accountUuid

当用 `CLAUDE_CODE_OAUTH_TOKEN` 环境变量登录时，`accountUuid` 不会写入 `~/.claude.json`，`ch1()` 会 fallback 到 `userID` 字段。

```bash
# 1. 获取 token
claude setup-token

# 2. 重置配置
cp ~/.claude.json ~/.claude.json.bak
echo '{"hasCompletedOnboarding": true, "theme": "dark"}' > ~/.claude.json

# 3. 用 token 启动（生成新的 .claude.json，不含 accountUuid）
export CLAUDE_CODE_OAUTH_TOKEN=<your-token>
claude   # 直接退出，不要 /buddy

# 4. 用一键脚本改 userID（此时脚本会自动走 userID 分支）
node /tmp/buddy-onestep.js legendary dragon --shiny

# 5. 重启 claude，输入 /buddy
```

此方案更安全（不动 accountUuid），但步骤更多。

---

## 注意事项

- **seed 后缀 `friend-2026-401`** 可能随 CLI 版本更新变化，届时所有人的宠物会重置
- **暴力脚本速度**：普通 legendary 约 0.01 秒；legendary + 物种 + 眼 + 帽 + shiny ≈ 5-30 秒
- **Claude Code 更新后**：如果宠物变了，说明 seed 后缀或算法改了，需要重新逆向 cli.js
- **恢复**：随时 `cp ~/.claude.json.bak ~/.claude.json` 恢复原配置

---

## 已验证的精品 ID（2026-04-01 版本）

| userID | 物种 | 稀有度 | 眼睛 | 帽子 | Shiny | 总属性 |
|--------|------|--------|------|------|-------|--------|
| `cced1e80-f1d3-4815-9621-4bfe3dca5815` | dragon | legendary | ✦ | crown | Yes | 348 |
| `54753225-8593-4aea-8326-336094ea4e71` | dragon | legendary | ° | none | Yes | 398 |
| `5de0cdb8-09eb-46af-8c87-057cd60022cf` | axolotl | legendary | @ | propeller | Yes | 354 |
| `456b2ae6-c2aa-4eb0-a1cd-62f103dcb2fb` | axolotl | legendary | ◉ | propeller | No | 392 |
| `268a9b63-85c0-4644-82e6-481769503e39` | robot | legendary | @ | tinyduck | No | 360 |
| `c617b844-f248-40a6-859e-a034f62280c6` | dragon | legendary | ◉ | crown | No | 348 |
| `77745fd7-fcc1-4a69-bc0f-56d30fb991ba` | axolotl | legendary | ✦ | crown | Yes | 392 |


## 写在最后
如果担心有封号的风险，并且已经运行过，可以通过 cp ~/.claude.json.bak ~/.claude.json 恢复原始配置。
