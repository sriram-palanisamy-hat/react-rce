# The Invisible Handshake: Understanding the RCE Flaw in React Server Functions

A critical Remote Code Execution (RCE) vulnerability — **CVE‑2025‑55182** — recently shook the React ecosystem. It targeted **React Server Components (RSC)** and **React Server Functions**, exposing how a subtle deserialization bug could lead to arbitrary code execution on the server.

---

## Table of Contents

- [Affected Versions](#affected-versions)
- [Why This Matters](#why-this-matters)
- [A New Rendering Paradigm](#a-new-rendering-paradigm)
- [What Is the React Flight Protocol?](#what-is-the-react-flight-protocol)
- [Why React Server Components Exist](#why-react-server-components-exist)
- [Where Things Went Wrong](#where-things-went-wrong)
- [Prototype Pollution as the Root Cause](#prototype-pollution-as-the-root-cause)
- [From Pollution to RCE](#from-pollution-to-rce)
- [The Real‑World Exploit Path](#the-real-world-exploit-path)
- [The Fix: What Changed in PR #35277](#the-fix-what-changed-in-pr-35277)
- [Mitigation and What You Should Do](#mitigation-and-what-you-should-do)
- [Final Thoughts](#final-thoughts)

---

## Affected Versions

The vulnerability affects **React 19 Server Component infrastructure** prior to the December 2025 security patch.

### Vulnerable React Versions

- `react@19.0.0`
- `react@19.1.0`
- `react@19.1.1`
- `react@19.2.0`

### Affected Packages

- `react-server-dom-webpack`
- `react-server-dom-turbopack`
- `react-server-dom-parcel`

### Fixed Versions

Upgrade to **one** of the following:

- `19.0.1`
- `19.1.2`
- `19.2.1`

Frameworks such as **Next.js** that bundle RSC functionality released coordinated fixes. If you are using Server Actions, upgrading React alone is **not sufficient** — the framework must also be patched.

---

## Why This Matters

This vulnerability is **unauthenticated** and **pre‑render**. An attacker does not need:

- User credentials
- A valid session
- JavaScript execution in the browser

A single crafted HTTP request to a Server Function endpoint is enough.

This places CVE‑2025‑55182 in the highest severity category for production applications.

---

## A New Rendering Paradigm

Historically, React applications lived in one of two worlds:

- **Client-Side Rendering (CSR)** — logic runs entirely in the browser
- **Server-Side Rendering (SSR)** — the server renders HTML for each request

Both models have trade‑offs. CSR shifts work to the client but delays meaningful content. SSR improves first paint but still ships _all_ component logic to the browser.

**React Server Components (RSC)** introduce a fundamentally different idea:

> _Not all components need to exist in the browser._

With RSC, some components run **only on the server**. They can:

- Access databases directly
- Read from the filesystem
- Use secrets and credentials
- Execute heavy computation

…and none of that code is sent to the client.

Instead of HTML, the server returns a **description of the component tree** using a compact binary protocol. The browser receives instructions — not markup — and stitches together the final UI.

This model unlocks performance and architectural benefits that neither CSR nor SSR could achieve alone.

---

## What Is the React Flight Protocol?

The **React Flight Protocol (RFP)** is the transport layer that makes Server Components possible.

It is not JSON. It is a **streaming, chunk-based serialization format**.

### Example: Logical Representation

```txt
Chunk 0 → $1
Chunk 1 → { "message": "$2" }
Chunk 2 → "Hello"
```

### Example: What React Is Describing

```jsx
<ServerComponent message="Hello" />
```

Instead of HTML, the server sends instructions:

- Render this component
- With these props
- Insert client boundaries here

On the client, React reconstructs the tree:

```js
const element = createElement(ServerComponent, { message: "Hello" });
```

This reconstruction step is where deserialization occurs — and where security boundaries matter.

---

## Why React Server Components Exist

Before diving into the vulnerability, it helps to understand _why_ RSC exists at all.

### 1. Smaller JavaScript Bundles

Server Components never ship to the browser.

```jsx
// Server Component (no "use client")
import { db } from "@/db";

export async function UserProfile({ id }) {
  const user = await db.users.find(id);
  return <div>{user.name}</div>;
}
```

The above component:

- Runs only on the server
- Can access databases and secrets
- Adds **zero bytes** to the client bundle

Only **Client Components** — explicitly marked — are bundled for the browser:

```jsx
"use client";

import { useState } from "react";

export function LikeButton() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}</button>;
}
```

---

### 2. Direct Data Access

With RSC, components can access data sources like databases or filesystems directly, eliminating the need for a separate API layer. Data fetching logic co-exists with the component that uses it.

```jsx
// Server Component
const user = await db.users.find(123);
```

This simplifies the architecture and removes the serialization hop between an API route and the UI.

---

### 3. How Server Actions Trigger RSC Updates

Server Actions are the bridge from client-side interactions to server-side code execution within the RSC paradigm. They allow client components to securely call functions that run exclusively on the server.

When a Server Action is invoked (e.g., via a form submission), React serializes the function call and its arguments into the **React Flight Protocol** format and sends it to the server in an HTTP request.

The server-side React runtime then:

1.  **Deserializes** the incoming request payload.
2.  **Locates** the corresponding Server Action function (e.g., `likePost`).
3.  **Executes** the function with the provided arguments.
4.  **Re-renders** the affected Server Components to reflect any state changes.
5.  **Streams** the new UI description back to the client as a Flight response.

The client receives this stream and seamlessly updates the DOM. From a developer’s perspective, it feels like a local function call, but it’s a full-cycle RPC mechanism that is central to the vulnerability. This tight integration of serialization and execution is what the exploit targets.

It's critical to understand that you are vulnerable if your server handles React Server Components (RSC), even if you define zero Server Functions. The exploit occurs during the initial handling of the `next-action` header, before any action validation even begins.

#### Example: A `like` Button

```jsx
"use server";

import { revalidatePath } from "next/cache";
import { db } from "@/db";

async function likePost(postId: string) {
  "use server";
  await db.posts.update({
    where: { id: postId },
    data: { likes: { increment: 1 } },
  });
  revalidatePath("/");
}

export function LikeButton({ postId }: { postId: string }) {
  return (
    <form action={likePost.bind(null, postId)}>
      <button type="submit">Like</button>
    </form>
  );
}
```

---

### 4. Server-First Mental Model

RSC flips the default assumption:

```txt
Old: UI → API → DB
New: UI (server) → DB
```

The server becomes the primary execution environment. The client is for interactivity only.

---

### 5. Progressive Streaming

Server Components stream incrementally:

```jsx
import { Suspense } from "react";

export default async function Page() {
  return (
    <>
      <Header />
      <Suspense fallback={<Spinner />}>
        <SlowContent />
      </Suspense>
    </>
  );
}
```

The browser receives UI as soon as each piece is ready.

---

## Where Things Went Wrong

During deserialization, React resolves _paths_ like:

```
$1:propertyA:propertyB
```

If `propertyA` does not exist, JavaScript’s normal behavior applies:

> Walk up the prototype chain.

This is usually harmless — until user‑controlled input can influence the path.

---

## Prototype Pollution as the Root Cause

During deserialization, React resolves reference paths like:

```txt
$1:propertyA:propertyB
```

If `propertyA` does not exist, JavaScript walks the prototype chain.

The `__proto__` property (or `[[Prototype]]` internal slot) in JavaScript is a reference to the prototype object of an object. When you try to access a property on an object, and that property isn't directly on the object itself, JavaScript will automatically look up the `__proto__` chain to find it on the prototype object, then its prototype, and so on, until it reaches `null`. This mechanism allows objects to inherit properties and methods from other objects.

### Malicious Traversal

```txt
Object
→ Object.__proto__
→ Object.__proto__.constructor
→ Object.__proto__.constructor.constructor
```

Which resolves to:

```js
Function;
```

This happens because:

```js
({}).constructor.constructor === Function;
```

Once an attacker gains access to `Function`, arbitrary code execution becomes possible.

---

## From Pollution to RCE

Access to the `Function` constructor enables runtime code execution.

### Execution Primitive

```js
const fn = Function("return process")();
fn.mainModule.require("child_process").execSync("id");
```

### How React Was Tricked

The exploit injected `Function` into an internal call path that React _expects_ to invoke:

```js
response._formData.get(response._prefix + "0");
```

After prototype pollution, this effectively became:

```js
Function(attackerControlledString);
```

At that point, deserialization turns into execution.

The attack arrives as a `multipart/form-data` request that looks like a legitimate Server Function call.

Key traits:

- Crafted Flight chunks
- Fake promise‑like objects
- Self‑referential chunk loops

React attempts to resolve the data — and executes attacker‑controlled code instead.

---

## Effects: The Cloudflare Outage

The Cloudflare outage on December 5, 2025, was a direct, unintentional consequence of an emergency response to this RCE. Cloudflare increased their WAF buffer to 1MB to catch these complex RFP payloads but accidentally triggered a 15-year-old nil value bug in their legacy Lua proxy when they used a "killswitch" to disable an incompatible testing tool. This resulted in 500 errors for 28% of global traffic.

---

## Reproducing the Vulnerability

A sample project is available on GitHub to demonstrate this vulnerability.

**1. Clone the repository:**

```bash
git clone git@github.com:sriram-palanisamy-hat/react-rce.git
cd react-rce
```

**2. Install dependencies:**

This project uses `pnpm` for package management.

```bash
pnpm install
```

**3. Run the vulnerable server:**

The exploit script targets port 3001. Run the Next.js development server on this port.

```bash
pnpm dev --port 3001
```

**4. Execute the exploit:**

In a separate terminal, run the exploit script:

```bash
node rce-exploit.js
```

If successful, the script will create a file named `test.txt` in the project root with the content `hi how you doing`, confirming that Remote Code Execution was achieved.

## The Fix: What Changed in PR #35277

The core fix is deceptively small.

**Before:**

```js
moduleExports[name];
```

**After:**

```js
if (hasOwnProperty.call(moduleExports, name)) {
  return moduleExports[name];
}
return undefined;
```

### Why This Matters

- Prevents prototype chain traversal
- Blocks `__proto__`, `constructor`, and inherited access
- Turns dangerous paths into harmless `undefined`

This change was applied consistently across all Server DOM implementations.

---

## Mitigation and What You Should Do

1. Upgrade React to a patched version
2. Upgrade your framework (Next.js, Remix, etc.)
3. Audit any custom Server Function handlers
4. Treat deserialization as **untrusted input**

If you cannot upgrade immediately, **disable Server Functions** as a temporary mitigation.

---

## Final Thoughts

This vulnerability wasn’t caused by unsafe code execution APIs — it emerged from _perfectly valid JavaScript behavior_ interacting with a powerful serialization format.

As React moves deeper into server‑first architectures, security boundaries matter more than ever. **Serialization is code execution adjacent** — and must be treated with the same caution.

The fix in [PR #35277](https://github.com/facebook/react/pull/35277) closes this chapter, but the lesson remains:

> Complexity is where security bugs hide.

Stay patched. Stay curious.
