# discord-py-components-v2

A [Claude Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview) that teaches Claude everything it needs to write and debug Discord bot UIs using **discord.py's Message Components V2** (`ui.LayoutView`) in Python.

Built from:
- discord.py official changelog — [v2.6.0](https://discordpy.readthedocs.io/en/latest/whats_new.html#v2-6-0) and [v2.7.0](https://discordpy.readthedocs.io/en/latest/whats_new.html#v2-7-0)
- discord.py Interactions API Reference — [v2.6.4](https://discordpy.readthedocs.io/en/v2.6.4/interactions/api.html)
- AbstractUmbra's [Components V2 writeup](https://about.abstractumbra.dev/discord.py/2025/08/17/components-v2.html)
- FallenDeity's [discord.py Masterclass — Components V2](https://fallendeity.github.io/discord.py-masterclass/components-v2/)
- Discord's official [Component Reference](https://discord.com/developers/docs/components/reference)

## What's inside

```
discord-py-components-v2/
├── SKILL.md                        — entry point: mental model, nesting cheat sheet, quick start
└── references/
    ├── component-types.md          — full parameter spec for every component type + limits
    ├── layout-guide.md             — LayoutView patterns, find_item, walk_children, attachments, full example
    ├── interactions.md             — callbacks, modals, interaction_check, on_timeout, BaseLayoutView
    └── troubleshooting.md          — version requirements, common errors with fixes, View → LayoutView migration
```

`SKILL.md` is concise — it carries the decision-making info needed on every task (nesting rules, quick-start snippet, key rules). The `references/` files are loaded on demand for exhaustive detail.

## Requirements

- **discord.py 2.6.0+** for all Components V2 features (LayoutView, Container, Section, TextDisplay, Thumbnail, MediaGallery, File, Separator, Label, FileUpload).
- **discord.py 2.7.0+** for `ui.CheckboxGroup`, `ui.Checkbox`, and `ui.RadioGroup` in modals.

Install or upgrade:
```bash
pip install -U discord.py
```

## Installing this skill

**Claude.ai / Claude apps:** zip the `discord-py-components-v2/` folder and upload it under *Settings → Capabilities → Skills*.

**Claude Code:** copy to your skills directory:
```bash
cp -r discord-py-components-v2 ~/.claude/skills/
```

**API:** include the skill's contents per the [Agent Skills docs](https://docs.claude.com/en/docs/agents-and-tools/agent-skills/overview).

Once installed, Claude will automatically consult this skill when you talk about discord.py Components V2, LayoutView, ui.Container, ui.Section, or related topics.

## Related skill

If you use **C# with Discord.Net** instead of Python, see the companion skill: [`discord-net-components-v2`](../discord-net-components-v2/).

## Contributing

PRs welcome — especially for:
- New examples for v2.7 modal components (`CheckboxGroup`, `RadioGroup`)
- Corrections to parameter names/defaults verified against source
- Edge cases discovered with specific Discord client versions

## License

MIT — see [`LICENSE`](./LICENSE).
