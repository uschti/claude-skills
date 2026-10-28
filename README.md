# claude-skills

Claude Code skills I use across my projects, published as a public plugin marketplace named
`uschti`. The procedure lives here, the facts of a project (branches, verification commands, rule
files) live in that project's `CLAUDE.md`, so one skill serves every repository.

## Skills

Each skill has its own README: what it does, how to trigger it, what it requires and what for,
what a project must provide.

| Skill | Say (Italian or English) | Requires | Docs |
|---|---|---|---|
| `shipping-pull-requests` | "fai la PR", "do the PR", "review the PR", "ship it" | `gh` logged in to GitHub, write access to the repo, the [superpowers](https://github.com/obra/superpowers) plugin, a `## Shipping pull requests` section in the project's `CLAUDE.md` | [README](plugins/shipping-pull-requests/README.md#requirements) |

## Install

This repository is a Claude Code **plugin marketplace** (`.claude-plugin/marketplace.json`); each
skill ships as a plugin under `plugins/<name>/`. The repository is public, so no credential is
needed to add it.

### Per developer (every project on the machine sees it)

```bash
claude plugin marketplace add uschti/claude-skills
claude plugin install <skill>@uschti
```

`uschti/claude-skills` is the GitHub shorthand; the full form
`https://github.com/uschti/claude-skills.git` works too (keep the `.git` suffix, without it the
URL is read as a hosted `marketplace.json` file).

A new session then lists the skill as `/<skill>:<skill>`.

### Per project (declared in the repository, collaborators get it by trusting the folder)

Run the two commands above inside the project with `--scope project` and commit the
`.claude/settings.json` they write (`extraKnownMarketplaces` with the GitHub source,
`enabledPlugins` with the plugin). Whoever opens the project and trusts the folder gets the
marketplace registered; if Claude Code reports the plugin as "enabled in project settings but not
installed", `claude plugin install <skill>@uschti --scope project` once per machine settles it.

### Update

```bash
claude plugin marketplace update uschti
claude plugin update <skill>@uschti
```

then `/reload-plugins` in the session (or a new session). Plugin manifests carry no `version` on
purpose: the installed commit is the version, so every merged change is an update.

## Working on the skills

Edit in your clone, validate, push, then run the two update commands above. Do not register the
clone's folder as the marketplace: a marketplace name has one source per machine, and a project
that declares the GitHub source refuses a `uschti` that user settings know by a folder (`Cannot
add marketplace "uschti": its source doesn't match its extraKnownMarketplaces entry`).

Validate before pushing: `claude plugin validate .` and `claude plugin validate ./plugins/<skill>`
(the "no version" warning is intended).

### Adding a skill

One plugin folder `plugins/<name>/` with:

- `.claude-plugin/plugin.json` (`name`, `description`, `author`, no `version`);
- `skills/<name>/SKILL.md` whose frontmatter has `name` and `description` (the description is
  what makes Claude pick the skill: say when to use it, in the words a developer would say);
- `README.md` for humans: what it does, the trigger phrases (Italian and English), a
  requirements table (each tool or plugin, what the skill uses it for, whether the skill checks
  it), the install command, what it builds on, and the `CLAUDE.md` section a project must
  provide, with an example.

Register it in `.claude-plugin/marketplace.json` (`plugins[]`, `source: ./plugins/<name>`) and add
a row to the table above. Keep project facts out of the skill and in the project's `CLAUDE.md`.
Everything in this repository is written in English.

## License

This project is licensed under the **MIT License** (see [LICENSE](LICENSE)).

## Development & Support

For bugs or feature requests, please open an issue on the
[GitHub repository](https://github.com/uschti/claude-skills/issues).

Contributions are welcome!

---

Made with ❤️ by Andrea Pellegrini ([@uschti](https://github.com/uschti))
