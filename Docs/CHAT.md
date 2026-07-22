# In-match chat (turnkey, Blueprint-only)

The plugin ships a complete, replicated player-chat system. **No C++ required to use or skin it.**

## What you get for free
- `URTSChatComponent` is **auto-created on `ARTSPlayerController`** — it's just there, one per player.
- Channels: **All**, **Team** (routes to the sender's team only), and **System** (server-only, e.g. "Red was defeated").
- Server-authoritative routing (client → Server RPC → team/all filter → Client RPC), length-clamped, schema-validated, and rate-limited against RPC abuse. Controls and repeated whitespace are canonicalized into one visual line, so a player cannot spoof extra chat rows. Clients cannot submit System messages. A capped local history survives the match.

## Wire a chat panel (3 Blueprint steps)
1. **Create a widget** — new Widget Blueprint, **parent class = `RTSChatWidget`**. Lay out your own visuals (a `ScrollBox` for lines, an `EditableTextBox` for input) with your fonts/colors.
2. **Show incoming lines** — implement the **`On Chat Message`** event (fires per line with `FRTSChatMessage`: `SenderName`, `Text`, `Channel`, `SenderColor`, `TimeSeconds`). Append a styled text block to your ScrollBox. Optionally call **`Replay History`** on Construct to show the backlog.
3. **Send** — on your input box's **OnTextCommitted (Enter)**, call **`Submit Chat`** (Text, Channel). Pick `Team` vs `All` from a toggle or a modifier key. The Boolean result reports local acceptance; clear the input only when it is true. Server rate limiting remains authoritative and asynchronous.

Add the widget to your HUD (e.g. `CreateWidget` → `AddToViewport`) and you have working chat.

## From gameplay Blueprints
- `Get Chat Component` (on the PlayerController) → **`Send Chat`** / **`Broadcast System Message`** (authority-only; e.g. call from your GameMode on a player defeat) / **`Get History`** / **`Clear History`**. Mutation nodes return whether the local operation was accepted.

## Skinning / extending
- Team color: `SenderColor` is neutral white by default (the core stays game-agnostic). Tint sender names by team in your `On Chat Message` event using your own team→color map (e.g. your `URTSHudStyle` palette).
- Input binding: bind an "Open Chat" key in your Input Mapping Context to focus your input box — standard Enhanced Input, no plugin change needed.
- Rate policy: tune `MessagesPerSecond` and `MessageBurstLimit` on the authoritative PlayerController's `RTSChatComponent` defaults. Production hard ceilings remain in force even if a derived Blueprint supplies extreme or non-finite values. Excess messages are dropped without disconnecting legitimate players.
- Whisper / more channels: extend `ERTSChatChannel` + the routing in `URTSChatComponent::RouteMessage` if you fork the plugin.
