# How to Extend This Template

Everything below assumes you've already renamed the namespace (see
[README.md](README.md)) and confirmed the extension loads cleanly with
just the console message and nothing else. Each section is additive -
none of them depend on the others, add whichever your extension
actually needs.

All calls below go through `stateEngine.*`
(`SillyTavern-StateEngine/src/api/index.js`) with the same
`(extensionId, instanceId, ...)` identity pattern this template's own
`src/api/*.js` wrappers already use - see
[architecture.md](architecture.md) if the shape looks unfamiliar. The
full, authoritative signatures live in the State Engine's own API
Reference; this file only shows the calls most relevant to each task.

The import path in each snippet below (`'../../SillyTavern-StateEngine/...'`)
assumes the code lives in a file directly under `src/` (one level deep,
like `src/extension.js` itself) - the same sibling-folder assumption
`src/api/*.js` makes, just one `../` shallower since those live under
`src/api/` instead. Adjust the number of `../` to match wherever you
actually put the code - or, better, add your own thin wrapper under
`src/api/` (matching `namespace.js`/`registration.js`/`capabilities.js`)
for anything you end up calling from more than one place.

## Adding presets

A preset is a named group of variables your extension owns - create one
once (e.g. from `initExtension()`, or lazily the first time your
extension needs it), then activate it for whichever chat should use it:

```js
import { stateEngine } from '../../SillyTavern-StateEngine/src/api/index.js';
import { ensureInstanceId } from '../../SillyTavern-StateEngine/src/api/identity.js';

const preset = stateEngine.createPreset(EXTENSION_ID, ensureInstanceId(), {
    namespace: NAMESPACE,
    name: 'Main',
    description: 'This extension\'s primary variable set',
    triggers: ['ai'], // which prompted-update triggers this preset's variables respond to
});

stateEngine.activatePreset(EXTENSION_ID, ensureInstanceId(), chatId, NAMESPACE, 'Main');
```

`createPreset` returns `null` (not a thrown error) for an ordinary
failure like a duplicate name - check for that before assuming it
worked. `activatePreset` seeds default values, recalculates any
calculated variables, and refreshes macros for you; nothing further to
call afterward.

## Adding variables

Variables live inside a preset. Nine types are supported - string,
number, boolean, enum, array, datetime, image, imageList, imageMap, and
calculated - see the State Engine's own "Variable Types" reference for
the full field shape of each. A minimal prompted string variable:

```js
stateEngine.createVariable(EXTENSION_ID, ensureInstanceId(), {
    namespace: NAMESPACE,
    presetName: 'Main',
    name: 'mood',
    type: 'string',
    defaultValue: 'calm',
    behaviors: { prompted: true, increment: false },
    prompted: { instructions: 'infer the character\'s current mood' },
});
```

The model then writes to it during the normal per-message prompted
update (or your own independent preset's update - see below) - never
call `createVariable`/`updateVariable` with a `value` field to set it
directly; a variable's value flows through chat state
(`stateEngine.getVariable`/the `{{getvar::myExtension__mood}}` macro),
never through its definition.

## Adding UI

The State Engine API itself is UI-agnostic - it has no opinion on how
(or whether) you show anything. Nothing in this template's scaffold
needs changing to add UI; you add your own HTML/CSS and jQuery wiring
the same way any SillyTavern extension does:

1. Add a `settings.html` (or similar) fragment, and load it the way
   SillyTavern's own extension settings panels do (fetched and injected
   into the extensions drawer) or via a modal/panel your extension opens
   itself.
2. Wire it up from `src/extension.js` (or a new module you import from
   there) - call `stateEngine.getVariable`/`listVariables`/`getVar`-style
   reads to populate it, and your own `updateVariable`/whatever write
   calls in response to user interaction.
3. Keep UI code out of `src/api/*.js` - those three files exist purely
   as identity-checked API wrappers, matching this template's own
   structure. A `src/ui/` folder, added when you need it, is a natural
   place to grow into.

## Adding independent presets

An independent preset runs its own prompted update on demand or on a
schedule, instead of participating in the standard per-message flow -
useful for a periodic summary, a background world-state tick, or
anything that shouldn't ride on every chat message:

```js
const indy = stateEngine.createIndependentPreset(EXTENSION_ID, ensureInstanceId(), {
    namespace: NAMESPACE,
    name: 'PeriodicSummary',
    connectionProfileId: '', // '' = use the currently active connection
});

// Optional: run it every real 30 minutes (natural-language time expression,
// same parser datetime variables use).
stateEngine.updateIndependentPresetSchedule(EXTENSION_ID, ensureInstanceId(), NAMESPACE, 'PeriodicSummary', {
    enabled: true,
    mode: 'interval',
    value: '30 minutes',
});

// Or run it on demand:
await stateEngine.runIndependentPreset(EXTENSION_ID, ensureInstanceId(), chatId, { namespace: NAMESPACE, name: 'PeriodicSummary' });
```

An independent preset automatically operates on every prompted/
incrementable variable IN THAT PRESET - there's no separate scoping
step; add variables to it exactly as in "Adding variables" above, just
naming `'PeriodicSummary'` as the `presetName`.

## Adding events

Events let your extension announce something happened, namespaced so
dispatching stays scoped to you while LISTENING stays open to anyone:

```js
// Declare it (optional, but makes it discoverable):
stateEngine.registerEventSource(EXTENSION_ID, ensureInstanceId(), {
    namespace: NAMESPACE,
    eventName: 'moodChanged',
});

// Fire it (namespace-qualified: "myExtension.moodChanged"):
stateEngine.fireEvent(EXTENSION_ID, ensureInstanceId(), chatId, `${NAMESPACE}.moodChanged`);

// Anyone - including your own extension - listens the ordinary
// SillyTavern way, not through the State Engine API:
SillyTavern.getContext().eventSource.on(`${NAMESPACE}.moodChanged`, () => { /* ... */ });
```

Only DISPATCHING is namespace-restricted to your extension; listening is
open to any code, since that's how other extensions would react to your
events in the first place.

## Integrating with Scenario Builder

There is no Scenario-Builder-specific API - it's expected to be just
another consumer of the same namespaced `stateEngine.*` surface this
template already uses, discovering and reading your extension's
variables/presets the ordinary way (`getRegisteredExtensions`,
`listVariables`, the capability graph below). If you want your
extension to be usable FROM Scenario Builder, the useful things to do
are the same ones that make any extension discoverable:

1. Register with a clear `description` and a `variables` list naming
   what you expose (see "Registration" in [architecture.md](architecture.md)).
2. Declare a capability describing what you provide, e.g.
   `"scenario.characterState"`, so Scenario Builder (or anything else)
   can find you via `stateEngine.getExtensionsProviding(...)` rather
   than needing to know your `extensionId` in advance.
3. If Scenario Builder later publishes its own capability name to
   depend on, declare it in your own `dependsOn` list - see "Declaring
   dependencies", directly below.

## Declaring dependencies

If your extension needs something ANOTHER extension provides, declare
it the same way you declare what you yourself provide - as a plain
capability string:

```js
stateEngine.declareDependencies(EXTENSION_ID, ensureInstanceId(), ['data.inventory']);

// Find out who (if anyone) actually provides it:
const providers = stateEngine.getExtensionsProviding('data.inventory'); // [] if nobody does yet
```

Nothing enforces that a dependency's provider is installed or loaded -
`declareDependencies` just records the string for the capability graph;
resolving it (deciding what to do if `providers` comes back empty) is
entirely up to your own extension's code. A natural place for this
check is right after `initExtension()` in `src/extension.js`, or lazily
right before you actually need the dependency.
