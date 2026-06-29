# Component Type Reference — discord.py

Field-by-field specification for every component type available through `discord.ui` in discord.py 2.6+/2.7+.  
Cross-referenced from: the discord.py changelog, AbstractUmbra's Components V2 writeup, and the discord.py Masterclass guide.

## Shared base: numeric `id`

Every component (V1 and V2) now has a numeric `id` field:
```python
display = discord.ui.TextDisplay("hello", id=1234567)
```
- Auto-assigned sequentially if omitted. `id=0` is treated as "unset" by Discord and replaced.
- Used with `LayoutView.find_item(id)` to retrieve a deeply nested component.
- **Different from `custom_id`** — `custom_id` is the developer string on interactive components that comes back in interaction payloads.

---

## Container (`discord.ui.Container`) — Type 17

Top-level or first-level inside another `LayoutView`. Cannot be nested inside another `Container`.

```python
# Declarative (class attribute)
class MyLayout(discord.ui.LayoutView):
    box = discord.ui.Container(
        discord.ui.TextDisplay("Inside the box"),
        accent_color=0x7289DA
    )

# Subclass pattern (for reuse)
class MyContainer(discord.ui.Container):
    text = discord.ui.TextDisplay("Hello!")

class MyLayout(discord.ui.LayoutView):
    box = MyContainer(accent_color=discord.Color.blurple())
```

| Parameter | Type | Notes |
|---|---|---|
| `*children` | positional | Any mix of: ActionRow, TextDisplay, Section, MediaGallery, Separator, File |
| `accent_color` | int or `discord.Color` (optional) | RGB color for the left border bar. `None` = no border / transparent. |
| `spoiler` | bool (optional) | Blurs the whole container. Default `False`. |
| `id` | int (optional) | Numeric ID for `find_item`. |

Valid children: `ActionRow`, `TextDisplay`, `Section`, `MediaGallery`, `Separator`, `File`. Does **not** accept another `Container`.

---

## Section (`discord.ui.Section`) — Type 9

Pairs 1–3 text pieces with one accessory (Button or Thumbnail) side by side.

```python
section = discord.ui.Section(
    "## Title",                               # string → wrapped in TextDisplay automatically
    discord.ui.TextDisplay("Description."),   # or pass TextDisplay directly
    accessory=discord.ui.Thumbnail("https://example.com/img.png")
)
```

| Parameter | Type | Notes |
|---|---|---|
| `*components` | positional | 1–3 items; each can be a `str` (auto-wrapped) or `TextDisplay` |
| `accessory` | `Button` or `Thumbnail` | Required. Exactly one. |
| `id` | int (optional) | — |

---

## TextDisplay (`discord.ui.TextDisplay`) — Type 10

Markdown text block. Valid top-level, inside Container, inside Section, or in Modals.

```python
discord.ui.TextDisplay("# H1\n**bold** _italic_\n> blockquote")
```

| Parameter | Type | Notes |
|---|---|---|
| `content` | str | Full Discord Markdown. Mentions ping unless `allowed_mentions` suppresses them on the message. |
| `id` | int (optional) | — |

**Character limit:** 4000 chars shared across **all** TextDisplay components in the same LayoutView (not per-component).  
Check with `view.content_length()` before sending.

---

## Thumbnail (`discord.ui.Thumbnail`) — Type 11

Small image, only valid as the `accessory` of a `Section`. Supports JPEG, PNG, GIF, WebP.

```python
# External URL
discord.ui.Thumbnail("https://i.imgur.com/abc.png")

# Uploaded attachment
discord.ui.Thumbnail(media=discord.File("avatar.png"))
# → then pass file=avatar_file to send_message / files=[avatar_file]

# With options
discord.ui.Thumbnail("https://example.com/img.png", description="Alt text", spoiler=True)
```

| Parameter | Type | Notes |
|---|---|---|
| `media` | str URL or `discord.File` | External URL or attachment reference. |
| `description` | str (optional) | Alt text, max 1024 chars. |
| `spoiler` | bool (optional) | Default `False`. |
| `id` | int (optional) | — |

---

## MediaGallery (`discord.ui.MediaGallery`) — Type 12

Gallery of 1–10 images/videos/GIFs. Top-level or inside a Container.

```python
gallery = discord.ui.MediaGallery(
    discord.MediaGalleryItem("https://example.com/a.png", description="Alt text"),
    discord.MediaGalleryItem("attachment://secret.png", spoiler=True),
    discord.MediaGalleryItem(file_object),   # discord.File shortcut
)
```

`MediaGalleryItem` fields:
| Parameter | Type | Notes |
|---|---|---|
| `media` | str URL or `discord.File` | External URL, `attachment://filename`, or `discord.File` (the file still needs to be in `files=`). |
| `description` | str (optional) | Alt text, max 1024 chars. |
| `spoiler` | bool (optional) | Default `False`. |

Runtime helpers on `ui.MediaGallery`: `.add_item(media=...)`, `.clear_items()`, `.attachments` (list of `discord.File` objects passed as media, useful for `files=` in the edit call).

---

## File (`discord.ui.File`) — Type 13

Displays a single uploaded file (non-image attachment, or image without gallery). Does **not** auto-preview.

```python
f = discord.File(fp=io.BytesIO(b"hello"), filename="notes.txt")

class MyLayout(discord.ui.LayoutView):
    attachment = discord.ui.File(media=f)

await interaction.response.send_message(view=MyLayout(), files=[f])
```

| Parameter | Type | Notes |
|---|---|---|
| `media` | str (`attachment://filename`) or `discord.File` | `attachment://` form only — no external URLs. Passing `discord.File` sets the `attachment://filename` automatically. |
| `spoiler` | bool (optional) | Default `False`. |
| `id` | int (optional) | — |

---

## Separator (`discord.ui.Separator`) — Type 14

Adds spacing (with or without a visible line) between components.

```python
discord.ui.Separator()                                          # default: small, visible
discord.ui.Separator(visible=True,  spacing=discord.SeparatorSpacing.large)
discord.ui.Separator(visible=False, spacing=discord.SeparatorSpacing.small)
```

| Parameter | Type | Notes |
|---|---|---|
| `visible` | bool (optional) | Whether a horizontal line is drawn. Default `True`. |
| `spacing` | `discord.SeparatorSpacing` (optional) | `.small` (default) or `.large`. |
| `id` | int (optional) | — |

---

## ActionRow (`discord.ui.ActionRow`) — Type 1

Container for interactive components. Top-level or inside a Container. Can be subclassed.

```python
# Direct usage (class attribute + decorator)
class MyLayout(discord.ui.LayoutView):
    row: discord.ui.ActionRow["MyLayout"] = discord.ui.ActionRow()

    @row.button(label="Click", style=discord.ButtonStyle.primary)
    async def on_click(self, interaction, button):
        await interaction.response.send_message("clicked!", ephemeral=True)

# Subclass pattern
class MyRow(discord.ui.ActionRow["MyLayout"]):
    @discord.ui.button(label="Go", style=discord.ButtonStyle.green)
    async def go(self, interaction, button):
        ...
```

Rules: ≤5 Buttons **or** exactly 1 Select (any type). Buttons and Selects cannot share a row.  
Constructor also accepts components positionally: `discord.ui.ActionRow(button1, button2)`.

---

## Button (`discord.ui.Button`) — Type 2

```python
# Decorator style (most common)
@row.button(label="Confirm", style=discord.ButtonStyle.success)
async def confirm(self, interaction: discord.Interaction, button: discord.ui.Button):
    ...

# Instance style
btn = discord.ui.Button(label="Docs", style=discord.ButtonStyle.link, url="https://discordpy.readthedocs.io")
```

| Parameter | Type | Notes |
|---|---|---|
| `label` | str (optional) | Max 80 chars. |
| `style` | `discord.ButtonStyle` | `.primary`(blue) `.secondary`(grey) `.success`(green) `.danger`(red) `.link`(URL) |
| `custom_id` | str (optional) | Required for clickable styles; auto-generated if not set. 1–100 chars. |
| `url` | str (optional) | Required for `.link` style. Max 512 chars. Link buttons never trigger a callback. |
| `emoji` | `discord.Emoji` / `discord.PartialEmoji` / str (optional) | — |
| `disabled` | bool (optional) | Default `False`. |
| `row` | int (optional) | Legacy `ui.View` placement hint; not needed in `LayoutView`. |

As a Section **accessory**: pass a `Button` instance to `accessory=`. Note that the button there must **not** be inside an ActionRow — it's placed directly as the accessory.

---

## Select menus — Types 3, 5, 6, 7, 8

All five types (`StringSelect`, `UserSelect`, `RoleSelect`, `MentionableSelect`, `ChannelSelect`) use `@row.select(cls=...)` or `discord.ui.<Type>Select`:

```python
class MyLayout(discord.ui.LayoutView):
    row = discord.ui.ActionRow()

    @row.select(cls=discord.ui.StringSelect, options=[
        discord.SelectOption(label="Option A", value="a"),
        discord.SelectOption(label="Option B", value="b"),
    ])
    async def on_select(self, interaction, select: discord.ui.StringSelect):
        chosen = select.values[0]
        await interaction.response.send_message(f"You picked {chosen}", ephemeral=True)
```

Shared parameters:
| Parameter | Type | Notes |
|---|---|---|
| `custom_id` | str | 1–100 chars. |
| `placeholder` | str (optional) | Max 150 chars. |
| `min_values` / `max_values` | int (optional) | 0–25 / 1–25; default 1/1. |
| `required` | bool (optional) | Modal-only. **Do not** set `disabled` on a modal-bound select. |
| `disabled` | bool (optional) | Message-only. |

`StringSelect` additionally takes `options` (list of `SelectOption`, 1–25 items; each label/value/description ≤ 100 chars).  
`ChannelSelect` takes `channel_types` to restrict the selectable channel types.

**Since v2.6:** `Select` menus can also be used directly in **Modals** (wrap in `ui.Label` for best results).

---

## Modal-only components (v2.6 / v2.7)

### Label (`discord.ui.Label`) — Type 18

Wraps a modal input component with a label and optional description. Preferred over raw ActionRow in modals since v2.6.

```python
class MyModal(discord.ui.Modal, title="Submit"):
    note_input = discord.ui.TextInput(label="", style=discord.TextStyle.paragraph)

    def __init__(self):
        super().__init__()
        self.add_item(discord.ui.Label(
            text="Your note",
            component=self.note_input,
            description="Max 500 characters."
        ))
```

| Parameter | Type | Notes |
|---|---|---|
| `text` | str | Label title, max 45 chars. |
| `component` | `TextInput`, `FileUpload`, `StringSelect`, or `UserSelect` etc. | The wrapped input. |
| `description` | str (optional) | Max 100 chars. |

### FileUpload (`discord.ui.FileUpload`) — Type 19 (v2.6+)

Lets users upload files via a modal. Max 10 files.

```python
file_upload = discord.ui.FileUpload(min_values=1, max_values=1, required=True)
label = discord.ui.Label(text="Attach image", component=file_upload)
```

After modal submit: `file_upload.values` is a list of `Attachment`; convert with `await attachment.to_file()`.

### CheckboxGroup / Checkbox (`discord.ui.CheckboxGroup`, `discord.ui.Checkbox`) — v2.7+

Multi-select group (like checkboxes). Modal-only, must be inside a `Label`.

```python
group = discord.ui.CheckboxGroup(
    discord.CheckboxGroupOption(label="Option A", value="a"),
    discord.CheckboxGroupOption(label="Option B", value="b"),
    min_values=1, max_values=2
)
```

### RadioGroup (`discord.ui.RadioGroup`) — v2.7+

Exactly-one-of-N selection. Modal-only, inside a `Label`.

```python
group = discord.ui.RadioGroup(
    discord.RadioGroupOption(label="Yes", value="yes"),
    discord.RadioGroupOption(label="No",  value="no"),
)
```

---

## Limits reference

| Limit | Value |
|---|---|
| Total components per message | 40 (including nested) |
| TextDisplay total chars (shared) | 4000 |
| MediaGallery items | 1–10 |
| Section text children | 1–3 |
| ActionRow buttons | ≤5 |
| ActionRow selects | exactly 1 (alone) |
| Button label | 80 chars |
| Select placeholder | 150 chars |
| Select options (StringSelect) | 1–25 |
| custom_id | 1–100 chars |
| Thumbnail / MediaGalleryItem description | 1024 chars |
| Label text | 45 chars |
| Label description | 100 chars |
| FileUpload max files | 10 |
