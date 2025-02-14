# phalanx

Project Phalanx

This is a project to create an alternative roster editor that uses BattleScribe-format datafiles.

Our main communication channel beside Issues, PRs and Discussions on GitHub is the `#project-phalanx` channel on BSData Discord server.
[![BSData Discord link](https://img.shields.io/discord/558412685981777922?style=popout-square)](https://www.bsdata.net/discord)

Please see [Wiki](https://github.com/BSData/phalanx/wiki) for more information.

## Technical description

The following stack of app layers is currently (at least partially) implemented:

- DTO to (de)serialize XML content into typed objects (WarHub.ArmouryModel.Source.*Core types). This layer is a tree where children don't have a reference to their parent.
- SourceNode layer, lazy-initialized immutable wrappers for DTOs (WarHub.ArmouryModel.Source.*Node types). This layer is a tree where children have a reference to their parent.
- Symbol layer, immutable object graph. Internal implementation of upper ISymbol layer, which builds an actual object graph, by resolving references (e.g. link targetId is resolved to an actual symbol for the target entry) - this operation is called binding. This layer also generates diagnostics (e.g. bad links, invalid enum values, etc).
- ISymbol layer is an interface view of the Symbol layer, creating a fully bound object graph.
- Compilation is a container for a set of data roots (gamesystem, catalogues, rosters), in which a binding happends.
- RosterEditor and RosterState (from `EditorServices` namespace) is a heavily WIP layer that manages actual editing of roster, via IRosterOperations which encapsulate roster edit actions.

## Development

You need one of the following setups:

- Visual Studio 2022 (latest) - open the `Phalanx.sln` with it.
- VSCode and .NET 6 SDK (latest) - open the repository folder in it.

The main app to run is in `src/Phalanx.App` - select this project as startup project in Visual Studio. VSCode has a defined debug configuration.

## TODO

- test default group entry binding - how it works in BattleScribe (allowed selections), does it work same in Phalanx
- consider removing "deeper" types than IResourceDefinition/IResourceEntry/IContainerEntry
  - simpler usage
  - requires moving their "custom" properties somewhere: "ContentFields" collection of custom name-value wrappers, maybe strongly typed?
- symbol layer:
  - symbolinfo that might be an error
  - implementation for: constraints, logic (conditions, modifiers)
  - usage/references (UseSiteInfo?)
- add default subselections:
  - for entries with constraints min > 0 (constraints need symbols?)
  - entry groups
  - should that be in a separate "RosterEditor" module?
- implement Logic symbols
- semantic model that retrieves symbols/symbol info from `SourceNode`s
- use ObjectPool for performance
- Analyzer-like API/plugin support
- print nicer diagnostics + usable location
- support for speculative semantic model (simulation)
- more declaration diagnostics
- diagnostic fixers

## Symbols

The symbols are the basis of semantic model object graph.

### Symbol notes

- `ISymbol` is the root type.
- Links (BS EntryLink/InfoLink/CategoryLink) are represented as the symbol type
that the link points to, its `IsReference` is `true` and `ReferencedEntry` points
to the link target.

### Symbol list

`ISymbol` is the top-level symbol interface. All symbols are an `ISymbol`

- `IGamesystemNamespaceSymbol` - root symbol, contains all loaded data.
- `IModuleSymbol` - data module root, most often one file is one module.
  - `ICatalogueSymbol` - contains data: definitions, entries.
  - `IRosterSymbol`

- `ICatalogueReferenceSymbol` - references other catalogues from a catalogue.
- `IResourceDefinitionSymbol` - defines a resource type: publication, cost, profile, characteristic.
  - `IPublicationSymbol` - special resource definition with additional details, like publisher.
- `IEntrySymbol` // contains effects
  - `IResourceEntrySymbol` (can be a group, or group link)
    - `ICharacteristicSymbol`
    - `ICostSymbol`
    - `IProfileSymbol` (can be a link)
    - `IRuleSymbol` (can be a link)
  - `IContainerEntrySymbol` // contains constraints, resources
    - `ICategoryEntry` (can be a link)
    - `IForceEntry`
    - `ISelectionEntryContainerSymbol` // contains other SelectionEntry-like stuff
      - `ISelectionEntrySymbol` (can be a link)
      - `ISelectionEntryGroupSymbol` (can be a link)
- `ILogicSymbol`
  - `IConstraintSymbol` // constraint, contains query
  - `IEffectSymbol` // modifier/modifier group/repeat, contains query and condition
  - `IConditionSymbol` // condition/condition group, contains query
  - `IQuerySymbol` // condition/constaint/repeat "query" part

- `IRosterCostSymbol` // both value and limit (if set)
- `IEntryInstanceSymbol` // has SourceEntry
  - `IResourceSymbol`
    - `IRosterProfileSymbol`
    - `IRosterRuleSymbol`
  - `IContainerEntryInstanceSymbol` // has custom name, notes
    - `ICategorySymbol`
    - `ISelectionContainerSymbol` // has selections
      - `IForceSymbol`
      - `ISelectionSymbol`

## Implementation notes

BattleScribe has some unnatural or unexpected, or just unobvious behavior.
Here we try to gather such cases:

- Editing a roster results in Categories having new IDs, even if they/selections didn't change.
