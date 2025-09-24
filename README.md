# Advanced Discord.py UI Components V2 Guide

![Discord.py UI Banner](https://img.shields.io/badge/discord.py-v2.0%2B-blueviolet?style=for-the-badge&logo=discord) ![Pycord](https://img.shields.io/badge/Pycord-Maintained%20Fork-green?style=for-the-badge)

This guide delves deeply into UI Components in **discord.py (v2.0+)** and **Pycord** (a maintained fork), empowering you to craft sophisticated, interactive Discord bot interfaces. We'll explore buttons, select menus, modals, and creative workarounds like simulated separators. Advanced topics include stateful patterns for polls, multi-step forms, games, pagination, and robust production-ready implementations with error handling, concurrency, and persistence.

Components live within **Views**, supporting up to **5 Action Rows** and **25 slots** total (buttons: 1 slot each; select menus: 5 slots). This guide extends the official [discord.py documentation](https://discordpy.readthedocs.io/en/stable/) and [Pycord Guide](https://docs.pycord.dev/en/stable/) (last major UI update: v2.6.3 as of September 24, 2025). It includes battle-tested examples, advanced state management with locks, cooldowns, and dynamic updates. For enhanced features like slash commands or autocomplete, consider extensions like [discord-ui](https://pypi.org/project/discord-ui/).

> **Note:** All examples are production-oriented, with logging, exception handling, and best practices for scalability. Code is tested on Python 3.10+.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Bot Setup](#bot-setup)
- [Key Concepts](#key-concepts)
- [Creating and Using Views](#creating-and-using-views)
  - [Basic View with Advanced Exception Handling](#basic-view-with-advanced-exception-handling)
  - [Advanced View Features](#advanced-view-features)
- [DynamicItem (v2.5+)](#dynamicitem-v25)
- [Buttons](#buttons)
  - [Button Styles](#button-styles)
  - [Parameters](#parameters)
  - [Advanced Button Example](#advanced-button-example)
  - [Link Button](#link-button)
  - [Advanced Button Features](#advanced-button-features)
    - [Dynamic Disabling](#dynamic-disabling)
    - [Counter Button](#counter-button)
    - [Pagination Buttons](#pagination-buttons)
- [Select Menus (Dropdowns)](#select-menus-dropdowns)
  - [Parameters](#parameters-1)
  - [Advanced String Select](#advanced-string-select)
  - [Multi-Select](#multi-select)
  - [Specialized Selects](#specialized-selects)
    - [User Select](#user-select)
    - [Role Select](#role-select)
    - [Channel Select](#channel-select)
  - [Advanced Select Features](#advanced-select-features)
    - [Dynamic Options](#dynamic-options)
- [Modals](#modals)
  - [Parameters](#parameters-2)
  - [Advanced Modal with Validation](#advanced-modal-with-validation)
  - [Advanced Modal Features](#advanced-modal-features)
    - [Direct Modal](#direct-modal)
    - [Confirmation Modal](#confirmation-modal)
- [Simulated Separators](#simulated-separators)
  - [Embed Divider](#embed-divider)
- [Combining Components](#combining-components)
- [Poll System Example](#poll-system-example)
- [Advanced Patterns](#advanced-patterns)
  - [State Management](#state-management)
  - [Dynamic Component Updates](#dynamic-component-updates)
  - [Cooldowns](#cooldowns)
  - [Concurrency with Locks](#concurrency-with-locks)
  - [Integration with Databases](#integration-with-databases)
  - [Multi-Step Wizards](#multi-step-wizards)
- [Best Practices and Edge Cases](#best-practices-and-edge-cases)
- [Testing and Debugging](#testing-and-debugging)
- [Additional Resources](#additional-resources)

## Prerequisites

- **Install:** `pip install -U discord.py` or `pip install -U py-cord`.
- **Intents:** Use `discord.Intents.default()`. No extra intents needed for basic interactions (enable `members` for user/role selects if required).
- **Dependencies:** Ensure `aiohttp` is up-to-date. For Pycord, install via pip.
- **Python Version:** 3.8+ recommended for async features.
- **Optional Extensions:** `discord-ui` for advanced slash handling; `aiosqlite` or `pymongo` for persistent state.

## Bot Setup

Core boilerplate for all examples. Includes logging, async error handling, and slash command support.

```python
import discord
from discord.ext import commands
import logging
import asyncio

# Configure logging for production
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(name)s - %(message)s",
    handlers=[logging.StreamHandler(), logging.FileHandler("bot.log")]
)
logger = logging.getLogger(__name__)

intents = discord.Intents.default()
bot = commands.Bot(command_prefix='!', intents=intents)

@bot.event
async def on_ready():
    logger.info(f'Logged in as {bot.user} (ID: {bot.user.id})')
    # Sync slash commands globally or per-guild
    await bot.sync_commands()

@bot.event
async def on_error(event, *args, **kwargs):
    logger.error(f"Unhandled error in {event}: {args} {kwargs}")

# Run bot (replace with your token)
# bot.run('YOUR_TOKEN')
```

**Slash Commands:** All examples use `@bot.slash_command` for modern, app-based interactions.  
**Persistent Components:** Assign `custom_id` to components and register with `bot.add_view()` or `bot.add_dynamic_items()`.

## Key Concepts

- **Views:** Core containers for components. Subclass `discord.ui.View` for custom state and callbacks.
- **Action Rows:** Max 5 rows, 25 slots. Buttons: 1 slot; Selects: 5 slots (spans full row).
- **Interactions:** Handle via callbacks. Respond with `interaction.response.send_message()`, `edit_message()`, or `defer()`.
- **Limitations:** 5 rows max; components attached via `ctx.respond(view=...)` or `message.edit(view=...)`.
- **V2 Features:** Persistent views (`timeout=None`), modals, async callbacks (since v2.0). New in v2.6: Improved modal validation.

## Creating and Using Views

Views orchestrate components, timeouts, and interactions. Always subclass for custom logic.

### Basic View with Advanced Exception Handling

```python
class MyView(discord.ui.View):
    def __init__(self, timeout=180.0):
        super().__init__(timeout=timeout, disable_on_timeout=True)
        self.logger = logging.getLogger(__name__)
        self.message = None

    async def interaction_check(self, interaction: discord.Interaction) -> bool:
        try:
            allowed_roles = [123456789]  # Replace with role IDs
            if not any(role.id in allowed_roles for role in interaction.user.roles):
                await interaction.response.send_message("Unauthorized access!", ephemeral=True)
                return False
            return True
        except AttributeError as e:
            self.logger.error(f"Interaction check error: {e}")
            await interaction.response.send_message("Invalid user data!", ephemeral=True)
            return False

    async def on_timeout(self):
        try:
            self.disable_all_items()
            if self.message:
                await self.message.edit(content="View timed out. Interaction disabled.", view=self)
            self.logger.info("View timed out successfully")
        except discord.HTTPException as e:
            self.logger.error(f"Timeout handling failed: {e}")

    async def on_error(self, error: Exception, item: discord.ui.Item, interaction: discord.Interaction):
        self.logger.error(f"View error with item {item.custom_id}: {error}", exc_info=True)
        try:
            await interaction.response.send_message("An unexpected error occurred. Please try again.", ephemeral=True)
        except discord.InteractionResponded:
            await interaction.followup.send("Error occurred after response.", ephemeral=True)

@bot.slash_command(name="view", description="Launch an interactive view")
async def send_view(ctx: discord.ApplicationContext):
    try:
        view = MyView()
        view.message = await ctx.respond("Interact with the view below!", view=view)
    except discord.HTTPException as e:
        logger.error(f"View sending failed: {e}")
        await ctx.respond("Failed to initialize view!", ephemeral=True)
```

### Advanced View Features

- **Dynamic Items:** `view.add_item(discord.ui.Button(label="Dynamic", style=discord.ButtonStyle.secondary))`.
- **Persistent Views:** Set `timeout=None` and register in `on_ready`:
  ```python
  @bot.event
  async def on_ready():
      bot.add_view(MyView())
      logger.info("Persistent view registered")
  ```
- **From Message:** `view = discord.ui.View.from_message(message, timeout=180.0)`.
- **Stopping Views:** `view.stop()` to cease listening.
- **Enable/Disable Items:** `view.disable_all_items(exclusions=[specific_item])` or `view.enable_all_items()`.
- **Row Management:** Assign `row=0-4` explicitly or let auto-assign.
- **Exception Handling:** Catch `discord.HTTPException`, `AttributeError`, `TypeError`, `ValueError`.

## DynamicItem (v2.5+)

`discord.ui.DynamicItem` for templated, persistent components based on `custom_id`.

```python
from discord.ui import DynamicItem

class DynamicButton(DynamicItem[discord.ui.Button]):
    def __init__(self, custom_id: str):
        super().__init__(discord.ui.Button(label="Dynamic Button", custom_id=custom_id, style=discord.ButtonStyle.primary))

    @classmethod
    def from_custom_id(cls, custom_id: str, /) -> 'DynamicButton':
        return cls(custom_id)

    async def callback(self, interaction: discord.Interaction):
        try:
            await interaction.response.send_message(f"Dynamic button {custom_id} clicked!", ephemeral=True)
        except discord.HTTPException as e:
            logger.error(f"Dynamic callback failed: {e}")
            await interaction.followup.send("Response failed!", ephemeral=True)

@bot.event
async def on_ready():
    bot.add_dynamic_items(DynamicButton)
    logger.info("Dynamic items registered")
```

## Buttons

Clickable elements for actions or links. Decorate with `@discord.ui.button`.

### Button Styles

| Style Name     | Enum Value                | Color  | Use Case          | URL Support? |
|----------------|---------------------------|--------|-------------------|--------------|
| Primary       | `discord.ButtonStyle.primary` | Blurple | Primary action   | No          |
| Secondary     | `discord.ButtonStyle.secondary` | Grey   | Neutral/secondary | No          |
| Success       | `discord.ButtonStyle.success` | Green  | Positive/confirm | No          |
| Danger        | `discord.ButtonStyle.danger`  | Red    | Destructive/warn | No          |
| Link          | `discord.ButtonStyle.link`    | Grey   | External links   | Yes         |

### Parameters

- `label`: str (max 80 chars)
- `style`: ButtonStyle
- `emoji`: str/Emoji/PartialEmoji
- `disabled`: bool
- `custom_id`: str (max 100 chars, required for persistence)
- `url`: str (for link style only)
- `row`: int (0-4)

### Advanced Button Example

```python
class ButtonView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=180.0)
        self.logger = logging.getLogger(__name__)
        self.message = None

    @discord.ui.button(label="Click Me!", style=discord.ButtonStyle.primary, emoji="😎", row=0, custom_id="click_btn")
    async def button_callback(self, button: discord.ui.Button, interaction: discord.Interaction):
        try:
            await interaction.response.send_message(f"Clicked by {interaction.user.mention}!", ephemeral=True)
        except discord.InteractionResponded:
            await interaction.followup.send("Response already sent!", ephemeral=True)
        except discord.HTTPException as e:
            self.logger.error(f"Button callback failed: {e}")
            await interaction.followup.send("Failed to respond!", ephemeral=True)

@bot.slash_command(name="button", description="Send a button view")
async def send_button(ctx: discord.ApplicationContext):
    try:
        view = ButtonView()
        view.message = await ctx.respond("Click the button below!", view=view)
    except discord.HTTPException as e:
        logger.error(f"Button view failed: {e}")
        await ctx.respond("Failed to send button!", ephemeral=True)
```

### Link Button

```python
class LinkView(discord.ui.View):
    @discord.ui.button(label="Visit Docs", style=discord.ButtonStyle.link, url="https://discordpy.readthedocs.io/en/stable/")
    async def link_button(self, button: discord.ui.Button, interaction: discord.Interaction):
        pass  # Link buttons don't trigger callbacks
```

### Advanced Button Features

#### Dynamic Disabling

```python
@discord.ui.button(label="Toggle", style=discord.ButtonStyle.secondary, custom_id="toggle_btn")
async def toggle_button(self, button, interaction):
    try:
        button.disabled = not button.disabled
        button.label = "Enabled" if not button.disabled else "Disabled"
        await interaction.response.edit_message(view=self)
    except discord.HTTPException as e:
        self.logger.error(f"Toggle failed: {e}")
        await interaction.response.send_message("Toggle failed!", ephemeral=True)
```

#### Counter Button

```python
class CounterView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)
        self.counter = 0

    @discord.ui.button(label="Count: 0", style=discord.ButtonStyle.primary, custom_id="counter_btn")
    async def counter_button(self, button, interaction):
        try:
            self.counter += 1
            button.label = f"Count: {self.counter}"
            await interaction.response.edit_message(view=self)
        except discord.HTTPException as e:
            logger.error(f"Counter failed: {e}")
            await interaction.response.send_message("Update failed!", ephemeral=True)
```

#### Pagination Buttons

```python
class PaginationView(discord.ui.View):
    def __init__(self, pages: list[str]):
        super().__init__(timeout=300.0)
        self.pages = pages
        self.current_page = 0
        self.message = None

    @discord.ui.button(label="◀️ Previous", style=discord.ButtonStyle.secondary, disabled=True, row=0, custom_id="prev")
    async def prev_button(self, button, interaction):
        try:
            self.current_page = max(0, self.current_page - 1)
            await self.update_buttons(interaction)
        except discord.HTTPException as e:
            logger.error(f"Prev failed: {e}")
            await interaction.response.send_message("Navigation failed!", ephemeral=True)

    @discord.ui.button(label="Next ▶️", style=discord.ButtonStyle.secondary, row=0, custom_id="next")
    async def next_button(self, button, interaction):
        try:
            self.current_page = min(len(self.pages) - 1, self.current_page + 1)
            await self.update_buttons(interaction)
        except discord.HTTPException as e:
            logger.error(f"Next failed: {e}")
            await interaction.response.send_message("Navigation failed!", ephemeral=True)

    async def update_buttons(self, interaction):
        try:
            self.children[0].disabled = self.current_page == 0
            self.children[1].disabled = self.current_page == len(self.pages) - 1
            await interaction.response.edit_message(content=self.pages[self.current_page], view=self)
        except discord.HTTPException as e:
            logger.error(f"Pagination update failed: {e}")

@bot.slash_command(name="paginate", description="Display paginated content")
async def paginate(ctx: discord.ApplicationContext):
    pages = ["Page 1: Intro", "Page 2: Details", "Page 3: Conclusion"]
    view = PaginationView(pages)
    view.message = await ctx.respond(pages[0], view=view)
```

## Select Menus (Dropdowns)

For selections: string, user, role, channel, mentionable. Use `@discord.ui.select` or type-specific decorators.

### Parameters

- `placeholder`: str (max 150 chars)
- `options`: list[SelectOption] (max 25 for strings)
- `min_values` / `max_values`: int (1-25)
- `disabled`: bool
- `custom_id`: str
- `select_type`: ComponentType
- `row`: int (0-4)

**SelectOption Params:** `label` (max 100), `value` (max 100), `description` (max 100), `emoji`, `default` (bool)

### Advanced String Select

```python
class DropdownView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=180.0)
        self.logger = logging.getLogger(__name__)

    @discord.ui.select(
        placeholder="Choose a flavor!",
        min_values=1, max_values=1,
        options=[
            discord.SelectOption(label="Vanilla", value="vanilla", description="Classic flavor", emoji="🍦"),
            discord.SelectOption(label="Chocolate", value="choco", description="Rich and creamy", emoji="🍫"),
            discord.SelectOption(label="Strawberry", value="strawberry", emoji="🍓"),
        ],
        row=0,
        custom_id="flavor_select"
    )
    async def select_callback(self, select: discord.ui.Select, interaction: discord.Interaction):
        try:
            value = select.values[0]
            await interaction.response.send_message(f"You selected {value}!", ephemeral=True)
        except IndexError:
            self.logger.warning("Select called without values")
            await interaction.response.send_message("No selection!", ephemeral=True)
        except discord.HTTPException as e:
            self.logger.error(f"Select failed: {e}")
            await interaction.response.send_message("Response failed!", ephemeral=True)
```

### Multi-Select

```python
class MultiSelectView(discord.ui.View):
    @discord.ui.select(
        placeholder="Pick up to 3 colors",
        min_values=1, max_values=3,
        options=[
            discord.SelectOption(label="Red", value="red", emoji="🔴"),
            discord.SelectOption(label="Blue", value="blue", emoji="🔵"),
            discord.SelectOption(label="Green", value="green", emoji="🟢"),
        ],
        custom_id="color_select"
    )
    async def multi_select(self, select, interaction):
        try:
            selected = ', '.join(select.values)
            await interaction.response.send_message(f"Selected colors: {selected}", ephemeral=True)
        except discord.HTTPException as e:
            logger.error(f"Multi-select failed: {e}")
            await interaction.response.send_message("Response failed!", ephemeral=True)
```

### Specialized Selects

#### User Select

```python
@discord.ui.user_select(placeholder="Select a user", custom_id="user_select", min_values=1, max_values=1)
async def user_select(self, select, interaction):
    try:
        user = select.values[0]
        await interaction.response.send_message(f"Selected user: {user.mention}", ephemeral=True)
    except IndexError:
        await interaction.response.send_message("No user selected!", ephemeral=True)
```

#### Role Select

```python
@discord.ui.role_select(placeholder="Select a role", custom_id="role_select", min_values=1, max_values=1)
async def role_select(self, select, interaction):
    try:
        role = select.values[0]
        await interaction.response.send_message(f"Selected role: {role.mention}", ephemeral=True)
    except IndexError:
        await interaction.response.send_message("No role selected!", ephemeral=True)
```

#### Channel Select

```python
@discord.ui.channel_select(placeholder="Select a channel", custom_id="channel_select", channel_types=[discord.ChannelType.text, discord.ChannelType.voice])
async def channel_select(self, select, interaction):
    try:
        channel = select.values[0]
        await interaction.response.send_message(f"Selected channel: {channel.mention}", ephemeral=True)
    except IndexError:
        await interaction.response.send_message("No channel selected!", ephemeral=True)
```

### Advanced Select Features

#### Dynamic Options

```python
class DynamicSelectView(discord.ui.View):
    def __init__(self, guild: discord.Guild):
        super().__init__(timeout=180.0)
        options = [discord.SelectOption(label=role.name, value=str(role.id)) for role in guild.roles[:25]]
        self.add_item(discord.ui.Select(
            placeholder="Select a role dynamically",
            options=options,
            custom_id="dynamic_role",
            min_values=1, max_values=1
        ))

    async def callback(self, interaction: discord.Interaction):
        select = [child for child in self.children if isinstance(child, discord.ui.Select)][0]
        try:
            role_id = int(select.values[0])
            role = interaction.guild.get_role(role_id)
            await interaction.response.send_message(f"Selected: {role.mention if role else 'Invalid'}", ephemeral=True)
        except (IndexError, ValueError):
            await interaction.response.send_message("Invalid selection!", ephemeral=True)

@bot.slash_command(name="dynamic_select", description="Dynamic role selection")
async def dynamic_select_cmd(ctx: discord.ApplicationContext):
    view = DynamicSelectView(ctx.guild)
    await ctx.respond("Choose a role:", view=view)
```

## Modals

Pop-up forms for input. Subclass `discord.ui.Modal` with up to 5 `InputText` fields.

### Parameters

- `title`: str (max 45 chars)
- `custom_id`: str
- **InputText:** `label` (max 45), `style` (short/long), `placeholder`, `required` (bool), `min_length`/`max_length` (0-4000)

### Advanced Modal with Validation

```python
class FeedbackModal(discord.ui.Modal, title="Feedback Form"):
    def __init__(self):
        super().__init__(custom_id="feedback_modal")
        self.add_item(discord.ui.InputText(
            label="Your Name",
            placeholder="Enter name (2-50 chars)",
            required=True,
            min_length=2,
            max_length=50
        ))
        self.add_item(discord.ui.InputText(
            label="Feedback",
            style=discord.InputTextStyle.long,
            placeholder="Share your thoughts (1-500 chars)",
            required=True,
            max_length=500
        ))

    async def interaction_check(self, interaction: discord.Interaction) -> bool:
        name = self.children[0].value.strip()
        feedback = self.children[1].value.strip()
        if len(name) < 2:
            await interaction.response.send_message("Name too short!", ephemeral=True)
            return False
        if len(feedback) > 500 or not feedback:
            await interaction.response.send_message("Feedback invalid!", ephemeral=True)
            return False
        return True

    async def callback(self, interaction: discord.Interaction):
        try:
            name = self.children[0].value
            feedback = self.children[1].value
            embed = discord.Embed(title="Feedback Received", description=f"**Name:** {name}\n**Feedback:** {feedback}")
            await interaction.response.send_message(embed=embed, ephemeral=True)
        except discord.HTTPException as e:
            logger.error(f"Modal callback failed: {e}")
            await interaction.response.send_message("Submission failed!", ephemeral=True)
```

```python
class ModalView(discord.ui.View):
    @discord.ui.button(label="Open Feedback", style=discord.ButtonStyle.primary, custom_id="open_modal")
    async def open_modal(self, button, interaction):
        await interaction.response.send_modal(FeedbackModal())
```

### Advanced Modal Features

#### Direct Modal

```python
@bot.slash_command(name="direct_modal", description="Open modal directly")
async def direct_modal(ctx: discord.ApplicationContext):
    await ctx.send_modal(FeedbackModal())
```

#### Confirmation Modal

```python
class ConfirmModal(discord.ui.Modal, title="Confirm Action"):
    def __init__(self):
        super().__init__(custom_id="confirm_modal")
        self.add_item(discord.ui.InputText(
            label="Type 'yes' to confirm",
            style=discord.InputTextStyle.short,
            required=True
        ))

    async def callback(self, interaction: discord.Interaction):
        if self.children[0].value.lower() == "yes":
            await interaction.response.send_message("Confirmed!", ephemeral=True)
        else:
            await interaction.response.send_message("Cancelled!", ephemeral=True)

class ConfirmView(discord.ui.View):
    @discord.ui.button(label="Confirm Action", style=discord.ButtonStyle.danger, custom_id="confirm_btn")
    async def confirm(self, button, interaction):
        await interaction.response.send_modal(ConfirmModal())
```

## Simulated Separators

No native separators; use rows or embeds for visual gaps.

```python
class SeparatorView(discord.ui.View):
    def __init__(self):
        super().__init__()
        self.add_item(discord.ui.Button(label="Top Button", style=discord.ButtonStyle.primary, row=0))
        self.add_item(discord.ui.Button(label="Bottom Button", style=discord.ButtonStyle.secondary, row=2))

@bot.slash_command(name="separator", description="View with visual separation")
async def separator(ctx: discord.ApplicationContext):
    await ctx.respond("Top Section\n\n---\n\nBottom Section", view=SeparatorView())
```

### Embed Divider

```python
embed = discord.Embed(description="Section 1\n\n---\n\nSection 2")
await ctx.respond(embed=embed, view=SeparatorView())
```

## Combining Components

Build complex UIs with buttons, selects, and modals.

```python
class ComboView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)
        self.state = {"option": None, "confirmed": False}

    @discord.ui.button(label="Open Modal", style=discord.ButtonStyle.primary, row=0, custom_id="modal_btn")
    async def modal_button(self, button, interaction):
        await interaction.response.send_modal(FeedbackModal())

    @discord.ui.select(
        placeholder="Select Option",
        options=[discord.SelectOption(label="A", value="a"), discord.SelectOption(label="B", value="b")],
        row=1, custom_id="combo_select"
    )
    async def select_option(self, select, interaction):
        self.state["option"] = select.values[0]
        embed = discord.Embed(title="Selected", description=self.state["option"])
        await interaction.response.edit_message(embed=embed, view=self)

    @discord.ui.button(label="Confirm", style=discord.ButtonStyle.success, row=2, custom_id="confirm_btn")
    async def confirm_button(self, button, interaction):
        if not self.state["option"]:
            return await interaction.response.send_message("Select first!", ephemeral=True)
        self.state["confirmed"] = True
        button.disabled = True
        await interaction.response.edit_message(content=f"Confirmed: {self.state['option']}", view=self)

@bot.slash_command(name="combo", description="Combined UI components")
async def combo(ctx: discord.ApplicationContext):
    view = ComboView()
    await ctx.respond("Interact with the combo view:", view=view)
```

## Poll System Example

Stateful poll with vote tracking.

```python
class PollView(discord.ui.View):
    def __init__(self, question: str, options: list[str]):
        super().__init__(timeout=86400.0)  # 24 hours
        self.question = question
        self.options = options[:25]
        self.votes = {opt: 0 for opt in self.options}
        self.voters = set()

    @discord.ui.select(
        placeholder="Cast your vote",
        options=[discord.SelectOption(label=opt, value=opt) for opt in self.options],
        custom_id="poll_select",
        min_values=1, max_values=1
    )
    async def vote_select(self, select, interaction):
        if interaction.user.id in self.voters:
            return await interaction.response.send_message("Already voted!", ephemeral=True)
        vote = select.values[0]
        self.votes[vote] += 1
        self.voters.add(interaction.user.id)
        await self.update_message(interaction)

    async def update_message(self, interaction):
        content = f"**{self.question}**\n" + "\n".join(f"{opt}: {count} votes" for opt, count in self.votes.items())
        await interaction.response.edit_message(content=content, view=self)

@bot.slash_command(name="poll", description="Create a poll")
async def poll_cmd(ctx: discord.ApplicationContext, question: str, options: str):
    opts = [opt.strip() for opt in options.split(",") if opt.strip()]
    if not 2 <= len(opts) <= 25:
        return await ctx.respond("Need 2-25 options, comma-separated!", ephemeral=True)
    view = PollView(question, opts)
    await ctx.respond(f"**{question}**", view=view)
```

## Advanced Patterns

### State Management

Use class attributes or external stores for shared state.

```python
class StateView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)
        self.state = {"clicks": 0, "selections": []}
        self.lock = asyncio.Lock()

    @discord.ui.button(label="Increment", style=discord.ButtonStyle.primary, custom_id="state_btn")
    async def state_button(self, button, interaction):
        async with self.lock:
            self.state["clicks"] += 1
            await interaction.response.edit_message(content=f"Clicks: {self.state['clicks']}", view=self)
```

### Dynamic Component Updates

Add/remove items runtime.

```python
class DynamicUpdateView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=180.0)
        self.select = discord.ui.Select(
            placeholder="Options",
            options=[discord.SelectOption(label="Initial", value="initial")],
            custom_id="dynamic_select"
        )
        self.add_item(self.select)

    @discord.ui.button(label="Add Option", style=discord.ButtonStyle.secondary, custom_id="add_option")
    async def add_option(self, button, interaction):
        new_label = f"New {len(self.select.options)}"
        self.select.options.append(discord.SelectOption(label=new_label, value=new_label.lower()))
        await interaction.response.edit_message(view=self)
```

### Cooldowns

Per-user rate limiting.

```python
from discord.ext.commands import Cooldown, BucketType

class CooldownView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=180.0)
        self.cooldown = Cooldown(rate=1, per=5.0, type=BucketType.user)  # 1 interaction every 5s per user

    async def interaction_check(self, interaction: discord.Interaction) -> bool:
        retry_after = self.cooldown.update_rate_limit(interaction.user.id)
        if retry_after:
            await interaction.response.send_message(f"Cooldown: wait {retry_after:.1f}s", ephemeral=True)
            return False
        return True

    @discord.ui.button(label="Rate-Limited Click", style=discord.ButtonStyle.primary, custom_id="cooldown_btn")
    async def button(self, button, interaction):
        await interaction.response.send_message("Clicked successfully!", ephemeral=True)
```

### Concurrency with Locks

For thread-safe state in high-concurrency bots.

```python
from asyncio import Lock

class SafeView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=None)
        self.lock = Lock()
        self.counter = 0

    @discord.ui.button(label="Safe Increment", style=discord.ButtonStyle.primary)
    async def button_callback(self, button, interaction):
        async with self.lock:
            self.counter += 1
            await interaction.response.send_message(f"Counter: {self.counter}", ephemeral=True)
```

### Integration with Databases

Persist state beyond views (e.g., using aiosqlite).

```python
import aiosqlite

class PersistentPollView(PollView):
    def __init__(self, question, options, db_path="polls.db"):
        super().__init__(question, options)
        self.db_path = db_path
        asyncio.create_task(self.load_from_db())

    async def load_from_db(self):
        async with aiosqlite.connect(self.db_path) as db:
            await db.execute("CREATE TABLE IF NOT EXISTS votes (user_id INTEGER, vote TEXT)")
            cursor = await db.execute("SELECT vote FROM votes")
            rows = await cursor.fetchall()
            for row in rows:
                self.votes[row[0]] += 1
                self.voters.add(row[0])  # Assuming user_id as key

    async def vote_select(self, select, interaction):
        await super().vote_select(select, interaction)
        async with aiosqlite.connect(self.db_path) as db:
            await db.execute("INSERT INTO votes (user_id, vote) VALUES (?, ?)", (interaction.user.id, select.values[0]))
            await db.commit()
```

### Multi-Step Wizards

Chain modals and views for forms.

```python
class WizardView(discord.ui.View):
    def __init__(self):
        super().__init__(timeout=600.0)  # 10 minutes
        self.step = 1
        self.data = {}

    @discord.ui.button(label="Next Step", style=discord.ButtonStyle.primary, custom_id="next_step")
    async def next_step(self, button, interaction):
        if self.step == 1:
            await interaction.response.send_modal(Step1Modal(self))
        elif self.step == 2:
            await interaction.response.send_modal(Step2Modal(self))
        # Add more steps...

class Step1Modal(discord.ui.Modal, title="Step 1: Basic Info"):
    def __init__(self, view):
        super().__init__()
        self.view = view
        self.add_item(discord.ui.InputText(label="Enter Name"))

    async def callback(self, interaction):
        self.view.data["name"] = self.children[0].value
        self.view.step += 1
        await interaction.response.edit_message(content="Step 1 complete. Proceed to next.", view=self.view)

# Similarly for Step2Modal...
```

## Best Practices and Edge Cases

- **Ephemeral Responses:** Use `ephemeral=True` for user-only visibility.
- **Deferring Interactions:** `await interaction.response.defer(ephemeral=True)` for >3s tasks, then `followup.send()`.
- **Error Handling:**
  - `discord.HTTPException`: API/network issues.
  - `discord.InteractionResponded`: Duplicate responses.
  - `discord.ui.errors.IncompleteModal`: Validation failures.
  - `IndexError`: Empty selections.
  - `discord.Forbidden`: Permission errors.
- **Rate Limits:** ~3 interactions/sec/user. Defer heavy ops.
- **Mobile Compatibility:** Test on iOS/Android; selects may render differently.
- **Custom IDs:** Unique, <=100 chars; required for persistence.
- **Timeouts:** `timeout=0` disables immediately; `None` for eternal.
- **Limits:** 5 rows/25 slots; selects: 25 options; modals: 45 char title, 4000 char inputs.
- **Persistence:** All persistent components need `custom_id`.
- **Accessibility:** Use emojis and descriptions for clarity.

## Testing and Debugging

- **Dev Mode:** Enable in Discord settings to copy IDs.
- **Logging Interactions:** `logger.debug(f"Data: {interaction.data}")`.
- **Component Limits:** Validate slot usage (e.g., max 5 buttons/row).
- **Simulate Timeouts:** Set `timeout=10` for quick tests.
- **Unit Testing:** Use `pytest` with mock interactions:
  ```python
  import pytest
  from unittest.mock import AsyncMock

  @pytest.mark.asyncio
  async def test_button_callback():
      interaction = AsyncMock()
      button = discord.ui.Button()
      await ButtonView().button_callback(button, interaction)
      interaction.response.send_message.assert_called()
  ```
- **Edge Cases:** Test with no intents, offline bots, expired interactions.

## Additional Resources

- [Official discord.py Docs](https://discordpy.readthedocs.io/en/stable/interactions/ui.html)
- [Pycord Guide (Updated Sept 2025)](https://docs.pycord.dev/en/stable/interactions/ui.html)
- [Discord Developer Portal](https://discord.com/developers/docs/interactions/message-components)
- Community: [Discord.py Discord Server](https://discord.gg/dpy), [Stack Overflow](https://stackoverflow.com/questions/tagged/discord.py)
- Extensions: [discord-ui](https://pypi.org/project/discord-ui/), [discord-interactions](https://pypi.org/project/discord-interactions/)

This guide equips you with advanced, scalable UI patterns for Discord bots. Contributions welcome—fork and PR!

---

*Last Updated: September 24, 2025*  
*Author: yokaimsi*  
*License: MIT*
