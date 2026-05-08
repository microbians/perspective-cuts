# Fork Changes

This document tracks changes made in this fork that are **not yet upstream**.
Upstream: https://github.com/taylorarndt/perspective-cuts

Forked from commit `0caa14c` ("Mirror AI contributions policy into CONTRIBUTING").

---

## Workflow surface metadata directives

Added six new top-level metadata directives so a `.perspective` file can fully
describe where a shortcut shows up (Share Sheet, Quick Actions, menu bar,
widget, watch) and which content types it accepts as input. Previously these
settings had to be configured manually in the Shortcuts app after import.

### New directives

| Directive | Effect | Default |
|---|---|---|
| `#input: <type>[, <type>...]` | Sets `WFWorkflowInputContentItemClasses` | full list (any) |
| `#quickaction: true\|false` | Adds `QuickActionsService` to `WFWorkflowTypes` | `false` |
| `#sharesheet: true\|false` | Adds `ActionExtension` to `WFWorkflowTypes` | `false` |
| `#menubar: true\|false` | Adds `MenuBar` to `WFWorkflowTypes` | `false` |
| `#widget: true\|false` | Toggles `NCWidget` in `WFWorkflowTypes` | `true` |
| `#watch: true\|false` | Toggles `WatchKit` in `WFWorkflowTypes` | `true` |
| `#noinput: continue\|ask [<type>]\|clipboard\|cancel` | Sets `WFWorkflowNoInputBehavior` (what to do when run without input) | not set |

#### `#noinput: ask <type>` picker shorthands

The bare `ask` form prompts the user to pick anything. A second token
selects the content-type picker Shortcuts.app presents:

| Form | Picker |
|---|---|
| `#noinput: ask files` | file picker |
| `#noinput: ask images` | photo picker |
| `#noinput: ask media` | media picker |
| `#noinput: ask url` | URL prompt |
| `#noinput: ask text` | text prompt |
| `#noinput: ask date` | date picker |
| `#noinput: ask number` | number prompt |
| `#noinput: ask contact` | contact picker |

Maps to `WFWorkflowNoInputBehaviorAskForInput` with
`Parameters.ItemClass` and `Parameters.SerializedParameters.WFPickingMode`
set per Apple's expected values.

When `#sharesheet` or `#quickaction` is enabled, the compiler also sets
`WFWorkflowHasShortcutInputVariables` to `true` automatically.

### `#input` type tokens

Multiple tokens may be combined separated by commas or spaces. Special tokens
`any` (full default list) and `none` (empty list) are supported.

| Token | Maps to |
|---|---|
| `url` | `WFURLContentItem`, `WFSafariWebPageContentItem` |
| `webpage` | `WFSafariWebPageContentItem` (Safari pages only) |
| `text` | `WFStringContentItem`, `WFRichTextContentItem` |
| `string` | `WFStringContentItem` |
| `richtext` | `WFRichTextContentItem` |
| `image` | `WFImageContentItem` |
| `file` | `WFGenericFileContentItem`, `WFPDFContentItem` |
| `pdf` | `WFPDFContentItem` |
| `media` | `WFAVAssetContentItem` |
| `contact` | `WFContactContentItem` |
| `location` | `WFLocationContentItem`, `WFDCMapsLinkContentItem` |
| `date` | `WFDateContentItem` |
| `email` | `WFEmailAddressContentItem` |
| `phone` | `WFPhoneNumberContentItem` |
| `app` | `WFAppStoreAppContentItem`, `WFiTunesProductContentItem` |
| `article` | `WFArticleContentItem` |

### Example

```
import Shortcuts
#color: red
#icon: download
#name: Youtube to MP4
#input: url, text
#sharesheet: true
#quickaction: true
#noinput: clipboard

// ... actions ...
```

---

## Bug fixes

### Magic Variables lost on raw `is.workflow.actions.*` calls

When an action was called by raw identifier (e.g.
`is.workflow.actions.runshellscript(Input: someVar)`) the compiler
incorrectly took the 3rd-party path because the heuristic for "third
party" was simply "name contains a dot". That path called
`expressionToPlainValue` which does not consult the `outputMap`, so any
variable reference passed in as an argument was serialised as a bare
string and the resulting shortcut showed disconnected boxes ("no line"
between the producer and the consumer in Shortcuts.app).

Fix: identifiers in the `is.workflow.actions.` namespace are now treated
as built-in even when the registry does not know about them. Real 3rd
party identifiers (e.g. `com.openai.chat.AskIntent`) still take the App
Intent path.

### Plain-string parameters tokenised on raw built-in calls

Once the previous fix routed raw built-ins through the built-in path, a
secondary issue surfaced: parameters that must be plain strings
(`WFVariableName`, `Shell`, `InputMode`, `Script`,
`WFCommentActionText`, `WFTextActionText`) were being wrapped in a
`WFTextTokenString` envelope because there is no `ActionParameter`
definition to tell the compiler their type. Shortcuts.app then displayed
those fields with the wrong widget.

Fix: `Compiler.appleBuiltinPlainKeys` lists the parameter labels that
must be emitted as plain values when the action is a raw `is.workflow.actions.*`
call without a registry entry.

The list currently includes: `WFVariableName`, `WFInputVariable`, `Shell`,
`InputMode`, `Script`, `WFTextActionText`, `WFNotificationActionTitle`,
`WFContentItemPropertyName`.

### `ShortcutInput` inside string interpolation

`ShortcutInput` referenced inside an interpolated string (e.g.
`"\(ShortcutInput)"` as a notification body) now serialises as the
`ExtensionInput` token, not as a regular named-variable reference.
This was already correct for bare references; this commit aligns
interpolation with the same behaviour.

### Raw glyph numbers in `#icon`

`#icon: <number>` now accepts a raw `ZGLYPHNUMBER` integer in addition
to the named glyphs, useful when targeting a specific SF Symbol that
doesn't have a friendly alias yet.

### `let X = Y` lost references to action outputs

A `let X = Y` (or any non-dictionary variable declaration whose right
hand side is a reference) was emitting a Get Text action whose body
resolved `Y` against an empty output map, so references to action
outputs ended up serialised as plain `Variable` lookups. Shortcuts.app
then could not find the named variable — there is no Set Variable for
it, only a `CustomOutputName` on the producing action — and the field
came up empty at runtime.

Fix: thread the caller's `outputMap` through `buildTextAction` so any
reference to a previously named action output is emitted as
`Type: ActionOutput` with the correct `OutputUUID`.

---

## Property access syntax (inline aggrandizement)

Apple's gallery shortcuts (e.g. *Music to YouTube*) read properties
inline off a media/file/contact reference by attaching a
`WFPropertyVariableAggrandizement` decorator to the variable, instead
of inserting a separate Get Details Of action. This produces a leaner
plist and is the only form Shortcuts.app's iTunes media engine
resolves correctly when the source is `Get Current Song`.

Same affordance is now available in `.perspective`:

```
getCurrentSong() -> song
searchWeb(
    query: "\(song.Title) \(song.Artist)",
    destination: "YouTube"
)
```

`song.Title` and `song.Artist` parse as a new `propertyAccess`
expression and emit the canonical aggrandizement attachment. Both
bare references (`var.Property`) and references inside string
interpolation (`\(var.Property)`) are supported. Falls back to a plain
`Variable` reference when the base isn't in the output map.

---

## New built-in actions

### `searchWeb`

Wraps `is.workflow.actions.searchweb` with two parameters:

| Parameter | Type | Plist key |
|---|---|---|
| `query` (required) | string | `WFInputText` |
| `destination` (optional) | plainString | `WFSearchWebDestination` |

Destination accepts the same string values Shortcuts.app exposes
(`"Google"`, `"YouTube"`, `"DuckDuckGo"`, `"Bing"`, …).

---

## CLI

### `compile --open`

After signing, automatically `open` the resulting `.shortcut` so
Shortcuts.app picks up the import dialog. Combine with `--sign`:

```bash
perspective-cuts compile --sign --open my.perspective
```

For subsequent updates use `--install` (already in upstream) to overwrite
the existing shortcut in the Shortcuts.app database without going
through the import dialog again.

---

## Implementation notes

### `Sources/perspective-cuts/Parser/Parser.swift`

`parseMetadata` now also accepts `comma` and `boolLiteral` tokens inside the
metadata value, so directives like `#input: url, text` and
`#sharesheet: true` parse cleanly. Previously the metadata value collector
only accepted identifiers, numbers, and strings.

### `Sources/perspective-cuts/Compiler/Compiler.swift`

- New per-compile state: `widgetEnabled`, `watchEnabled`, `quickActionEnabled`,
  `shareSheetEnabled`, `menuBarEnabled`, `explicitInputClasses`.
- `WFWorkflowTypes` is now built from those toggles via `buildWorkflowTypes(...)`
  instead of a hardcoded `["NCWidget", "WatchKit"]`.
- `WFWorkflowInputContentItemClasses` uses `explicitInputClasses` when set,
  otherwise falls back to the previous default list (now exposed as
  `Compiler.defaultInputContentClasses`).
- New helpers: `parseBoolDirective(_:)` and
  `parseInputDirective(value:location:)`.
- `WFWorkflowHasShortcutInputVariables` is emitted as `true` whenever
  `#sharesheet` or `#quickaction` is enabled.

No lexer or AST changes were required — the existing `metadata(key, value)`
node already carries arbitrary key/value pairs.
