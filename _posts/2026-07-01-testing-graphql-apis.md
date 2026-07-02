---
layout: post
title: "How I Test a GraphQL API (When Introspection Is Off)"
date: 2026-07-01 09:00:00 -0700
categories: [web, methodology]
tags: [graphql, api, idor, access-control, methodology, bug-bounty]
---

More and more apps I run into put their *entire* front-end on a single GraphQL endpoint —
`/graphql`, `/api/graphql`, something like that. One URL, and behind it every query and
mutation the app can do. That's a huge, structured attack surface hiding behind one door.
Here's the methodology I've settled on for working that door — including when the easy way in
is bolted shut.

> **Scope note:** everything below is generic technique, written against no specific target.
> Only ever run this against assets you're explicitly authorized to test.

![Testing a GraphQL API — one endpoint, big surface](/assets/img/graphql-testing.svg)

## 1. Find the endpoint and read the wire format

Open the app, watch the network tab, and look for `POST` requests to a `*/graphql` URL. Before
anything clever, look at the **shape of the request body** — it tells you a lot:

- `{"query": "...", "variables": {...}}` — a single operation.
- `[{...}, {...}]` — a **JSON array**, which means **query batching is enabled**. Note that; it
  becomes an attack of its own later.
- Only a hash in `extensions` (no `query` text) — **persisted queries** (APQ). The server only
  accepts pre-registered operations by hash… unless it *also* accepts a full query string. Test
  both.

## 2. Try to dump the schema (the fast wins)

GraphQL can hand you the entire schema if it's misconfigured. Always ask first:

```graphql
query { __schema { queryType { name } mutationType { name } types { name kind } } }
```

If introspection is on, you now have the full map — every type, field, and mutation. If it's
off, you'll get something like `INTROSPECTION_DISABLED`. Not the end of the road — try two more
cheap things:

- **Field suggestions.** Send a deliberately misspelled field (`restauran` instead of
  `restaurant`). Some servers reply *"Did you mean `restaurant`?"* — leaking real field names
  even with introspection disabled.
- **Error verbosity.** Do the errors carry helpful messages, or are they stripped to a bare
  code? Verbose errors leak structure; stripped ones mean someone hardened this on purpose.

A mature target will have introspection **off**, suggestions **off**, and errors **stripped**.
That's good defense — and it's not a wall, it's a detour.

## 3. Harvest the real operations

Here's the key idea when introspection is off: **the front-end already knows every valid
operation** — they ride inside its own traffic. So I let the app hand them to me. A small hook
on the browser's request functions logs each GraphQL body as the app makes it:

```js
// Hook fetch AND XMLHttpRequest — apps use either. And handle fetch(new Request(...)),
// where the body lives on the Request object, not the options argument:
const of = window.fetch;
window.fetch = async (...a) => {
  const req = a[0], o = a[1] || {};
  const u = typeof req === "string" ? req : req?.url;
  if (u?.includes("/graphql")) {
    let b = o.body;
    if (!b && req?.clone) b = await req.clone().text();
    console.log("GQL", b);
  }
  return of(...a);
};
```

Then I just *use the app* — search, open items, paginate, edit the profile — and it emits its
own queries and mutations, complete with variable names and the shape of the data. That's your
schema, reconstructed from behavior instead of introspection.

One gotcha that cost me time: some apps call `fetch(new Request(url, {body}))`, so the body
isn't on the second argument at all — it's on the Request object. Clone the request and read
`.text()` or you'll capture a pile of empty bodies. (A note on *where* you run this: injecting
JavaScript like this can trip bot-detection — more on that in a separate post. On a protected
target, capture the same traffic through an intercepting proxy instead.)

## 4. Attack what you found

Now the operations are the map. Two lines of attack pay off most often:

**IDOR / broken access control.** Any operation that takes an explicit `id`, `uuid`, or similar
is a question: *what happens if I change it to a value that isn't mine?* Make two accounts, do
the action as account A, then replay it with account B's identifier — or just walk the ID space.
If you get back data that isn't yours, that's the bug. A tell I've learned to look for:
ownership fields baked into responses, like `isYou` or `isOwner`. Those are exactly the checks
developers implement by hand, and exactly the ones they get wrong.

One nuance worth internalizing: an operation with **no** ID parameter — say a `myReservations`
query the server scopes to your session token — usually *can't* be IDOR'd, because you never got
to name the object. The vulnerable ones are the by-ID operations: fetch-one, cancel, edit,
share/invite. Spend your time there.

**Batching abuse.** If step 1 showed batching is on, you can put an **array of the same
mutation** into a single HTTP request — many logins, many OTP checks, many coupon redemptions in
one shot. That can slip straight past rate limits and anti-brute-force controls that only count
*requests*, not *operations inside a request*.

## The takeaway

Introspection being off doesn't mean a GraphQL API is secure — it just hides the map. The app
draws that map for you every time it loads. Harvest the operations, then treat every `id:` as an
access-control question and every batch as a rate-limit question. Same instinct as everything
else I've been learning: *untrusted input reaching a trusted context* — here, an identifier you
control reaching a query that forgot to check whether it's yours.
