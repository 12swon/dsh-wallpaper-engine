# dsh-wallpaper-engine

[English](README.md) | [中文](README.zh.md)

A DSH bundle that turns your **Wallpaper Engine** wallpapers into the **background of the DSH web GUI** (`dsh web`).

It discovers the Wallpaper Engine install on your machine, lists its wallpapers, and renders the *portable* ones (Video `.mp4` and Web/HTML) behind the DSH chat interface with an iOS-style **liquid glass** effect. You pick the wallpaper from a settings row, fine-tune it with four sliders, and pause/clear it anytime.

## Why only Video and Web wallpapers?

Wallpaper Engine wallpapers come in four types:

| Type | Rendered by | Portable to DSH? |
|---|---|---|
| **Scene** | Wallpaper Engine's own 3D engine | ❌ No — native 3D (`.obj`/shaders), only WE can render it |
| **Video** | a plain `.mp4` file | ✅ Yes — plays in a `<video>` tag |
| **Web** | a Chromium (`webwallpaper64.exe`) host for HTML | ✅ Yes — loads in an `<iframe>` |
| **Application** | an injected external window | ❌ No |

This is the same fundamental limit that applies to **mineradio** and every other
third-party Wallpaper Engine integration: only *Video* and *Web* wallpapers are
portable. Scene wallpapers still show up in the picker (shown as `[不可播放]`)
so you can see what you have, but they cannot be used as a live background here.

## How it works

- **Host half** (`lib/index.js`): a Cordis plugin that
  1. locates the Wallpaper Engine install by reading Steam's `libraryfolders.vdf`
     (so it works even when Steam is on a non-default drive),
  2. enumerates wallpapers from `projects/defaultprojects`, `projects/myprojects`,
     and `steamapps/workshop/content/431960/*`,
  3. registers same-origin HTTP routes on the DSH webserver so the browser half
     can fetch data and stream media directly:
     - `GET /wallpaper-engine/inventory` → JSON list of wallpapers
     - `GET /wallpaper-engine/media/<token>` → video / HTML (Range supported)
     - `GET /wallpaper-engine/preview/<token>` → preview image
- **Client half** (`lib/client.js`): a browser module that fetches the inventory
  and renders the selected wallpaper into a fixed layer *behind* the app columns,
  plus a "Wallpaper Engine" row in General settings with a picker.

## Install

```sh
# from a local checkout of this bundle
dsh plugin --profile web add file:<absolute-path-to>/dsh-wallpaper-engine

# (or, once published)
dsh plugin --profile web add dsh-wallpaper-engine
```

Then restart `dsh web`. The host plugin becomes a bundle layer and the client
plugin auto-loads (`dsh.client.immediately: true`).

If your machine has Steam installed in a non-standard location, the host auto-detects
via `libraryfolders.vdf`. Nothing further is required.

## Usage

1. Open `dsh web` → the DSH GUI.
2. Open **Settings → General** and find the **Wallpaper Engine** row.
3. Pick a Video or Web wallpaper from the dropdown. It appears behind the app.
4. Use **暂停/播放** to pause a video wallpaper, and **关闭** to clear it.
   The choice is remembered in your browser's `localStorage` (key
   `dsh-wallpaper-engine:selection`).

### The four sliders

While a wallpaper is active, four sliders let you tune how it blends with the UI:

| Slider | What it controls | Range | Default |
|---|---|---|---|
| **壁纸模糊** (wallpaper blur) | Blurs the wallpaper itself | 0–60 px | 0 |
| **暗化** (scrim) | Darkens the overlay between wallpaper and text | 0–90 % | 25 % |
| **边框** (border) | Raises border/divider contrast | 0–90 % | 35 % |
| **玻璃** (glass) | Blur radius of the frosted-glass panels (composer, bubbles) | 0–40 px | 24 |

> **Light vs. dark mode** — Wallpapers differ wildly in colour and brightness, so
> there is no one mode that fits every wallpaper. Switch DSH's theme between
> **light** and **dark** to find which suits the current wallpaper. If text or
> hairlines become hard to read on a bright or busy wallpaper, raise the
> **暗化 / 边框** sliders (and optionally add a little **壁纸模糊**) until it is
> comfortable. All four sliders apply instantly — no page refresh needed.

## Configuration

There is no model-visible tool or prompt text. The bundle adds zero tokens to the
agent. All state is process-local/browser-local; no durable DSH settings are written.

## Limitations

- Scene (native 3D) and Application wallpapers cannot be embedded; they appear as
  `[不可播放]` in the picker. Their live render remains Wallpaper Engine's desktop job.
- The browser must be able to autoplay muted `<video>` (DSH runs on loopback; muted
  autoplay is allowed by modern browsers).
- Media is served from your local Wallpaper Engine install paths; the host only
  serves files it has already enumerated (no arbitrary filesystem exposure).
- The picker is English/Chinese mixed (this bundle is not yet wired into DSH's
  locale namespaces).

## Development / rebuild

The host half (`lib/index.js`) is plain ESM with no build step. The client half
(`lib/client.js`) is a **compiled artifact** produced from the canonical source
`src/client.js` by `scripts/build-client.mjs`, which emits the exact
`window.__ModuleLoader__.load({ id, factory })` envelope the DSH module loader
consumes (the same shape `tsdown` emits for in-box client packages).

```sh
npm run build      # regenerate lib/client.js from src/client.js
npm run verify     # materialize the emitted bundle and assert its exports
```

Edit `src/client.js`, then `npm run build`. Do not hand-edit `lib/client.js`.
`npm install`/`pnpm install` runs `prepare` → `build` automatically, so a
fresh checkout always ships a current `lib/client.js`.

The host↔browser contract is plain same-origin HTTP, so the two halves are
developed independently: rebuild the host by restarting `dsh web`, and rebuild
the client with `npm run build` before re-running `dsh web`.

