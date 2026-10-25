# WebTerminal — releases

Public distribution channel for **WebTerminal**, a passkey-protected web
terminal with persistent tmux sessions, packaged as a signed and notarized
macOS menu-bar app. The application source lives in a separate, private repository; this repo
holds only the built releases and their notes.

## Download

Grab the latest Apple Silicon build from
[Releases](https://github.com/SaschaHenning/web-terminal-releases/releases):

- `WebTerminal-<version>-arm64-mac.dmg` — open it and drag **WebTerminal** into
  **Applications**.

No GitHub account or token is needed — the releases are public.

First launch: the app is signed with a Developer ID and notarized by Apple, so
macOS opens it directly.

Setting up several devices, your own hub and paired devices: see
[INSTALL.md](INSTALL.md).

## Updates

The app checks this repository for newer versions on its own and installs them
in place — no token, no manual download. Each release also carries a
`WebTerminal-<version>-arm64-mac.zip`, which is the archive the in-app updater
downloads and swaps in.

## What is not here

No source code and no build pipeline — those stay in the private repository.
This repo is releases only.
