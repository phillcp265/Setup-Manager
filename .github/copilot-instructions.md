# Setup Manager - AI Coding Instructions

## Project Overview

Setup Manager is a macOS enrollment tool for IT service providers using Jamf Pro or Jamf School. It runs during device setup before user creation to install software, execute policies, and apply configurations without interfering with MDM or FileVault token flow.

**Key constraint**: Must not run with Auto Advance enabled (only at enrollment or loginwindow); requires at least one Setup Assistant screen enabled before user creation.

## Configuration-Driven Architecture

Setup Manager is entirely **configuration-driven**. The binary reads a plist/mobileconfig file with key `com.jamf.setupmanager` containing all behavior:

- **UI elements** (title, message, icon, banner, colors) sourced from plist
- **Enrollment actions** (sequence of tasks) defined as an array of dicts in `enrollmentActions` key
- **Runtime behavior** (when to launch, debug mode, network checks) controlled via top-level keys

See [ConfigurationProfile.md](../ConfigurationProfile.md) for the complete schema (1294 lines). This is the single source of truth for feature documentation.

## Action Types (Core Workflow Pattern)

Enrollment actions are polymorphic dicts with a `type` field determining behavior. Common types:

- `policy` - Triggers Jamf Pro custom policy by name
- `shell` - Executes shell command; respects `requiresRoot` flag
- `watchPath` - Waits for file/app appearance (watches `/Applications` for Mac App Store installs)
- `wait` - Pauses for specified seconds
- `installomator` - Uses Installomator script to install apps
- `recon` - Runs Jamf Pro inventory update
- `waitForUserEntry` - Pauses until user provides input

Each action includes: `label` (localized), `icon` (SF Symbols or URL), optional `timeout`. All keys are **case-sensitive**.

## Key Design Patterns

1. **Localization Everywhere**: Labels, titles, messages, and image sources all support language codes (e.g., `en`, `de`, `fr`, `sv`). See sample at [Examples/sample-com.jamf.setupmanager.plist](../Examples/sample-com.jamf.setupmanager.plist).

2. **Icon Sources**: Multiple formats supported:
   - SF Symbols: `symbol:clock`, `symbol:network`
   - System images: `name:NSComputer`, `name:AppIcon`
   - URLs: HTTPS image paths
   - Specific variant support in dark mode

3. **Substitutions in Text**: Messages support `%model%`, `%serialNumber%`, etc. for dynamic device info.

4. **Markdown in Messages**: Message field supports `**bold**`, italics in UI text (truncated to one line during action progress).

5. **Debug Mode**: When `DEBUG: true`, no disk changes occur; reads local prefs; accepts test configuration. Useful for screenshots and testing without enrollment.

6. **User Entry System**: Optional `userEntry` dict can prompt for fields (department, computer name, asset tag, userID) with validation. Data passed to `webhooks` on completion.

## Critical Integration Points

1. **Jamf Pro Policies**: Setup Manager triggers policies via custom trigger name; policies can use `enrollmentComplete` trigger (but this may delay Setup Manager).

2. **Webhooks**: On start/finish, sends JSON POST with device info, duration, user entry data to configurable URLs. Used for audit, logging, or downstream automation.

3. **Configuration Profile Delivery**: Profile delivered by Jamf Pro at enrollment; Setup Manager auto-detects and reads it. The profile must be scoped correctly; Prestage deployment workflow is primary use case.

4. **Recon Workflow**: `recon` actions or `waitForUserEntry` trigger inventory update back to JSS; older `jamf` binary API changed in recent versions (see ChangeLog 1.4.1).

5. **Network Quality Checks**: Can be suppressed with `networkQualityCheck: false` for poor connectivity environments.

## Documentation Structure

- **[ConfigurationProfile.md](../ConfigurationProfile.md)** - Complete plist schema reference (THE authoritative source)
- **[Docs/JamfPro-QuickStart.md](../Docs/JamfPro-QuickStart.md)** - Step-by-step setup guide (Prestage workflow, scoping, testing)
- **[Docs/Extras.md](../Docs/Extras.md)** - Keyboard shortcuts (cmd-L logs, cmd-B battery, space barcode), debug tips, iMazing support
- **[Docs/Webhooks.md](../Docs/Webhooks.md)** - Webhook schema (started/finished events with device metadata)
- **Examples/** - Sample plist configs (Jamf Pro, Jamf School, two-phase user entry workflow)

## When Analyzing/Updating Documentation

1. **ConfigurationProfile.md is authoritative** - always reference it for feature details, not README
2. **ChangeLog.md tracks behavioral changes** - version 1.4 added banner/colors, 1.4.1 fixed recon workflow
3. **Known issues are documented in README** - e.g., Auto Advance incompatibility, Jamf.app check disabled
4. **Localization is pervasive** - check ConfigurationProfile for lang-code support in any text field

## Developer Workflows (from docs, not visible in repo)

1. **Testing**: Set `DEBUG: true` in profile, optionally set `simulateMDM` to "Jamf Pro" or "Jamf School" to test on unenrolled Macs
2. **Logging**: Cmd-L opens live log; logs written to `/Library/Logs/Setup Manager.log` (ISO8601 timestamp, log level, category, message); also in macOS unified log under `com.jamf.setupmanager` subsystem
3. **Debugging Prestage**: Wipe test Mac, enroll to Prestage, approve MDM, Setup Manager auto-launches if profile/package in Prestage
4. **Profile Export**: Jamf Pro custom schema available for JSON-based configuration (but does not support `wait` actions or localization)

## Edge Cases & Workarounds

- **Profile Not Detected**: Ensure profile is scoped correctly and delivered *before* enrollment completes
- **Actions Not Running**: Verify Jamf binary exists, policy triggers match exactly, shell commands use correct paths
- **Policy Delays**: Disable or unscope `enrollmentComplete`-triggered policies to avoid blocking Setup Manager
- **Auto Advance Conflict**: Use `runAt: loginwindow` instead of enrollment when Auto Advance is enabled
- **Localization Issues**: Use language codes from ISO 639-1 (e.g., `nb` for Norwegian Bokmål); missing code defaults to English

## Files to Examine for Pattern Depth

- [Examples/sample-com.jamf.setupmanager.plist](../Examples/sample-com.jamf.setupmanager.plist) - Real-world multi-action config with localized labels
- [Examples/sample-waitForUserEntry.plist](../Examples/sample-waitForUserEntry.plist) - Two-phase workflow pattern
- [Docs/JamfPro-LoginWindow.md](../Docs/JamfPro-LoginWindow.md) - Handsfree Auto Advance workflow
- [Docs/Network.md](../Docs/Network.md) - Network quality check architecture
