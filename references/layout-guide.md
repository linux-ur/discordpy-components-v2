# Layout Guide — discord.py

How to build Components V2 messages with `discord.ui.LayoutView`, including the declarative pattern, `add_item`, `find_item`, generic syntax, attachment handling, and a complete worked example (a paginated avatar gallery panel).

## Two ways to add components to a LayoutView

### 1. Declarative class attributes (preferred for static layouts)

```python
class MyPanel(discord.ui.LayoutView):
    header = discord.ui.TextDisplay("# My Bot")
    sep    = discord.ui.Separator(spacing=discord.SeparatorSpacing.small)
    body   = discord.ui.TextDisplay("Choose an action below.")
    row    = discord.ui.ActionRow()

    @row.button(label="Ping", style=discord.ButtonStyle.primary)
    async def ping(self, interaction: discord.Interaction, button: discord.ui.Button):
        await interaction.response.send_message("Pong!", ephemeral=True)
```

Class attributes are processed in **declaration order** — that's the order they appear in the message. Decorators like `@row.button` attach a callback to the button and add it to the named `row` ActionRow.

### 2. `add_item` in `__init__` (preferred for dynamic layouts)

```python
class DynamicPanel(discord.ui.LayoutView):
    def __init__(self, items: list[str]):
        super().__init__()
        self.add_item(discord.ui.TextDisplay("## Pick one:"))
        row = discord.ui.ActionRow()
        for label in items[:5]:   # max 5 buttons per row
            row.add_item(discord.ui.Button(label=label, custom_id=f"pick-{label}"))
        self.add_item(row)
```

Both styles can be mixed in the same class — class attributes are processed first, then `__init__` additions are appended.

## Generic type parameter

`discord.ui.Container["MyView"]`, `discord.ui.ActionRow["MyView"]`, etc. let type-checkers infer `self.view` as `MyView` inside callbacks:

```python
class MyRow(discord.ui.ActionRow["MyPanel"]):
    view: "MyPanel"   # typed by the generic

    @discord.ui.button(label="Go")
    async def go(self, interaction: discord.Interaction, button: discord.ui.Button["MyPanel"]):
        self.view.count += 1   # type-checker knows view is MyPanel
```

At runtime, `["MyPanel"]` is a no-op — it only helps type analysis.

## Subclassing components

Both `Container` and `ActionRow` support subclassing for reusable building blocks:

```python
class CounterRow(discord.ui.ActionRow["CounterView"]):
    @discord.ui.button(label="+1", style=discord.ButtonStyle.success)
    async def increment(self, interaction, button):
        self.view.count += 1
        await self.view.refresh(interaction)

class HeaderSection(discord.ui.Container["CounterView"]):
    title = discord.ui.TextDisplay("## Counter")
    sep   = discord.ui.Separator()

class CounterView(discord.ui.LayoutView):
    header = HeaderSection(accent_color=discord.Color.green())
    row    = CounterRow()
```

## `find_item` — reading back nested components

Assign an explicit numeric `id` at build time, then retrieve the component later:

```python
import zlib

def stable_id(name: str) -> int:
    """Derive a stable 31-bit int from a string."""
    return zlib.crc32(name.encode()) & 0x7FFF_FFFF

COUNT_DISPLAY_ID = stable_id("count_display")

class CounterView(discord.ui.LayoutView):
    display = discord.ui.TextDisplay("Count: 0", id=COUNT_DISPLAY_ID)
    row     = discord.ui.ActionRow()

    @row.button(label="+1", style=discord.ButtonStyle.primary)
    async def increment(self, interaction, button):
        td = self.find_item(COUNT_DISPLAY_ID)   # returns the TextDisplay
        n  = int(td.content.split()[-1]) + 1
        td.content = f"Count: {n}"
        await interaction.response.edit_message(view=self)
```

`find_item` searches the **entire nested tree** (Containers, Sections, ActionRows) — no need to store references separately if you set explicit IDs.

## `walk_children` — iterating all components

Use `walk_children()` to flatten the whole tree, e.g. to disable every button on timeout:

```python
async def on_timeout(self) -> None:
    for item in self.walk_children():
        if isinstance(item, (discord.ui.Button, discord.ui.Select)):
            item.disabled = True
    if self.message:
        await self.message.edit(view=self)
```

`walk_children` is recursive — it descends into Containers, Sections, and ActionRows automatically.

## Sending attachments with LayoutView

Attachments must be explicitly exposed via a component — they don't auto-preview.

```python
f = discord.File("avatar.png")

class AvatarView(discord.ui.LayoutView):
    gallery = discord.ui.MediaGallery(
        discord.MediaGalleryItem(media=f)
    )

await interaction.response.send_message(view=AvatarView(), files=[f])
```

The `discord.File` object must appear **both** in `files=` on the send/edit call **and** as the `media=` argument of the component. discord.py translates the `discord.File` into `attachment://filename` for you.

For `ui.File` (single non-gallery attachment):
```python
f = discord.File(fp=io.BytesIO(data), filename="report.csv")

class ReportView(discord.ui.LayoutView):
    report = discord.ui.File(media=f)

await interaction.response.send_message(view=ReportView(), files=[f])
```

When **editing** a message to update attachments, pass `attachments=[new_file]`:
```python
await interaction.response.edit_message(view=updated_view, attachments=[new_file])
```

## Complete worked example — a paginated avatar gallery

This builds a panel that shows a user select, displays the selected user's avatar in a media gallery, and lets the invoker apply image filters via buttons. It demonstrates: Container, Section, Thumbnail, MediaGallery, ActionRow (subclassed), UserSelect, and `find_item` / attachment swapping.

```python
from __future__ import annotations
import io
import discord
from discord.ext import commands

GALLERY_ID = 1001  # stable numeric id for the MediaGallery


class FilterRow(discord.ui.ActionRow["AvatarPanel"]):
    view: "AvatarPanel"

    @discord.ui.button(label="Grayscale", style=discord.ButtonStyle.secondary)
    async def grayscale(self, interaction: discord.Interaction, button: discord.ui.Button):
        await self.view.apply_filter(interaction, lambda img: img.convert("L").convert("RGB"))

    @discord.ui.button(label="Invert", style=discord.ButtonStyle.secondary)
    async def invert(self, interaction: discord.Interaction, button: discord.ui.Button):
        from PIL import ImageOps
        await self.view.apply_filter(interaction, lambda img: ImageOps.invert(img.convert("RGB")))

    @discord.ui.button(label="Reset", style=discord.ButtonStyle.danger)
    async def reset(self, interaction: discord.Interaction, button: discord.ui.Button):
        await self.view.reset_gallery(interaction)


class UserRow(discord.ui.ActionRow["AvatarPanel"]):
    view: "AvatarPanel"

    @discord.ui.select(cls=discord.ui.UserSelect, placeholder="Select a user…")
    async def user_select(
        self, interaction: discord.Interaction, select: discord.ui.UserSelect
    ):
        await interaction.response.defer()
        user = select.values[0]
        asset = user.display_avatar.with_size(256).with_format("png")
        data  = await asset.read()
        f     = discord.File(fp=io.BytesIO(data), filename="avatar.png")
        gallery: discord.ui.MediaGallery = self.view.find_item(GALLERY_ID)
        gallery.clear_items()
        gallery.add_item(media=f)
        self.view._original_bytes = data
        await self.view._edit(interaction, attachments=[f])


class AvatarPanel(discord.ui.LayoutView):
    """Paginated avatar panel — shows a selected user's avatar with filter buttons."""

    def __init__(
        self,
        user: discord.User | discord.Member,
        default_file: discord.File,
        default_bytes: bytes,
    ):
        super().__init__(timeout=120.0)
        self.invoker       = user
        self._original_bytes = default_bytes
        self.message: discord.Message | None = None

        gallery = discord.ui.MediaGallery["AvatarPanel"](
            discord.MediaGalleryItem(media=default_file),
            id=GALLERY_ID,
        )
        container = discord.ui.Container["AvatarPanel"](
            discord.ui.Section(
                "## Avatar Gallery",
                "Select a user and apply a filter.",
                accessory=discord.ui.Thumbnail(user.display_avatar.url),
            ),
            gallery,
            FilterRow(),
            UserRow(),
            accent_color=discord.Color.blurple(),
        )
        self.add_item(container)

    # ── helpers ──────────────────────────────────────────────────────────

    async def interaction_check(self, interaction: discord.Interaction) -> bool:
        if interaction.user.id != self.invoker.id:
            await interaction.response.send_message(
                "Only the command invoker can use this panel.", ephemeral=True
            )
            return False
        return True

    async def on_timeout(self) -> None:
        for item in self.walk_children():
            if isinstance(item, (discord.ui.Button, discord.ui.Select)):
                item.disabled = True
        if self.message:
            await self.message.edit(view=self)

    async def _edit(self, interaction: discord.Interaction, **kwargs):
        try:
            await interaction.response.edit_message(view=self, **kwargs)
        except discord.InteractionResponded:
            if self.message:
                await self.message.edit(view=self, **kwargs)

    async def apply_filter(self, interaction, filter_fn):
        from PIL import Image
        await interaction.response.defer()
        with Image.open(io.BytesIO(self._original_bytes)) as img:
            out = io.BytesIO()
            filter_fn(img).save(out, format="PNG")
            out.seek(0)
        f = discord.File(fp=out, filename="avatar.png")
        gallery: discord.ui.MediaGallery = self.find_item(GALLERY_ID)
        gallery.clear_items()
        gallery.add_item(media=f)
        await self._edit(interaction, attachments=[f])

    async def reset_gallery(self, interaction):
        await interaction.response.defer()
        f = discord.File(fp=io.BytesIO(self._original_bytes), filename="avatar.png")
        gallery: discord.ui.MediaGallery = self.find_item(GALLERY_ID)
        gallery.clear_items()
        gallery.add_item(media=f)
        await self._edit(interaction, attachments=[f])


@bot.tree.command(name="avatar", description="Browse and filter a user's avatar")
async def avatar_cmd(interaction: discord.Interaction):
    asset = interaction.user.display_avatar.with_size(256).with_format("png")
    data  = await asset.read()
    f     = discord.File(fp=io.BytesIO(data), filename="avatar.png")
    panel = AvatarPanel(interaction.user, f, data)
    resp  = await interaction.response.send_message(view=panel, file=f)
    panel.message = await interaction.original_response()
```

Key things this example shows:
- `add_item` in `__init__` for a dynamic layout.
- `find_item(GALLERY_ID)` to mutate a deeply nested `MediaGallery` on each button click.
- `gallery.clear_items()` + `gallery.add_item(media=f)` to swap images in-place.
- `attachments=[f]` on the edit call to swap the uploaded file.
- `walk_children()` in `on_timeout` to disable everything.
- `interaction_check` to restrict use to the invoker.
- A safe `_edit` helper that handles `InteractionResponded` for the deferred case.
