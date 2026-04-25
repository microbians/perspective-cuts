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
| `#noinput: continue\|ask\|clipboard\|cancel` | Sets `WFWorkflowNoInputBehavior` (what to do when run without input) | not set |

When `#sharesheet` or `#quickaction` is enabled, the compiler also sets
`WFWorkflowHasShortcutInputVariables` to `true` automatically.

### `#input` type tokens

Multiple tokens may be combined separated by commas or spaces. Special tokens
`any` (full default list) and `none` (empty list) are supported.

| Token | Maps to |
|---|---|
| `url` | `WFURLContentItem`, `WFSafariWebPageContentItem` |
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
