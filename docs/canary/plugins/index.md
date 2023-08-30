---
description: Plugins can add new functionality to Fresh without requiring significant complexity.
---

Plugins can dynamically add new functionality to Fresh without exposing
significant complexity to the user.

## Using Plugins

Users can add plugins by importing and initializing them in their `main.ts`
file:

```ts main.ts
import {start} from '$fresh/server.ts';
import manifest from './fresh.gen.ts';

import twindPlugin from '$fresh/plugins/twind.ts';
import twindConfig from './twind.config.js';

await start(manifest, {
	plugins: [
		// This line configures Fresh to use the first-party twind plugin.
		twindPlugin(twindConfig),
	],
});
```

## What Plugins can do

Plugins register some hooks which are provided by fresh core to attach
functionality to your fresh app. For example, the first-party plugin "Twind"
uses the "render" hook to inject additional css and js into the output for
styling.

Currently, they can use the following hooks:

- `render` and `renderAsync` - allow plugins to alter html output generation
- `routes` - allows adding pages and api routes to your app.
- `middlewares` - allows adding fresh middlewares to paths to provide additional
  data for your pages or functionality, like authentication

> ℹ️ To learn more about the technical details of these hooks, go to the Page
> [Creating a plugin][creating-a-plugin]

Plugin hooks are executed in the order that the plugins are defined in the
`plugins` array. This means that the first plugin in the array will be executed
first, and the last plugin in the array will be executed last. For many plugins,
this does not matter, but for some plugins it may.

## Available Plugins (WIP)

Currently, the only available first-party plugin is the Twind plugin.
Third-party plugins are also supported - they can be imported from any HTTP
server, like any other Deno module.

- tailwind V0
- tailwind v1

<!-- Links-->

[creating-a-plugin]: /docs/canary/plugins/creating-a-plugin
