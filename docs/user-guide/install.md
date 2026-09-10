# Installing the InterruptingCow App

> **Status: design phase.** The app doesn't exist yet. This describes how it will
> work so the design can be reviewed. There is nothing to install today.

InterruptingCow's client is a **Progressive Web App (PWA)**. You open it in a
browser and "install" it to your device, after which it runs in its own window
like any other app — no app store, no separate download.

Your operator (the person running the InterruptingCow server) will give you:

- a **web address** (something like `https://icow.example.net`),
- an **account** (a handle and a password) — there is no self-service signup.

## iPhone / iPad (iOS / iPadOS 16.4 or later)

1. Open the web address in **Safari** (it must be Safari for installation).
2. Sign in with your handle and password.
3. Tap the **Share** button (the square with an arrow).
4. Tap **Add to Home Screen**, then **Add**.
5. Launch InterruptingCow from the new home-screen icon.

**Why the home screen matters on iOS:** Apple only allows a web app to receive
**push notifications** when it's been added to the Home Screen and launched from
there — not from a Safari tab. So even if you'd normally just bookmark it, add it
to the Home Screen if you want to be notified of pages and tells later.

## Android

1. Open the web address in **Chrome** (or another browser that supports PWA
   installation).
2. Sign in.
3. You'll see an **Install app** prompt, or choose **Install app** / **Add to
   Home screen** from the browser menu (⋮).
4. Confirm. Launch it from your app drawer.

## Desktop (Chrome, Edge, and other Chromium browsers)

1. Open the web address and sign in.
2. Click the **install icon** in the address bar (a small screen-with-arrow
   icon), or choose **Install InterruptingCow…** from the browser menu.
3. It opens in its own window and appears alongside your other applications.

Firefox and Safari on desktop can use InterruptingCow in a normal tab; dedicated
"install" support varies by version.

## Signing in

- You sign in **once** per device. The app keeps you signed in afterward
  (it stores only a login token, nothing else about you locally).
- Your worlds, your scrollback, and your settings all live on the server, so they
  follow you to every device you sign in on.
- If you sign out, or your session expires, you'll sign in again with your handle
  and password.

## Staying connected

The whole point of InterruptingCow: **closing the app does not disconnect you
from your worlds.** The server holds those connections. When you reopen the app —
on the same device or another one — you get the lines you missed and the live
conversation resumes.

The exception is if a **game server** disconnects you (a reboot, an idle kick, a
ban). InterruptingCow will show that world as **disconnected** and wait for you
to reconnect it; it doesn't automatically reconnect in that case, on purpose.
See [Managing worlds](worlds.md).

## One thing to know about shared instances

If your operator runs InterruptingCow for more than one person, every user's game
connections leave the server from **the same IP address**. Games that limit alts
by IP, or that ban by IP, will apply those limits and bans to your whole instance
collectively. If that matters for a game you play, read
[Everyone on this instance shares one IP address](worlds.md#before-you-start-everyone-on-this-instance-shares-one-ip-address)
before you depend on InterruptingCow for it.

## Getting the source

InterruptingCow is [free software](../../LICENSE) (AGPL-3.0-or-later). The app has
a **Source** link in its settings that takes you to the exact source code the
server is running, including any changes your operator has made. You're entitled
to that by the license.

## Troubleshooting

- **"Add to Home Screen" is missing on iOS** — you're not in Safari, or not on a
  regular tab (e.g. you're in a private window or another browser).
- **No install prompt on Android/desktop** — some browsers only offer it after a
  few seconds or a second visit; use the browser menu instead.
- **It won't load / "not secure"** — the server must be reached over `https://`.
  Check the address with your operator.
- **Signed out unexpectedly** — sign in again; if it keeps happening, tell your
  operator (their token settings or a clock problem can cause it).
