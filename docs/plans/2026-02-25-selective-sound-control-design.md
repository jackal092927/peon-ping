# Selective Sound Control: Documentation & Discoverability Improvements

**Date:** 2026-02-25
**Status:** Approved
**Type:** Documentation + Minor CLI Enhancement

## Problem Statement

Users want to keep peon-ping voice reminders (audio feedback) while disabling desktop notification popups that can be distracting. The feature already exists via `desktop_notifications: false` config, but lacks discoverability and clear documentation.

**User request:** "Keep voice reminder, but disable sound notifications"
**Reality:** Feature works, but users don't know it exists or how to use it.

## Current Implementation Analysis

### How It Works Today

The architecture already separates audio playback from notification popups:

1. **Python block** (line 2230 in `peon.sh`) reads `desktop_notifications` from config
2. **Python outputs** (line 2771) `DESKTOP_NOTIF=true` or `DESKTOP_NOTIF=false`
3. **Shell eval** (line 2208) sets bash variable `$DESKTOP_NOTIF`
4. **Notification logic** (lines 2919, 2965) checks:
   ```bash
   if [ -n "$NOTIFY" ] && [ "$PAUSED" != "true" ] && [ "${DESKTOP_NOTIF:-true}" = "true" ]; then
     send_notification ...
   fi
   ```
5. **Sound logic** (line 2915) plays independently:
   ```bash
   play_sound "$SOUND_FILE" "$VOLUME"
   ```

### Behavior Verification

When `desktop_notifications: false`:
- ✅ Sounds play (voice reminders continue)
- ❌ Desktop notifications suppressed (no popup banners)
- ✅ Mobile notifications still work (separate `MOBILE_NOTIF` flag)

**Conclusion:** Implementation is correct. Problem is discoverability.

## Proposed Solution

Improve documentation and CLI feedback to make the feature discoverable. No code changes to core functionality needed.

### Three Independent Toggle System

peon-ping has three independent controls:

| Config Key | Controls | Affects Sounds | Affects Desktop Popups | Affects Mobile Push |
|------------|----------|----------------|------------------------|---------------------|
| `enabled` | Master audio switch | ✅ Yes | ❌ No | ❌ No |
| `desktop_notifications` | Desktop popup banners | ❌ No | ✅ Yes | ❌ No |
| `mobile_notify.enabled` | Phone push notifications | ❌ No | ❌ No | ✅ Yes |

Users can mix and match these to achieve their desired behavior.

## Implementation Plan

### 1. CLI Enhancements (`peon.sh`)

#### A. Improve help text for `peon notifications`

**Current:**
```bash
peon notifications on|off  # Toggle desktop notifications
```

**Proposed:**
```bash
peon notifications on|off  # Toggle desktop notification popups (sounds continue playing)
```

**Files to modify:**
- `peon.sh` (help text around line 1900)

#### B. Add verbose status output

**Current `peon status` output:**
```
peon-ping: enabled
peon-ping: volume 0.5
peon-ping: pack peon
```

**Proposed with `--verbose` flag:**
```
peon-ping: enabled
peon-ping: volume 0.5
peon-ping: pack peon
peon-ping: desktop notifications off (sounds still play)
peon-ping: mobile notifications off
```

**Implementation:**
- Add `--verbose` flag to `peon status` command
- Read `desktop_notifications` and `mobile_notify.enabled` from config
- Display human-readable status with clarifying text

**Files to modify:**
- `peon.sh` (status command section around line 800)

#### C. Optional: Add `peon popups` alias

**Rationale:** "notifications" is ambiguous (could mean sounds). "popups" is clearer.

**Implementation:**
```bash
peon popups on|off  # Alias for 'peon notifications on|off'
```

**Files to modify:**
- `peon.sh` (command routing section)

### 2. README.md Updates

#### A. Add "Common Use Cases" section

Insert after Configuration section (~line 270):

```markdown
## Common Use Cases

### Sounds without popups

Want voice feedback but no visual distractions?

```bash
peon notifications off
```

This keeps all sound categories playing while suppressing desktop notification banners. Mobile notifications (if configured) continue working.

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

#### B. Enhance Configuration section with clarity table

Add the independent controls table (shown above in "Three Independent Toggle System") to the Configuration section.

**Files to modify:**
- `README.md` (~line 270)

### 3. Skill Documentation Updates

#### A. Update `/peon-ping-toggle` skill

**Current (implied):**
> Toggles peon-ping sounds on/off

**Proposed (explicit):**
> Toggles peon-ping sounds AND notifications on/off. For notification-only control, use /peon-ping-config to set desktop_notifications: false

**Files to modify:**
- `skills/peon-ping-toggle/SKILL.md`

#### B. Update `/peon-ping-config` skill

Add example use case:

```markdown
## Example: Disable notification popups but keep sounds

User request: "Disable desktop notifications"

Action:
→ Set `desktop_notifications: false`

Result:
→ Sounds continue playing
→ Desktop popups stop
→ Mobile notifications unaffected
```

**Files to modify:**
- `skills/peon-ping-config/SKILL.md`

### 4. Localization Updates

#### Update Chinese README (`README_zh.md`)

Apply equivalent changes in Chinese translation:
- Common Use Cases section
- Independent controls table
- Configuration clarifications

**Files to modify:**
- `README_zh.md`

### 5. LLM Context Updates

Update `docs/public/llms.txt` to include the "sounds without popups" use case so AI assistants recommend the correct solution.

**Files to modify:**
- `docs/public/llms.txt`

## Testing Strategy

### Manual Verification Tests

**Test Case 1: Verify desktop_notifications=false behavior**
```bash
# Setup
peon notifications off

# Trigger a hook event
echo '{"event":"Stop","sessionId":"test123"}' | ~/.claude/hooks/peon-ping/peon.sh

# Expected results:
# ✅ Sound plays (hear peon voice)
# ❌ No desktop notification popup
# ✅ Mobile notification still sent (if configured)
```

**Test Case 2: CLI status output**
```bash
peon status
# Should show current notification state clearly

peon status --verbose
# Should show detailed breakdown of all toggles
```

**Test Case 3: Help text clarity**
```bash
peon help
peon notifications --help
# Should clearly explain sound vs popup distinction
```

### Documentation Review

- Read through updated README sections to ensure clarity
- Verify Chinese translation matches English meaning
- Check that examples are copy-pasteable and work
- Test skill invocations in Claude Code

### Existing Test Coverage

No new BATS tests needed. Existing tests in `tests/peon.bats` already cover:
- Sound playback logic
- Notification gating via `DESKTOP_NOTIF` variable
- Config loading and parsing

The behavior already works correctly; we're only improving documentation and CLI feedback.

### Edge Cases to Verify

- **Config migration:** Users upgrading from old versions still work
- **Dual control methods:** Both `peon notifications off` command and manual `config.json` edit achieve same result
- **Skill accuracy:** Skills properly reference the updated documentation

## Files Changed

### Core Implementation (Minor)
- `peon.sh` — Add `--verbose` flag to status, improve help text, optional `popups` alias

### Documentation (Major)
- `README.md` — Add Common Use Cases, independent controls table
- `README_zh.md` — Chinese translation of new sections
- `docs/public/llms.txt` — Add use case context
- `skills/peon-ping-toggle/SKILL.md` — Clarify what gets toggled
- `skills/peon-ping-config/SKILL.md` — Add notification example

## Success Criteria

1. ✅ Users can easily discover how to disable popups while keeping sounds
2. ✅ CLI provides clear feedback about notification state
3. ✅ Documentation clearly explains the three independent toggles
4. ✅ Skills guide users to the correct solution
5. ✅ All language variants (EN, ZH) updated consistently

## Non-Goals

- No changes to core sound/notification logic (already works)
- No new config keys (use existing `desktop_notifications`)
- No breaking changes to existing behavior
- No new BATS tests (existing coverage sufficient)

## Implementation Effort

**Estimated time:** 1-2 hours

**Breakdown:**
- CLI enhancements: 20 minutes
- README updates (EN): 30 minutes
- README updates (ZH): 20 minutes
- Skill documentation: 15 minutes
- Testing & verification: 15 minutes

## Future Enhancements (Out of Scope)

These are NOT part of this design but could be considered later:

1. Interactive setup wizard: `peon setup` that asks about notification preferences
2. Per-category notification control (e.g., desktop popup for errors only)
3. Notification preview command: `peon notifications test`

## References

- CESP v1.0 spec: https://github.com/PeonPing/openpeon
- Existing config: `config.json`
- Implementation: `peon.sh` lines 2230, 2771, 2919, 2965
