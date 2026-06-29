---
name: discord-py-components-v2
description: Reference and implementation guide for building Discord bot UIs with discord.py's Message Components V2 system (ui.LayoutView) in Python. Covers every V2-only component (Container, Section, TextDisplay, Thumbnail, MediaGallery, File, Separator), the shared V1/V2 interactive components (ActionRow, Button, all Select types, TextInput), nesting rules, the discord.py class-based declarative syntax, find_item / walk_children, and modal components (Label, FileUpload, CheckboxGroup, RadioGroup added in v2.7). Use this skill whenever the user is writing or debugging a Discord bot in Python with discord.py and mentions "Components V2", "LayoutView", "ui.Container", "ui.Section", "ui.TextDisplay", or asks about errors with nesting, find_item, walk_children, edit_message with LayoutView, or migrating from ui.View to ui.LayoutView.
---

# discord.py — Message Components V2

## What this skill covers

Discord's **Message Components V2** landed in **discord.py 2.6** (PR #10166), with modal-component additions in 2.7. It replaces the classic `content` + `embeds` model with a composable tree of Python classes: `ui.LayoutView` is the new root, and all layout/display/interactive components hang off it as class-level attributes or constructor arguments.

| File | Open it when you need… |
|---|---|
| `references/component-types.md` | Full field/param spec for every component type with limits, parent rules, and the discord.py class signature. |
| `references/layout-guide.md` | The declarative class pattern, `add_item`, `find_item`, `walk_children`, generic syntax, attachment files, and a complete worked example. |
| `references/interactions.md` | Interaction callbacks, `on_timeout`, `interaction_check`, `edit_message` vs `edit_message`, deferred responses, BaseLayoutView pattern. |
| `references/troubleshooting.md` | Common errors (nesting, content/embeds conflict, missing files, `None` parent), version changelog notes, migration from `ui.View`. |

## Mental model

There are two kinds of components:

- **Shared (V1 + V2)** — `ActionRow`, `Button`, the five `Select` types, `TextInput`. These exist since discord.py 2.0 and are registered the same way — but inside a `LayoutView` they **must** be manually placed inside an `ActionRow`. The `@ui.button` / `@ui.select` shortcut decorators still work when attached to a named `ActionRow` class attribute.
- **V2-only** — `Container`, `Section`, `TextDisplay`, `Thumbnail`, `MediaGallery`, `File`, `Separator`. Only available when the root class is `LayoutView` (not `View`).

Once a message is sent with `LayoutView`, **`content`, `embeds`, `stickers`, and `polls` stop working** — `TextDisplay` is the content replacement; `Container` is the embed replacement. Once sent, you cannot convert the message back to classic components.

Sending a `LayoutView` **automatically sets the `IS_COMPONENTS_V2` flag** — you don't need to set it manually on first send. When editing a non-V2 message to a V2 one, you must clear `content` and `embeds` (set to `None`).

## Nesting cheat sheet

Get this right and 90 % of errors disappear.

| Component | Top-level in LayoutView | Inside Container | Inside Section | Inside ActionRow | Notes |
|---|---|---|---|---|---|
| `Container` | ✅ | ❌ no nesting | ❌ | ❌ | Can hold: ActionRow, TextDisplay, Section, MediaGallery, Separator, File |
| `Section` | ✅ | ✅ | ❌ | ❌ | Holds 1–3 TextDisplays + 1 accessory (Button or Thumbnail) |
| `ActionRow` | ✅ | ✅ | ❌ | ❌ | Holds ≤5 Buttons **or** 1 Select |
| `TextDisplay` | ✅ | ✅ | ✅ | ❌ | Also valid in Modals |
| `MediaGallery` | ✅ | ✅ | ❌ | ❌ | 1–10 `MediaGalleryItem`s |
| `Separator` | ✅ | ✅ | ❌ | ❌ | — |
| `File` | ✅ | ✅ | ❌ | ❌ | `attachment://` only |
| `Thumbnail` | ❌ | ❌ | Accessory only | ❌ | — |
| `Button` | ❌ | ❌ | Accessory only | ✅ | Link/Premium styles never fire a callback |
| Any Select | ❌ | ❌ | ❌ | ✅ (alone) | One select per ActionRow, no buttons |
| `TextInput` | ❌ | ❌ | ❌ | ❌ (deprecated) | Use `Label` + `TextInput` in Modals (v2.6+) |
| `Label` | Modal only | — | — | — | Wraps TextInput/FileUpload/Select in modals |
| `FileUpload` | Modal only | — | — | — | v2.6+, requires `Label` |
| `CheckboxGroup` | Modal only | — | — | — | v2.7+ |
| `RadioGroup` | Modal only | — | — | — | v2.7+ |

Max **40 total components** per message (counts nested components). All TextDisplay content across the whole LayoutView shares a **4000-char** pool.

## Quick start

```python
import discord

class TicketPanel(discord.ui.LayoutView):
    # Class-level attributes = declarative component tree
    header = discord.ui.TextDisplay("# Support Center")
    info   = discord.ui.TextDisplay("Pick a category and our team will jump in.")

    row = discord.ui.ActionRow()

    @row.button(label="Billing", style=discord.ButtonStyle.primary)
    async def billing(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.send_message("Opening a billing ticket…", ephemeral=True)

    @row.button(label="Technical", style=discord.ButtonStyle.secondary)
    async def technical(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.send_message("Opening a technical ticket…", ephemeral=True)

@bot.tree.command()
async def ticket(interaction: discord.Interaction):
    await interaction.response.send_message(view=TicketPanel())
```

No flag-setting needed — `send_message(view=<LayoutView>)` handles it automatically.

## Key rules of thumb

- **Two kinds of IDs**: Every component has a **numeric `id`** (auto-assigned or set by you, used with `find_item()`). Interactive components additionally have a **`custom_id`** (string, returned in the interaction payload). Don't confuse them.
- **`find_item(id)`** searches all nested components by numeric `id`. Set explicit IDs (`id=MY_CONST`) only when you need to mutate a deeply nested component later; otherwise let discord.py auto-assign.
- **`walk_children()`** iterates over every component in the tree (flattened), including those inside Containers and Sections — necessary for disabling all buttons on timeout.
- **Attachments don't auto-preview**. Pass `discord.File(...)` both in the `files=` argument of your send call and as the `media=` argument of `ui.File`, `ui.Thumbnail`, or `discord.MediaGalleryItem(...)`.
- **Generics are optional but nice**: `discord.ui.Container["MyView"]` lets type-checkers infer that `self.view` is `MyView`. At runtime it has no effect.
- **`ui.View` still works** — `LayoutView` is strictly additive. Migrate at your own pace; existing `ui.View` bots are unaffected.
- **3-second rule applies**: The first response to any interaction must happen within 3 s. Use `await interaction.response.defer()` for slow operations, then `await interaction.followup.send(...)` or `await interaction.edit_original_response(...)`.
