# DevOps Daily Skills

Open-source [Agent Skills](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) for DevOps engineers, from [DevOps Daily](https://devops-daily.com).

Each skill packages the checks and workflows a good platform engineer carries in their head into something Claude can apply consistently. Drop one into Claude Code (or any agent that supports skills) and it triggers automatically when the work matches.

## Skills

| Skill | What it does |
| --- | --- |
| [`kubernetes-manifest-auditor`](skills/kubernetes-manifest-auditor) | Reviews Kubernetes YAML for security, reliability, and production-readiness issues and reports findings by severity with concrete fixes. |

More on the way. Ideas and contributions welcome.

## Using a skill with Claude Code

Skills live in a directory with a `SKILL.md` file. Claude reads the description and loads the skill when a request matches.

**Per project** (checked in for your team):

```bash
mkdir -p .claude/skills
cp -r skills/kubernetes-manifest-auditor .claude/skills/
```

**For your user** (available everywhere):

```bash
mkdir -p ~/.claude/skills
cp -r skills/kubernetes-manifest-auditor ~/.claude/skills/
```

Then just ask, for example: *"audit these Kubernetes manifests before I apply them."* Claude picks up the skill on its own. See the [Agent Skills docs](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) for the full mechanism, including plugins and other agents.

## What is a skill?

A skill is a folder with instructions (`SKILL.md`) and optional supporting files (reference docs, examples, scripts). The `SKILL.md` frontmatter has a `name` and a `description` that tells the agent when to use it. Supporting files are loaded only when needed, so the agent stays focused. These skills follow the format from Anthropic's [skill-creator](https://github.com/anthropics/skills).

## Contributing

Found a gap in a checklist, or have a DevOps skill worth sharing? Open an issue or a pull request. Keep skills focused, give every finding a fix, and stay defensive (these harden systems, they do not attack them).

## About

Built and maintained by [DevOps Daily](https://devops-daily.com), a blog and set of free tools for DevOps and platform engineers: Kubernetes, Docker, Terraform, CI/CD, Linux, cloud, and security. If a skill's checks interest you, the [Kubernetes](https://devops-daily.com/categories/kubernetes) and [Security](https://devops-daily.com/categories/security) sections go deeper.

## License

[MIT](LICENSE) © DevOps Daily
