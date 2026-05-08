# translucid-svelte

Svelte 5 adapter for [Translucid](https://github.com/EmilienKopp/translucid) — real-time model sync over Laravel Echo.

Models stay in sync across tabs and users automatically. No polling, no manual re-fetching.

## Requirements

- Svelte 5
- Laravel Echo (`laravel-echo` v2+) configured with Reverb, Pusher, or Ably
- The [Translucid Laravel package](https://github.com/EmilienKopp/translucid) broadcasting events from your backend

## Installation

```bash
npm install translucid-svelte
```

## Setup

Configure once in your root layout (e.g. `AppLayout.svelte`). The space is resolved lazily on every subscription, so it stays correct across Inertia navigations.

```svelte
<script>
  import { translucid } from 'translucid-svelte';
  import { page } from '@inertiajs/svelte';

  // Resolve space from Inertia page props
  translucid.configure(() => page.props.space);
</script>
```

If you don't use a multi-tenant space, pass a static string:

```svelte
translucid.configure('default');
```

To inject a custom Echo instance instead of relying on `window.Echo`:

```svelte
translucid.configure({ space: () => page.props.space, echo: myEcho });
```

## Usage

### Watch a single model (`$state`)

Reactively sync a model declared with `$state`. Updates from the server mutate the object in place — Svelte's reactivity picks them up automatically.

```svelte
<script>
  import { translucid } from 'translucid-svelte';

  let { user } = $props();
  const currentUser = $state(user);

  translucid.table('users').watch(currentUser);
</script>

<p>{currentUser.name}</p>
```

`watch()` returns a cleanup function and is safe to call at the component's top level — it registers and deregisters listeners automatically.

### Watch a derived or prop-sourced model (`$derived`)

Use `watchLive()` when the model reference comes from a getter or `$derived`. It wraps `$effect` internally and re-subscribes whenever the reference changes.

```svelte
<script>
  import { translucid } from 'translucid-svelte';
  import { page } from '@inertiajs/svelte';

  const user = $derived(page.props.auth.user);

  translucid.table('users').watchLive(() => user);
</script>
```

### Watch an array of models

```svelte
<script>
  import { translucid } from 'translucid-svelte';

  let { posts } = $props();
  const livePosts = $state(translucid.table('posts').watchAll(posts));
</script>

{#each livePosts as post}
  <p>{post.title}</p>
{/each}
```

### Override space per-call

```svelte
translucid.table('posts').scope('acme').watch(post);
```

### Stop watching manually

```svelte
translucid.table('users').unwatch(user);
```

## Functional API

For use outside of the `translucid` singleton — useful in plain TypeScript files or non-reactive contexts.

### `watchCollection(table, opts)`

Listens for `created` events on a table. Filters incoming records client-side against the current URL query string, so only records that belong on the current page are surfaced.

```ts
import { watchCollection } from 'translucid-svelte';
import { onDestroy } from 'svelte';

const stop = watchCollection('posts', {
  onCreated(payload) {
    items = [payload.data, ...items];
  },
});

onDestroy(stop);
```

URL filter convention:
- `?status=published` → only records where `status === 'published'`
- `?tag[]=svelte&tag[]=laravel` → only records where `tag` is in the set

### `watchId(table, id, opts)`

Listens for `updated` and `deleted` events on a specific record.

```ts
import { watchId } from 'translucid-svelte';
import { onDestroy } from 'svelte';

const stop = watchId('posts', post.id, {
  onUpdated(payload) { Object.assign(post, payload.changes); },
  onDeleted()        { goto('/posts'); },
});

onDestroy(stop);
```

## API Reference

### `translucid`

Pre-built singleton — import and use directly.

| Method | Description |
|--------|-------------|
| `.configure(resolver)` | Set global space string, resolver function, or `{ space, echo }` config object |
| `.scope(space)` | Override space for the next chain only |
| `.table(name)` | Set table context (chainable) |
| `.watch(model, index?)` | Subscribe to updates/deletes for a `$state` model. Returns cleanup function. |
| `.watchLive(getter, index?)` | Reactive variant for `$derived` models — wraps `$effect` internally |
| `.watchAll(arr)` | Subscribe to all models in an array. Returns a live array. |
| `.unwatch(model)` | Stop listening for a model |

### Standalone functions

| Export | Description |
|--------|-------------|
| `watchCollection(table, opts)` | Table-wide `created` events with URL-based filtering |
| `watchId(table, id, opts)` | Per-record `updated` / `deleted` events |

## How it works

The backend broadcasts events on a private channel `translucid.{space}` using custom event names:

```
.translucid.updated.{table}.{id}
.translucid.deleted.{table}.{id}
.translucid.created.{table}
```

On the client, `watch()` subscribes to these events and mutates the model object directly. Because Svelte 5's `$state` proxy intercepts property assignments, the UI updates without any manual triggering.

Echo is resolved lazily — `window.Echo` is read on first use, so it does not matter if this package is imported before Echo is initialized.

## License

ISC
