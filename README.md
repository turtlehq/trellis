# Trellis Manifesto

_A compiler-first stack for human‑centered software._

---

## 1) Problem Statement

Modern web development is a Rube Goldberg machine: five languages, seven runtimes, a dozen libraries to do basics (data, auth, reactivity, styling, streaming). Teams drown in hydration taxes, ad‑hoc security, and brittle sync code. We want to build products, not plumbing.

**Thesis:** Collapse the stack into one language + compiler + runtime that proves security, plans data, and ships the minimum code for the job—HTML when static, tiny bytecode when interactive.

---

## 2) Non‑Negotiable Principles

1. **Single mental model.** One reactivity model (`state/derive/effect`) across server and client.
2. **Security as types.** Capabilities and access rules verified at compile time; authority is explicit and non‑serializable.
3. **Static by default.** Zero JS for static views; resumable islands only where interaction demands it.
4. **Offline‑first substrate.** CRDT‑backed docs and queries; conflict‑free by construction, with explicit business invariants.
5. **Explainable magic.** Every optimization is inspectable: what shipped, why, and with which proofs.
6. **Performance as a contract.** Budgets are declared; the compiler enforces them with actionable diffs.
7. **Interop, not isolation.** Clear FFI for UI and data; escape hatches and a degraded fallback target.

---

## 3) Core Guarantees (What Trellis Promises)

- **Capability safety:** You cannot compile code that reads/writes data without the correct proof token. Proofs cannot cross trust boundaries or serialize into client bytecode.
- **Deterministic derives:** `derive` blocks are pure; effects carry IO/time; purity violations are compile errors.
- **Minimal shipping unit:** Static → HTML. Interactive → resumable bytecode scoped to the island. No hydration cliffs.
- **Planned replication:** `query<T>` compiles to a materialized view with indexes; clients receive diffs, not dumps.
- **Invariant dignity:** Business rules are declared and enforced with clear repair strategies or typed conflicts.

---

## 4) Language Surface (tight and small)

### Reactivity

```trellis
state count = 0
derive doubled = count * 2
effect log = console.info(doubled)
```

### Data & Rules

```trellis
type Todo { id: id; title: text; done: bool; owner: user }
doc todos: Set<Todo>

rule Read(u): Rule<Todo>  = todos.where(t => t.owner == u)
rule Write(u): Rule<Todo> = todos.where(t => t.owner == u)
```

### Server Blocks with Capabilities

```trellis
script server with (db) {
  checker session(): Proof<Read(currentUser)> & Proof<Write(currentUser)>

  query list() using session() returns Subset<Todo> {
    return db.select(todos).where(t => t.owner == currentUser)
  }

  action add(title: text, can: Proof<Write(currentUser)>) {
    db.insert(todos, { id: newId(), title, done: false, owner: currentUser }, can)
  }
}
```

### UI & Progressive Enhancement

```trellis
view App {
  let data = list()
  <form enhance on:submit={add(title.value)}>
    <Input name=title />
    <Button>Add</Button>
  </form>
  <ul> for t in data stream { <li>{t.title}</li> } </ul>
}
```

### Invariants & Repair

```trellis
invariant WIP on todos ensure count(t where !t.done) <= 100 repair queueUntilOnline()
```

### Streams (files/SSE/WS)

```trellis
script server with (db) {
  pipe exportCSV() -> stream<bytes> {
    for row in db.scan(todos) { yield encodeCSV(row) }
  }
}

view Export { <Button download from={exportCSV}>Download</Button> }
```

---

## 5) Execution Model (how it actually runs)

1. **Capability slicing:** The compiler walks the graph, annotates nodes with required capabilities, and splits server/client by what code _does_, not where it lives.
2. **Auto‑islands & resumption:** Views without handlers/state serialize to HTML; islands capture a pure, serializable closure graph into a small bytecode blob. Resumption rebinds events and resumes effects.
3. **Replication graphs:** Each `query<T>` emits a plan (filter, order, projection, window) and picks indexes. Server maintains materialized views, clients keep causally consistent replicas.
4. **Backpressure by default:** `pipe` yields await downstream demand; navigation cancels streams; resources clean up via `onCancel`.
5. **Invariant enforcement:** Merges apply CRDT policies per field, then invariants. Violations trigger configured repair or raise typed conflicts into the UI.

---

## 6) Security Model (concise but formal)

- **Proofs as witnesses:** `Proof<R>` is an unforgeable witness that rule `R` holds.
- **Minting:** Proofs are minted by checkers at trust boundaries (e.g., `session()`), never by user code.
- **Flow:** A function that requires capability `R` must receive `Proof<R>`. Call sites without the proof do not type‑check.
- **Non‑serialization:** Proofs (and other resources: db handles, sockets) are `!Serialize`. Capturing them in resumable closures is a compile error with a precise trace.

_Sketch:_ `Γ ⊢ e : τ ▷ C`. Serialize if and only if `C = ∅` and `e` captures no `Proof<?> | Resource`.

---

## 7) Data Model & CRDTs (pragmatic set)

- **Shipped in v0:** LWW register (scalars), OR‑Set (tags), RGA list (ordered sequences).
- **Indexing:** Declarative covering indexes (`index TodosByOwner on todos (owner) covering (id,title,done)`).
- **Planner:** Chooses server vs client execution, exposes plan in dev tools. No “full table sync.”

---

## 8) Performance Contract

- **Budgets:** `@budget(js=40kb, css=12kb, rAF=1)` per view.
- **Enforcement:** Compile fails with a diff: which modules, by how much, suggested fixes.
- **Streaming SSR:** HTML first paint ASAP; islands fetch bytecode lazily (idle/visible/interaction heuristics with manual overrides).

---

## 9) DX: Tools that respect your time

- **Explain Panel:** shows shipped unit (HTML/bytecode), capability proofs in scope, replication plan, invariant events.
- **Time‑travel:** Scrub CRDT history and effects; export repro bundles.
- **Errors with fixes:** Diagnostics include one‑line code mods where safe.
- **Test kit:** Deterministic VM with DOM semantics; snapshot without hydration flake.

---

## 10) Interop & Escape Hatches

- **UI FFI:** Mount web components; adapters for React/Vue via `<Mount>` (no JSX in Trellis itself).
- **Data interop:** Import/export Automerge/Yjs; protocol documented.
- **Fallback target:** `trellis build --target=react-node` emits SSR + fetch endpoints (degraded, but portable).

---

## 11) What Trellis Refuses (anti‑features)

- Runtime CSS engines; styling is compile‑time or ordinary CSS.
- Untyped network calls from UI code.
- Global mutable singletons for shared state.
- “Magic” server components that smuggle authority to the client.

---

## 12) Governance & Compatibility

- **SemVer with receipts:** Every breaking change ships codemods and lints.
- **Stability windows:** LTS releases; security patches backported.
- **Transparency:** Roadmaps, RFCs, and compiler internals are public. No telemetry by default.

---

## 13) Adoption Wedge (v0 demo)

A **shared Kanban**: offline, presence‑optional, CSV export, WIP limits. One page proves:

- CRDT list ordering, invariants, optimistic updates/rollback
- Auto‑islands + resumability
- Planned replication with indexes
- Backpressured export stream

---

## 14) Roadmap (skeletal)

- **v0**: Signals, server blocks, proofs, 3 CRDTs, planner, islands, pipe, invariants, Explain Panel, Kanban demo.
- **v0.1**: Presence, design tokens (compile‑time), React adapter.
- **v0.2**: Jobs/cron, view transitions, more CRDTs, migration tooling.

---

## 15) FAQ (blunt)

- **Isn’t this lock‑in?** Interop + export paths are first‑class. Worst case, target React/Node.
- **Can I break invariants offline?** You can try. The merge layer repairs or returns a typed conflict you must resolve.
- **Where’s the DB?** Any pluggable store with index support. The compiler targets a narrow planner contract.
- **How big is the runtime?** Tiny. HTML for static; islands ship a small interpreter + graph state.
- **Debugging “magic”?** The Explain Panel shows every decision the compiler made. No black boxes.

---

## 16) Call to Builders

If the web is to remain our medium, the tools must serve product thinking, not ceremony. Trellis is a bet: that correctness, performance, and DX are not trade‑offs if the compiler carries its weight.

**Build fewer layers. Grow a Trellis.**
