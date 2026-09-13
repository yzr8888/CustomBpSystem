# Custom Blueprint Runtime

Standalone UE 5.3-5.7 runtime plugin for versioned node schemas and multiple execution
backends. It does not depend on ScriptRunner, NodeToCode, or another runtime plugin.

## Implemented runtime

- Typed `FCBPValue` values.
- Versioned `FCBPNodeDefinition` and stable semantic pin IDs.
- Schema validation and deterministic schema hashes.
- JSON developer manifest registry.
- Cooked `UCBPNodeDefinitionAsset` registry with automatic asset discovery.
- Per-invocation `UCBPNodeExecutor` Blueprint classes selected by each node
  definition asset.
- Replicated graph and variable storage using Fast Array serialization.
- Replicated collaboration presence using Fast Array deltas: customizable
  player name/color/avatar, selected nodes, edited variables, and
  editing/moving/connecting state.
- Bright selected-node outlines, per-node player badges, and advisory warnings
  when multiple players select or manipulate the same node.
- Server-authoritative graph editing through a player-owned CBP component
  gateway, including permission hooks and configurable graph limits.
- Token-based execution scheduler with per-tick and per-run step limits.
- Recursive pure-node evaluation. Pure results are cached only for the current
  impure execution token, while impure outputs live in the execution context.
- Execution events for start, node start/completion, debug output, failure,
  finish, and cancellation.
- Shared execution contract for Native, TrustedExternal, PlayerVM, and Subgraph backends.
- Replicated reusable functions created by collapsing a node selection. Calls
  preserve execution and data boundaries and expand into an authority-side
  execution snapshot.
- Server-authoritative `Event Construction`, `Event Begin Play`, and `Event
  Tick` entry nodes. Tick owns an editable interval and exposes trigger delta.
- Profile-based external process subsystem with:
  - server/standalone enforcement;
  - Shipping opt-in per profile;
  - separate stdout and stderr pipes;
  - continuous non-blocking pipe draining;
  - timeout, cancellation, concurrency, and output limits;
  - executable paths selected only by developer settings.
- Explicit compiler and VM boundaries for a future player-safe bytecode language.

The native runtime currently includes:

- `core.start`
- `event.construction`, `event.begin_play`, and `event.tick`
- `variable.get` and `variable.set`
- `debug.print`
- `math.add`, `math.subtract`, `math.multiply`, and `math.divide`
- `vector.split` and `vector.merge`
- `world.get_player_location`
- `world.spawn_actor`
- `flow.parallel`

All built-in types are represented by two editable plugin assets:

- `Content/Nodes/Definitions/DA_CBPNode_*`
- `Content/Nodes/Executors/BP_CBPExec_*`

The DataAsset owns identity, presentation, pin schemas, permissions, purity, and
the executor class reference. Each executor Blueprint is a separate class. The
built-in executor Blueprints inherit tested native behavior; developer nodes may
override `Execute Node` directly in Blueprint.

Executor Blueprint helpers:

- `Get Input Value`
- `Complete`
- `Fail`

The execution context provides the runtime component, owning Actor, node
definition asset, stable IDs, custom name, and resolved input map.

The core execution behavior that previously lived in `AC_CBP` is implemented in
C++. The graph widget and component now replace the legacy `AC_Player` graph
RPC facade; existing save-game structs still need an explicit versioned importer.

## Blueprint use

1. Add `Custom Blueprint Runtime` to an Actor.
2. Select a `CBPNodeDefinitionAsset` and call `Request Create Node From
   Definition`.
3. Read `Get Nodes` to obtain generated node and pin GUIDs.
4. Set unconnected input values with `Request Set Pin Default`.
5. Join an output pin to an input pin with `Request Connect Pins`.
6. Call `Request Start Execution`.
7. Bind `On Execution Event` for status and debug messages.

`Get Nodes`, `Get Variables`, `On Graph Changed`, and `On Variables Changed` are
the intended bridge for a runtime editor UI.

For a new developer-authored node:

1. Create a Blueprint class derived from `CBPNodeExecutor`.
2. In Class Settings > Overrides, implement the `Execute Node` **function**.
   It has one `Context` input and an `FCBPNodeExecutionResult` return value.
3. Read values with `Get Input Value`. Its `PinId` must be the stable pin ID
   from the definition asset, not the localized display name.
4. On success, create an output map whose keys are output pin IDs, call
   `Complete(Context, Outputs, ExecutionOutputs)`, and connect its result to the
   function Return Node. Pure nodes normally pass an empty execution-output
   array. Impure nodes pass the execution pin IDs to continue, for example
   `then`, `completed`, or `failed`.
5. On failure, connect `Fail(Context, ErrorMessage)` to the Return Node instead.
6. Create a `CBPNodeDefinitionAsset`.
7. Give it a globally stable `NodeTypeId`, define its pins, and select the
   executor Blueprint class.
8. Reload the registry or restart PIE. The node becomes available without a
   scheduler code change.

Conceptually, an Add executor is:

`Execute Node(Context) -> Get Input Value("a"/"b") -> calculate -> Outputs["result"] -> Complete -> Return Value`

`Execute Node` used to expose an output reference named `Result`. It now returns
the result structure directly. If an executor Blueprint was created against the
old API, delete its red `Event ExecuteNode` node, compile/refresh the Blueprint,
then add `Execute Node` again from Class Settings > Overrides. Connect the output
of `Complete` or `Fail` to the override function's Return Value pin.

For variable nodes, use the node custom name as the variable name, or provide a
non-empty value on the node's `Name` input pin.

In multiplayer, mutations execute on the server and graph/variable state
replicates to clients. Add a second CBP component to the locally owned
PlayerController or Pawn; it automatically acts as the command gateway when the
target graph Actor is not client-owned. Override `Can Remote Player Edit` on the
target component for project-specific authorization.

The graph widget also publishes local selection and interaction presence. Call
`Set Local Player Collaboration Profile` once for each local player to set the
replicated display name, avatar, and color. Other clients see the player row above the selected node. A shared
node displays a warning strip and fires `On Collaboration Conflict` when a
remote player is already editing, moving, or connecting it. This is an advisory
warning rather than a hard lock; accepted graph mutations remain
server-authoritative.

See [Docs/Integration.md](Docs/Integration.md) for the minimal Blueprint setup
and migration from the legacy `AC_Player` RPC chains.

User guides with screenshots:

- [中文使用指南](Docs/UserGuide.zh-CN.md)
- [English User Guide](Docs/UserGuide.en.md)

## Runtime model

An execution begins at the first `core.start` node. Every connected execution
output schedules the downstream node. Before an impure node runs, each data input
resolves its connected source:

- a pure source is recursively evaluated from its earliest pure dependency;
- an impure source reads the value saved in the current execution context;
- an unconnected input uses its graph default.

This preserves the important behavior of the original Blueprint implementation
without storing temporary runtime values inside replicated editable pins.

## UI foundation

The plugin exposes identity-preserving ViewModels:

- `UCBPGraphViewModel`
- `UCBPNodeViewModel`
- `UCBPPinViewModel`

and concrete runtime widgets:

- `UCBPGraphWidget`
- `UCBPNodeWidget`
- `UCBPPinWidget`
- `UCBPCreatePanel`
- `UCBPVariableDetailsPanel`
- `UCBPFunctionToolbarPanel`
- `UCBPGraphToolbarPanel`
- `UCBPGraphEditorWidget`

`UCBPGraphEditorWidget` is a ready-to-use UE-style multi-panel editor. Its
left palette creates DataAsset nodes, the three built-in events, replicated
variable Get/Set references, and calls to stored functions. The independent
function panel lists functions and edits names plus input/output signatures.
The variable panel creates/edits variable names, types, and default values.
Its top toolbar provides `RUN`, `STOP`, `SAVE`, and `LOAD` actions. Run validates the
interpreted graph and starts the server-authoritative Construction, Begin Play,
then Tick session. Stop cancels the session, clears runtime visuals, and restores
the pre-run variable snapshot. Save uses the server-authoritative component path.

For custom layouts, place `CBP Graph Toolbar Panel`, `CBP Graph Widget`, `CBP
Create Panel`, `CBP Variable Details Panel`, and `CBP Function Details Panel`
independently in any UMG containers. Initialize the editor panels normally,
then call `Initialize Toolbar` with the initialized graph widget. Call
`Initialize Independent Editor Panels From Actor` once with
those references and the graph-owning Actor. Every panel exposes an
`Appearance` struct. Disable `Use Built-in Layout` in a Blueprint subclass only
when replacing the complete panel WidgetTree; its public create/apply/collapse
functions remain available to custom controls. The native function-panel class
keeps the old `UCBPFunctionToolbarPanel` name for Blueprint compatibility.

The variable panel has a prominent `+ ADD VARIABLE` action and type selector.
The Create panel observes replicated variables and automatically contributes a
typed `Get <Name>` and `Set <Name>` entry for each one. Variable references use
stable IDs, follow variable renames/type changes, and are removed from the main
graph when their variable is deleted.

`UCBPGraphWidget` can be created and used directly. Call
`InitializeGraphFromActor`; it creates the grid canvas, nodes, pins, and
connection layer automatically. Mouse wheel zooms around the cursor, right or
middle mouse drag pans the canvas, dragging a title moves a node, and dragging a
pin previews and creates a connection. A right click without drag emits only
`On Graph Pointer Context`; a right drag only pans and emits nothing. The plugin
never opens a menu automatically. Switch on the event's `Context` enum and call
`Open Create Node Menu`, `Open Node Context Menu`, or `Open Pin Context Menu`.
`ResetView` returns to 100% zoom.

Bind `On Graph Menu Item Selected` to receive the chosen action type,
display name, source variable/function ID, created node ID, graph position, and
success state after any built-in menu item is clicked. Menus use compact desired
sizes, a bounded scroll region, and screen-work-area placement.

Palette entries use the last cursor/right-click graph position, falling back to
the view center before the graph has received pointer input. Custom popup menus
can call `Set Node Spawn Graph Position` with the context-click position before
invoking a public Create-panel action.

Unconnected input pins use inline value editors. This includes `Interval
Seconds` on `Event Tick`: `0` means every server frame, while a positive value
throttles that event independently. Pin-default edits use the same
server-authoritative gateway and FastArray replication path as graph edits.
The built-in pin class exposes `Input Socket Label Spacing`, `Input Label Editor
Spacing`, and `Default Value Editor Width` for compact custom skins.

### Functions and prebuilt events

Select non-entry nodes and explicitly call `Collapse Selection To Function` on
the graph widget from your context menu, shortcut, or own button. Cross-boundary pins are generated
automatically, the selected subgraph moves into replicated function storage,
and the main graph receives a reusable call node. Existing functions appear in
the Create panel. Entry events and existing function calls are rejected inside
a collapse operation to prevent recursive definitions in this version.

The function details panel lists functions and their input/output pins. One
wireable `Function Entry` node contains all public inputs and one `Return` node
contains all public outputs, so signatures are not limited by the original
selection's pin count. Renaming/removing a pin updates every call node by stable
interface identity. Double-clicking a function call or list row switches the
main graph widget to the editable function canvas; its node/link/default
mutations follow the same server-authoritative function FastArray path, and
boundary nodes are bypassed during executable expansion.

Initialize a standalone function panel with `Initialize Function Panel` and
only the initialized graph widget, or use `Initialize Function Panel From
Actor`; no manual runtime-component lookup is required. Call `Return To Main
Graph` to leave the function canvas. Disable in-place switching and bind `On
Function Editor Opened` to host the returned function graph in a custom UI.
The built-in function panel also exposes an `< EVENT GRAPH` button while the
same graph widget is editing a function.

### Graph saving

Call `Save Graph To Slot` / `Load Graph From Slot` on the target
`CBPRuntimeComponent`. Client calls use the replicated edit gateway and execute
on the server. Snapshots include graph links, variables and current
values, function bodies, and signatures. Load validates the snapshot and then
replicates the restored FastArray state to clients. `Export Graph Save Data`
and authority-only `Import Graph Save Data` are available for custom save
systems.

The variable panel's `Save Variable` button reports validation/network status.
Its list displays the saved default value; when the runtime value still equals
the previous default, saving a new default updates both immediately.
The details area also lists every player editing the selected variable and
shows a shared-edit warning when more than one player has it open.

`Event Construction` and `Event Begin Play` fire once, in that order, when the
component begins play on the server. `Event Tick` is scheduled only by the
server. Graphs, variables, and functions replicate persistently; transient
execution/debug events multicast unreliably for client-side visualization.

### Widget Blueprint design preview

The native graph, node, and pin layouts also render in the UMG designer. Their
Class Defaults expose editor-only preview settings:

- a graph widget uses `Preview Node Definitions` to place sample DataAsset
  nodes on its canvas;
- a node widget uses `Preview Node Definition`;
- a pin widget uses `Preview Node Definition` plus `Preview Pin ID`.

These preview references are never used by the runtime graph and are removed
from non-editor builds. With no graph preview assets selected, the designer
shows a guide card explaining where to add them.

`Use Built-in Graph Canvas` and `Use Built-in DataAsset Layout` should normally
remain enabled. They exist only as escape hatches for a project that deliberately
replaces the complete native layout with its own UMG WidgetTree.

The node and pin widgets render directly from the node-definition DataAsset:
title, header color, compact mode, pin names, directions, types, and default
values all use the same schema shown in the CBP Node Designer. No
`WBP_CBPNode_Test` or `WBP_CBPPin_Test` is required.

The ViewModel translates FastArray updates into stable UI objects.
`UCBPConnectionLayerWidget` draws all output links in one retained layer and
supports connection-drag preview. Pin widgets report only their local anchors;
connection geometry is not replicated or stored in the graph.

For a palette, call `Get Available Node Categories` and `Get Available Node
Definitions` on the graph widget. The latter supports category and text search
against asset IDs, display names, descriptions, and keywords. Pass the selected
asset to `Create Node At View Center`; no node-name string is required.

## Runtime modeling prototype

`UCBPRuntimeModelingSubsystem` exposes safe, Blueprint-callable OBJ import/export
under `Saved/CBPModels`, plus box creation. Models spawn as
`ACBPRuntimeModelActor` and support server-authoritative vertex translation,
scaling, and single-vertex changes. Transform and small mesh payloads replicate
to clients. The prototype limit is 4,096 vertices and 8,192 triangles; it is not
intended as a large-mesh streaming solution.

## Visual node definition designer

Double-click a `CBPNodeDefinitionAsset` to open the dedicated **CBP Node
Designer** instead of the generic DataAsset editor. The designer contains:

- a live node card using the asset's title and header color;
- separate input and output columns with type colors;
- direct editing of node name, type ID, category, version, and purity;
- direct editing of pin names, stable IDs, types, directions, required/variadic
  flags, and typed default values;
- `+ Pin`, move-up, move-down, and remove controls;
- optional previews of the top, whole-node background, and pin-background style
  layers;
- live schema errors and warnings;
- the complete Details panel for advanced properties.

Pin edits are transactional, update `SchemaHash`, dirty the package, and support
the editor's normal undo/redo workflow. The Content Browser also exposes a
direct `CBP Node Definition` asset factory.

### Optional custom style layers

Enable `Presentation > Custom Style > Enable Custom Style Slots`. The three
class fields remain hidden until this option is enabled:

- `Top Style Widget Class`
- `Node Background Widget Class`
- `Pin Background Widget Class`

Top and node-background widgets must derive from `CBPNodeExtensionWidget`. They
receive the owning `CBPNodeViewModel`. Pin-background widgets must derive from
`CBPPinExtensionWidget` and receive the individual `CBPPinViewModel`. Each base
provides a `Refresh From View Model` event. The class pickers reject unrelated
`UserWidget` classes.

The concrete C++ node and pin widgets place these style layers automatically.
If a project deliberately replaces the complete layout with a UMG skin, the
optional skin should disable `Use Default Layout` and add the following named
slots.

In the custom skin derived from `CBPNodeWidget`, add:

- `TopStyleSlot`, layered over the title/header region;
- `NodeBackgroundStyleSlot`, stretched behind the complete node content.

In the custom skin derived from `CBPPinWidget`, add:

- `PinBackgroundStyleSlot`, stretched behind the visible pin content.

Use an Overlay (or another Z-order-aware UMG panel) so the two background slots
are behind interactive content and do not consume layout space. The plugin
creates and refreshes these widgets only while the DataAsset option is enabled.
All bindings remain optional so older skins still load.

## Developer node manifests

Manifests are read from `DeveloperNodes` by default and must end with
`.cbpnode.json`. The directory is fixed by project settings; callers cannot pass an
arbitrary filesystem path.

See:

- `DeveloperNodes/Examples/PlayerClamp.cbpnode.json`
- `DeveloperNodes/Templates/TrustedExternal.cbpnode.json`

Pin `id` values are stable contracts. Display names may change, but published pin
IDs should not be reused for different meanings.

## External process profiles

Configure profiles under:

`Project Settings > Plugins > Custom Blueprint Runtime`

Supported path tokens:

- `$(ProjectDir)`
- `$(ProjectContentDir)`
- `$(ProjectSavedDir)`
- `$(PluginDir)`

Trusted external node definitions refer to a `ProfileId`; they never contain or
receive an executable path from a player.

The generic `cmd.exe /C <player input>` model is intentionally unsupported.

## Player scripting boundary

PlayerVM definitions cannot request privileged permissions. The future VM must:

- compile source to platform-independent bytecode on the server;
- expose no filesystem, network, process, pointer, or UObject reflection API;
- enforce per-tick and per-execution instruction budgets;
- return `Pending` when a tick budget is exhausted;
- resume through the common execution backend contract.

The first player-authoring backend should be Subgraph packaging. The textual DSL
can be added after the scheduler and graph versioning are operational.

## Current integration boundaries

- Trusted external manifests and the protected process runner exist, but external
  nodes are not yet dispatched by the graph scheduler.
- PlayerVM and Subgraph are contracts only; no player bytecode compiler is wired.
- Legacy `S_NodeSave`, pin, variable, and save-game data need an explicit
  versioned importer.
- The modeling prototype replicates complete small mesh payloads; large models
  need chunking or operation-based replication.

## Verification

The plugin source has been compiled successfully for UE 5.3, 5.4, 5.5, 5.6,
and 5.7 on Win64 in Editor, Development, and Shipping configurations. Each
release archive carries binaries and content produced or validated by its target
engine version. UE 5.3 and 5.4 use node assets regenerated natively by that
engine so newer package versions are never copied backward.

Project-local test graphs and node executors with hard references to `/Game`
assets are intentionally excluded from marketplace archives. Author reusable
developer nodes against plugin or soft project inputs, and save cross-version
node assets in the oldest supported engine before migrating them forward.

Automation tests:

- `CBP.Schema.Validation`
- `CBP.Runtime.PureEvaluationAndExecution`
- `CBP.Runtime.GraphEditing`
- `CBP.Modeling.ObjRoundTrip`
- `CBP.Network.MultiplayerPresenceReplication`
- `CBP.UI.DocumentationScreenshots`

The network test starts one authority and two real clients, then verifies that
batched node positions and the two-player conflict state reach both clients.

The runtime execution test also verifies that every built-in node resolves from
a DataAsset and references a generated `BP_CBPExec_*` class before evaluating
the graph.

Copyright 2026 RuiZhongMing. All Rights Reserved.  
[Fab Marketplace](https://fab.com/s/96b5c2931a7b)
