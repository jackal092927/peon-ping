# Selective Sound Control: Documentation & CLI Improvements Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Improve discoverability of the existing `desktop_notifications: false` feature that lets users keep voice reminders while disabling desktop notification popups.

**Architecture:** Pure documentation and CLI enhancement. The core feature already works (sounds and notifications are independent). We're adding help text, verbose status output, README sections, and skill documentation updates.

**Tech Stack:** Bash (peon.sh CLI), Markdown (README, skills), no build tools

---

## Task 1: Add verbose status output to CLI

**Files:**
- Modify: `peon.sh` (status command section around line 800-850)

**Step 1: Locate the status command section**

Search for the `status)` case in peon.sh:

```bash
grep -n "status)" peon.sh | head -3
```

Expected: Find the status command handler around line 800-850

**Step 2: Read current status implementation**

Read the status command section to understand current output format:

```bash
# Look for lines like:
# echo "peon-ping: enabled"
# echo "peon-ping: volume $vol"
```

**Step 3: Add --verbose flag support**

Modify the status command to support `--verbose` flag:

```bash
status)
    # Existing status output
    eval "$(_py_status)"

    # Add verbose flag support
    if [ "${1:-}" = "--verbose" ]; then
      # Read desktop_notifications and mobile_notify from config
      _verbose_out="$(python3 -c "
import json
try:
    cfg = json.load(open('$CONFIG_PY'))
    dn = cfg.get('desktop_notifications', True)
    mn = cfg.get('mobile_notify', {})
    mobile_on = bool(mn and mn.get('service') and mn.get('enabled', True))

    dn_status = 'on' if dn else 'off (sounds still play)'
    mobile_status = 'on' if mobile_on else 'off'

    print('peon-ping: desktop notifications ' + dn_status)
    print('peon-ping: mobile notifications ' + mobile_status)
except Exception:
    pass
")"
      echo "$_verbose_out"
    fi
    exit 0 ;;
```

**Step 4: Test the verbose output**

Run the status command with and without verbose:

```bash
peon status
# Expected: Current output (enabled, volume, pack)

peon status --verbose
# Expected: Above + desktop notifications status + mobile notifications status
```

**Step 5: Commit**

```bash
git add peon.sh
git commit -m "feat: add --verbose flag to peon status command

Shows desktop and mobile notification states with clarifying text
that sounds continue when desktop notifications are disabled."
```

---

## Task 2: Improve help text for notifications command

**Files:**
- Modify: `peon.sh` (help text section around line 1900-1910)

**Step 1: Locate notifications help text**

Search for the help text:

```bash
grep -n "notifications on\|off" peon.sh | grep -v "^#"
```

Expected: Find help text around line 1905

**Step 2: Update help text**

Modify the help text to clarify that sounds continue:

```bash
# Before:
# notifications on|off  # Toggle desktop notifications

# After:
notifications on|off  # Toggle desktop notification popups (sounds continue playing)
```

**Step 3: Verify help output**

```bash
peon help | grep notifications
# Expected: See updated text with "(sounds continue playing)"
```

**Step 4: Commit**

```bash
git add peon.sh
git commit -m "docs: clarify notifications command help text

Makes it clear that toggling notifications only affects popups,
not audio playback."
```

---

## Task 3: Add popups alias command

**Files:**
- Modify: `peon.sh` (command routing section where notifications command is handled)

**Step 1: Locate notifications command handler**

```bash
grep -n "notifications)" peon.sh
```

Expected: Find the case statement around line 900-940

**Step 2: Add popups alias**

Add a new case that routes to the same handler:

```bash
# Find this section:
notifications)
  case "${1:-status}" in
    on)
      # ... existing code ...
    off)
      # ... existing code ...
  esac ;;

# Add right after:
popups)
  # Alias for 'notifications' command - same behavior
  case "${1:-status}" in
    on)
      # Call same Python code as notifications on
      python3 -c "
import json
try:
    cfg = json.load(open('$CONFIG_PY'))
except Exception:
    cfg = {}
cfg['desktop_notifications'] = True
json.dump(cfg, open('$CONFIG_PY', 'w'), indent=2)
print('peon-ping: desktop notifications on')
"
      sync_adapter_configs; exit 0 ;;
    off)
      python3 -c "
import json
try:
    cfg = json.load(open('$CONFIG_PY'))
except Exception:
    cfg = {}
cfg['desktop_notifications'] = False
json.dump(cfg, open('$CONFIG_PY', 'w'), indent=2)
print('peon-ping: desktop notifications off')
"
      sync_adapter_configs; exit 0 ;;
    *)
      echo "Usage: peon popups on|off" >&2
      exit 1 ;;
  esac ;;
```

**Step 3: Add to help text**

Add the popups alias to the help output:

```bash
# In the help section, add:
popups on|off         # Alias for 'notifications' - toggle desktop notification popups
```

**Step 4: Test the alias**

```bash
peon popups off
# Expected: "peon-ping: desktop notifications off"

peon status --verbose
# Expected: Shows "desktop notifications off (sounds still play)"

peon popups on
# Expected: "peon-ping: desktop notifications on"
```

**Step 5: Commit**

```bash
git add peon.sh
git commit -m "feat: add 'peon popups' alias for notifications command

Provides clearer alternative to 'peon notifications' since popups
is more specific than the ambiguous 'notifications' term."
```

---

## Task 4: Update README.md with Common Use Cases section

**Files:**
- Modify: `README.md` (insert after Configuration section around line 270)

**Step 1: Find insertion point**

```bash
grep -n "^## Configuration" README.md
```

Expected: Find the Configuration section heading

**Step 2: Add Common Use Cases section**

After the Configuration section and before the next ## heading, insert:

```markdown
## Common Use Cases

### Sounds without popups

Want voice feedback but no visual distractions?

```bash
peon notifications off
```

This keeps all sound categories playing while suppressing desktop notification banners. Mobile notifications (if configured) continue working.

You can also use the alias:

```bash
peon popups off
```

### Silent mode with notifications only

Want visual alerts but no audio?

```bash
peon pause  # or set "enabled": false in config
```

With `desktop_notifications: true`, you'll get popups but no sounds.

### Complete silence

Disable everything:

```bash
peon pause
peon notifications off
peon mobile off
```
```

**Step 3: Verify markdown formatting**

```bash
# Preview the section (if you have a markdown viewer)
# Or just verify syntax manually
grep -A20 "^## Common Use Cases" README.md
```

**Step 4: Commit**

```bash
git add README.md
git commit -m "docs: add Common Use Cases section to README

Highlights the 'sounds without popups' use case and clarifies
the three independent toggle controls (enabled, desktop_notifications,
mobile_notify)."
```

---

## Task 5: Add Independent Controls table to README

**Files:**
- Modify: `README.md` (Configuration section around line 240)

**Step 1: Locate the Configuration section details**

Find where config keys are documented:

```bash
grep -n "desktop_notifications" README.md | head -3
```

**Step 2: Add table before the config key descriptions**

Insert this table at the start of the Configuration section, right after the JSON example:

```markdown
### Independent Controls

peon-ping has three independent controls that can be mixed and matched:

| Config Key | Controls | Affects Sounds | Affects Desktop Popups | Affects Mobile Push |
|------------|----------|----------------|------------------------|---------------------|
| `enabled` | Master audio switch | ✅ Yes | ❌ No | ❌ No |
| `desktop_notifications` | Desktop popup banners | ❌ No | ✅ Yes | ❌ No |
| `mobile_notify.enabled` | Phone push notifications | ❌ No | ❌ No | ✅ Yes |

This means you can:
- Keep sounds but disable desktop popups: `peon notifications off`
- Keep desktop popups but disable sounds: `peon pause`
- Enable mobile push without desktop popups: set `desktop_notifications: false` and `mobile_notify.enabled: true`
```

**Step 3: Update desktop_notifications description**

Find the existing `desktop_notifications` description and enhance it:

```markdown
- **desktop_notifications**: `true`/`false` — toggle desktop notification popups independently from sounds (default: `true`). When disabled, sounds continue playing but visual popups are suppressed. Mobile notifications are unaffected.
```

**Step 4: Verify table renders correctly**

```bash
# Check the markdown syntax
grep -A10 "^### Independent Controls" README.md
```

**Step 5: Commit**

```bash
git add README.md
git commit -m "docs: add Independent Controls table to README

Clarifies that enabled, desktop_notifications, and mobile_notify
are three separate toggles that can be combined for different
notification strategies."
```

---

## Task 6: Update Chinese README (README_zh.md)

**Files:**
- Modify: `README_zh.md`

**Step 1: Find corresponding sections in Chinese README**

```bash
grep -n "## Configuration\|## 配置" README_zh.md
```

**Step 2: Add Chinese translation of Common Use Cases**

Insert after the Configuration section:

```markdown
## 常见用例

### 保留声音但禁用弹窗

想要语音反馈但不想要视觉干扰？

```bash
peon notifications off
```

这会保持所有声音类别的播放，同时禁用桌面通知横幅。手机通知（如果已配置）继续工作。

您也可以使用别名：

```bash
peon popups off
```

### 静音模式但保留通知

想要视觉提醒但不要音频？

```bash
peon pause  # 或在配置中设置 "enabled": false
```

当 `desktop_notifications: true` 时，您将收到弹窗但没有声音。

### 完全静音

禁用所有功能：

```bash
peon pause
peon notifications off
peon mobile off
```
```

**Step 3: Add Chinese translation of Independent Controls table**

```markdown
### 独立控制

peon-ping 有三个独立的控制开关，可以混合使用：

| 配置键 | 控制项 | 影响声音 | 影响桌面弹窗 | 影响手机推送 |
|--------|--------|----------|--------------|--------------|
| `enabled` | 主音频开关 | ✅ 是 | ❌ 否 | ❌ 否 |
| `desktop_notifications` | 桌面弹窗横幅 | ❌ 否 | ✅ 是 | ❌ 否 |
| `mobile_notify.enabled` | 手机推送通知 | ❌ 否 | ❌ 否 | ✅ 是 |

这意味着您可以：
- 保留声音但禁用桌面弹窗：`peon notifications off`
- 保留桌面弹窗但禁用声音：`peon pause`
- 启用手机推送但不显示桌面弹窗：设置 `desktop_notifications: false` 和 `mobile_notify.enabled: true`
```

**Step 4: Update desktop_notifications description in Chinese**

Find and enhance the Chinese description of `desktop_notifications`.

**Step 5: Commit**

```bash
git add README_zh.md
git commit -m "docs: add Chinese translations for Common Use Cases and Independent Controls

Maintains documentation parity between English and Chinese versions."
```

---

## Task 7: Update /peon-ping-toggle skill documentation

**Files:**
- Modify: `skills/peon-ping-toggle/SKILL.md`

**Step 1: Read current skill content**

```bash
cat skills/peon-ping-toggle/SKILL.md
```

**Step 2: Add clarification about what gets toggled**

Update the description or add a note section:

```markdown
## What This Toggles

This command toggles the **master audio switch** (`enabled` config). When disabled:
- ❌ Sounds stop playing
- ❌ Desktop notifications also stop (they require sounds to be enabled)
- ❌ Mobile notifications also stop

**For notification-only control**, use `/peon-ping-config` to set `desktop_notifications: false`. This keeps sounds playing while suppressing desktop popups.

## Examples

"Mute peon-ping completely" → Sets `enabled: false`
"Just disable the popups but keep sounds" → Sets `desktop_notifications: false` (use `/peon-ping-config` instead)
```

**Step 3: Commit**

```bash
git add skills/peon-ping-toggle/SKILL.md
git commit -m "docs: clarify peon-ping-toggle behavior

Explains that toggle affects master switch, and directs users
to peon-ping-config for notification-only control."
```

---

## Task 8: Update /peon-ping-config skill documentation

**Files:**
- Modify: `skills/peon-ping-config/SKILL.md`

**Step 1: Read current skill content**

```bash
cat skills/peon-ping-config/SKILL.md
```

**Step 2: Add example for notification control**

Add to the examples section (or create one if it doesn't exist):

```markdown
## Common Configuration Examples

### Disable desktop notification popups but keep sounds

**User request:** "Disable desktop notifications"

**Action:**
Set `desktop_notifications: false` in config

**Result:**
- ✅ Sounds continue playing (voice reminders)
- ❌ Desktop notification popups suppressed
- ✅ Mobile notifications unaffected (separate toggle)

**Alternative CLI command:**
```bash
peon notifications off
# or
peon popups off
```

### Adjust volume

**User request:** "Set volume to 30%"

**Action:**
Set `volume: 0.3` in config

### Enable round-robin pack rotation

**User request:** "Enable round-robin pack rotation with peon and glados"

**Action:**
Set:
```json
{
  "pack_rotation": ["peon", "glados"],
  "pack_rotation_mode": "round-robin"
}
```
```

**Step 3: Commit**

```bash
git add skills/peon-ping-config/SKILL.md
git commit -m "docs: add notification control examples to peon-ping-config skill

Shows how to disable desktop popups while keeping sounds, and
clarifies the independent toggle system."
```

---

## Task 9: Update docs/public/llms.txt

**Files:**
- Modify: `docs/public/llms.txt`

**Step 1: Read current llms.txt structure**

```bash
head -50 docs/public/llms.txt
```

**Step 2: Add Common Use Cases section**

Find an appropriate location (likely near configuration or features) and add:

```
## Common Use Cases

Sounds without popups:
- User wants voice feedback but no visual distractions
- Solution: peon notifications off
- Result: Sounds play, desktop popups suppressed, mobile notifications continue
- Config: desktop_notifications: false

Silent mode with notifications only:
- User wants visual alerts but no audio
- Solution: peon pause (or enabled: false in config)
- Result: Desktop popups show, no sounds play

Independent Controls:
- enabled: master audio switch (affects sounds only)
- desktop_notifications: desktop popup banners (affects popups only)
- mobile_notify.enabled: phone push notifications (affects mobile only)
```

**Step 3: Commit**

```bash
git add docs/public/llms.txt
git commit -m "docs: add notification control patterns to llms.txt

Helps AI assistants recommend correct solution for sounds-without-popups
use case."
```

---

## Task 10: Manual verification testing

**Files:**
- None (testing only)

**Step 1: Test desktop_notifications=false behavior**

```bash
# Setup
peon notifications off

# Verify config
grep desktop_notifications ~/.claude/hooks/peon-ping/config.json
# Expected: "desktop_notifications": false

# Trigger a hook event (Stop event)
echo '{"event":"Stop","sessionId":"test-$(date +%s)"}' | ~/.claude/hooks/peon-ping/peon.sh

# Manual verification:
# ✅ Did you hear a sound?
# ❌ Did you see a desktop notification popup?
```

**Step 2: Test CLI commands**

```bash
# Test status
peon status
# Expected: Shows enabled, volume, pack

peon status --verbose
# Expected: Above + "desktop notifications off (sounds still play)"

# Test notifications command
peon notifications on
peon status --verbose
# Expected: "desktop notifications on"

peon notifications off
peon status --verbose
# Expected: "desktop notifications off (sounds still play)"

# Test popups alias
peon popups on
peon status --verbose
# Expected: "desktop notifications on"

peon popups off
peon status --verbose
# Expected: "desktop notifications off (sounds still play)"
```

**Step 3: Test help output**

```bash
peon help | grep -A1 "notifications"
# Expected: See "(sounds continue playing)" in help text

peon help | grep "popups"
# Expected: See popups alias listed
```

**Step 4: Verify documentation**

```bash
# Check README has new sections
grep "## Common Use Cases" README.md
grep "### Independent Controls" README.md

# Check Chinese README
grep "## 常见用例" README_zh.md
grep "### 独立控制" README_zh.md

# Check skills
grep "notification-only control" skills/peon-ping-toggle/SKILL.md
grep "desktop_notifications: false" skills/peon-ping-config/SKILL.md
```

**Step 5: Document test results**

Create a test report comment or note:

```
Manual testing completed:
✅ desktop_notifications=false keeps sounds, suppresses popups
✅ peon status --verbose shows notification state
✅ peon notifications on/off works correctly
✅ peon popups on/off alias works correctly
✅ Help text shows clarifying text
✅ README sections added (EN and ZH)
✅ Skills updated with examples
✅ llms.txt updated
```

---

## Task 11: Final commit and summary

**Files:**
- None (wrap-up)

**Step 1: Review all commits**

```bash
git log --oneline -11
# Expected: See all 10 commits from this plan
```

**Step 2: Verify working tree is clean**

```bash
git status
# Expected: "nothing to commit, working tree clean"
```

**Step 3: Create summary of changes**

Document what was changed:

```
## Summary

Enhanced discoverability of desktop_notifications feature:

CLI Enhancements:
- Added peon status --verbose flag
- Improved notifications command help text
- Added peon popups alias

Documentation:
- Added Common Use Cases section to README (EN + ZH)
- Added Independent Controls table to README (EN + ZH)
- Updated skill documentation (toggle + config)
- Updated llms.txt with use case patterns

Testing:
- Manual verification confirms feature works correctly
- Sounds play independently from desktop notifications
- CLI provides clear feedback

No code changes to core functionality - feature already worked correctly.
```

**Step 4: Optional: Update CHANGELOG.md**

If this warrants a version bump, add to CHANGELOG.md:

```markdown
## [Unreleased]

### Added
- `peon status --verbose` flag showing desktop and mobile notification states
- `peon popups` alias for `peon notifications` command
- Common Use Cases section in README (sounds without popups, etc.)
- Independent Controls table clarifying the three toggle system

### Changed
- Improved help text for notifications command to clarify sounds continue
- Enhanced skill documentation with notification control examples
```

**Step 5: Done**

Implementation complete! All documentation and CLI improvements are in place.
