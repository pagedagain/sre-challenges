# Contributing

This repository is public. Commits, PRs, and review comments are part of the product. Write like an SRE posting a runbook, not like a chatbot.

Pages are generated from the private IncidentForge exporter, then edited here only when the public page needs a catalogue-specific change (MkDocs, CTA, site chrome). Do not paste sandbox Dockerfiles, `setup.sh`, `validator.sh`, or `solution.sh`.

## Voice

- Plain English. Short sentences. Name the command, path, or failure.
- No emoji in copy, commits, PR titles, or review comments.
- No cheerleading or filler: "excited to", "happy to help", "let's dive in", "hope this helps", "great question".
- No brochure words: "robust", "seamless", "leverage", "cutting-edge", "delve", "unlock", "empower", "landscape", "game-changer".
- No padding: "It's important to note", "In conclusion", "This PR introduces a comprehensive solution".
- Do not add "Made with" banners, Co-authored-by AI lines, or Key takeaways boxes.

Match the tone of `docs/linux/disk-full.md`: scenario, facts, commands, then the lesson.

```markdown
Bad: 🚀 Supercharge your SRE journey with this seamless disk-full adventure!
Good: The log volume is full. Use du to find the bloat under /var/log. Do not delete important-service.log.
```

## Code and docs

- Small diffs. One concern per PR when you can.
- Prefer the existing MkDocs Material patterns. Do not add themes, plugins, or CSS frameworks "while you are here".
- CSS stays in `docs/stylesheets/extra.css`. Keep selectors few and named for purpose.
- Challenge pages: keep the Interactive Sandbox CTA as a `.challenge-cta` block and a primary button. Do not replace it with screenshots or extra badges.

## Git

Commit subject: sentence case, imperative, no emoji.

```
Bad: ✨ Add amazing CTA!!
Good: Highlight the live sandbox CTA on disk-full.
```

PR title follows the same rule. PR body: what changed, how to check it, nothing else required. Use `.github/pull_request_template.md`.

## Agents

Read this file before editing. If a draft sounds like generated marketing copy, rewrite it before you open a PR.
