# Troubleshooting & Version Notes — discord.py

## Version requirements

| Feature | Minimum discord.py version |
|---|---|
| `ui.LayoutView`, `ui.Container`, `ui.Section`, `ui.TextDisplay`, `ui.Thumbnail`, `ui.MediaGallery`, `ui.File`, `ui.Separator`, `ui.ActionRow` (in LayoutView), `ui.Label`, `ui.FileUpload`, Select in modals | **2.6.0** |
| `ui.CheckboxGroup`, `ui.Checkbox`, `ui.RadioGroup` in modals | **2.7.0** |

Install latest: `pip install -U discord.py`  
Check version: `python -m discord --version`

discord.py 2.6 was released alongside Discord's own Components V2 rollout (April 2025 API change). If you're on 2.5.x or earlier, none of the V2 UI classes exist.

---

## `IS_COMPONENTS_V2` flag — when you need to set it manually

**You usually don't.** `send_message(view=<LayoutView>)`, `followup.send(view=...)`, and `ctx.send(view=...)` all set the flag automatically.

The one case where you must clear `content`/`embeds` manually is when **editing** a pre-existing message that used classic `content`/`embeds`:

```python
# Converting a classic message to V2 — must clear the old fields:
await message.edit(
    content=None,
    embeds=[],
    attachments=[],
    view=my_layout_view,
)
```

Once a message has the V2 flag, it can **never** be reverted to classic components. `content` and `embeds` will be silently ignored on any subsequent edits.

---

## Common errors and fixes

### `AttributeError: module 'discord.ui' has no attribute 'LayoutView'`
**Cause:** discord.py < 2.6.  
**Fix:** `pip install -U discord.py`

---

### `TypeError: ... is not a valid child for LayoutView` / nesting exception
**Cause:** A component is placed where it isn't allowed — most often:
- `Thumbnail` placed directly in a `LayoutView` (it must be a `Section` accessory)
- Two components sharing a row (a Button and a Select in the same `ActionRow`)
- A `Container` nested inside another `Container`
- A `Button` or `Select` placed directly in a `LayoutView` without an `ActionRow`

**Fix:** Check the nesting table in `SKILL.md`. discord.py raises the exception locally before any network call — the message is usually specific about which component is invalid.

---

### `HTTPException: 400 Bad Request — content/embeds not allowed with IS_COMPONENTS_V2`
**Cause:** Passing `content="..."` or `embeds=[...]` when sending a `LayoutView` message.  
**Fix:** Remove `content` and `embeds` from the send/edit call. Use `ui.TextDisplay` for text and `ui.Container` for embed-like blocks.

---

### `HTTPException: 400 Bad Request — Invalid Form Body` (from Discord)
**Cause:** A field limit was exceeded — Discord validated the payload but found something out of range. Common triggers:
- TextDisplay total chars > 4000 (shared pool across all TextDisplays)
- MediaGallery with 0 or > 10 items
- Section with 0 or > 3 text children
- ActionRow with > 5 buttons or a select + buttons mixed
- `custom_id` > 100 chars

**Fix:** Check the limits table in `references/component-types.md`. Use `view.content_length()` to check total text chars before sending.

---

### `ui.Section.children` or `ui.Section.accessory` has `None` as `Item.parent`
**Cause:** A known bug fixed in discord.py **2.6.1**.  
**Fix:** Upgrade to 2.6.1+.

---

### Memory leak when removing items from `LayoutView`
**Cause:** A known bug fixed in discord.py **2.7.1**.  
**Fix:** Upgrade to 2.7.1+.

---

### Interaction never fires / "This interaction failed" in Discord
**Cause:** Most common reasons:
1. Button is `style=discord.ButtonStyle.link` — link buttons never fire a callback.
2. The view's `timeout` expired and the view is no longer registered.
3. `interaction_check` returned `False` without sending a response — Discord shows the error to the user.
4. The callback raised an exception before responding — discord.py calls `on_error`, but if that also fails without responding, Discord shows the default failure message.

**Fix:** For case 3: always `await interaction.response.send_message(...)` before returning `False` in `interaction_check`. For case 2: use a persistent view (`add_view`) or set `timeout=None`.

---

### `discord.InteractionResponded` raised inside a callback
**Cause:** The callback tried to respond twice — e.g. called `defer()` then also called `edit_message()` as if it were the first response.  
**Fix:** Use the `_edit` helper pattern from `references/interactions.md`:
```python
async def _edit(self, interaction, **kwargs):
    try:
        await interaction.response.edit_message(**kwargs)
    except discord.InteractionResponded:
        if self.message:
            await self.message.edit(**kwargs)
```

---

### `ui.DynamicItem` not working inside `Section` accessory
**Cause:** Known bug fixed in discord.py **2.6.1**.  
**Fix:** Upgrade to 2.6.1+.

---

### Attachments not showing / files appear as generic download links
**Cause:** Attachments require explicit component references — discord.py Components V2 doesn't auto-preview.  
**Fix:** Pass the `discord.File` both in `files=` on the send call and as `media=` on `ui.File`, `ui.Thumbnail`, or `discord.MediaGalleryItem`. discord.py converts it to `attachment://filename` automatically.

---

### `ui.Select.required` not applied in a modal
**Cause:** Known bug fixed in discord.py **2.6.4**.  
**Fix:** Upgrade to 2.6.4+.

---

## Migration from `ui.View`

The old system still works. There's no forced migration. Differences to be aware of:

| `ui.View` | `ui.LayoutView` |
|---|---|
| `@ui.button` / `@ui.select` at class level | Must be attached to a named `ActionRow` attribute: `@row.button` |
| `add_item(button)` directly | `add_item(ActionRow(button))` or use `ActionRow.add_item(button)` |
| `content=` / `embeds=` work alongside components | Not allowed — use `TextDisplay` / `Container` instead |
| `walk_children()` only iterates top-level items | `walk_children()` recursively descends into all containers |
| Persistent view: `custom_id` on each button | Same — `custom_id`s survive restarts the same way |

If you migrate and keep the same `custom_id` values, existing users of a persistent view can keep clicking buttons on old messages after you deploy.
