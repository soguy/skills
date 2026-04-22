# claude-plugins

Guy Soffer's Claude Code plugin marketplace — AI-assisted development workflow tools.

## Skills

### blueprint
Sets up a structured AI-assisted development workflow for any project. Installs agent roles, a three-track process, project-doctor deep review, and configures everything for your tech stack. Works on new and existing projects. Re-run anytime to update.

### project-doctor
Deep codebase health check: dead code removal, bug detection, security audit, sanity checks, and documentation sync. Works with any tech stack.

---

## Install via Claude Code plugin marketplace

Add this marketplace once:
```
/plugin marketplace add guy-soffer/claude-plugins
```

Then install individual skills:
```
/plugin install blueprint
/plugin install project-doctor
```

---

## Manual install (no plugin system needed)

Copy the skill file directly into your project:

**blueprint:**
```bash
mkdir -p .claude/skills/blueprint
curl -o .claude/skills/blueprint/SKILL.md \
  https://raw.githubusercontent.com/guy-soffer/claude-plugins/main/plugins/blueprint/.claude/skills/blueprint/SKILL.md
```

**project-doctor:**
```bash
mkdir -p .claude/skills/project-doctor
curl -o .claude/skills/project-doctor/SKILL.md \
  https://raw.githubusercontent.com/guy-soffer/claude-plugins/main/plugins/project-doctor/.claude/skills/project-doctor/SKILL.md
```

Then invoke in Claude Code:
```
/blueprint
/project-doctor
```

---

## Usage

### Set up a new or existing project
```
/blueprint
```
blueprint will detect your stack, ask about dependencies, and write the full workflow configuration.

### Run a deep code review
```
/project-doctor
```
project-doctor runs a structured pipeline: dead code removal, bug detection, security audit, sanity checks, and documentation sync.

---

## Adding skills to this marketplace

1. Create `plugins/<skill-name>/.claude/skills/<skill-name>/SKILL.md`
2. Add an entry to `.claude-plugin/marketplace.json`
3. Push to GitHub — existing users get the update automatically
