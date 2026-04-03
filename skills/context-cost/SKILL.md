---
name: tortuga:context-cost
description: Break down exactly what's eating the Claude Code context window using actual API token counts. Shows skills, memory, CLAUDE.md, and MCP costs with an interactive cleanup menu.
license: MIT
metadata:
  author: tortuga-ai
  version: "1.0.0"
  organization: Tortuga AI
  public: true
  repo: https://github.com/tortuga-ai/skills
  date: March 2026
  abstract: Context window cost analyzer for Claude Code. Runs a single Python script to measure token usage from CLAUDE.md files, memory files, skills (global, plugin, builtin), and project commands. Reads deferred MCP tools from conversation context. Outputs a ranked breakdown and presents an interactive cleanup menu to remove skills, trim CLAUDE.md, or prune memory.
---

Break down exactly what's eating the Claude Code context window using actual API token counts.

## Step 1: Run a single Python script to get all data

```bash
python3 - << 'EOF'
import os, re, glob, json

home = os.path.expanduser("~")
cwd = os.getcwd()
user = os.environ.get("USER", os.path.basename(home))
proj_suffix = cwd.replace(f"/Users/{user}", "").replace("/", "-")
proj_key = f"-Users-{user}{proj_suffix}"
proj_dir = os.path.join(home, ".claude", "projects", proj_key)

# --- Session tokens ---
jsonl_files = sorted(glob.glob(os.path.join(proj_dir, "*.jsonl")), key=os.path.getmtime, reverse=True)
initial, current = None, None
if jsonl_files:
    lines = open(jsonl_files[0]).readlines()
    for line in lines:
        m = re.search(r'"cache_creation_input_tokens":(\d+)', line)
        if m and initial is None:
            initial = int(m.group(1))
    for line in reversed(lines):
        m = re.search(r'"cache_read_input_tokens":(\d+)', line)
        if m:
            current = int(m.group(1))
            break

print(f"Initial context : {initial or 'N/A'} tokens")
print(f"Current context : {current or 'N/A'} tokens\n")

# --- File sizes ---
def measure(path, label):
    path = os.path.expanduser(path)
    if not os.path.isfile(path): return
    tokens = os.path.getsize(path) // 4
    print(f"  {label:<42} {tokens:>5} tokens")

measure("~/.claude/CLAUDE.md", "CLAUDE.md (user)")
measure(os.path.join(cwd, "CLAUDE.md"), "CLAUDE.md (project)")
mem_dir = os.path.join(proj_dir, "memory")
for f in sorted(glob.glob(os.path.join(mem_dir, "*.md")) + glob.glob(os.path.join(mem_dir, "*.json"))):
    measure(f, f"memory/{os.path.basename(f)}")

# --- Skills ---
def get_description(path):
    try:
        text = open(path).read()
        if text.startswith('---'):
            fm_match = re.match(r"^---\s*\n(.*?)\n---", text, re.DOTALL)
            if fm_match:
                fm = fm_match.group(1)
                m = re.search(r"^description:\s*[>|]?\s*\n((?:[ \t]+.+\n?)+)", fm, re.MULTILINE)
                if m:
                    return re.sub(r"[ \t]+", " ", m.group(1).replace("\n", " ")).strip()
                m = re.search(r"^description:\s*[\x22\x27]?(.+?)[\x22\x27]?\s*$", fm, re.MULTILINE)
                if m:
                    return m.group(1).strip()
            text = re.sub(r"^---.*?---\s*", "", text, flags=re.DOTALL)
        paras = [p.strip() for p in text.split('\n\n') if p.strip()]
        return paras[0] if paras else ''
    except:
        return ''

total = 0
rows = []

# User-level and project-level skills
for scope, skills_dir in [("user", os.path.join(home, ".claude", "skills")), ("project", os.path.join(cwd, ".claude", "skills"))]:
    for skill_dir in sorted(glob.glob(os.path.join(skills_dir, "*"))):
        name = os.path.basename(skill_dir)
        real = os.path.realpath(skill_dir)
        for f in glob.glob(os.path.join(real, "SKILL.md")) + glob.glob(os.path.join(real, "*.md")):
            desc = get_description(f)
            if desc:
                t = len(desc) // 4
                total += t
                rows.append((f"skill({scope}):{name}", t))
                break

# --- Plugins (from installed_plugins.json) ---
seen = set()
try:
    installed = json.load(open(os.path.join(home, ".claude", "plugins", "installed_plugins.json")))
    for key, installs in installed.get("plugins", {}).items():
        plugin_name = key.split("@")[0]
        for inst in installs:
            install_path = inst.get("installPath", "")
            for f in glob.glob(os.path.join(install_path, "skills", "**", "SKILL.md"), recursive=True) + \
                     glob.glob(os.path.join(install_path, "skills", "**", "*.md"), recursive=True):
                parts = f.split(os.sep)
                if "skills" not in parts: continue
                skill_idx = max(i for i, p in enumerate(parts) if p == "skills")
                if skill_idx + 1 >= len(parts) - 1: continue
                skill_name = parts[skill_idx + 1]
                uid = f"{plugin_name}:{skill_name}"
                if uid in seen: continue
                seen.add(uid)
                desc = get_description(f)
                if desc:
                    t = len(desc) // 4
                    total += t
                    rows.append((f"plugin:{plugin_name}:{skill_name}", t))
except:
    pass

BUILTIN_SKILLS = {
    "keybindings-help": "Use when the user wants to customize keyboard shortcuts, rebind keys, add chord bindings, or modify ~/.claude/keybindings.json.",
    "simplify": "Review changed code for reuse, quality, and efficiency, then fix any issues found.",
    "loop": "Run a prompt or slash command on a recurring interval (e.g. /loop 5m /foo, defaults to 10m).",
    "claude-api": "Build apps with the Claude API or Anthropic SDK. TRIGGER when: code imports anthropic or claude_agent_sdk.",
}
for name, desc in BUILTIN_SKILLS.items():
    t = len(desc) // 4
    total += t
    rows.append((f"builtin:{name}", t))

for f in glob.glob(os.path.join(cwd, ".claude", "commands", "**", "*.md"), recursive=True):
    name = os.path.relpath(f, os.path.join(cwd, ".claude", "commands")).replace(".md", "")
    desc = get_description(f)
    t = len(desc) // 4
    total += t
    rows.append((f"cmd:{name}", t))

print()
rows.sort(key=lambda x: -x[1])
for label, t in rows:
    print(f"  {label:<45} {t:>4} tokens")
print(f"\n  TOTAL skills                              {total:>5} tokens")
EOF
```

## Step 2: Count deferred MCP tools from context

Read the `<available-deferred-tools>` block in the conversation. Group `mcp__*` entries by server prefix (between first and second `__`). All listed = 0 tokens.

## Step 3: Output the report and present an interactive cleanup menu

Print the report, then build a numbered action list based on actual findings (only include actions that apply). After printing, ask the user which numbers to run (or "all").

```
══════════════════════════════════════════════════════
  Context Cost  —  actual from Anthropic API
  Initial: XX,XXX tokens   Current: XX,XXX tokens
══════════════════════════════════════════════════════

  OPTIMIZABLE
  ────────────────────────────────────────────────
  Skills system-reminder        ~X,XXX tokens  <- #1
  memory/MEMORY.md               X,XXX tokens
  CLAUDE.md (project)            X,XXX tokens
  CLAUDE.md (user)                   X tokens
  MCP pre-loaded                     0 tokens

  FIXED (cannot reduce)
  ────────────────────────────────────────────────
  Claude Code system prompt    ~10,000 tokens
  Built-in tool definitions     ~8,000 tokens
  gitStatus / injections        ~1,000 tokens

  MCP DEFERRED (0 tokens each)
  ────────────────────────────────────────────────
  posthog  XX   supabase  XX   qm    XX
  figma    XX   pencil    XX
  Total: XXX tools, 0 tokens

══════════════════════════════════════════════════════
  TOP ACTIONS  (reply with numbers or "all")
  ────────────────────────────────────────────────
  1. Remove skills             (~XXX tokens total — list all user
                               skills found, grouped, largest first)
  2. Trim memory/MEMORY.md    (XXX tokens, watch 200-line cap)
  3. Trim CLAUDE.md (project) (XXX tokens every turn)
══════════════════════════════════════════════════════
```

Fill in the actual token counts from the script output. Then use the `AskUserQuestion` tool to present the choices:

```
Which actions do you want to run?
1. Remove skills (~XXX tokens/turn)
2. Trim CLAUDE.md (project) (XXX tokens)
3. Trim memory/MEMORY.md (XXX tokens)
all. Do all three
```

When the user responds, execute the chosen actions:
- **1 (skills)**: list ALL removable user skills from `~/.claude/skills/` sorted by token cost, use `AskUserQuestion` to ask which to remove, then `rm -rf` the chosen dirs
- **2 (CLAUDE.md)**: read project CLAUDE.md, trim without removing critical info, rewrite it
- **3 (memory)**: read MEMORY.md, remove redundant/stale content, rewrite it shorter
- **all**: do all three in order
- Confirm what was done and tokens saved
