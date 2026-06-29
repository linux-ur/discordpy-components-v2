# Interactions — discord.py

How to handle button clicks, select changes, and modal submissions from a Components V2 `LayoutView`, including the interaction lifecycle, the `BaseLayoutView` pattern, deferred responses, and modals.

## Interaction lifecycle

Every interaction on a component has a **3-second window** to send the first response. After that, you have up to **15 minutes** to edit or follow up on that interaction token.

The four response types you'll use for component interactions:

| Method | When to use |
|---|---|
| `await interaction.response.send_message(...)` | Reply in a new (ephemeral or public) message |
| `await interaction.response.edit_message(view=self, ...)` | Rebuild the current message in-place — the most common one for button clicks |
| `await interaction.response.defer()` | Ack within 3 s; work continues; then call `edit_original_response` or `followup.send` |
| `await interaction.response.send_modal(modal)` | Open a modal — must be the **first** response to the interaction |

## Button callback

```python
class MyLayout(discord.ui.LayoutView):
    row = discord.ui.ActionRow()

    @row.button(label="Confirm", style=discord.ButtonStyle.success)
    async def confirm(self, interaction: discord.Interaction, button: discord.ui.Button):
        button.disabled = True
        button.label    = "Confirmed ✓"
        await interaction.response.edit_message(view=self)
```

## Select callback

```python
class MyLayout(discord.ui.LayoutView):
    row = discord.ui.ActionRow()

    @row.select(cls=discord.ui.StringSelect, options=[
        discord.SelectOption(label="Red",   value="red"),
        discord.SelectOption(label="Blue",  value="blue"),
    ])
    async def on_select(self, interaction: discord.Interaction, select: discord.ui.StringSelect):
        colour = select.values[0]
        await interaction.response.send_message(f"You chose {colour}!", ephemeral=True)
```

## Modals

### Opening from a button

```python
@row.button(label="Open form", style=discord.ButtonStyle.primary)
async def open_form(self, interaction: discord.Interaction, button: discord.ui.Button):
    await interaction.response.send_modal(MyModal())
```

A modal can only be opened as the **first** response to an interaction — you can't defer first and then send a modal.

### Defining and handling a modal (v2.6 style with `ui.Label`)

```python
class FeedbackModal(discord.ui.Modal, title="Feedback"):
    def __init__(self):
        super().__init__()
        self.body_input = discord.ui.TextInput(
            style=discord.TextStyle.paragraph,
            max_length=500
        )
        self.add_item(discord.ui.Label(
            text="Your feedback",
            component=self.body_input,
            description="Max 500 characters.",
        ))

    async def on_submit(self, interaction: discord.Interaction):
        text = self.body_input.value
        await interaction.response.send_message(
            f"Thanks! You said:\n> {text}", ephemeral=True
        )

    async def on_error(self, interaction: discord.Interaction, error: Exception):
        await interaction.response.send_message("Something went wrong.", ephemeral=True)
```

### FileUpload modal (v2.6+)

```python
class UploadModal(discord.ui.Modal, title="Upload Image"):
    def __init__(self):
        super().__init__()
        self.file_upload = discord.ui.FileUpload(min_values=1, max_values=1, required=True)
        self.add_item(discord.ui.Label(
            text="Image file",
            component=self.file_upload,
            description="PNG or JPEG, max 8 MB."
        ))

    async def on_submit(self, interaction: discord.Interaction):
        if not self.file_upload.values:
            await interaction.response.send_message("No file received.", ephemeral=True)
            return
        # values[0] is a discord.Attachment
        f = await self.file_upload.values[0].to_file(filename="upload.png")
        await interaction.response.send_message(file=f)
```

### RadioGroup / CheckboxGroup (v2.7+)

```python
class PollModal(discord.ui.Modal, title="Quick poll"):
    def __init__(self):
        super().__init__()
        self.vote = discord.ui.RadioGroup(
            discord.RadioGroupOption(label="Option A", value="a"),
            discord.RadioGroupOption(label="Option B", value="b"),
        )
        self.add_item(discord.ui.Label(text="Your vote", component=self.vote))

    async def on_submit(self, interaction: discord.Interaction):
        chosen = self.vote.values[0] if self.vote.values else "none"
        await interaction.response.send_message(f"Voted: {chosen}", ephemeral=True)
```

## `interaction_check` — restricting who can interact

```python
class MyLayout(discord.ui.LayoutView):
    def __init__(self, invoker: discord.User):
        super().__init__()
        self.invoker = invoker

    async def interaction_check(self, interaction: discord.Interaction) -> bool:
        if interaction.user.id != self.invoker.id:
            await interaction.response.send_message(
                "Only the person who ran this command can interact with it.", ephemeral=True
            )
            return False
        return True
```

Return `True` to allow the interaction, `False` to block it (discord.py will **not** respond automatically — you must send a response before returning `False` to avoid a "This interaction failed" error).

## `on_timeout` — cleaning up after inactivity

```python
async def on_timeout(self) -> None:
    for item in self.walk_children():
        if isinstance(item, (discord.ui.Button, discord.ui.Select)):
            item.disabled = True
    if self.message:
        await self.message.edit(view=self)
```

Store `self.message` when sending:
```python
resp = await interaction.response.send_message(view=panel)
panel.message = await interaction.original_response()
```

Or with `ctx.send`:
```python
panel.message = await ctx.send(view=panel)
```

## `on_error` — handling callback exceptions

```python
async def on_error(
    self, interaction: discord.Interaction, error: Exception, item: discord.ui.Item
) -> None:
    import traceback
    tb = "".join(traceback.format_exception(type(error), error, error.__traceback__))
    self._disable_all()
    self.add_item(discord.ui.TextDisplay(f"An error occurred:\n```py\n{tb[:1800]}\n```"))
    try:
        await interaction.response.edit_message(view=self)
    except discord.InteractionResponded:
        if self.message:
            await self.message.edit(view=self)
    self.stop()
```

## The `BaseLayoutView` pattern

A reusable base class that handles the common boilerplate:

```python
class BaseLayoutView(discord.ui.LayoutView):
    """Reusable base: restricts interactions to invoker, disables on timeout, handles errors."""
    interaction: discord.Interaction | None = None
    message:     discord.Message     | None = None

    def __init__(self, invoker: discord.User | discord.Member, timeout: float = 60.0):
        super().__init__(timeout=timeout)
        self.invoker = invoker

    async def interaction_check(self, interaction: discord.Interaction) -> bool:
        if interaction.user.id != self.invoker.id:
            await interaction.response.send_message("Not your panel.", ephemeral=True)
            return False
        self.interaction = interaction
        return True

    def _disable_all(self) -> None:
        for item in self.walk_children():
            if isinstance(item, (discord.ui.Button, discord.ui.Select)):
                item.disabled = True

    async def _edit(self, *args, **kwargs) -> None:
        if self.interaction is not None:
            try:
                await self.interaction.response.edit_message(*args, **kwargs)
                return
            except discord.InteractionResponded:
                pass
        if self.message is not None:
            await self.message.edit(*args, **kwargs)

    async def on_timeout(self) -> None:
        self._disable_all()
        await self._edit(view=self)

    async def on_error(self, interaction, error, item) -> None:
        import traceback
        tb = "".join(traceback.format_exception(type(error), error, error.__traceback__))
        self._disable_all()
        self.add_item(discord.ui.TextDisplay(f"```py\n{tb[:1800]}\n```"))
        await self._edit(view=self)
        self.stop()
```

Usage:
```python
class MyPanel(BaseLayoutView):
    header = discord.ui.TextDisplay("# My Panel")
    row    = discord.ui.ActionRow()

    @row.button(label="Do thing", style=discord.ButtonStyle.primary)
    async def do_thing(self, interaction, button):
        # business logic
        await self._edit(view=self)

@bot.tree.command()
async def show(interaction: discord.Interaction):
    panel = MyPanel(interaction.user)
    await interaction.response.send_message(view=panel)
    panel.message = await interaction.original_response()
```

## Editing a message to use LayoutView

If the original message had `content` or `embeds`, you **must** clear them:

```python
await message.edit(
    content=None,
    embeds=[],
    attachments=[],
    view=my_layout_view
)
```

discord.py will set the `IS_COMPONENTS_V2` flag automatically. You cannot reverse this — once V2, always V2.

## Persistent LayoutViews (survives bot restart)

If all interactive components have **explicit `custom_id`s**, a `LayoutView` can survive restarts the same way a persistent `View` does:

```python
class PersistentPanel(discord.ui.LayoutView):
    row = discord.ui.ActionRow()

    @row.button(label="Ping", custom_id="persistent_ping")
    async def ping(self, interaction, button):
        await interaction.response.send_message("Pong!", ephemeral=True)

# On bot start:
bot.add_view(PersistentPanel())
```

`custom_id`s must be identical to those used when the message was originally sent. `ui.DynamicItem` also works within `LayoutView` the same way it does in `View`.
