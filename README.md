# hermes-skills

Custom [Hermes Agent](https://hermes-agent.nousresearch.com) skills by [@izzyco](https://github.com/izzyco).

## Install

```bash
# Install a specific skill
hermes skills install izzyco/hermes-skills/desloppify

# Or add as a tap to browse all skills
hermes skills tap add izzyco/hermes-skills
hermes skills browse --source izzyco/hermes-skills
```

## Skills

### 🧹 [desloppify](./desloppify/SKILL.md)

Multi-language codebase health scanner and fix orchestrator. Combines mechanical detection (dead code, duplication, complexity) with LLM-based subjective review (naming, abstractions, error handling, module design) into a scored improvement loop.

- 29 languages supported
- Score > 98 = *"a codebase a seasoned engineer would call beautiful"*
- Scoring: 25% mechanical + 75% subjective LLM review
- Full fix orchestration with triage, planning, and parallel subagent review

```bash
pip install --upgrade "desloppify[full]"
desloppify scan --path .
desloppify next
```

## Contributing

Skills follow the [agentskills.io](https://agentskills.io/specification) open standard — each skill is a `SKILL.md` file with YAML frontmatter.
