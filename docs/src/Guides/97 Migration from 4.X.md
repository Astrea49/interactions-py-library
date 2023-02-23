# Migrating from interactions.py 4.X

Version 5.X and beyond can be considered a major rewrite of the library. It is essentially a new library - a library that still focuses on interactions and Discord bot making, but with major changes to allow for more stability and flexibilty.

Basically: **you *will* have to update, and likely rewrite, your bot in some fashion.** The changes are too massive to get away with changing only a couple of imports, and you should be prepared to analyze what your code is doing and rewrite it to modern standards.

**This guide does not cover everything.** That simply is not possible - again, v5 is basically a new library, and it's virtually impossible to cover every aspect of two different libraries and how they're different. Read this guide, read on the examples and documentation, and don't be afraid to ask for help in the support server if you need to.

Now, let's get started, shall we?

## Python Version Change

v5 and beyond *requires* Python 3.10 or higher, as compared to v4's 3.8+. This is to necessitate a lot of features that have been introduced since 3.8, including match statements and union types.

For Windows users, this is usually as simple as downloading 3.10+ (ideally the latest version for the most speed and features) and possibly removing the old version if you have no other projects that depend on older versions.

TODO: Any Mac users that can tell me how to upgrade?

For Linux users, this may be more complicated. You almost *never* want to uninstall the current Python you have and replace it wih a new one - you usually want to add a new one and keep the old one, to ensure system stability (lots of Linux tools depend on Python).
We suggest using [pyenv](https://github.com/pyenv/pyenv) for easy management between multiple Python versions, use Deadsnakes' PPA for Ubuntu devices (you should do a search to find information on how to install the latest version, though [this guide can install 3.11 on Ubuntu 20.04/22.04](https://tecadmin.net/how-to-install-python-3-11-on-ubuntu-22-04/)), build Python from source, or research other, *safe* ways of getting it.

*Please be careful when doing this.* Especially on Linux, it is easy to mess something up. Consult trusted guides if necessary.

## Slash Commands

Slash commands function differently from v4's commands - it's worth taking a good look at the guide to see how they work in the library now (TODO: link to it, I'm on mobile lol).

Big changes include the fact that `@bot.command` (we'll get to extensions later) is now `@interactions.slash_command`, and `CommandContext` is now `InteractionContext`. There may be some slight renamings elsewhere too in the decorators itself - it's suggested you look over the options for the new decorator and approiately adapt your code.

Arguably the biggest change involves how v5 handles slash options. v5's primary method relies heavily on decorators to do the heavy lifting, though there are other methods you may prefer - again, consult the guide, as that will tell you every method. A general rule of thumb is that if you did not use the "define options as a list right in the slash command decorator" choice, you will have to make some changes to adjust to the new codebase.
Subcommands also cannot be defined as an option in a command. We encourage you to use a subcommand decorator instead, as seen in the guide.

If you were using some of the more complex features of slash commands in v4, it's important to note: *v5 only runs the subcommand, not the base-command-then-subcommands that you could do with v4.* This was mostly due to the logic being too complex to maintain - it is encouraged that you use checks to either add onto base commands or the subcommands you want to add them to, as will be talked about in an upcoming section. `StopIteration` also doesn't exist in v5 due to this change.

Autocomplete *is* different. v5 encourages you to tie autocompletes to specific commands in a different manner than v4, like seen in the guide (TODO: link). There is `interactions.global_autocomplete` too.

Autodeferring is also pretty similar, although there's more control, with options to allow for global autodefers and extension-wide ones.

## Other Types of Interactions (Context Menus, Components, Modals)

These should be a lot more familiar to you - many interactions in v5 that aren't slash commands are similar to v4, minus name changes (largely to the decorators and classes you use). They should still *function* similarly though, but it's never a bad idea to consult the various guides that are on the sidebar to gain a better picture of how they work.

You can use `ActionRow` directly instead of using `ActionRow.new()` now.

# WIP - these next sections are not in order of their final appearance

## Extensions

Honestly, this is mostly the same to most people. `await teardown(...)` is now just `drop(...)` (note how drop is *not* async), and you use `bot.load_extension`/`bot.unload_extension` instead of `bot.load`/`bot.unload`.

One *major* difference though that isn't fully related to extensions themselves - *you use the same decorator for both commands/events in your main file and commands/events in extensions in v5.* Basically, instead of having `bot.command` and `interactions.extension_command`, you *just* have `interactions.slash_command` (and so on for context menus, events, etc.), which functions seemlessly in both contexts.

## Cache and interactions.get

interactions.get is a mistake
todo because im running out of battery—

## asyncio Changes

In recent Python versions, `asyncio` has gone through a major change on how it treats its "loops," the major thing that controls asynchronous programming. Instead of allowing libraries to create and manage their own loops, `asyncio` now encourages (and soon will enforce) users to use one loop managed by `asyncio` itself.

What this means to you is that *the `Client` does not have a loop variable, and no `asyncio` loop exists until the bot is started (if you use `bot.start()`).*

For accessing the loop itself, there is [`asyncio.get_running_loop()`](https://docs.python.org/3/library/asyncio-eventloop.html#asyncio.get_running_loop) to, well, get the running loop, though you're probably using the loop to run a task - it's better to use [`asyncio.create_task(...)`](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task) instead. 

However, as for the second poont... it shouldn't impact most users, but this may if you use `create_task` to run an asynchronous function before the bot starts - *this including loading in an extension that uses it before the bot is properly started.* Both of the above functions will error out if used, so using them isn't an option.

So what do you do? Simple - create the loop "yourself" and use `bot.astart()` instead!

Before:
```python
import interactions

# if there's no loop detected, v4 would create the loop for you at this point
# it also stores the loop in bot._loop
bot = interactions.Client(...)

bot._loop.create_task(some_func())
bot.load("an_ext_that_uses_the_event_loop")

bot.start()
```

After:
```python
import asyncio
import interactions

# no bot._loop, loop also does not exist yet
bot = interactions.Client(...)

async def main():
    # loop now exists, woo!
    asyncio.create_task(some_func())
    bot.load_extension("an_ext_that_uses_the_event_loop")
    await bot.astart()

# a function in asyncio that creates the loop for you and runs
# the function within
asyncio.run(main())
```

It's worth noting that you can continue to use `bot.start()` and not change your code if you never relied on `asyncio` like this.
