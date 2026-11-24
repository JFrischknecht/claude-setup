# claude-setup

Personal setup notes for working with [Claude Code](https://code.claude.com/docs/en/setup#install-claude-code) in the terminal.

## Once per machine

### Configure the status line

At the Claude Code prompt, run this command to configure a persistent status line showing the model name, effort level, and context size in thousands of tokens. It will be available in future Claude Code sessions:

```text
/statusline show model name, effort and context size in k
```

### Install my preferred Agent Engineering skills

Matt Pocock's skills are my preferred skill set for Agent Engineering.

Run these commands in your terminal, not at the Claude Code prompt:

```bash
claude plugin marketplace add mattpocock/skills
claude plugin install mattpocock-skills@claude-plugins-official
```

## For every new project

### Add `CLAUDE.md` and the coding standards

Copy these two files into the root of the project:

- `CLAUDE.md`: a template for the project description, repository layout, and any project conventions that override the standards.
- `CODING_STANDARDS.md`: the R coding standards, loaded automatically through the `@CODING_STANDARDS.md` reference in `CLAUDE.md`.

You don't have to fill in `CLAUDE.md` by hand. Start Claude Code in the project and ask it to fill in the template with you, for example:

```text
Fill in CLAUDE.md with me: look at the repo, then ask me what you can't find out yourself.
```

### Configure project workflows for Matt Pocock's skills

From the project, run this command at the Claude Code prompt. It can configure project workflows such as issue tracking and triage labels:

```text
/setup-matt-pocock-skill
```

## Learn more about Matt Pocock's skills

- [AI Hero: Skills](https://www.aihero.dev/skills)
- [Matt Pocock's skills repository](https://github.com/mattpocock/skills)

### See the skills in action

- [Video walkthrough](https://www.youtube.com/watch?v=jVBlkOF8sHk)

## Additional Agent Engineering resources

### Understand tokens, context windows, and agents

- [Keep context down](https://code.claude.com/docs/en/best-practices#keep-cost-and-context-down)
- [Reduce token usage](https://code.claude.com/docs/en/costs#reduce-token-usage)
- [Video: tokens, context windows, and agents](https://www.youtube.com/watch?v=-uW5-TaVXu4)

### Work in parallel with Git worktrees

- [Git worktrees tutorial](https://www.datacamp.com/tutorial/git-worktree-tutorial)
- [Video: Claude worktrees](https://www.youtube.com/watch?v=yv8VZpov8bk)