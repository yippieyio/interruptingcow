# Notifications

> **Status: planned — not built.** This describes how notifications are intended
> to work. They depend on the **triggers** feature, which comes first. Nothing
> here is available yet. See the [roadmap](../../ROADMAP.md).

The goal: your phone buzzes when someone **pages** you or sends you a **tell** —
but stays quiet for room chatter, weather, and movement.

## Why notifications need triggers first

InterruptingCow could, in principle, notify you on every line from a world. That
would be useless — a busy room produces hundreds of lines an hour. So
notifications are **driven by triggers**: rules you set for what actually matters
to you.

A **trigger** matches lines (for example, lines containing "pages:" directed at
your character) and can do several things — highlight the line, hide it, send a
command in response, or **notify** you. Only lines that match a "notify" trigger
produce a notification.

Triggers are a feature in their own right and arrive before notifications. Once
they exist you'll be able to, for instance:

- notify me when a line matches `<my name> pages:` or `You sense that <my name>`
- notify me when a specific friend connects
- **don't** notify me for anything in a particular busy channel

## How delivery will work

InterruptingCow will use **Web Push**, the browser standard for notifications to
an installed web app. When it's built:

- You'll grant notification permission once, from the app's settings.
- The InterruptingCow server sends the notification to your device even when the
  app is closed.
- Tapping it opens the app to the relevant world.

## Platform notes

- **iPhone / iPad:** the app **must be added to the Home Screen** and opened from
  there — iOS does not deliver Web Push to a web app running in a Safari tab.
  Requires iOS/iPadOS 16.4 or later. See [Installing the app](install.md).
- **Android / desktop:** Web Push works from the installed app normally.
- **European Union:** Apple currently does not offer Web Push for home-screen web
  apps in the EU. This is an Apple platform restriction, not an InterruptingCow
  limitation, and affects iOS only.

## Controlling the noise

When notifications exist you'll be able to:

- enable or disable them per world,
- set quiet hours,
- tune which triggers are allowed to notify,

so the buzz means "someone's talking to me" and nothing else.
