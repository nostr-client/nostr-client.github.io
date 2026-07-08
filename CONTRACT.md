# The nostr-client contract

Five conventions make independently-developed parts compose. Anything that
honors them can replace anything here.

## 1. Hex is the primitive

All APIs, element attributes, DOM events, filters, and storage use **64-char
lowercase hex** pubkeys and event ids — exactly what relays speak.

NIP-19 strings (`npub…`, `nsec…`, `note…`) are **presentation only**: encode at
the last moment before display, decode at the first moment after user input.
[nip19](https://github.com/nostr-client/nip19) does both.

A pubkey is also a W3C DID — `did:nostr:<hex>` — for interop with DID/Solid
tooling. See [did](https://github.com/nostr-client/did) and
[did-nostr.com](https://did-nostr.com/).

## 2. The signer

The page-wide signer is **NIP-07-shaped** — the same interface browser
extensions already implement:

```js
window.nostrSigner  // { type, getPublicKey(): hex, signEvent(evt): signed event } | null
window.nostrPubkey  // hex | null
```

Login components announce changes on `window`:

```js
window.dispatchEvent(new CustomEvent('nostr:login',  { detail: { pubkey, signer } }))
window.dispatchEvent(new CustomEvent('nostr:logout'))
```

Consumers check `window.nostrSigner` on connect and listen for both events.
Any login UI that does this replaces
[login](https://github.com/nostr-client/login) — every other part keeps
working. Components also accept a `.signer` property for explicit injection.

## 3. The pool

One page, one websocket per relay — no matter how many components are
composed. Components default to the shared singleton:

```js
import { defaultPool } from 'https://nostr-client.github.io/pool/pool.js'
```

The pool interface: `subscribe(filters, {onEvent, onEose, relays})` →
`{close()}`, `list(filters)`, `get(filter)`, `publish(event)` → per-relay
`[{relay, ok, message}]`.

Swap globally by assigning `globalThis.__nostrClientPool` before components
load, per-component via the `relays="wss://…"` attribute or the `.pool`
property.

## 4. Events

Components communicate outward with `CustomEvent`s, namespaced `nostr:*`,
dispatched with `bubbles: true, composed: true` (and on `window` where global
interest is expected): `nostr:login`, `nostr:logout`, `nostr:published`,
`nostr:profile-saved`, …

Data in `detail` uses the primitives: hex keys, raw nostr events.

## 5. Swapping

Parts import each other by **absolute URL**:

```js
import { defaultPool } from 'https://nostr-client.github.io/pool/pool.js'
```

Because import maps can remap absolute URLs, a composing page can replace any
dependency without forking:

```html
<script type="importmap">
{ "imports": {
    "https://nostr-client.github.io/pool/pool.js": "https://my.site/fancier-pool.js"
} }
</script>
```

Rules for a part to be a good citizen:

- **one repo, one thing**, with a live demo `index.html` on gh-pages
- **no build step** — the repo is the deployment
- **no dependencies**, or pinned CDN ESM URLs loaded lazily and only for
  optional paths
- render user content with DOM APIs, never `innerHTML`
- AGPL-3.0-or-later
