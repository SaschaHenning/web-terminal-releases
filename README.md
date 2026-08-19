# WebTerminal — releases

Public distribution channel for **WebTerminal**, a passkey-protected web
terminal with persistent tmux sessions, packaged as an unsigned macOS menu-bar
app. The application source lives in a separate, private repository; this repo
holds only the built releases and their notes.

## Download

Grab the latest Apple Silicon build from
[Releases](https://github.com/SaschaHenning/web-terminal-releases/releases):

- `WebTerminal-<version>-arm64-mac.dmg` — open it and drag **WebTerminal** into
  **Applications**.

No GitHub account or token is needed — the releases are public.

First launch: macOS blocks an unsigned app. Right-click the app icon → **Open**,
then confirm in the dialog (or System Settings → Privacy & Security →
**Open Anyway**). This is a one-time step per installed build.

## Updates

The app checks this repository for newer versions on its own and installs them
in place — no token, no manual download. Each release also carries a
`WebTerminal-<version>-arm64-mac.zip`, which is the archive the in-app updater
downloads and swaps in.

## What is not here

No source code and no build pipeline — those stay in the private repository.
This repo is releases only.
