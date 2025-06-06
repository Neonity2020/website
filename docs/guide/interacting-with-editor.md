# Interacting with Editor

This guide covers the essential ways to interact with the Milkdown editor, including initialization, content management, and editor lifecycle.

## Using Crepe Editor

---

Crepe is a high-level wrapper around Milkdown that provides a simpler API for common editor operations. Here's how to use it:

```typescript
import { Crepe } from "@milkdown/crepe";

// Create a new editor instance
const editor = new Crepe({
  // Optional: specify root element (DOM node or selector)
  root: "#editor",

  // Optional: set default content, supports markdown, json and dom.
  defaultValue: "# Hello Crepe!",
});

// Create the editor
await editor.create();

// Get markdown content
const markdown = editor.getMarkdown();

// Set readonly mode
editor.setReadonly(true);

// Register event listeners
editor.on((listener) => {
  listener.markdownUpdated((ctx, markdown) => {
    console.log("Content updated:", markdown);
  });

  listener.focus((ctx) => {
    console.log("Editor focused");
  });

  listener.blur((ctx) => {
    console.log("Editor blurred");
  });

  listener.selectionUpdated((ctx, selection, prevSelection) => {
    console.log("Selection updated:", selection);
  });

  listener.updated((ctx, doc, prevDoc) => {
    console.log("Document updated:", doc);
  });
});

// Destroy the editor when done
await editor.destroy();
```

## Register to DOM

---

By default, milkdown will create editor on the `document.body`. Alternatively, you can also point out which dom node you want it to load into:

```typescript
import { rootCtx } from "@milkdown/kit/core";

Editor.make().config((ctx) => {
  ctx.set(rootCtx, document.querySelector("#editor"));
});
```

It's also possible to just pass a selector to `rootCtx`:

> The selector will be passed to `document.querySelector` to get the dom.

```typescript
import { rootCtx } from "@milkdown/kit/core";

Editor.make().config((ctx) => {
  ctx.set(rootCtx, "#editor");
});
```

## Setting Default Value

---

We support three types of default values:

- Markdown strings
- HTML DOM
- Prosemirror documentation JSON

### Markdown

You can set a markdown string as the default value of the editor.

```typescript
import { defaultValueCtx } from "@milkdown/kit/core";

const defaultValue = "# Hello milkdown";
Editor.make().config((ctx) => {
  ctx.set(defaultValueCtx, defaultValue);
});
```

### Dom

You can also use HTML as default value.

Let's assume that we have the following html snippets:

```html
<div id="pre">
  <h1>Hello milkdown!</h1>
</div>
```

Then we can use it as a defaultValue with a `type` specification:

```typescript
import { defaultValueCtx } from "@milkdown/kit/core";

const defaultValue = {
  type: "html",
  dom: document.querySelector("#pre"),
};
Editor.make().config((ctx) => {
  ctx.set(defaultValueCtx, defaultValue);
});
```

### JSON

We can also use a JSON object as a default value.

This JSON object can be obtained by a listener through the [listener-plugin](https://www.npmjs.com/package/@milkdown/plugin-listener), for example:

```typescript
import { listener, listenerCtx } from "@milkdown/kit/plugin/listener";

let jsonOutput;

Editor.make()
  .config((ctx) => {
    ctx.get(listenerCtx).updated((ctx, doc, prevDoc) => {
      jsonOutput = doc.toJSON();
    });
  })
  .use(listener);
```

Then we can use this `jsonOutput` as default Value:

```typescript
import { defaultValueCtx } from "@milkdown/kit/core";

const defaultValue = {
  type: "json",
  value: jsonOutput,
};
Editor.make().config((ctx) => {
  ctx.set(defaultValueCtx, defaultValue);
});
```

## Inspecting Editor Status

---

You can inspect the editor's status through the `status` property.

```typescript
import { Editor, EditorStatus } from "@milkdown/kit/core";

const editor = Editor.make().use(/* some plugins */);

assert(editor.status === EditorStatus.Idle);

editor.create().then(() => {
  assert(editor.status === EditorStatus.Created);
});

assert(editor.status === EditorStatus.OnCreate);

editor.destroy().then(() => {
  assert(editor.status === EditorStatus.Destroyed);
});

assert(editor.status === EditorStatus.OnDestroyed);
```

You can also listen to the status changes:

```typescript
import { Editor, EditorStatus } from "@milkdown/kit/core";

const editor = Editor.make().use(/* some plugins */);

editor.onStatusChange((status: EditorStatus) => {
  console.log(status);
});
```

### Status Lifecycle

1. `Idle`: Initial state
2. `OnCreate`: During creation
3. `Created`: Successfully created
4. `OnDestroyed`: During destruction
5. `Destroyed`: Successfully destroyed

## Adding Listeners

---

As mentioned above, you can add a listener to the editor, in order to get its value when needed.
You can add as many listeners as you want, all the listeners will be triggered at once.

### Markdown Listener

You can add markdown listener to get the editor's contents as a markdown string.

> ⚠️ Markdown listener will influence the performance for large documents, please use it carefully.
> If you have a large document, I suggest you to only `parse` and `serialize` the document when needed.

```typescript
import { listener, listenerCtx } from "@milkdown/kit/plugin/listener";

let output = "";

Editor.make()
  .config((ctx) => {
    ctx.get(listenerCtx).markdownUpdated((ctx, markdown, prevMarkdown) => {
      output = markdown;
    });
  })
  .use(listener);
```

### Doc Listener

You can also listen to the [raw prosemirror document node](https://prosemirror.net/docs/ref/#model.Node), and do things you want from there.

```typescript
import { listener, listenerCtx } from "@milkdown/kit/plugin/listener";

let jsonOutput;

Editor.make()
  .config((ctx) => {
    ctx.get(listenerCtx).updated((ctx, doc, prevDoc) => {
      jsonOutput = doc.toJSON();
    });
  })
  .use(listener);
```

### Selection Listener

You can track changes to the editor's selection using the `selectionUpdated` event. This is useful for implementing features like:

- Custom toolbars that update based on selection
- Context menus
- Selection-based formatting controls

```typescript
import { listener, listenerCtx } from "@milkdown/kit/plugin/listener";
import { Selection, TextSelection } from "@milkdown/prose/state";

Editor.make()
  .config((ctx) => {
    ctx.get(listenerCtx).selectionUpdated((ctx, selection, prevSelection) => {
      if (selection instanceof TextSelection) {
        // Get selection range
        const { from, to } = selection;

        // Example: Update toolbar based on selection
        updateToolbar({
          hasSelection: from !== to,
          selectionStart: from,
          selectionEnd: to,
        });
      }
    });
  })
  .use(listener);
```

The selection listener will be triggered when the selection is changed.
So you don't need to compare them manually.

For more details about listeners, please check [Using Listeners](/docs/api/plugin-listener).

## Readonly Mode

---

You can set the editor to readonly mode by setting the `editable` property.

```typescript
import { editorViewOptionsCtx } from "@milkdown/kit/core";

let readonly = false;

const editable = () => !readonly;

Editor.make().config((ctx) => {
  ctx.update(editorViewOptionsCtx, (prev) => ({
    ...prev,
    editable,
  }));
});

// set to readonly after 5 secs.
setTimeout(() => {
  readonly = true;
}, 5000);
```

### Use Cases for Readonly Mode

- Preview mode
- Document review
- Print-friendly views
- Mobile device optimization

## Using Actions

---

You can use an action to get the context value in a running editor on demand.

For example, to get the markdown string by running an action:

```typescript
import { Editor, editorViewCtx, serializerCtx } from "@milkdown/kit/core";

async function playWithEditor() {
  const editor = await Editor.make().use(commonmark).create();

  const getMarkdown = () =>
    editor.action((ctx) => {
      const editorView = ctx.get(editorViewCtx);
      const serializer = ctx.get(serializerCtx);
      return serializer(editorView.state.doc);
    });

  // get markdown string:
  getMarkdown();
}
```

We provide some macros out of the box, you can use them as actions:

```typescript
import { insert } from "@milkdown/kit/utils";

editor.action(insert("# Hello milkdown"));
```

### Common Actions

- Insert content
- Get current selection
- Apply formatting
- Execute commands

For more details about macros, please check [macros](/docs/guide/macros).

## Destroying

---

You can call `editor.destroy` to destroy an existing editor. You can create a new editor again with `editor.create`.

```typescript
await editor.destroy();

// Then create again
await editor.create();
```

If you just want to recreate the editor, you can use `editor.create`, it will **destroy the old editor and create a new one**.

```typescript
await editor.create();

// This equals to call `editor.destroy` and `editor.create` again.
await editor.create();
```

If you want to **clear the plugins and configs for the editor** when calling `editor.destroy`, you can pass `true` to `editor.destroy`.

```typescript
await editor.destroy(true);
```
