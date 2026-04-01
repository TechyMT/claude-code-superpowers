# Installing Claude Code Superpowers for Codex

## Installation

1. **Clone the repository:**
   ```bash
   git clone https://github.com/TechyMT/claude-code-superpowers.git ~/.codex/claude-code-superpowers
   ```

2. **Create the skills symlink:**
   ```bash
   mkdir -p ~/.agents/skills
   ln -s ~/.codex/claude-code-superpowers/skills ~/.agents/skills/claude-code-superpowers
   ```

   **Windows (PowerShell):**
   ```powershell
   New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.agents\skills"
   cmd /c mklink /J "$env:USERPROFILE\.agents\skills\claude-code-superpowers" "$env:USERPROFILE\.codex\claude-code-superpowers\skills"
   ```

3. **Restart Codex** to discover the skills.

## Verify

```bash
ls -la ~/.agents/skills/claude-code-superpowers
```

## Updating

```bash
cd ~/.codex/claude-code-superpowers && git pull
```

## Uninstalling

```bash
rm ~/.agents/skills/claude-code-superpowers
rm -rf ~/.codex/claude-code-superpowers
```
