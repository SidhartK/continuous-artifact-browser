# How Chromium Builds the Accessibility Tree (from the DOM)

A curated, visualization-first walkthrough of the Blink module that constructs
the **accessibility tree** (the "AX tree") out of the DOM + Layout trees in the
open-source Chromium browser.

> **What "the accessibility tree parser" is, concretely:**
> It is not a single parser class. It is a Blink module —
> `third_party/blink/renderer/modules/accessibility/` — whose central engine is
> the class **`AXObjectCacheImpl`**. It walks the DOM/Layout trees, decides
> which nodes deserve an accessibility object, builds a tree of **`AXObject`s**,
> and (via **`BlinkAXTreeSource`**) serializes them into the cross-process
> **`ui::AXNodeData`** format. This doc reverse-documents that engine.

- The curated upstream source lives in [`chromium-source/`](./chromium-source/)
  (see [`PROVENANCE.md`](./chromium-source/PROVENANCE.md) for exact paths/license).
- Everything below is original explanation; **all C++/code excerpts are quoted
  from the real Chromium source** and cite their file.

---

## Table of contents

1. [Executive summary](#1-executive-summary)
2. [Where the code lives (module map)](#2-where-the-code-lives-module-map)
3. [The three trees: DOM → Layout → AX](#3-the-three-trees-dom--layout--ax)
4. [The cast of classes](#4-the-cast-of-classes)
5. [The construction algorithm = the lifecycle state machine](#5-the-construction-algorithm--the-lifecycle-state-machine)
6. [Per-node decision: `DetermineAXObjectType`](#6-per-node-decision-determineaxobjecttype)
7. [`AXObject` creation: `GetOrCreate` / `CreateAndInit`](#7-axobject-creation-getorcreate--createandinit)
8. [Ignored vs. Included: the two-axis visibility model](#8-ignored-vs-included-the-two-axis-visibility-model)
9. [Serialization & incremental updates](#9-serialization--incremental-updates)
10. [End-to-end worked example](#10-end-to-end-worked-example)
11. [Glossary & references](#11-glossary--references)

---

## 1. Executive summary

Chromium does **not** expose the DOM directly to screen readers. Instead, the
Blink renderer maintains a parallel **accessibility tree** that is *derived
from* the DOM and Layout trees, cached, and shipped — as incremental deltas —
to the browser process, which then drives each OS's native accessibility API.

The engine that builds this tree is **`AXObjectCacheImpl`** (a cache + tree
builder + event router rolled into one). Its construction model has five
defining ideas:

1. **One wrapper per relevant node.** For each DOM `Node` / `LayoutObject` that
   matters for accessibility, the cache creates one **`AXObject`** (almost
   always an **`AXNodeObject`**). Irrelevant subtrees (`<head>`, `<script>`,
   `display:none`, `aria-hidden`, etc.) are *pruned*.

2. **Layout-aware, not DOM-shaped.** The AX tree resembles the DOM but is
   reshaped by CSS and ARIA: generated content (list bullets, `::before`) adds
   nodes with no DOM node; `display:none` removes them; `aria-owns` reparents
   them. Whether a node exists at all often depends on its **`LayoutObject`**.

3. **Deferred, batched, lifecycle-gated.** Blink fires hundreds of
   `Handle…`/`…Changed` notifications as the page mutates. The cache does **not**
   react immediately — it *queues* "tree update" work and processes it only when
   layout/style is **clean**, marching through an explicit five-state
   **lifecycle**: `DeferTreeUpdates → ProcessDeferredUpdates → FinalizingTree →
   Serialize` (then back to deferring). Each state strictly governs what the
   code is allowed to touch.

4. **Ignored ≠ absent.** A node can be *ignored* (hidden from assistive tech)
   yet still *included* in the serialized tree (so the browser side can compute
   names, line breaks, relationships). Inclusion and ignored-ness are two
   independent bits.

5. **Serialize deltas, not snapshots.** A general-purpose `AXTreeSerializer`
   diffs the tree against the last-sent state and emits the smallest possible
   `AXTreeUpdate`. **`BlinkAXTreeSource`** translates each `AXObject` into a
   compact, attribute-array-based **`ui::AXNodeData`**, which crosses the
   process boundary via a Mojo IPC (`RenderAccessibilityHost::HandleAXEvents`).

**One-line mental model:**

```
DOM + Layout  ──AXObjectCacheImpl──▶  AXObject tree  ──BlinkAXTreeSource──▶
  ui::AXNodeData deltas  ──Mojo IPC──▶  browser cache  ──▶  OS a11y APIs / screen reader
```

The remaining sections unpack each arrow.

---

## 2. Where the code lives (module map)

The "parser" is the **renderer-side Blink module**. The pipeline then continues
into the content layer and browser process.

```
RENDERER PROCESS (sandboxed, one per site)
│
├─ third_party/blink/renderer/core/dom/                 ← Node, Document (the DOM)
├─ third_party/blink/renderer/core/layout/              ← LayoutObject (the Layout tree)
│
├─ third_party/blink/renderer/modules/accessibility/    ◀══ THE AX TREE PARSER MODULE
│    ├─ ax_object_cache_impl.{h,cc}      ← engine: builds/caches/updates the tree  (~270KB .cc)
│    ├─ ax_object_cache_lifecycle.h      ← the 5-state construction state machine
│    ├─ ax_object.{h,cc}                 ← abstract base node of the AX tree
│    ├─ ax_node_object.{h,cc}            ← concrete node wrapping a Node (+ LayoutObject)
│    ├─ ax_relation_cache.{h,cc}         ← aria-owns / labelledby / describedby graph
│    ├─ blink_ax_tree_source.{h,cc}      ← AXObject ─▶ ui::AXNodeData translation
│    └─ inspector_accessibility_agent.*  ← DevTools "Full accessibility tree"
│
├─ content/renderer/accessibility/
│    ├─ render_accessibility_impl.{h,cc}     ← drives serialization, owns the serializer
│    └─ render_accessibility_manager.{h,cc}  ← Mojo endpoint to the browser
│
─────────────────────────── Mojo IPC boundary ───────────────────────────
│
BROWSER PROCESS (one, trusted, talks to the OS)
├─ content/browser/renderer_host/render_frame_host_impl.*  ← receives AX events
└─ ui/accessibility/platform/
     ├─ browser_accessibility_manager.*   ← merges per-frame trees into ONE window tree
     └─ ax_platform_node.*                ← implements IAccessible2 / NSAccessibility / ATK
```

Cross-cutting data type (shared, renderer **and** browser):

```
ui/accessibility/ax_node_data.h        ← ui::AXNodeData  (the wire/cache node format)
ui/accessibility/ax_tree_serializer.h  ← generic incremental tree-diff serializer
```

---

## 3. The three trees: DOM → Layout → AX

A web page is represented by **three** parallel trees. The AX tree is the third,
and it is *built from the first two*.

```
        HTML  ─parse─▶   DOM tree        (semantic structure: elements & text)
                              │
        CSS   ──────────▶  + style  ─▶   LAYOUT tree   (boxes that are actually rendered;
                              │                          display:none → NO layout object)
                              │
                              ▼
                          AX tree        (what assistive tech sees)
```

Why the AX tree needs **both** inputs (paraphrasing
[`how_a11y_works.md`](./chromium-source/docs/how_a11y_works.md) and
[`overview.md`](./chromium-source/docs/overview.md)):

| Phenomenon | Effect on AX tree | Driven by |
|---|---|---|
| `<head>`, `<script>`, `<style>` | pruned (never rendered) | no LayoutObject |
| `display:none` | node **absent** from AX tree | no LayoutObject |
| `visibility:hidden` | node ignored, *descendants may still show* | Layout/style |
| CSS `::before` / `::after`, list bullets | **extra** AX nodes with *no DOM node* | LayoutObject only |
| `aria-hidden="true"` | subtree removed from AX tree (still rendered!) | ARIA |
| `role="presentation"`/`none` | node removed, **children kept** | ARIA |
| `aria-owns` | subtree **reparented** in AX tree | ARIA |

> This is exactly *why* the wrapper for a node is usually an **`AXNodeObject`
> that also holds a `LayoutObject`** — Blink frequently needs layout to decide
> if/how a node appears. Quoting the overview:
>
> > *"the most commonly used concrete subclass … is `AXNodeObject`, which wraps
> > a `Node`. In turn, most AXNodeObjects are actually `AXLayoutObject`s, which
> > wrap both a `Node` and a `LayoutObject`. Access to the LayoutObject is
> > important because some elements are only in the AXObject tree depending on
> > their visibility, geometry, linewrapping, and so on."*

ASCII version of the "not 1:1" reality:

```
   DOM                          AX tree
   ───                          ───────
   <ol>                         list
    ├─ <li>Apple                 ├─ listItem
    │                            │    ├─ listMarker  "1."   ◀─ NO DOM node (generated)
    │                            │    └─ staticText  "Apple"
    └─ <li>Pear                  └─ listItem
                                      ├─ listMarker  "2."   ◀─ NO DOM node (generated)
                                      └─ staticText  "Pear"

   <aside aria-hidden="true">   (entire subtree pruned — present in DOM,
      …sidebar…                  rendered on screen, but invisible to AT)
   </aside>
```

---

## 4. The cast of classes

```
                         ┌──────────────────────────────────────────┐
   Node / LayoutObject   │            AXObjectCacheImpl              │
   (DOM + Layout) ──────▶│  • owns AXID → AXObject maps              │
                         │  • DeferTreeUpdate() queues work          │
                         │  • CommitAXUpdates() runs the lifecycle   │
                         │  • routes a11y events to the content layer│
                         └───────────────┬──────────────────────────┘
                                         │ creates / owns
                                         ▼
                          ┌──────────────────────────────┐
                          │           AXObject            │   abstract base
                          │  role, name, bounds, children │
                          │  IsIgnored(), IsIncludedInTree│
                          └───────────────┬──────────────┘
                                          │ (almost always)
                                          ▼
                          ┌──────────────────────────────┐
                          │         AXNodeObject          │   concrete; wraps a Node,
                          │  (usually also a LayoutObject)│   computes role/name from DOM+CSS+ARIA
                          └───────────────┬──────────────┘
                                          │ serialized by
                                          ▼
   ┌──────────────────────┐   translate  ┌──────────────────────────────┐
   │   BlinkAXTreeSource   │◀────────────│  reads role/name/bounds/etc.  │
   │  AXObject ▶ AXNodeData│             └──────────────────────────────┘
   └──────────┬───────────┘
              ▼
        ui::AXNodeData   (compact, attribute-array node → crosses the process boundary)
```

**`AXObjectCacheImpl`** — the engine. Its own header documents its dual role
([`ax_object_cache_impl.h`](./chromium-source/blink-accessibility/ax_object_cache_impl.h)),
and the overview summarizes it:

> > *"The central class responsible for dealing with accessibility events in
> > Blink is `AXObjectCacheImpl`, which is responsible for caching the
> > corresponding AXObjects for Nodes or LayoutObjects. This class has many
> > methods named `handleFoo`, which are called throughout Blink to notify the
> > AXObjectCacheImpl that it may need to update its tree."*

**`BlinkAXTreeSource`** — the translator. It implements the generic
`ui::AXTreeSource` interface so the serializer can walk it
([`blink_ax_tree_source.h`](./chromium-source/blink-accessibility/blink_ax_tree_source.h)):

```cpp
// blink_ax_tree_source.h  — the tree-walking + node-serializing contract
class MODULES_EXPORT BlinkAXTreeSource
    : public ui::AXTreeSource<const AXObject*, ui::AXTreeData*, ui::AXNodeData> {
  const AXObject* GetRoot() const override;                 // tree root
  const AXObject* GetFromId(int32_t id) const override;     // AXID → AXObject
  size_t GetChildCount(const AXObject* node) const override;
  AXObject* ChildAt(const AXObject* node, size_t) const override;
  AXObject* GetParent(const AXObject* node) const override;
  // THE translation step: one AXObject → one ui::AXNodeData
  void SerializeNode(const AXObject* node, ui::AXNodeData* out_data) const override;
  void Freeze();   // lock the tree so a consistent snapshot can be serialized
  void Thaw();
};
```

**`ui::AXNodeData`** — the destination format. To avoid hundreds of fields per
node, attributes are stored in **typed arrays** keyed by an enum
(from [`overview.md`](./chromium-source/docs/overview.md)):

```cpp
struct AXNodeData {
  int32_t id;
  ax::mojom::Role role;
  // …everything else is sparse, stored by attribute key:
  std::vector<std::pair<ax::mojom::StringAttribute, std::string>> string_attributes;
  std::vector<std::pair<ax::mojom::IntAttribute,    int32_t>>     int_attributes;
  // … float_attributes, bool_attributes, intlist_attributes, …
};
```

---

## 5. The construction algorithm = the lifecycle state machine

**The single most important file for understanding *how the tree is built* is
[`ax_object_cache_lifecycle.h`](./chromium-source/blink-accessibility/ax_object_cache_lifecycle.h).**
Tree construction is not a one-shot recursive descent; it is a *state machine*
that the cache marches through every frame, mirroring Blink's `DocumentLifecycle`.

The header's own words:

> > *"AXObjectCacheLifecycle describes which step of the a11y tree update
> > algorithm is currently running. It borrows concepts from DocumentLifecycle.
> > Benefits: Ensure that code paths do not attempt to access data, such as
> > layout or style, when it is unsafe to do so … Ensure completeness of
> > generated data, such as the tree structure and cached properties …"*

### The states

```cpp
// ax_object_cache_lifecycle.h
enum LifecycleState {
  kUninitialized,
  kDeferTreeUpdates,        // (1) listen & QUEUE work; layout is DIRTY, do not read it
  kProcessDeferredUpdates,  // (2) layout now CLEAN; apply queued updates, build subtrees
  kFinalizingTree,          // (3) make tree final: every node has children & cached values
  kSerialize,               // (4) tree FROZEN; emit AXNodeData deltas + events
  kDisposing,
  kDisposed,
};
```

### The cycle (runs once per rendering update, then loops)

```
                 DOM / Layout / ARIA mutations fire Handle…() / …Changed()
                                       │
                                       ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │ (1) kDeferTreeUpdates                                                 │
  │     • DeferTreeUpdate(reason, node) pushes a TreeUpdateReason onto    │
  │       a callback queue (main doc + popup doc).                        │
  │     • Layout/style are DIRTY here → must NOT be read.                 │
  │     • This is the *only* state where new updates may be queued.       │
  └───────────────────────────────┬─────────────────────────────────────┘
                                   │  CommitAXUpdates() — once layout is clean
                                   ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │ (2) kProcessDeferredUpdates                                          │
  │     • Drain the queues; for each reason, update structure & cached   │
  │       values. Create AXObjects (GetOrCreate), reparent, remove.      │
  │     • Layout is CLEAN → safe to read geometry/style.                  │
  │     • Dirty objects + events are queued for the serializer.          │
  └───────────────────────────────┬─────────────────────────────────────┘
                                   │  FinalizeTree()
                                   ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │ (3) kFinalizingTree                                                  │
  │     • Walk the tree so EVERY node has computed its children and      │
  │       no node is orphaned. Tree model is now final.                  │
  │     • Reparenting still allowed; result must be a clean acyclic tree.│
  └───────────────────────────────┬─────────────────────────────────────┘
                                   │  SerializeAXUpdatesIfNeeded()
                                   ▼
  ┌─────────────────────────────────────────────────────────────────────┐
  │ (4) kSerialize    (tree is FROZEN)                                   │
  │     • NO AXObject creation or cached-value changes allowed.          │
  │     • BlinkAXTreeSource.SerializeNode() → ui::AXNodeData per dirty   │
  │       node; AXTreeSerializer produces a minimal AXTreeUpdate delta.  │
  │     • Send to browser via Mojo HandleAXEvents().                      │
  └───────────────────────────────┬─────────────────────────────────────┘
                                   │  EnsureStateAtMost(kDeferTreeUpdates)
                                   └──────────────▶ back to (1)
```

### The states are *enforced*, not advisory

The lifecycle exposes `StateAllows…()` predicates that the rest of the code
`CHECK()`s against. This is the formal contract for "what may happen when":

```cpp
// ax_object_cache_lifecycle.h  — capabilities per state (verbatim)
bool StateAllowsImmediateTreeUpdates() const {            // build/modify nodes now?
  return state_ == kProcessDeferredUpdates || state_ == kFinalizingTree;
}
bool StateAllowsReparentingAXObjects() const {            // move subtrees (aria-owns)?
  return state_ == kDeferTreeUpdates || state_ == kProcessDeferredUpdates;
}
bool StateAllowsSerialization() const {                   // emit AXNodeData?
  return state_ == kSerialize;
}
bool StateAllowsAXObjectsToBeDirtied() const {            // mark node needs-resend?
  return state_ == kDeferTreeUpdates || state_ == kProcessDeferredUpdates ||
         state_ == kFinalizingTree;                       // …but NOT during kSerialize
}
```

### The same machine, seen in `CommitAXUpdates()`

The transitions above are literally the body of `AXObjectCacheImpl::CommitAXUpdates()`
in `ax_object_cache_impl.cc` (state advances quoted verbatim, interleaving
trimmed):

```cpp
// ax_object_cache_impl.cc  — AXObjectCacheImpl::CommitAXUpdates(...)
lifecycle_.AdvanceTo(AXObjectCacheLifecycle::kProcessDeferredUpdates);   // (2)
  // … drain tree_update_callback_queue_main_ / _popup_ …
  lifecycle_.AdvanceTo(AXObjectCacheLifecycle::kFinalizingTree);         // (3)
  FinalizeTree();   // "Build out tree, such that each node has computed its children."
  CHECK(tree_update_callback_queue_main_.empty());
  DCHECK(!IsDirty());
// …later, in SerializeAXUpdatesIfNeeded():
lifecycle_.AdvanceTo(AXObjectCacheLifecycle::kSerialize);                // (4)
```

> **Why deferral matters (performance):** web pages mutate constantly. Reacting
> to every mutation synchronously would thrash. Batching all changes and
> applying them once layout is clean is what keeps accessibility cheap enough to
> leave on. The cache even rate-limits how often it serializes.

---

## 6. Per-node decision: `DetermineAXObjectType`

When the builder reaches a `Node`/`LayoutObject`, the first question is:
**"Should this become an AXObject — and if so, built from the Node or the
LayoutObject — or should the whole subtree be pruned here?"** That is
`DetermineAXObjectType()` in `ax_object_cache_impl.cc`. Its result enum:

```cpp
// ax_object_cache_impl.h
enum AXObjectType {
  kPruneSubtree = 0,   // create nothing; cut the subtree at this point
  kCreateFromNode,     // make an AXNodeObject backed by the DOM Node
  kCreateFromLayout,   // make an AXNodeObject backed by the LayoutObject (preferred)
};
```

### Decision tree (ASCII)

```
DetermineAXObjectType(node, layout_object, ax_mode, parent_known)
│
├─ display-locked (content-visibility) & no screen reader?  ── yes ─▶ PRUNE
│      └─ (screen reader on) → ignore layout_object, judge by node
│
├─ node present?
│   ├─ not isConnected()                                     ── yes ─▶ PRUNE
│   ├─ in shadow tree & not a11y-relevant                    ── yes ─▶ PRUNE
│   ├─ not Element and not Text (e.g. document/doctype)
│   │       └─ has layout? CREATE_FROM_LAYOUT : PRUNE
│   ├─ Element in a forbidden/pruned subtree                 ── yes ─▶ PRUNE
│   │       └─ else node is "relevant"
│   └─ Text node:
│           has layout? → layout-relevant? CREATE_FROM_LAYOUT : PRUNE
│           no layout?  → hidden-text-relevant? mark node relevant
│
├─ layout_relevant := layout_object && IsLayoutObjectRelevantForAccessibility(...)
│
├─ neither layout_relevant nor node_relevant                 ──────▶ PRUNE
│
├─ not layout_relevant AND inside hidden head/style/script
│        or DOM-descendant of an iframe                       ──────▶ PRUNE
│
└─ layout_relevant ? CREATE_FROM_LAYOUT : CREATE_FROM_NODE
```

### The rules, in the source's own comment + the decisive lines

```cpp
// ax_object_cache_impl.cc  (comment verbatim)
// DetermineAXObjectType() determines what type of AXObject should be created
// for the given node and layout_object.
// * If neither the node nor layout object are relevant for accessibility, will
//   return kPruneSubtree … result in the entire subtree being pruned …
// * If both the node and layout are relevant, kCreateFromLayout is preferred,
//   otherwise: kCreateFromNode for relevant nodes, kCreateFromLayout for layout.
AXObjectType DetermineAXObjectType(const Node* node,
                                   const LayoutObject* layout_object,
                                   ui::AXMode ax_mode,
                                   bool parent_ax_known) {
  // … (connection / shadow / element-vs-text checks above) …

  bool is_layout_relevant =
      layout_object && IsLayoutObjectRelevantForAccessibility(*layout_object);

  // Prune if neither the LayoutObject nor Node are relevant.
  if (!is_layout_relevant && !is_node_relevant)
    return kPruneSubtree;

  // If a node is not rendered, prune if it is in head/style/script or a DOM
  // descendant of an iframe.
  if (!is_layout_relevant && IsInPrunableHiddenContainerInclusive(
                                 *node, parent_ax_known, is_display_locked)) {
    return kPruneSubtree;
  }

  return is_layout_relevant ? kCreateFromLayout : kCreateFromNode;   // ◀ the verdict
}
```

**Takeaway:** "preferred to build from the LayoutObject" is the mechanical
restatement of "the AX tree follows what is *rendered*." Nodes with no layout
survive only as a deliberate exception (e.g. hidden text needed for a name, or
nodes referenced by `aria-labelledby`).

---

## 7. `AXObject` creation: `GetOrCreate` / `CreateAndInit`

Once the type is decided, the cache materializes the wrapper. The header is
explicit that creation is *not guaranteed* —
[`ax_object_cache_impl.h`](./chromium-source/blink-accessibility/ax_object_cache_impl.h):

```cpp
// ax_object_cache_impl.h
// Create an AXObject, and do not check if a previous one exists.
// Also, initialize the object and add it to maps for later retrieval.
AXObject* CreateAndInit(Node*, LayoutObject*, AXObject* parent);

// Note that these functions do NOT guarantee that an AXObject will
// be created. For instance, not all HTMLElements can have an AXObject,
// such as <head> or <script> tags.
AXObject* GetOrCreate(LayoutObject*, AXObject* parent);
AXObject* GetOrCreate(const Node*, AXObject* parent) override;
```

Pseudocode of the create path (synthesized from the header + `.cc`):

```
GetOrCreate(node_or_layout, parent):
    existing = Get(node_or_layout)          # look up AXID → AXObject map
    if existing: return existing

    type = DetermineAXObjectType(node, layout, ax_mode, parent_known)
    if type == kPruneSubtree:
        return nullptr                      # subtree stops here

    obj = CreateAndInit(node, layout, parent):
        ax_object = new AXNodeObject(...)   # concrete subclass chosen by type
        ax_id     = GetOrCreateAXID(ax_object)     # assign unique-within-frame AXID
        objects_[ax_id] = ax_object                # register in cache maps
        ax_object.Init(parent)                     # compute cached role/ignored/…
        return ax_object
    return obj
```

Identity & lookup are by **`AXID`** (an `int32`, unique *within a frame*). The
cache holds the authoritative `AXID → AXObject` maps; `BlinkAXTreeSource.GetFromId()`
and `GetId()` ride on top of them. Children are computed lazily and finalized in
the `kFinalizingTree` stage so that *every* included node ends with a settled
child list.

---

## 8. Ignored vs. Included: the two-axis visibility model

A frequent source of confusion: a node being *hidden from a screen reader* and a
node being *absent from the tree* are **different things**. Blink models them as
two independent booleans on every `AXObject`
([`ax_object.h`](./chromium-source/blink-accessibility/ax_object.h)):

```cpp
// ax_object.h
// Whether objects are included in the tree. Nodes that are included in the
// tree are serialized, even if they are ignored. This allows browser-side
// accessibility code to have a more accurate representation of the tree …
bool IsIncludedInTree() const;

// Whether objects are ignored, i.e. hidden from the AT.
bool IsIgnored() const;

// Whether an ignored object should still be included in the serialized tree.
bool IsIgnoredButIncludedInTree() const;
```

The 2×2 it produces:

```
                         IsIncludedInTree?
                    ┌──────────────┬──────────────┐
                    │     true     │     false    │
        ┌───────────┼──────────────┼──────────────┤
        │  Ignored  │ in tree but  │ not present  │
 Is     │  = true   │ hidden from  │ at all       │
 Ignored│           │ AT (kept for │ (truly       │
        │           │ name calc /  │ pruned)      │
        │           │ line breaks /│              │
        │           │ relations)   │              │
        ├───────────┼──────────────┼──────────────┤
        │  Ignored  │ NORMAL: a    │  (n/a — an   │
        │  = false  │ real exposed │   exposed    │
        │           │ control      │   node is    │
        │           │              │   included)  │
        └───────────┴──────────────┴──────────────┘
```

Why keep ignored-but-included nodes (from the same header)? For *recursive name
computation* over hidden subtrees, to mark *line-breaking* positions, to host
*language* info, and for *bookkeeping* when DOM traversal isn't safe. The browser
side wants a richer tree than the screen reader ultimately hears.

The ignored decision itself fans out into composable predicates — e.g.
`ComputeIsIgnored()`, `ComputeIsAriaHidden()`, `ComputeIsInert()`,
`IsHiddenViaStyle()` — each cached on the object and recomputed during
`kProcessDeferredUpdates`.

---

## 9. Serialization & incremental updates

The tree is built so it can be **shipped cheaply and repeatedly**. Three ideas
do the heavy lifting (all from
[`overview.md`](./chromium-source/docs/overview.md)).

### (a) Generic incremental serializer

> > *"Chromium has a general-purpose tree serializer class that's designed to
> > send small incremental updates of a tree from one process to another."*

Its only requirements:

```
• every node has a unique integer ID
• the tree is acyclic
• the serializer is told when a node's data changes
• the serializer is told when a node's child-ID list changes
```

It diffs against the previously sent state and walks only what changed:

```
   last-sent tree            current tree            AXTreeUpdate (sent)
   ──────────────            ────────────            ───────────────────
        A                         A                   { update node C,
       / \                       / \                    add node E under C }
      B   C                     B   C                  (A, B, D untouched →
          |                        / \                  not serialized)
          D                       D   E◀new
```

### (b) Compact, sparse node format

`ui::AXNodeData` stores only the attributes a node actually has, in typed arrays
(see §4). A placeholder string, for instance, is just one entry appended to
`string_attributes` under `kPlaceholder`.

### (c) Relative geometry (no re-serialize on scroll)

Each node's bounds are stored **relative to an offset container**, optionally
with scroll offsets and a 4×4 transform that apply to the whole subtree.
Global screen coordinates are computed by walking ancestors:

```
                      screen_bounds(n) = bounds(n)
   for each ancestor a from n up to root:
       screen_bounds(n) = T(a) · ( screen_bounds(n) − scroll(a) ) + offset(a)

   where  T(a) = 4×4 transform of offset-container a   (identity if none)
```

So scrolling or animating a container updates **only that container's** node, not
its entire subtree.

### (d) The handoff out of the renderer

```
AXObject (frozen)
   │  BlinkAXTreeSource::SerializeNode()
   ▼
ui::AXNodeData  ──▶ AXTreeSerializer diffs ──▶ AXTreeUpdate (delta)
   │
   │  Mojo: ax.mojom.RenderAccessibilityHost::HandleAXEvents()
   ▼
RenderFrameHostImpl (browser) ──▶ BrowserAccessibilityManager
   • merges per-frame trees into ONE window tree (site isolation: 1 tree / frame)
   • fires native events → IAccessible2 / NSAccessibility / ATK / AccessibilityNodeInfo
```

The serialization queue is itself lifecycle-aware. From `ax_object_cache_impl.cc`:

> > *"1) Dirty objects and events are fired through
> > `AXObjectCacheImpl::PostPlatformNotification` which in turn makes a call to
> > `AXObjectCacheImpl::AddEventToSerializationQueue` to queue it. 2) When the
> > lifecycle is ready to be serialized, `AXObjectCacheImpl::CommitAXUpdates` is
> > called which first checks if it's time to make a new serialization, and if
> > not, it will early return in order to add a delay between serializations."*

---

## 10. End-to-end worked example

Take the form from the Chromium overview doc:

```html
<html>
<head><title>How old are you?</title></head>
<body>
  <label for="age">Age</label>
  <input id="age" type="number" name="age" value="42">
  <div>
    <button>Back</button>
    <button>Next</button>
  </div>
</body>
</html>
```

**Step 1 — DOM + Layout exist.** `<head>`/`<title>` have no rendered layout;
`<body>` and its children do.

**Step 2 — cache walks from the root, calling `DetermineAXObjectType` per node:**

```
<html>   → root WebArea            (kCreateFromLayout)
  <head>   → PRUNE (in hidden head container, no layout)
  <body>
    <label>   → relevant           (kCreateFromLayout)  role=Label
    <input>   → relevant           (kCreateFromLayout)  role=TextField
    <div>     → relevant           (kCreateFromLayout)  role=Group
      <button>Back</button> → relevant (kCreateFromLayout) role=Button
      <button>Next</button> → relevant (kCreateFromLayout) role=Button
```

**Step 3 — finalize tree; relation cache resolves `for="age"` → labelledby.**

**Step 4 — serialize each `AXObject` to `ui::AXNodeData`.** Result (the exact
tree the overview shows):

```
id=1 role=WebArea   name="How old are you?"
    id=2 role=Label     name="Age"
    id=3 role=TextField labelledByIds=[2]  value="42"
    id=4 role=Group
        id=5 role=Button name="Back"
        id=6 role=Button name="Next"
```

Note the reshaping vs. the DOM: `<head>` is gone; the text field's *name* is not
inline but a **relation** (`labelledByIds=[2]`) pointing at the label node;
structure is otherwise DOM-like but "slightly simplified."

**Step 5 — a later edit** (user types in the field) fires a `value changed`
notification → queued in `kDeferTreeUpdates` → on the next clean frame only
node `id=3` is re-serialized and shipped as a one-node delta.

---

## 11. Glossary & references

### Glossary

| Term | Meaning |
|---|---|
| **AX tree** | The accessibility tree: derived from DOM+Layout, consumed by AT. |
| **`AXObjectCacheImpl`** | Blink's engine that builds, caches, updates, and routes the AX tree. |
| **`AXObject`** | Abstract base node of the AX tree (role, name, bounds, children). |
| **`AXNodeObject`** | Concrete `AXObject` wrapping a DOM `Node` (usually + `LayoutObject`). |
| **`BlinkAXTreeSource`** | Translates `AXObject` → `ui::AXNodeData`; the serializer's view of the tree. |
| **`ui::AXNodeData`** | Compact, cross-process node format (sparse attribute arrays). |
| **`AXID`** | `int32` node id, unique within a frame; key of the cache's maps. |
| **Lifecycle** | The 5-state machine (`Defer→Process→Finalize→Serialize`) the build runs through. |
| **Ignored** | Node is hidden from assistive tech. |
| **Included** | Node is serialized into the tree (may be ignored-but-included). |
| **Prune** | Decision to create no AXObject and cut the subtree there. |
| **Deferred update** | A queued `TreeUpdateReason` processed later when layout is clean. |
| **Offset container** | Ancestor that a node's bounds are stored relative to. |

### Curated source (in this repo)

- [`chromium-source/docs/overview.md`](./chromium-source/docs/overview.md) — Chromium "Accessibility Overview"
- [`chromium-source/docs/how_a11y_works.md`](./chromium-source/docs/how_a11y_works.md) — "How Chrome Accessibility Works"
- [`chromium-source/blink-accessibility/ax_object_cache_lifecycle.h`](./chromium-source/blink-accessibility/ax_object_cache_lifecycle.h) — **the construction state machine**
- [`chromium-source/blink-accessibility/ax_object_cache_impl.h`](./chromium-source/blink-accessibility/ax_object_cache_impl.h) — the engine's interface
- [`chromium-source/blink-accessibility/ax_object.h`](./chromium-source/blink-accessibility/ax_object.h) — base node; ignored/included model
- [`chromium-source/blink-accessibility/ax_node_object.h`](./chromium-source/blink-accessibility/ax_node_object.h) — concrete node
- [`chromium-source/blink-accessibility/blink_ax_tree_source.h`](./chromium-source/blink-accessibility/blink_ax_tree_source.h) — the translator
- [`chromium-source/PROVENANCE.md`](./chromium-source/PROVENANCE.md) — exact upstream paths, retrieval date, license

### Upstream (canonical)

- Module dir: `third_party/blink/renderer/modules/accessibility/`
  <https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/modules/accessibility/>
- `ax_object_cache_impl.cc` (the implementation behind everything above):
  <https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/modules/accessibility/ax_object_cache_impl.cc>
- Accessibility docs index:
  <https://chromium.googlesource.com/chromium/src/+/main/docs/accessibility/>

> Quotes and line-level details reflect the `main` branch as fetched on
> 2026-06-28; upstream moves quickly, so reconcile against current source for
> exact line numbers.
