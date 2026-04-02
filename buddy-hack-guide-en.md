# Claude Code Buddy (Companion) Reverse-Engineering Reset Guide

> Version: 2026-04-01 | CLI version 2.1.87 | seed suffix `friend-2026-401`

## Quick Start (One-Click Pet Swap)

### Step 1: Choose Your Desired Buddy Configuration

| Option | Available Values |
|--------|-----------------|
| **Species** | duck, goose, blob, cat, dragon, octopus, owl, penguin, turtle, snail, ghost, axolotl, capybara, cactus, robot, rabbit, mushroom, chonk |
| **Rarity** | common, uncommon, rare, epic, **legendary** |
| **Eyes** | `·`(1) `✦`(2) `×`(3) `◉`(4) `@`(5) `°`(6) |
| **Hat** | none, crown, tophat, propeller, halo, wizard, beanie, tinyduck (common is always none) |
| **Shiny** | Add `--shiny` flag (1% chance, the script will brute-force it for you) |

### Step 2: Run the One-Click Script

Save the script below as `/tmp/buddy-onestep.js`, then run it — **no manual confirmation needed**:

```bash
# Example: legendary shiny dragon with crown and ✦ eyes
node /tmp/buddy-onestep.js legendary dragon --shiny --eye "✦" --hat crown

# Example: legendary shiny axolotl with propeller hat and @ eyes
node /tmp/buddy-onestep.js legendary axolotl --shiny --eye "@" --hat propeller

# Example: just a legendary dragon, no picky details
node /tmp/buddy-onestep.js legendary dragon
```

The script will output:
```
✅ Done! Restart Claude Code and type /buddy to hatch your new pet.
   To restore original config: cp ~/.claude.json.bak ~/.claude.json
```

**Then restart Claude Code, type `/buddy`, and you're done.**

---

## One-Click Script Source Code

```javascript
#!/usr/bin/env node
// Claude Code Buddy One-Click Pet Swap Script
// Usage: node buddy-onestep.js <rarity> [species] [--shiny] [--eye "X"] [--hat name]
// Fully automated: brute-force ID → backup → modify config → delete old pet, no manual steps

const fs = require("fs");
const path = require("path");
const { randomUUID } = require("crypto");

// ─── Constants (reverse-engineered from cli.js) ───
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

// ─── Core Algorithm (reverse-engineered from cli.js) ───
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

// ─── Parse Command-Line Arguments ───
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

// ─── Argument Validation ───
if (!RARITY_ORDER.includes(TARGET_RARITY)) {
  console.error(`❌ Invalid rarity: ${TARGET_RARITY}\n   Options: ${RARITY_ORDER.join(", ")}`);
  process.exit(1);
}
if (TARGET_SPECIES && !SPECIES.includes(TARGET_SPECIES)) {
  console.error(`❌ Invalid species: ${TARGET_SPECIES}\n   Options: ${SPECIES.join(", ")}`);
  process.exit(1);
}
if (TARGET_EYE && !EYES.includes(TARGET_EYE)) {
  console.error(`❌ Invalid eyes: ${TARGET_EYE}\n   Options: ${EYES.join(" ")}`);
  process.exit(1);
}
if (TARGET_HAT && !HATS.includes(TARGET_HAT)) {
  console.error(`❌ Invalid hat: ${TARGET_HAT}\n   Options: ${HATS.join(", ")}`);
  process.exit(1);
}

// ─── Phase 1: Brute-Force Search for Matching ID ───
console.log("🔍 Searching for matching Buddy ID...");
console.log(`   Target: rarity=${TARGET_RARITY} species=${TARGET_SPECIES || "any"} eye=${TARGET_EYE || "any"} hat=${TARGET_HAT || "any"} shiny=${WANT_SHINY}`);

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
  console.error(`❌ No match found in ${MAX_ATTEMPTS.toLocaleString()} attempts. Try relaxing conditions (remove --eye / --hat / --shiny).`);
  process.exit(1);
}

// Sort by total stats, pick the highest
found.sort((a, b) => b.totalStats - a.totalStats);
const best = found[0];

console.log(`\n🎯 Found ${found.length} matches, selecting the one with highest stats:`);
console.log(`   UUID:    ${best.uid}`);
console.log(`   Species: ${best.buddy.species}`);
console.log(`   Rarity:  ${best.buddy.rarity}`);
console.log(`   Eye:     ${best.buddy.eye}`);
console.log(`   Hat:     ${best.buddy.hat}`);
console.log(`   Shiny:   ${best.buddy.shiny}`);
console.log(`   Stats:   ${JSON.stringify(best.buddy.stats)} (total: ${best.totalStats})`);

// ─── Phase 2: Backup and Modify ~/.claude.json ───
const claudeJsonPath = path.join(process.env.HOME, ".claude.json");

if (!fs.existsSync(claudeJsonPath)) {
  console.error(`❌ Cannot find ${claudeJsonPath}. Please launch Claude Code normally at least once first.`);
  process.exit(1);
}

// Backup
const backupPath = claudeJsonPath + ".bak";
fs.copyFileSync(claudeJsonPath, backupPath);
console.log(`\n💾 Backed up to ${backupPath}`);

// Read and modify
const data = JSON.parse(fs.readFileSync(claudeJsonPath, "utf-8"));

// Modify accountUuid (highest priority seed source)
if (data.oauthAccount) {
  data.oauthAccount.accountUuid = best.uid;
  console.log(`🔧 accountUuid → ${best.uid}`);
} else if (data.userID !== undefined) {
  data.userID = best.uid;
  console.log(`🔧 userID → ${best.uid}`);
} else {
  data.userID = best.uid;
  console.log(`🔧 Writing userID → ${best.uid}`);
}

// Delete existing companion
if (data.companion) {
  delete data.companion;
  console.log("🗑️  Deleted old companion");
}

// Write back
fs.writeFileSync(claudeJsonPath, JSON.stringify(data, null, 4), "utf-8");

// ─── Phase 3: Verification ───
const verify = JSON.parse(fs.readFileSync(claudeJsonPath, "utf-8"));
const verifyId = verify.oauthAccount?.accountUuid ?? verify.userID ?? "anon";
const verifyBuddy = generateBuddy(verifyId);

console.log(`\n🔍 Verification result:`);
console.log(`   seed ID:  ${verifyId}`);
console.log(`   Species:  ${verifyBuddy.species} | Rarity: ${verifyBuddy.rarity} | Eye: ${verifyBuddy.eye} | Hat: ${verifyBuddy.hat} | Shiny: ${verifyBuddy.shiny}`);

if (verifyBuddy.rarity === TARGET_RARITY &&
    (!TARGET_SPECIES || verifyBuddy.species === TARGET_SPECIES) &&
    (!TARGET_EYE || verifyBuddy.eye === TARGET_EYE) &&
    (!TARGET_HAT || verifyBuddy.hat === TARGET_HAT) &&
    (!WANT_SHINY || verifyBuddy.shiny)) {
  console.log(`\n✅ Done! Restart Claude Code and type /buddy to hatch your new pet.`);
  console.log(`   To restore original config: cp ~/.claude.json.bak ~/.claude.json`);
} else {
  console.error(`\n❌ Verification failed! Config may have been written incorrectly. Please manually check ~/.claude.json`);
  process.exit(1);
}
```

---

## How It Works

### Core Logic (Reverse-Engineered from cli.js)

```
ch1() → oauthAccount?.accountUuid ?? userID ?? "anon"
seed   = ch1() + "friend-2026-401"          // append fixed suffix
hash   = FNV-1a(seed)                       // 32-bit FNV-1a hash
rng    = Mulberry32(hash)                   // Mulberry32 PRNG
buddy  = generateBones(rng)                 // deterministically generate all attributes
```

**Key Points**:
- As long as `accountUuid` (or `userID`) stays the same, `/buddy` will always generate the exact same pet
- Changing this ID changes the pet
- `oauthAccount.accountUuid` has the highest priority, followed by `userID`, with `"anon"` as the final fallback

### Attribute Generation Rules

| Attribute | Source |
|-----------|--------|
| **Rarity** | Weighted random: common:60, uncommon:25, rare:10, epic:4, legendary:1 |
| **Species** | 18 types (see table above) |
| **Eyes** | 6 types: `·` `✦` `×` `◉` `@` `°` |
| **Hat** | common is always none; others randomly selected from 8 options |
| **Shiny** | 1% chance |
| **Stats** | 5 categories (DEBUGGING, PATIENCE, CHAOS, WISDOM, SNARK), base values determined by rarity |

**Stat Base Values**: common=5, uncommon=15, rare=25, epic=35, legendary=50

### Why Changing accountUuid Is Sufficient

The `companion` field only stores `name` and `personality` (AI-generated text during hatching). The actual skeleton attributes (rarity/species/eye/hat/shiny/stats) are **computed in real-time** from the ID every time. Therefore:

1. Change `accountUuid` → changes the seed → changes all skeleton attributes
2. Delete `companion` → next `/buddy` triggers re-hatching (generates new name/personality)
3. **Don't restore the original accountUuid** → pet stays permanently

---

## Alternative Approach: Bypass accountUuid via OAUTH_TOKEN

When logging in with the `CLAUDE_CODE_OAUTH_TOKEN` environment variable, `accountUuid` is not written to `~/.claude.json`, and `ch1()` falls back to the `userID` field.

```bash
# 1. Get token
claude setup-token

# 2. Reset config
cp ~/.claude.json ~/.claude.json.bak
echo '{"hasCompletedOnboarding": true, "theme": "dark"}' > ~/.claude.json

# 3. Launch with token (generates new .claude.json without accountUuid)
export CLAUDE_CODE_OAUTH_TOKEN=<your-token>
claude   # exit immediately, don't use /buddy

# 4. Use the one-click script to change userID (script will automatically use the userID branch)
node /tmp/buddy-onestep.js legendary dragon --shiny

# 5. Restart claude, type /buddy
```

This approach is safer (doesn't touch accountUuid) but involves more steps.

---

## Important Notes

- **Seed suffix `friend-2026-401`** may change with CLI version updates, which would reset everyone's pets
- **Brute-force speed**: A plain legendary takes ~0.01s; legendary + species + eye + hat + shiny takes ~5-30s
- **After Claude Code updates**: If your pet changes, the seed suffix or algorithm was modified — you'll need to reverse-engineer cli.js again
- **Recovery**: You can always restore with `cp ~/.claude.json.bak ~/.claude.json`

---

## Verified Premium IDs (2026-04-01 Version)

| userID | Species | Rarity | Eyes | Hat | Shiny | Total Stats |
|--------|---------|--------|------|-----|-------|-------------|
| `cced1e80-f1d3-4815-9621-4bfe3dca5815` | dragon | legendary | ✦ | crown | Yes | 348 |
| `54753225-8593-4aea-8326-336094ea4e71` | dragon | legendary | ° | none | Yes | 398 |
| `5de0cdb8-09eb-46af-8c87-057cd60022cf` | axolotl | legendary | @ | propeller | Yes | 354 |
| `456b2ae6-c2aa-4eb0-a1cd-62f103dcb2fb` | axolotl | legendary | ◉ | propeller | No | 392 |
| `268a9b63-85c0-4644-82e6-481769503e39` | robot | legendary | @ | tinyduck | No | 360 |
| `c617b844-f248-40a6-859e-a034f62280c6` | dragon | legendary | ◉ | crown | No | 348 |
| `77745fd7-fcc1-4a69-bc0f-56d30fb991ba` | axolotl | legendary | ✦ | crown | Yes | 392 |


## Final Notes
If you're worried about account risks and have already run the script, you can restore the original config with `cp ~/.claude.json.bak ~/.claude.json`.
