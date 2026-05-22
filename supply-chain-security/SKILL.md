---
name: supply-chain-security
description: Supply Chain Security Skill - Enforces minimum 7-day release age for package installations for python UV, Node NPM, pnpm, bun, and .net dependencies.
---

# Supply Chain Security Skill

## Overview

This document specifies **workspace-specific** supply chain security configurations that must be implemented in every workspace. All package installations must enforce a **minimum 7-day release age** to protect against recently published malicious packages.

## Workflow

1. **Provisioning Phase**: An agent or the user sets up the workspace with the configuration files specified in this document
2. **Usage Phase**: After provisioning, all agents must follow this skill file and respect the workspace security settings

**This document contains no global settings** - all configurations are workspace-specific and self-contained within each project.

> ⚠️ **Known anti-pattern**: Do NOT set `minimum-release-age=7d` in any `.npmrc` (local or global). This is not a valid pnpm setting and causes `ERR_PNPM_NO_MATURE_MATCHING_VERSION` errors even for packages that are hundreds of days old. The correct setting is `minimumReleaseAge: 10080` in `pnpm-workspace.yaml`.

The workspace supports multiple package managers:
- **JavaScript/TypeScript**: pnpm, npm, Bun
- **Python**: UV (replaces pip)
- **.NET**: NuGet (via Dependabot - limited support)

## Why This Matters

Supply chain attacks on npm, Python, and .NET packages are a critical threat. Attackers compromise packages by:
- Stealing maintainer credentials
- Publishing malicious updates to popular packages  
- Taking over abandoned packages

**Key insight**: Most malicious packages are detected and removed within 24-48 hours. A 7-day minimum age provides a "cooling off period" that protects against recently compromised packages.

Recent incidents:
- `axios` compromise (100M+ weekly downloads) pulled in malicious `plain-crypto-js@4.2.1`
- `debug`, `chalk` packages affected per CISA alerts
- Python `shai-hulud` incident (registry compromise)

## Workspace Configuration Files

### JavaScript/TypeScript

| Package Manager | Config File | Setting |
|----------------|-------------|---------|
| **pnpm** | `pnpm-workspace.yaml` | `minimumReleaseAge: 10080` (7 days in minutes) — requires pnpm v10.16+ |
| **npm** | `npm-install-safe.js` | Script with `--before` flag |
| **Bun** | `bunfig.toml` | `minimumReleaseAge = 604800` (7 days in seconds) |

### Python

| Package Manager | Config Method | Setting |
|----------------|---------------|---------|
| **UV** | `pyproject.toml` or env var | `exclude-newer = "7 days"` |
| **pip** | ⚠️ Not supported | Use UV instead |

### .NET

| Package Manager | Config Method | Setting |
|----------------|---------------|---------|
| **NuGet** | `.github/dependabot.yml` | `cooldown.default-days: 7` |
| **NuGet CLI** | ⚠️ Not supported | Feature request: NuGet/Home#14657 |

## What Agents MUST Do

When helping with package installation in **this workspace**:

1. **Use workspace-specific configurations** - Respect the local config files:
   ```bash
   # pnpm - respects pnpm-workspace.yaml in workspace root (requires pnpm v10.16+)
   pnpm install
   
   # npm - use workspace script
   npm run install:safe
   
   # Bun - respects bunfig.toml
   bun install
   
   # UV - respects pyproject.toml or env var
   uv pip install package
   ```

2. **When adding new package managers** - Create workspace-specific config files:
   - Add `minimumReleaseAge` to `pnpm-workspace.yaml` for pnpm (v10.16+)
   - Add `bunfig.toml` for Bun
   - Add `pyproject.toml` section for UV
   - Add `.github/dependabot.yml` for NuGet

3. **Update the workspace README** - After creating config files, update `README.md`:
   - Add the "Supply Chain Security" section at the top
   - Include the quick reference table
   - Document which package managers are configured
   - See "Updating Workspace README" section below for template

4. **When suggesting age changes** - Modify workspace config files directly (edit `pnpm-workspace.yaml`, `bunfig.toml`, or `pyproject.toml`):
   - pnpm: change `minimumReleaseAge` value (14 days = `20160` minutes)
   - Bun: change `minimumReleaseAge` value (14 days = `1209600` seconds)
   - UV: change `exclude-newer = "14 days"`

5. **Document bypass methods** - Always include how to bypass when suggesting strict security:
   ```bash
   # pnpm - add package to minimumReleaseAgeExclude in pnpm-workspace.yaml
   # OR temporarily set minimumReleaseAge: 0
   
   # npm
   npm install --before=$(date +%Y-%m-%d)
   
   # Bun
   bun add package --force
   
   # UV
   UV_EXCLUDE_NEWER="1 day" uv pip install package
   ```

## What Agents MUST NOT Do

1. ❌ Forget to check for existing workspace config files (`pnpm-workspace.yaml`, `bunfig.toml`, etc.)
2. ❌ Suggest `pip` for Python packages (UV is configured and should be used)
3. ❌ Recommend removing workspace security settings without explaining trade-offs
4. ❌ Create new projects without supply chain protection configuration
5. ❌ Skip updating the README.md after provisioning security configs
6. ❌ Use `minimum-release-age=7d` in any `.npmrc` — it is an invalid pnpm setting that causes cryptic install failures
7. ❌ Edit the pnpm global config file directly using its OS-specific path — use `pnpm config` CLI instead

## Package Manager Details

### pnpm (`pnpm-workspace.yaml`)

> ⚠️ **Requires pnpm v10.16+**. This setting has no effect on pnpm v9 or earlier.

```yaml
minimumReleaseAge: 10080  # 7 days in minutes (7 * 24 * 60)
minimumReleaseAgeExclude:
  - pnpm  # always exclude pnpm itself to avoid self-install errors
```

**Why `pnpm` must be excluded**: pnpm manages its own version via the `packageManager` field in `package.json`. If pnpm is not excluded, pnpm v10.16+ will block installing the pinned pnpm version even when it is hundreds of days old (due to how the age check interacts with self-installs).

**Usage:**
```bash
pnpm install                    # Uses workspace config
```

**Bypass (add to minimumReleaseAgeExclude in pnpm-workspace.yaml):**
```yaml
minimumReleaseAgeExclude:
  - pnpm
  - urgent-package
```

**Docs:** https://pnpm.io/settings#minimumreleaseage

#### ❌ Anti-patterns to avoid

- **Do NOT use `minimum-release-age=7d` in `.npmrc`** (local or global) — this is not a valid pnpm v10 setting and will cause `ERR_PNPM_NO_MATURE_MATCHING_VERSION` for packages that are clearly old enough.
- **Do NOT edit the pnpm global config file directly** — the path is OS-specific:
  - macOS: `~/Library/Preferences/pnpm/rc`
  - Linux: `~/.config/pnpm/rc`
  - Windows: `%LOCALAPPDATA%\pnpm\config\rc`
  
  Use `pnpm config set` / `pnpm config delete` instead (OS-agnostic):
  ```bash
  # Check for rogue global settings
  pnpm config list --global
  
  # Remove a bad global setting
  pnpm config delete minimum-release-age --global
  ```

---

### npm (`npm-install-safe.js`)

Since npm lacks native minimum age support, this workspace provides a wrapper script:

```bash
npm run install:safe            # Install packages ≥7 days old
```

**File:** `npm-install-safe.js` - Calculates date 7 days ago, uses `--before` flag

**Manual override:**
```bash
npm install --before=$(date +%Y-%m-%d)
```

---

### Bun (`bunfig.toml`)

```toml
[install]
minimumReleaseAge = 604800  # 7 days in seconds
minimumReleaseAgeExcludes = []  # Packages to bypass
```

**Usage:**
```bash
bun install                     # Uses workspace config
bun add package --force         # Bypass
```

**Docs:** https://bun.sh/docs/runtime/bunfig

---

### UV (Python)

**pyproject.toml:**
```toml
[tool.uv]
exclude-newer = "7 days"

# Per-package exceptions
exclude-newer-package = { package-name = false }
```

**Usage:**
```bash
uv pip install package          # Uses workspace config
UV_EXCLUDE_NEWER="1 day" uv pip install package  # Override
```

**Docs:** https://docs.astral.sh/uv/reference/settings/#exclude-newer

---

### NuGet (.NET)

**⚠️ Native CLI support pending** (Issue: https://github.com/NuGet/Home/issues/14657)

Current workspace approach via Dependabot:
```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "nuget"
    directory: "/"
    schedule:
      interval: "weekly"
    cooldown:
      default-days: 7
```

For immediate CLI needs, use exact version pinning in `.csproj`:
```xml
<PackageReference Include="PackageName" Version="1.2.3" />
```

## Emergency Override Procedures

If a package is too new for the workspace policy:

### Option 1: Command-line override (recommended)
```bash
# pnpm - add to minimumReleaseAgeExclude in pnpm-workspace.yaml
# minimumReleaseAgeExclude:
#   - urgent-package

# npm
npm install --before=$(date +%Y-%m-%d) package

# Bun
bun add package --force

# UV
UV_EXCLUDE_NEWER="1 day" uv pip install package
```

### Option 2: Temporarily modify workspace config
```bash
# Edit pnpm-workspace.yaml (set minimumReleaseAge: 0), bunfig.toml, or pyproject.toml
# ... install package ...
# Revert config changes
```

### Option 3: Add to exceptions list
```yaml
# pnpm - edit pnpm-workspace.yaml
minimumReleaseAgeExclude:
  - urgent-package
```

```bash
# Bun - edit bunfig.toml
minimumReleaseAgeExcludes = ["urgent-package"]

# UV - edit pyproject.toml
exclude-newer-package = { urgent-package = false }
```

## Creating New Projects

When creating a new project for this user, **always** include workspace-specific supply chain protection:

1. **For pnpm projects (v10.16+):**
   ```bash
   # Add to pnpm-workspace.yaml (or create if it doesn't exist)
   cat >> pnpm-workspace.yaml << 'EOF'
   
   minimumReleaseAge: 10080
   minimumReleaseAgeExclude:
     - pnpm
   EOF
   ```

   **For npm projects:**
   ```bash
   # Create npm-install-safe.js (copy from template)
   # Add to package.json scripts: "install:safe": "node npm-install-safe.js"
   ```

2. **For Bun projects:**
   ```bash
   # Create bunfig.toml
   cat > bunfig.toml << 'EOF'
   [install]
   minimumReleaseAge = 604800
   EOF
   ```

3. **For Python projects:**
   ```bash
   # Add to pyproject.toml
   cat >> pyproject.toml << 'EOF'
   [tool.uv]
   exclude-newer = "7 days"
   EOF
   ```

4. **For .NET projects:**
   ```bash
   # Create .github/dependabot.yml
   mkdir -p .github
   cat > .github/dependabot.yml << 'EOF'
   version: 2
   updates:
     - package-ecosystem: "nuget"
       directory: "/"
       schedule:
         interval: "weekly"
       cooldown:
         default-days: 7
   EOF
   ```

## Updating Workspace README

After provisioning a workspace (creating the config files above), agents **MUST** update or create the workspace `README.md` with the supply chain security notice.

### If README.md exists:

Insert the security notice at the top (after the title):

```markdown
# Supply Chain Security

This workspace enforces a **minimum 7-day release age** for all package installations to protect against supply chain attacks.

## Quick Reference

| Package Manager | Config File | Install Command | Emergency Override |
|----------------|-------------|-----------------|-------------------|
| **pnpm** | `pnpm-workspace.yaml` | `pnpm install` | Add pkg to `minimumReleaseAgeExclude` |
| **npm** | `npm-install-safe.js` | `npm run install:safe` | `npm install --before=$(date +%Y-%m-%d)` |
| **Bun** | `bunfig.toml` | `bun install` | `bun add <pkg> --force` |
| **UV** | `pyproject.toml` or env | `uv pip install` | `UV_EXCLUDE_NEWER="1 day" uv pip install` |
| **NuGet** | `.github/dependabot.yml` | N/A (see notes) | Pin exact versions in `.csproj` |

[Rest of original README content follows...]
```

### If README.md does not exist:

Create it with the full security template:
- Quick reference table
- Why 7 days explanation  
- Configuration details for each package manager used
- Emergency override procedures
- How to change the minimum age
- References

**Purpose**: The README serves as immediate documentation for anyone using the workspace. The skill file (SUPPLY_CHAIN_SKILL.md) is the comprehensive specification - the README is the user-facing summary.

## Verification Commands

Check workspace configuration:
```bash
# pnpm workspace config
grep minimumReleaseAge pnpm-workspace.yaml

# Check for rogue global pnpm settings (OS-agnostic)
pnpm config list --global

# npm
grep MIN_AGE npm-install-safe.js

# Bun
grep minimumReleaseAge bunfig.toml

# UV
grep exclude-newer pyproject.toml

# Check all workspace configs exist
ls pnpm-workspace.yaml bunfig.toml pyproject.toml .github/dependabot.yml 2>/dev/null
```

## Related Tools & References

- **Socket.dev**: Dependency security scanning
- **Package Managers Need to Cool Down**: https://nesbitt.io/2026/03/04/package-managers-need-to-cool-down.html

## Additional Notes

- The 7-day default balances security with access to legitimate updates
- Most security researchers recommend 7-30 days as the sweet spot
- All configurations in this document are workspace-specific only
- Always provide bypass methods for urgent security patches
- pip and NuGet CLI lack native support - use UV for Python, Dependabot for .NET
