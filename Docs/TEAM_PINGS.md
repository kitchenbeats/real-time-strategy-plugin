# Multiplayer team pings

The plugin ships a complete server-routed map-ping path. In the bundled minimap, **Alt + click**
sends a ping to the sender's team and to observers. A blank-project RTS gets multiplayer pings
without adding a Blueprint graph.

## Blueprint workflow

Call **Issue Allied Ping** on `ARTSPlayerController` with a world location and label. The controller's
auto-created `RTSPingComponent` submits the request to authority, which validates and rate-limits it
before routing it to eligible owning clients.

To react outside the bundled HUD, choose either supported event surface:

- **Get Ping Component** → bind **On Player Ping Received** for the sender, world location, and text.
- **Get Local Player Subsystem** (`RTSEventFeedSubsystem`) → bind **On Ping** to use the same typed
  `FRTSHudEvent` stream as toasts and minimap effects. `Source` is the sending `ARTSPlayerState`.

`RTSEventFeedSubsystem::EmitPing` is intentionally a local presentation injection. Use it for
single-client markers, tutorial guidance, and QA. Use **Issue Allied Ping** for multiplayer traffic.

## Routing and extension

The authority evaluates two `ARTSPlayerController` Blueprint-native policies:

- **Can Send Player Ping** accepts active players with a team by default. Observers cannot send
  strategic pings unless the project explicitly overrides this policy.
- **Can Receive Player Ping** accepts the sender's teammates and observers by default.

Override these functions in a PlayerController Blueprint for alliances, parties, coaches, or a
custom spectator policy. In C++, override `CanSendPlayerPing_Implementation` and
`CanReceivePlayerPing_Implementation`. Routing still stays server-authoritative.

## Abuse protection

Every inbound request has a hard 256-character transport cap, finite bounded coordinates,
single-line control/whitespace sanitization, and a per-player token bucket. Tune
`MaxPingTextLength`, `PingsPerSecond`, and `PingBurstLimit` on the authoritative controller's
`RTSPingComponent` defaults. Production hard ceilings remain in force even if a derived Blueprint
supplies extreme or non-finite values. Requests beyond the token budget are dropped without
disconnecting a legitimate player; malformed transport payloads fail RPC validation.
