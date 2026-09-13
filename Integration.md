# CBP Runtime Integration

## Minimal graph editor setup

1. Prefer enabling replication on every Actor that owns an editable graph
   (`CBPRuntimeComponent` also enables it automatically on the server).
2. Add a `Custom Blueprint Runtime` component to that Actor.
3. Create a Widget Blueprint derived from `CBPGraphWidget`.
4. On the widget, call `Initialize Graph From Actor` and pass the graph-owning
   Actor.

For the complete built-in editor, derive the Widget Blueprint from
`CBPGraphEditorWidget` and call `Initialize Editor From Actor`. It already
contains a searchable Create panel, the graph canvas, a function-collapse
details panel, and a replicated variable details panel.

For a freely arranged editor, place these four independent widgets anywhere in
your own Widget Blueprint:

1. `CBPGraphWidget`
2. `CBPCreatePanel`
3. `CBPVariableDetailsPanel`
4. `CBPFunctionDetailsPanel` (native class name remains
   `CBPFunctionToolbarPanel` for Blueprint compatibility)

Mark them as variables and call `Initialize Independent Editor Panels From
Actor` once, passing the graph-owning Actor and all four references. The helper
initializes the graph first and binds the other panels to the same runtime
component, preventing inactive Create buttons caused by a missing graph target.
The function-panel parameter is optional.

Each panel exposes an `Appearance` struct for background, header, accent,
selection, destructive-action, text colors, padding, and item spacing. For a
complete UMG-authored skin, derive a Widget Blueprint from a panel, disable
`Use Built-in Layout`, build its WidgetTree, and call the panel's public
Blueprint actions from your controls. The convenience `CBPGraphEditorWidget`
also exposes child widget classes so custom panel subclasses can be used in its
prebuilt layout.

By default the runtime component enables replication on its owning Actor and
the server creates one replicated edit gateway on each PlayerController. A
manually added PlayerController/Pawn component is still supported but is no
longer required. Client edits made before the automatic gateway arrives are
queued and flushed in order.

The base graph widget already supplies the grid, pan/zoom, node rendering,
connection rendering, title-bar dragging, pin dragging, connection preview, and
right-click context events. An empty UMG designer hierarchy is expected
while `Use Built-in Graph Canvas` is enabled.

The variable panel owns variable creation through its visible type selector and
`+ ADD VARIABLE` button. The Create panel does not create variable definitions;
it watches the component's replicated variable list and adds typed `Get Name`
and `Set Name` entries automatically. Those entries create at the last pointer
or context-click position on the graph. Before any graph pointer input they
fall back to view center. A custom context menu can call `Set Node Spawn Graph
Position` before invoking `Create Built In Node`, `Create Node Asset`, `Create
Variable Reference`, or `Create Function Call`.

Custom classes derived from `CBPPinWidget` may override `Get Connection Anchor
Screen Position` when their visible socket is not located at the normal left or
right edge. The built-in pin layout targets the colored socket square
automatically.
The built-in layout also exposes socket-to-label spacing, label-to-editor
spacing, and default-editor width in Class Defaults.

For designer-time visuals, set `Preview Node Definitions` in Class Defaults.
These assets are preview-only and do not create runtime graph nodes.

## Shared graph style and interaction

Create `CBP Graph Style` from the Content Browser and assign it to the
`Graph Style` property of the `CBPGraphWidget`. It centralizes:

- finite canvas size, pan/zoom speed and zoom limits;
- background color or a Slate brush containing a texture/UI material;
- minor/major grid size and colors;
- marquee fill material, colors, and outline;
- straight, orthogonal, or spline wires and type-color overrides;
- moving dot/dash runtime flow, speed, spacing, and size;
- whether the palette uses all registered nodes, an explicit whitelist, or
  both.

The inline `Appearance` property remains a compatibility fallback when no
Graph Style asset is assigned.

Node-definition icons now render in both the definition preview and the runtime
node header. Enable `State Effect Slots` on a node definition to choose a
full-node selection overlay and running overlay. A full custom UMG node skin
can expose optional named slots called `SelectionEffectSlot` and
`RunningEffectSlot`; the built-in layout needs no manual slots.

Left-drag empty canvas space to marquee-select. Ctrl/Shift adds to the current
selection. Bind the single `On Graph Pointer Context` event. Its `Context` enum
distinguishes Canvas, Node, and Pin while the payload carries the corresponding
stable IDs and graph position. It fires once on a right click without drag;
left input and right-drag pan remain internal gestures.

`Use Built-in Node Action Menu` enables the three explicit menu functions but
does not open anything automatically. In the unified event, switch `Context`
and call `Open Create Node Menu`, `Open Node Context Menu`, or `Open Pin Context
Menu`. The Create popup matches the UE All Actions pattern with search,
collapsible categories, and a Context Sensitive checkbox. Right-button drag
continues to pan the canvas and emits no context event.

Bind `On Graph Menu Item Selected` to receive an
`FCBPNodeActionMenuSelection` after an entry is chosen. It reports the action
kind, display name, node type, node/pin IDs, variable/function source ID,
created node/function IDs, graph position, and success flag.

Clicking any part of a node, including a pin row or a custom child widget,
selects its owning node before the child handles the click. Dragging the title
of one selected node moves the complete selection without changing relative
spacing. Context-menu actions can call `Delete Selected Nodes`, `Disconnect
Selected Node Pins`, or `Move Selected Nodes By Offset` directly; they resolve
the current selection internally, so Blueprint menus do not need to copy or
loop over node IDs. `Select All Nodes` is also available. Pin right-click never
disconnects automatically: call `Disconnect Pin` with the event IDs or
`Disconnect Selected Node Pins` from your own menu action.

While dragging from a pin, right-click to fire `On Quick Create Node
Requested`. Build a filtered menu using the supplied type/direction, then pass
the selected definition to `Complete Quick Create Node`. The graph creates it
at the pointer position and, after the authoritative graph update arrives,
connects the first compatible opposite pin. Call `Cancel Quick Create Node`
when the menu closes without a choice.

## Node palette without string IDs

Call `Get Available Node Categories`, then `Get Available Node Definitions`.
Use the returned `CBPNodeDefinitionAsset` objects to build buttons or list
entries. When the user picks one, call `Create Node At View Center` or `Create
Node From Definition`.

When a Graph Style asset is assigned, these functions honor its `Available
Nodes` settings instead of always exposing the entire registry.

`Request Create Node` and `Create Node` remain as low-level compatibility APIs.
New UI should never ask a player or designer to type `math.add` or another node
type ID.

## Multiplayer editing

All graph state and variables live in the target `Custom Blueprint Runtime`
component and replicate with FastArray serialization. A client cannot normally
send an RPC through an arbitrary world Actor it does not own, so the component
on the player's owned PlayerController or Pawn acts as the edit gateway.

The persistent-state flow is:

`Graph widget -> target CBP component -> queued/owned CBP gateway -> server ->
target CBP component -> FastArray replication -> every relevant graph widget`

Create, delete, move, connect, disconnect, and pin-default operations all use
the same path. FastArray state replication is intentionally used instead of a
multicast RPC: late-joining and temporarily irrelevant clients receive the
complete current graph rather than only transient edit messages. Graph change
notification occurs after a received FastArray batch has fully applied, so a
delete never rebuilds the client ViewModel from the pre-removal array.

Variable definition edits, function collapse/create/delete, and function-call
creation/signature edits use the same owned gateway. Function bodies replicate through their
own FastArray, so late joiners receive the complete reusable subgraphs. Graph
execution and all lifecycle events remain authority-only; unreliable multicast
execution events are presentation/debug signals, not gameplay state.

### Collaboration presence and conflict warnings

`CBPGraphWidget` publishes its selected node IDs and current interaction mode
(editing, moving, or connecting) through a separate FastArray. This state is
ephemeral and is deliberately excluded from graph save files. Disconnected
players and deleted node IDs are pruned by the authority.

On each local graph widget:

1. Leave `Enable Collaboration Presence` enabled.
2. Call `Set Local Player Collaboration Profile` after initialization with the
   player's display name, avatar texture, and preferred color. Calling `Set
   Local Collaboration Profile` on the runtime component is equivalent.
3. If the avatar is empty, the widget renders a colored initial. An automatic
   deterministic color is used when color alpha is zero.
4. Bind `On Collaboration Conflict` if the game needs an additional toast,
   sound, or modal warning. The native node already shows an orange shared-node
   warning strip.

The variable details panel publishes the selected variable as ephemeral
presence and lists all players editing it. Player rows above graph nodes use a
negative visual layer, so their height never changes the node body's saved or
dragged graph coordinate.

The warning is advisory, not a distributed lock. The server still validates and
orders every mutation, and the last accepted mutation becomes authoritative.
Override `Can Remote Player Edit` for hard project rules such as roles, teams,
distance, or explicit ownership locks.

Multi-node dragging is sent as one `Move Nodes` request and one graph revision,
instead of one RPC/revision per node. Presence uses FastArray delta replication,
and graph widgets rebuild only nodes whose collaboration-state hash changed.

Override `Can Remote Player Edit` on the target component to enforce team,
distance, role, lock, or ownership rules. `Allow Remote Editing` is the simple
per-Actor switch. Graph limits are configured under:

`Project Settings > Plugins > Custom Blueprint Runtime > Graph Limits`

The legacy `AC_Player` networking facade is no longer required for graph data.
Its responsibilities can be reduced to creating/opening the editor widget and
choosing the target Actor. Remove the old create/move/connect/break/update-draw
and variable RPC chains after the new widget is initialized successfully.

## Lifecycle events

The Create panel exposes `Event Construction`, `Event Begin Play`, and `Event
Tick`. Only one node of each type is allowed per graph. Construction and Begin
Play run once on the authority when the component begins play. Tick's
unconnected `Interval Seconds` pin is editable inline: `0` runs every server
frame and positive values run at that interval. `Delta Seconds` reports the
time since that Tick node last fired.

## Functions

Collapsing is an explicit graph action, not a button owned by the function
details panel. Select nodes, then call `Collapse Selection To Function` on the
`CBPGraphWidget` from your context menu, shortcut, or own button. Pass the
desired function name. The new call node replaces the selection.

The function panel lists replicated functions. Selecting one exposes its name,
inputs, and outputs. Pin names can be edited inline and pins can be removed.
Choose a type and press `+ INPUT` or `+ OUTPUT` to expose the first unused,
type-compatible pin inside the stored function body. Existing call nodes are
updated by stable interface-pin ID and their compatible links are preserved.

## Saving and loading

Call `Save Graph To Slot` or `Load Graph From Slot` on the target
`CBPRuntimeComponent`; client calls are forwarded through the owned edit
gateway and executed on the server. Use a slot name such as `PlayerGraph` and a
user index such as `0`. The save includes
graph nodes and links, variable definitions/current values, function bodies,
and function signatures. Loading validates the snapshot before replacing live
state, then FastArray replication sends the restored result to clients.

`Does Graph Save Exist`, `Export Graph Save Data`, and `Import Graph Save Data`
are also available. Save existence and import are authority-only operations.

## Runtime modeling prototype

Get `CBPRuntimeModelingSubsystem` from the current World. It provides:

- `Create Box Model`
- `Import Obj Model`
- `Export Obj Model`
- `Load Obj Mesh Data`
- `Save Obj Mesh Data`

OBJ names are filenames, not arbitrary paths. All files live in
`Saved/CBPModels`. Imported models spawn as `CBPRuntimeModelActor`; their
Transform and small mesh payload are replicated. Geometry mutation functions
(`Set Vertex Position`, `Translate Vertices`, and `Scale Vertices`) are
server-authoritative.

This is deliberately a small-model prototype: 4,096 vertices and 8,192
triangles. A later production implementation should use chunked mesh streaming
or replicated edit operations rather than repeatedly replicating full large
meshes.
