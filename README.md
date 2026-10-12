# jevskill

An agent skill for thinking through where TypeSafe Jev belongs in a product or
codebase, and integrating it well.

Jev is a System One decision model: send a `state` and typed `questions`, get
typed `answers` with calibrated probabilities in about 100 ms. It does not
generate text. The skill is written as philosophy and method rather than a
fixed procedure, so an agent can adapt it to any repo. It covers:

- the idea underneath Jev and why typed decisions beat prompt-and-parse
- a thinking method for finding where it fits (product behavior, code
  consumers, primitive semantics, economics)
- principles for shaping judgments and integrating them
- the factual API contract: inputs, outputs, confidence, SDKs, limits

## Install

Copy or link the skill directory into any Agent Skills location:

```sh
# global (pi, Claude Code, and other agents that read ~/.agents/skills)
ln -s "$PWD/skills/jev-integration" "$HOME/.agents/skills/jev-integration"

# project-local
mkdir -p .agents/skills
ln -s "$(pwd)/skills/jev-integration" .agents/skills/jev-integration
```

Or use the skills CLI:

```sh
npx skills add Ami-Khokhar/jevskill --skill jev-integration
```

## Use

Ask the agent to scan a repo, or to use the skill directly:

```
Using the jev-integration skill, scan this repo and find where Jev fits.
```

See `skills/jev-integration/SKILL.md` for the full skill.

## License

MIT
