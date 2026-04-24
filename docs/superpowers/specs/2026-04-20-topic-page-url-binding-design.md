# Topic Page URL Binding

Date: 2026-04-20
Scope: SPA routing for the agent topic page view.

## Context

Today `/agent/:aid/:topicId/page` renders `TopicPage`, which maintains the
currently-active document in React state (hoisted to `(chat)/_layout` via
`PageDocContext`). Clicking a document in the resources sidebar swaps the
in-memory `active` and replaces canvas content. The URL never reflects which
document is being edited, so the state cannot be deep-linked, refreshed, or
navigated via browser back/forward.

## Goals

1. Make the document id a first-class URL segment: `/page/:docId`.
2. Keep `/page` (bare) as a convenient entry — it redirects to the topic's
   auto-created notebook document.
3. Gracefully fall back when the URL carries an invalid `docId`.
4. Remove the layout-level context now that URL is the source of truth.

## Decisions

- **Default document on entry**: the topic's auto-created notebook document
  (`useAutoCreateTopicDocument`). Agent documents opened via the sidebar are
  transient selections reflected only in URL; no cross-session memory.
- **Bare `/page`**: component mounts, waits for `useAutoCreateTopicDocument`
  to resolve a `topicDocId`, then `navigate(/page/<topicDocId>, { replace })`.
- **Invalid `/page/:docId`**: silent fallback to `/page/<topicDocId>` via
  `navigate(replace)`. No error UI.

## Architecture

### Route tree

Update `src/spa/router/desktopRouter.config.tsx` **and**
`src/spa/router/desktopRouter.config.desktop.tsx` together. Register a nested
route tree for `page`:

```
path: 'page'
├── { index: true }    → PageRedirect
└── { path: ':docId' } → TopicPage
```

### Components

**`src/routes/(main)/agent/features/Page/PageRedirect.tsx`** (new)

- Reads `topicId` from `useParams`.
- Uses `useAutoCreateTopicDocument(topicId)` to fetch or auto-create.
- Once a `document.id` is available, calls
  `navigate(`/agent/${aid}/${topicId}/page/${document.id}`, { replace: true })`.
- While loading, renders the existing editor skeleton (reuse
  `EditorCanvas`'s skeleton styling or a lightweight centered `Spin`).

**`src/routes/(main)/agent/features/Page/index.tsx`** (update)

- `docId` comes from `useParams<{ docId: string }>()`.
- Fetch metadata via
  `useClientDataSWR(['page-document-meta', docId], () => documentService.getDocumentById(docId))`.
- If the fetch resolves to `null`/throws, call
  `useAutoCreateTopicDocument(topicId)` and
  `navigate(`/agent/${aid}/${topicId}/page/${topicDocId}`, { replace: true })`.
- When valid, render `TopicCanvas` with `documentId={docId}` plus the
  debounced title save. Saving covers both doc kinds by:
  1. `documentService.updateDocument({ id, title, saveSource: 'autosave' })`
  2. `mutate(agentDocumentSWRKeys.documentsList(agentId))` (refreshes
     sidebar tiles when the doc is an agent-document)
  3. `mutate([SWR_USE_FETCH_NOTEBOOK_DOCUMENTS, topicId])` (refreshes
     notebook list so topic-notebook docs reflect the new title)
     The notebook key may be inlined or exported from `src/store/notebook/action.ts`.
- Reset `titleDraft` when `docId` changes.

**`src/routes/(main)/agent/features/Conversation/WorkingSidebar/ResourcesSection/AgentDocumentsGroup.tsx`** (update)

- `DocumentItem` becomes URL-aware: each leaf calls
  `useMatch('/agent/:aid/:topicId/page/:docId?')` itself.
- When the match hits, `handleOpen` calls
  `navigate(/agent/${aid}/${topicId}/page/${document.documentId})`
  instead of the old `onOpenDocument` / `openDocument(Portal)` path.
- `isActive` derived from the match's `params.docId`.
- Fall back to `openDocument` (Portal) only when no page-match (chat mode).
- Prop chain for `onOpenDocument` / `activeDocumentId` is dropped:
  intermediate components no longer need to know.

**`src/routes/(main)/agent/features/Conversation/WorkingSidebar/index.tsx`** (update)

- Remove the `onOpenDocument` / `activeDocumentId` props added earlier.
- No URL awareness lives at this level — the decision is pushed down to
  `DocumentItem` so intermediate components stay dumb.
- `useMatch('/agent/:aid/:topicId/page/*')` may still be used here **only**
  to decide whether the sidebar header, etc. should take page-specific
  styling (if needed); leave out if not necessary.
- Critically: `pathname.endsWith('/page')` must **not** be used — it
  breaks once the URL carries `/:docId`.

**`src/routes/(main)/agent/(chat)/_layout/index.tsx`** (update)

- Keep horizontal layout, keep `Portal` + `AgentWorkingSidebar` mounts.
- Remove the `PageDocContext` provider and the `useState` for `active`.

**`src/routes/(main)/agent/(chat)/_layout/pageDocContext.ts`** (delete)

**`src/features/TopicCanvas/useAutoCreateTopicDocument.ts`** (harden)

- Promote the `creatingRef` de-dup from component-instance scope to a
  module-level `Map<topicId, Promise<NotebookDocument>>`. Concurrent
  mounts / rapid remounts then share the same in-flight creation promise
  and never issue duplicate inserts.
- The module-level map entry is cleared on promise resolution or
  rejection.

**`src/features/Electron/titlebar/RecentlyViewed/plugins/agentTopicPagePlugin.ts`** (update)

- Extend `AGENT_TOPIC_PAGE_PATH_REGEX` to tolerate an optional trailing
  `/:docId` segment:
  `^\/agent\/([^/?]+)\/(tpc_[^/?]+)\/page(?:\/[^/?]+)?$`
- `generateUrl` continues to return bare `/page` (round-trips into the
  redirect); callers that want a deep link with docId can carry it in
  `reference.params` and include it in the path. Add
  `docId?: string` to `AgentTopicPageParams` (in `types.ts`) and plumb
  through `parseUrl` (capture 3rd group) and `generateUrl` (append
  `/${docId}` when present).

## Data Flow

```
URL /page/:docId  (source of truth)
   │
   ├─▶ TopicPage
   │     • fetch doc metadata + validate
   │     • render TopicCanvas with documentId
   │     • debounced title save via documentService
   │
   └─▶ AgentWorkingSidebar
         • reads docId via useParams
         • highlights matching DocumentItem
         • click → navigate(/page/:newId)
```

Clicking a doc in the sidebar causes a URL change; both `TopicPage` and
the sidebar re-render to match.

## Error Handling

| Case                                   | Behavior                                                                      |
| -------------------------------------- | ----------------------------------------------------------------------------- |
| Bare `/page`                           | Mount `PageRedirect`, auto-create if needed, `navigate(replace)` to concrete. |
| `/page/:docId` with nonexistent docId  | Fetch returns null → `navigate(replace)` to topic doc.                        |
| `/page/:docId` belonging to other user | Same as above (fetch returns null).                                           |
| `topicDocId` creation fails            | Existing `notebookStore.createDocument` error surface; no special UI.         |

## Testing

- Enter `/agent/:aid/:topicId/page` bare → URL rewrites to `/page/<topicDocId>`.
- Enter `/page/<topicDocId>` directly → canvas loads without redirect.
- Enter `/page/<invalid>` → URL rewrites to topic doc; no error banner.
- Click another document in sidebar → URL changes, canvas swaps, sidebar
  highlight follows (verifies `useMatch` detection works on `/page/:docId`).
- Browser back/forward cycles through visited docs.
- Refresh `/page/<docId>` → same doc loads.
- Switch topic (`:topicId` change) → URL reset to that topic's main doc;
  previous in-memory selection discarded.
- Chat route `/agent/:aid/:topicId` — Portal/Sidebar still behave as before;
  clicking sidebar doc opens Portal (not page).
- Title edit persists for **both** topic-notebook docs and agent-docs
  (verify notebook-store SWR cache + agent-documents SWR cache both
  show the new title after debounce).
- Rapid remount of the topic page (e.g., hot-reload or fast topic
  switching) does not produce duplicate topic-notebook docs (module-level
  de-dup map).
- Electron Recently Viewed entry for `/page/<docId>` round-trips: parse
  → resolve → generate yields the same URL including docId.

## Out of Scope

- Per-topic memory of last-viewed document (rejected in favor of A).
- Multi-document tabbing inside the page view.
- Emoji / icon persistence for topic docs.
- Cross-agent document sharing UX.
