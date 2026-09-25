# Blueprint Quick Reference

This page lists the Blueprint events and nodes most games need, where to find them, and where to
put your logic. Every plugin node lives under an **RTS** category in the node menu, for example
**RTS > Combat** or **RTS > Production**. Categories are written below as `RTS|Combat`.

## Where your Blueprint logic goes

| You want to change | Put the logic in |
| --- | --- |
| One unit or building type | Its **Custom Blueprint**. In the Content Set, open the entry and choose **Create Custom Blueprint** in its **Blueprint** group. See [The Content Set](CONTENT_SET_GUIDE.md#add-blueprint-logic-to-a-unit-or-building). |
| The HUD | A Widget Blueprint that binds the events below, added through a child of **RTSHUD**. See [UI Data API](UI_DATA_API.md). |
| Win conditions and match rules | A child of your generated game mode, saved outside the generated folders. See [AI and Match Policies](AI_MATCH_POLICIES.md#blueprint-win-conditions). |
| AI strategy | A child of **RTSPlayerAIController**. See [AI and Match Policies](AI_MATCH_POLICIES.md). |
| A new command | A child of **RTSOrder**. See [Orders](ORDERS.md). |

Do not add logic to the generated Blueprints (`BP_<Id>` in the `Generated` folders).
**Generate Game** rebuilds them and replaces your changes.

## How generated units and buildings are built

A generated unit is a **Character**; a generated building is a **Pawn**. Their behavior comes
from RTS components. In your Custom Blueprint these components appear in the **Components** panel,
inherited from the generated parent. The ones you will use most:

| Component (name in generated Blueprints) | Class | Used for |
| --- | --- | --- |
| `HealthComponent` | `RTSHealthComponent` | Health, shields, damage and death. |
| `OwnerComponent` | `RTSOwnerComponent` | Which player owns the actor, and team checks. |
| `SelectableComponent` | `RTSSelectableComponent` | Selection and mouse hover. |
| `AttackComponent` | `RTSAttackComponent` | Weapons and kill count. Units and buildings with attacks. |
| `GathererComponent` | `RTSGathererComponent` | Gathering and carrying resources. Workers. |
| `BuilderComponent` | `RTSBuilderComponent` | Constructing buildings. Workers. |
| `ProductionComponent` | `RTSProductionComponent` | Training queues and rally points. Buildings that train units. |
| `ConstructionSiteComponent` | `RTSConstructionSiteComponent` | Construction progress. Every building. |
| `NameComponent`, `DescriptionComponent`, `PortraitComponent` | `RTSNameComponent`, `RTSDescriptionComponent`, `RTSPortraitComponent` | The display name, description and portrait shown in the HUD. |

To handle a component event: select the component in the **Components** panel, scroll the
**Details** panel to **Events**, and click **+** next to the event. Unreal adds an event node such
as **On Killed (HealthComponent)** to the Event Graph.

A new Custom Blueprint already contains two example events in its Event Graph, each inside a
comment and wired to a **Print String**: **On Killed (HealthComponent)**, and, for a building that
trains units, **On Production Finished (ProductionComponent)**. Replace the **Print String** with
your own logic, or delete the example nodes. **Print String** shows its message only in
development builds.

## Networking in one paragraph

The server decides everything that matters for gameplay: damage, deaths, resources, training and
the match result. Many events fire on the server *and* again on players' machines when the change
replicates. Use them freely for effects, sounds and UI. When your logic changes gameplay (grants
resources, damages units, ends the match), put a **Switch Has Authority** node first and continue
from its **Authority** pin. Nodes marked **Authority Only** do nothing on a client. Player commands
from a client go through the player controller's **Issue ...** nodes, which ask the server to act.

## Events

### A unit or building dies

| | |
| --- | --- |
| Event | **On Killed** on `RTSHealthComponent` (`RTS|Combat`) |
| Pins | **Actor** (the one that died), **Previous Owner** (the controller that owned it), **Damage Causer** |
| Fires on | The server, and on every machine that can see the actor when its death replicates |

**Example: refund minerals when a unit dies.** In the unit's Custom Blueprint:

1. Find the example **On Killed (HealthComponent)** event in the Event Graph and delete its
   **Print String**. (If you removed the example, select `HealthComponent` and, under **Events**,
   click **+** next to **On Killed**.)
2. Add **Switch Has Authority** and continue from **Authority**.
3. From **Previous Owner**, add **Get Component by Class** with **Component Class** set to
   `RTSPlayerResourcesComponent` (listed as *RTSPlayer Resources Component*). Every RTS player
   controller, human or AI, has one.
4. From its return value, add **Add Resources** (`RTS|Economy`). Set **Resource Type** to your
   minerals resource (for the installed starter game, `BP_Res_Minerals` in
   `/Game/RTSStarterGame/Generated/Resources`) and **Resource Amount** to 10.

For an effect instead, connect **Spawn System at Location** or **Play Sound at Location** directly
to the event, without the authority check.

Related: **On Health Changed** (**Actor**, **Old Health**, **New Health**, **Damage Causer**) fires
whenever health changes, and **On Shield Changed** for shields.

### A building finishes training a unit

| | |
| --- | --- |
| Event | **On Production Finished** on `RTSProductionComponent` (`RTS|Production`) |
| Pins | **Actor** (the building), **Product** (the new unit), **Queue Index** |
| Fires on | The server, and again on the owning player's machine. On that player's machine **Queue Index** is -1. |

**Example: react to a trained unit.** In the building's Custom Blueprint (for example
`BP_Barracks_Custom`):

1. Use the example **On Production Finished (ProductionComponent)** event in the Event Graph, or
   select `ProductionComponent` and, under **Events**, click **+** next to
   **On Production Finished**.
2. For gameplay changes, add **Switch Has Authority** and continue from **Authority**. **Product**
   is the unit that was just spawned; cast it to your unit's Custom Blueprint or use
   **Get Component by Class** on it.

Related events on the same component: **On Product Queued**, **On Production Started**,
**On Production Progress Changed**, **On Production Canceled**, **On Production Failed** and
**On Rally Point Changed**. For buildings under construction, `RTSConstructionSiteComponent` has
**On Construction Started**, **On Construction Finished** and **On Construction Canceled**.

### The player's selection changes

| | |
| --- | --- |
| Event | **On Selection Changed Event** on `RTSPlayerController` (`RTS|UI Data`) |
| Pins | **Selection** (array of actors) |
| Fires on | The local player's machine only |

In a Widget Blueprint: on **Event Construct**, call **Get Owning Player**, **Cast To
RTSPlayerController**, then **Bind Event to On Selection Changed Event**. Unbind it in
**Event Destruct**. To read the selection at any time, use **Get Selection** on the controller, then
**Get Selected Actors** (`RTS|Selection`).

For one unit or building, `RTSSelectableComponent` has **On Selected**, **On Deselected**,
**On Hovered** and **On Unhovered**.

### The player's resources change

| | |
| --- | --- |
| Event | **On Resources Changed** on `RTSPlayerResourcesComponent` (`RTS|Economy`) |
| Pins | **Resource Type**, **Old Resource Amount**, **New Resource Amount**, **Synced from Server** |
| Fires on | The server and the owning player's machine |

The resources component sits on the player controller. In a Widget Blueprint: **Get Owning Player**,
then **Get Component by Class** (`RTSPlayerResourcesComponent`), then
**Bind Event to On Resources Changed**. Read an amount with **Get Resources**, and the name, icon and
color of a resource with **Get Resource Name**, **Get Resource Icon** and **Get Resource Color**.
**Synced from Server** is true when the value is the server's correction of a local prediction.

### The match ends

| | |
| --- | --- |
| Event | **On Match Ended** on `RTSGameState` (`RTS|UI Data`) |
| Pins | **Winner** (an `RTSPlayerState`; empty for a draw) |
| Fires on | Every machine, once |

In a Widget Blueprint: **Get Game State**, **Cast To RTSGameState**, then
**Bind Event to On Match Ended**. Compare **Winner** with **Get Owning Player** > **Player State**
to show victory or defeat. **On Player Defeated** (**Defeated Player**) fires on every machine when a
player is eliminated. **Is Match Over** and **Get Winning Player** read the result later, and
**Get Match Time Seconds** reads the match clock.

On the server, a game mode child also receives **Event OnPlayerDefeated** and
**Event OnMatchEnded**.

### A unit receives an order

| | |
| --- | --- |
| Event | **On Current Order Changed** on `RTSPawnAIController` (`RTS|AI`) |
| Pins | **Actor** (the unit), **New Order** (an `RTSOrderData`) |
| Fires on | The server only (unit AI controllers exist only on the server) |

In the unit's Custom Blueprint: add **Event Possessed**, **Cast To RTSPawnAIController** on
**New Controller**, then **Bind Event to On Current Order Changed**. Use **Break RTSOrder Data**
on **New Order** to read **Order Class**, **Target Actor**, **Target Location** and **Queued**.

On the player's own machine, **On Local Order Confirmed** on `RTSPlayerController` fires each time
the player gives an order. It is meant for effects such as a click marker.

### Alerts for the HUD

The `RTSEventFeedSubsystem` (a local player subsystem) turns gameplay events into ready-made HUD
alerts. Get it with the **Get RTSEventFeedSubsystem** node (node menu category
**LocalPlayer Subsystems**) and bind **On Under Attack**, **On Unit Lost**,
**On Production Complete**, **On Research Complete**, **On Supply Blocked**,
**On Player Eliminated** or **On Ping**. Each passes an `RTSHudEvent` with a location, text and the
actor involved. Under-attack alerts are already throttled, so binding them does not spam the player.
See [UI Data API](UI_DATA_API.md#2-the-event-feed--urtseventfeedsubsystem-publicfeedback).

**On Error Occurred Event** on `RTSPlayerController` passes player-facing error text, such as
"not enough minerals".

## Useful nodes

### Give orders

These are on `RTSPlayerController` and act on the player's current selection. Each returns
**Request Submitted**: true means the request was sent to the server, not that it succeeded.

| Node | Category |
| --- | --- |
| **Issue Move Order**, **Issue Attack Order**, **Issue Gather Order**, **Issue Stop Order** | `RTS|Player` |
| **Issue Production Order**, **Issue Cancel Production Order** | `RTS|Player`, `RTS|Production` |
| **Issue Research Order**, **Issue Cancel Research Order** | `RTS|Research` |
| **Issue Ability Order** | `RTS|Ability` |
| **Surrender** | `RTS|Player` |

On the server, **Issue Order** (`RTS|Orders`, **Authority Only**) gives one actor an order directly.
Make its **Order** input with **Make RTSOrder Data**: set **Order Class** (for example
`RTSMoveOrder` or `RTSAttackOrder`), **Target Actor** or **Target Location**, and **Queued** to add
it after the current order.

### Read and change units

| Node | On | Category |
| --- | --- | --- |
| **Get Current Health**, **Get Maximum Health**, **Is Dead** | `RTSHealthComponent` | `RTS|Combat`, `RTS|Health` |
| **Set Current Health**, **Kill Actor** (Authority Only) | `RTSHealthComponent` | `RTS|Health` |
| **Get Player Owner**, **Is Same Team as Actor** | `RTSOwnerComponent` | `RTS|Core` |
| **Is Owned by Local Player** | library | `RTS|UI Data` |
| **Is Ready to Use** (false while a building is under construction) | library | `RTS|Gameplay` |
| **Get Name** | `RTSNameComponent` | `RTS|Core` |
| **Get Kills** | `RTSAttackComponent` | `RTS|UI Data` |

### Economy and production

| Node | On | Category |
| --- | --- | --- |
| **Get Resources**, **Can Pay Resources** | `RTSPlayerResourcesComponent` | `RTS|Economy` |
| **Add Resources**, **Pay Resources** (Authority Only) | `RTSPlayerResourcesComponent` | `RTS|Economy` |
| **Get Player Supply** | library | `RTS|Gameplay` |
| **Get Available Products**, **Get Queued Products**, **Get Progress Percentage**, **Is Producing** | `RTSProductionComponent` | `RTS|Production`, `RTS|UI Data` |

### Match

| Node | On | Category |
| --- | --- | --- |
| **Eliminate Player**, **Commit Match Result** (Authority Only) | `RTSGameMode` | `RTS|Match` |
| **Transfer Ownership** (Authority Only) | `RTSGameMode` | `RTS|Ownership` |
| **Is Match Over**, **Get Winning Player**, **Get Match Time Seconds** | `RTSGameState` | `RTS|Match`, `RTS|UI Data` |

Cheats for testing, such as `Money`, `NoFog`, `God` and `Victory`, are console commands (press the
backtick key during Play in Editor). They are not in the node menu of ordinary Blueprints; as nodes
they are available only inside a Blueprint child of `RTSCheatManager`.

## Learn from the examples

The plugin includes small Blueprint examples in `/RealTimeStrategy/Examples/Blueprint` (turn on
**Show Plugin Content** in the Content Browser settings): a custom order, an ability rule, an AI
strategy, an objective-based win condition, a HUD panel, and a complete resource, unit and building
made by hand. Duplicate one into your project before changing it. See
[Blueprint and C++ Examples](EXAMPLES.md).

For the rules every node follows (authority, request results, failures), see
[Blueprint API Rules](BLUEPRINT_API_CONTRACT.md).
