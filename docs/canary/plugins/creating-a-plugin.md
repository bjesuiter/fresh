---
description: How to create a plugin for fresh
---

Fresh plugins are in essence a collection of hooks that allow the plugin to hook
into various systems inside of Fresh.

At the core a Fresh plugin is just a JavaScript object that conforms to the
[Plugin](https://deno.land/x/fresh/server.ts?s=Plugin) interface. The only
required property of a plugin is its name. Names must only contain the
characters `a`-`z`, and `_`.

```ts
import { Plugin } from "$fresh/server.ts";

const plugin: Plugin = {
  name: "my_plugin",
  // entrypoints?: Record<string, string>;
  // render?(ctx: PluginRenderContext): PluginRenderResult;
  // renderAsync?(ctx: PluginAsyncRenderContext): Promise<PluginRenderResult>;
  // routes?: PluginRoute[];
  // middlewares?: PluginMiddleware<State>[];
};
```

A plugin containing only a name is technically valid, but not very useful. To be
able to do anything with a plugin, it must register some hooks, middlewares, or
routes.

> ⚠️ HEADS UP! The format for the plugin definition is currently under discussion
> to hopefully make plugin support much more powerful, without adding huge
> support overhead. If you're interested, join
> [this github issue](https://github.com/denoland/fresh/issues/1602#issuecomment-1686535522)
> for the discussion.

## Taking Options

To take options for your plugin, do not create and export the
`const plugin = ...` object directly, but a factory function instead which
returns the plugin object:

```ts
// IMPORTANT: You need to use the raw fresh import here, instead of a importMap value,
// bc. import maps will not be resolved when loading your plugin from deno.land/x
// ./deps/fresh.ts contains: `export * from "https://deno.land/x/fresh@1.4.2/server.ts";`
import { Plugin } from "./deps/fresh.ts";

type YourPluginOptions = {
  option1: boolean;
  option2: string;
  // ...
};

export async function MyPlugin(options: YourPluginOptions) {
  // ... validate your options
  // ... init your plugin

  return {
    name: "openprops",
  } satisfies Plugin; // this `satisfies` ensures that your plugin object is correct without tricking the typescript compiler;
}
```

> ℹ️ TIP: You could use [zod][zod] to validate the options passed to your plugin,
> as well as setting defaults. You can go here to see an example:
> [Skeleton Plugin with zod options][fresh-plugin-w-zod-options]

## Available Hooks

### Hook: `render`

The render hook allows plugins to:

- Control timing of the synchronous render of a page.
- Inject additional CSS and JS into the rendered page.

This is commonly used to set thread local variables for the duration of the
render (for example preact global context, preact option hooks, or for style
libraries like Twind). After render is complete, the plugin can inject inline
CSS and JS modules (with attached state) into the page.

The render hook is called with the
[`PluginRenderContext`](https://deno.land/x/fresh/server.ts?s=PluginRenderContext)
object, which contains a `render()` method. This method must be invoked during
the render hook to actually render the page. It is a terminal error to not call
the `render()` method during the render hook.

The `render()` method returns a
[`PluginRenderFunctionResult`](https://deno.land/x/fresh/server.ts?s=PluginRenderFunctionResult)
object which contains the HTML text of the rendered page, as well as a boolean
indicating whether the page contains any islands that will be hydrated on the
client.

The `render` hook needs to synchronously return a
[`PluginRenderResult`](https://deno.land/x/fresh/server.ts?s=PluginRenderResult)
object. Additional CSS and JS modules can be added to be injected into the page
by adding them to `styles` and `scripts` arrays in this object.

`styles` are injected into the `<head>` of the page as inline CSS. Each entry
can define the CSS text to inject, as well as an optional `id` for the style
tag, and an optional `media` attribute for the style tag.

`scripts` define JavaScript/TypeScript modules to be injected into the page. The
possibly loaded modules need to be defined up front in the `Plugin#entrypoints`
property. Each defined module must be a JavaScript/TypeScript module that has a
default export of a function that takes one (arbitrary) argument, and returns
nothing (or a promise resolving to nothing). Fresh will call this function with
the state defined in the `scripts` entry. The state can be any arbitrary JSON
serializable JavaScript value.

For an example of a plugin that uses the `render` hook, see the first-party
[Twind plugin](https://github.com/denoland/fresh/blob/main/plugins/twind.ts).

### Hook: `renderAsync`

This hook is largely the same as the `render` hook, with a couple of key
differences to make asynchronous style and script generation possible. It must
asynchronously return its
[`PluginRenderResult`](https://deno.land/x/fresh/server.ts?s=PluginRenderResult),
either from an `async/await` function or wrapped within a promise.

The render hook is called with the
[`PluginAsyncRenderContext`](https://deno.land/x/fresh/server.ts?s=PluginAsyncRenderContext)
object, which contains a `renderAsync()` method. This method must be invoked
during the render hook to actually render the page. It is a terminal error to
not call the `renderAsync()` method during the render hook.

This is useful for when plugins are generating styles and scripts with
asynchronous dependencies based on the `htmlText`. Unlike the synchronous render
hook, async render hooks for multiple pages can be running at the same time.
This means that unlike the synchronous render hook, you can not use global
variables to propagate state between the render hook and the renderer.

The `renderAsync` hooks start before any page rendering occurs, and finish after
all rendering is complete -- they wrap around the underlying JSX->string
rendering, plugin `render` hooks, and the
[`RenderFunction`](https://deno.land/x/fresh/server.ts?s=RenderFunction) that
may be provided to Fresh's `start` entrypoint in the `main.ts` file.

### Adding Routes

To add a route to your plugin provide one or more `PluginRoute` definitions to
the `routes` property. Routes can be either component routes or plain handler
routes which can be used to build api endpoints inside a plugin.

```ts
const myHandlerRoute = {
  path: "/handler", // the plugin route will be attached to /handler in the host Fresh app.
  handler: async (
    req: Request,
    ctx: HandlerContext,
  ) => {
    // Do some things in your handler and return a response
    // This will be a GET handler 
    // It can return the Response directly or as a promise (a.k.a be an 'async' function)
    return new Response("Hello World", {
        headers: new Headers([
          ["Content-Type", "text/plain"]
        ]),
      }); 
  },
} satisfies PluginRoute; // this ensures that your route config matches the expected Fresh type

const myRoute2 = {
  path: "/component", // the plugin route will be attached to /component in the host Fresh app.
  component: // a normal Fresh component
} satisfies PluginRoute;

return {
  name: "openprops",
  routes: [myRoute1, myRoute2],
} satisfies Plugin;
```

TODO

### Adding Middlewares

TODO

### Adding Custom JS Entrypoints

TODO

### Routes and Middlewares - REWRITE!

You can create routes and middlewares that get loaded and rendered like the
normal [routes](/docs/concepts/routes) and
[middlewares](/docs/concepts/middleware).

The plugin routes and middlewares need a defined path in the format of a file
name without a filetype inside the routes directory(E.g. `blog/index`,
`blog/[slug]`).

For more examples see the [Concepts: Routing](/docs/concepts/routing) page.

To create a middleware you need to create a `MiddlewareHandler` function.

And to create a route you can create both a Handler and/or component.

<!-- Links-->

[zod]: https://deno.land/x/zod
[fresh-plugin-w-zod-options]: https://github.com/codemonument/deno_fresh_plugin_skeleton/blob/main/src/01a_skeleton_plugin_w_zod.ts
