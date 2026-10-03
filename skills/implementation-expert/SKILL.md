---
name: implementation-expert
description: |
  Authoritative map of everything the Sublay platform can do — every model, bundle,
  and feature an app can build on, grouped by domain with depth matched to each.
  Grounds the agent in Sublay's mental model (project scoping, bundles,
  the Provider+Hook SDK pattern) so it never invents a capability, never misses one
  that already exists, and knows which surface (SDK hook, REST route, dashboard) to
  reach for. First point of contact, not a spec — deep field/param/endpoint detail
  lives in the Sublay Docs MCP. Use when building on or integrating Sublay, wiring a
  Sublay-backed feature, choosing between Sublay capabilities, or answering "can
  Sublay do X / how do I do X in Sublay". Triggers: "Sublay", "@sublay/*",
  "entities/spaces/bundles/custom tables", "implement this with Sublay".
---

# Sublay Implementation Expert

You are the reference an agent consults before building anything on Sublay. Your job is to hold the complete, accurate menu of what the platform offers so the agent uses what already exists instead of reinventing it or inventing what isn't there. Depth here is deliberately uneven: obvious surfaces get a line, the genuinely confusing ones (entities, spaces, bundles, reputation, custom tables, the AI features) get real explanation. For exact hook signatures, params, endpoint shapes, or field references, go to the live docs at **https://docs.sublay.io** (current version is v7). The docs are also served as a **public, no-auth MCP server at `https://docs.sublay.io/mcp`** — if a Sublay docs MCP is connected in your environment, use its search tool (`search_sublay_documentation`); otherwise connect that endpoint or fetch pages directly from `docs.sublay.io`. Prefer docs-sourced detail over memory for any specific claim, and don't guess.

## The mental model

Sublay is **infrastructure for user-based products**: the pre-modeled data, APIs, and client SDKs for the layer a product needs once its users start *doing* things — accounts/auth, entities (content), comments, reactions, files, chat, spaces, workspaces, notifications, moderation, search — so a developer doesn't build that layer from scratch. It is a hosted, multi-tenant SaaS, not a library you self-host.

Three ideas unlock everything else:

**Everything is scoped to a project.** Every app is a **project** with its own `projectId`, and all API paths are namespaced under `/v7/:projectId/...`. Users, entities, spaces, and conversations exist only inside one project; multiple projects (staging, separate apps, white-label) are fully isolated from each other. You'll juggle a `projectId` in every call — that's the main day-to-day consequence.

**Features are BUNDLES, not a monolith.** A project is *assembled* from bundles — self-contained features (each with its own API endpoints and SDK hooks) that you install per project and add/remove anytime. Only `core` is mandatory. **If a feature's bundle isn't installed, its endpoints return `403 database/tables-not-available`** (with a `dashboardUrl` to install it), or `503 database/tables-provisioning` while mid-install. This is why a feature can be fully documented yet still error on a given project — the docs describe the whole platform; a project only runs the bundles it installed. Bundles are about *provisioning*, separate from *pricing/plan* (some features need both a bundle AND a paid plan — see AI). A few bundles depend on another: `comments` and `collections` require `entities`; `interest-matching` requires `ai-search`.

**One integration pattern: Provider + Hook.** The React/React Native SDK is the recommended path. Wrap the app once in `SublayProvider` (`projectId`, optional `signedToken` for external auth). The recurring shape is a **context provider** that loads a resource + a **hook** that reads its state and bound actions: `EntityProvider`/`useEntity`, `SpaceProvider`/`useSpace`, `EventProvider`/`useEvent`, `ChatProvider`, `CommentSectionProvider`, etc. Feeds use `useEntityList` / `useSpaceList` (each keyed by a `listId`, no provider needed); newer lists use a `useFetchMany…Wrapper` hook. The SDK auto-manages tokens, refresh, optimistic state, real-time (WebSocket for chat), and types. Non-React apps use the framework-agnostic `@sublay/js` SDK or raw REST; servers use `@sublay/node` (service keys, can act on behalf of any user).

### Integration surfaces

| Surface | Package / path | Use when |
|---|---|---|
| React web | `@sublay/react-js` | Next.js, Vite, CRA |
| React Native | `@sublay/react-native` | RN (Keychain tokens) |
| Expo | `@sublay/expo` | Expo (SecureStore tokens) |
| Core (shared) | `@sublay/core` | platform-agnostic hooks/types; rarely imported directly |
| Framework-agnostic JS | `@sublay/js` | Vue/Angular/Svelte/vanilla; the v7 API without React, authed as the end user |
| Server-side | `@sublay/node` | backends, webhooks, jobs; service key acts as any user |
| CLI components | `@sublay/cli` | copy editable UI source into a project (shadcn-style): `comments-threaded`, `comments-social` (React/RN/Expo), `notifications-control` (React) |
| Raw REST | `/v7/:projectId/…` | anything else; `Authorization: Bearer <token>` |

## The bundle map

`core` is always present and can't be removed. It provides **Users + auth/identity + OAuth** — the identity foundation every other model hangs off. Everything below is opt-in per project, **Entities included**: entities are their own bundle, not part of `core` (near-foundational, but optional). There are 18 bundles in all.

| Bundle | Feature | One-liner |
|---|---|---|
| `core` *(always)* | Users, Auth, OAuth | Identity foundation every model hangs off |
| `entities` | Entities | The base content unit; `comments` & `collections` require it |
| `comments` | Comments | Threaded comments on entities |
| `reactions` | Reactions | 8 reaction types (incl. up/down votes) on entities & comments |
| `files-images` | Storage | File/image uploads incl. avatars & banners |
| `follows` | Follows | Unidirectional follow graph |
| `connections` | Connections | Bidirectional (friend-style) requests |
| `spaces` | Spaces | Hierarchical group containers with membership, roles, rules |
| `workspaces` | Workspaces | Invite-only nestable teams/orgs with per-member authority |
| `chat` | Chat | Real-time 1:1 & group conversations |
| `collections` | Collections | User bookmarking/folders for entities |
| `moderation` | Moderation | Reports, resolution, suspensions, user blocking |
| `notifications` | App Notifications | In-app notification system |
| `push` | Push Notifications | Native OS push (iOS/Android/Web) |
| `ai-search` | Semantic Search & AI | Embeddings, semantic search, AI ask |
| `reputation` | Reputation | Per-space reputation buckets + overall total, reputation grants |
| `interest-matching` | Interest Matching | Activity-derived interest facets + user matching; requires `ai-search` |
| `events` | Events | Events, RSVPs, hosts/invites |

Removing a bundle is **destructive** — it drops the bundle's tables and clears references to it from other bundles (e.g. dropping `spaces` nulls `spaceId` everywhere). `reputation` is doubly lossy (also drops all grant history and zeroes every user's `reputation`). No undo. A bundle can't be removed while a dependent is installed (remove `comments`/`collections` before `entities`, `interest-matching` before `ai-search`). Bundles are managed on the project's **Database** page in the dashboard (`dash.sublay.io/<projectId>/database`) or via the bundle install API.

## Users & auth *(core)*

One unified identity per project — every other model hangs off a user. Fetch users by `id` / `foreignId` / `username`; user suggestions are available. Auth is deliberately flexible — you pick the model:

- **Built-in** — email/password sign-up, password reset/change, email verification, fully managed by Sublay.
- **OAuth** — Google, GitHub, Apple, Facebook; several OAuth identities (and a password) can link to one account, and an OAuth-only user can set an initial password. OAuth lives in `core`, so it works on every project without an extra bundle.
- **External JWT** — your backend signs a JWT and Sublay creates/matches the user from it. Keep your existing auth stack and run Sublay alongside it.

**Multi-account**: up to 5 accounts signed in at once on one device, with switching (`useAccounts`, `useSwitchAccount`, `useAddAccount`, …). **Self-service account deletion** is a two-step emailed-code flow (`useRequestAccountDeletion` → `useConfirmAccountDeletion`). A **service key** (project owner, `@sublay/node`) can act on behalf of any user via an explicit, gated `userId` — never ship a service key to a client.

## Content models

The content your users create. **Entities are the foundation** — comments and collections attach to an entity; reactions attach to entities and comments.

### Entities *(entities bundle)*

The **fundamental content unit**: an article, product listing, task, review, photo, forum thread — whatever the app centers on. Comments, reactions (including up/down votes), collections, and reputation grants hang off entities. Get this one right and the platform clicks.

**Entities is its own bundle**, not part of `core` — near-foundational but optional. **`comments` and `collections` depend on it** (they can't run without entities), and other bundles carry references *into* entities (reactions, reports, embeddings) that clean up if entities is removed. Most apps install it; a project that only needs, say, chat or a pure directory can skip it.

What trips people up:

- **Three identifiers.** `id` (UUID, Sublay's own), `foreignId` (**your** system's ID, or a static handle like `"about-page"` — how you attach Sublay to *existing* content without a migration; fetch by it directly), and `shortId` (auto short handle for share links). Any of the three can address an entity.
- **`sourceId`** separates distinct *kinds* of entity that coexist in one project but shouldn't mix. Example: a marketplace app stores seller listings as `sourceId: "listing"` (created by users, shown in the browse feed) **and** its help-center articles as `sourceId: "help-article"` (written by staff, surfaced only in a "Help" section). Both are entities in the same project — same reactions, comments, collections — but you filter feeds by `sourceId` so help articles never leak into the listing feed and vice versa. It lets one project host several content streams with different authors and audiences without spinning up separate projects.
- **Drafts & publishing.** Entities can be created as `isDraft` and published later. There's a drafts list for the current user.
- **Rich payload.** `title`, `content`, `keywords` (tags for discovery), `mentions`, `attachments` (flexible media metadata), managed `files` (via storage), `location` (GeoJSON point), and `metadata` (free-form key/value, max 10 KB) for app-specific fields that don't warrant a custom table.
- **Engagement fields are computed for you:** `reactionCounts` + `userReaction`, `repliesCount`, `views` (auto-increments on fetch), `score` (a decaying "hotness" for trending feeds).
- **`include` param** populates related data on fetch: `"user"`, `"space"`, `"topComment"`, `"saved"` (→ `isSaved`), `"files"`, `"grants"` (reputation-grant summary).
- **NSFW flagging.** Entities (and spaces) can be flagged NSFW; flagging a space cascades to all its content. Sublay only *labels and filters* (pass `nsfwFilter` on list queries) — blurring or age-gating is the app's job.
- **Deletion** is a hard delete by default; soft delete (`deletedAt`) is an opt-in project setting (`entityDeletion`, paid plans).
- Pattern: `EntityProvider`/`useEntity` for one; `useEntityList` (keyed by `listId`) for feeds. Hooks: `useCreateEntity`, `useUpdateEntity`, `useDeleteEntity`.

### Comments *(comments bundle)*

Threaded/nested discussions attached to an entity. Nested replies, reactions (incl. up/down votes, needs `reactions`), GIFs, `@mentions`, moderation. `CommentSectionProvider` + hooks; also shippable as **editable UI via the CLI** for React, React Native and Expo (`comments-threaded` = Reddit-style, `comments-social` = Instagram-style).

### Reactions *(reactions bundle)*

Reactions on entities and comments — a fixed set of 8 types: `upvote`, `downvote`, `like`, `love`, `wow`, `sad`, `angry`, `funny` (votes *are* reactions). Surfaced as `reactionCounts` plus the current user's `userReaction`; each type has a set effect on the author's reputation.

### Collections *(collections bundle)*

User-owned **bookmarking**: save entities into collections and sub-collections/folders. Check saved-state with `include: "saved"` on an entity.

## Spaces & relationships

Group containers, and how users relate to each other.

### Spaces *(spaces bundle)*

**Group containers** — a scoped place with its own membership, roles, rules, and moderation, where members create and interact with content. Think teams, cohorts, classes, forums, subreddits, clubs. A space owns its membership, roles, moderation, rules, and optionally an integrated chat conversation. Entities and conversations can belong to a space (`spaceId`).

The parts worth internalizing:

- **Two independent permission axes, plus a third visibility axis** — don't conflate them:
  - `readingPermission`: `"anyone"` | `"members"` — who can *see* content.
  - `postingPermission`: `"anyone"` | `"members"` | `"admins"` — who can *create content* in the space.
  - `visibility`: `"public"` | `"unlisted"` | `"private"` — whether the space is *listed/discoverable*. **Not access control** — `"unlisted"` just hides from discovery/search but stays reachable by link; `"private"` currently behaves like `"unlisted"`. Gate actual content with `readingPermission`.
- **Membership** has approval workflows (join/leave, approve/decline pending members, ban/unban) and **roles** — `admin` / `moderator` / `member` (`useSpace` resolves `isMember`, `canPost`, `canRead`, `canModerate`, `isAdmin`, `isModerator` for the current user).
- **Hierarchy**: nest up to **10 levels** via `parentSpaceId`. Only parent admins create child spaces; parent admins/mods cascade into children with their roles.
- **Sub-features**: moderation (approve/remove content, handle reports), **rules** (an ordered rules list), space-level chat (needs `chat` bundle too), and **space lists** (`useSpaceList`) for browsing/paginating spaces.
- Pattern: `SpaceProvider`/`useSpace`; `useCreateSpace`.

### Workspaces *(workspaces bundle)*

The **B2B/SaaS collaboration layer** — an account/team people are *invited into*, with granular per-member authority and optional self-nesting (org → client → project). Where **Spaces** is an *open-membership* shape (join, roles), **Workspaces** is **invite-only** with an **owner apex** and a capabilities/permissions model. Reach for it for teams, orgs, agencies, tenant accounts — anything with an ownership hierarchy and delegated access.

The authority model is the subtle part:

- **Owner apex + ranks.** Each workspace has exactly one **owner** (its creator), above everything. Members carry a numeric **rank** per workspace where **smaller = more senior** (owner sits above rank 0). Rank governs who can act on whom.
- **Two arrays, very different.** **`capabilities`** are a **closed, Sublay-enforced vocabulary** of 7 verbs — `view`, `invite`, `remove-member`, `edit-member-access`, `edit-member-profile`, `create-sub-workspace`, `edit-workspace` — Sublay reads and enforces these. **`permissions`** are **opaque free-form strings Sublay never interprets** (your app's own labels); per-node only, they don't cascade.
- **Reach & the wall/door rule** — the crux. Authority is **strict per-node by default**: a capability on one workspace does *not* leak into its children. A child can opt into **downward reach** by setting its own **`inheritsFromParent`** flag — an open "door" that lets a parent member's capabilities reach it. A node with the flag off is a **sealed wall** that shields itself and its subtree. The one exception: an **owner** (own or any ancestor) is a **god** over the whole subtree regardless of walls.
- **Invitations are identity-matched + verified-email, no token.** Invite by exactly one of `email` / `userId` / `username`; the invitee accepts from their own session (must be the matched identity with a verified email). Invites expire lazily. No magic-link token to leak.
- **Two service reads.** The **roster** (`useFetchWorkspaceMembers`) returns one entry per distinct user with *why* they're there (`owner` / `member` / `ancestor-owner` / `reach-holder` / `descendant-member`). **Authority-as-a-service** (`useFetchWorkspaceAuthority`) resolves the current user's standing on a node — reasons, effective capabilities, permissions, rank.
- **Owner-only vs self-service.** Deleting a workspace, transferring ownership, and flipping the inherit flag are **owner-only** (not capabilities). Accepting/declining your own invite, leaving, and editing your own `title` need no capability.
- **Client vs server.** The client hooks cover the full surface, always acting as the **signed-in user** (subject to capability + rank rules): `useCreateWorkspace`, `useFetchWorkspace`, `useFetchManyWorkspaces` (+`…Wrapper`), `useUpdateWorkspace`, `useDeleteWorkspace`, `useTransferWorkspaceOwnership`, `useFetchWorkspaceMembers`, `useUpdateWorkspaceMember`, `useRemoveWorkspaceMember`, `useLeaveWorkspace`, `useCreateWorkspaceInvite`, `useFetchMyWorkspaceInvites`, `useAcceptWorkspaceInvite`/`useDeclineWorkspaceInvite`, `useFetchWorkspaceAuthority`, and more. The only extra reach on **`@sublay/node`** is acting **as a different user** via `actingUserId` (service key). There's **no `WorkspaceProvider` and no Redux slice**: plain hooks + the `useFetchMany…Wrapper` list pattern.
- **Webhooks**: invite lifecycle (`workspace.invite.*`), roster sync (`workspace.member.added` / `.removed`), and `workspace.deleted`.

### Chat *(chat bundle)*

Real-time **1:1 and group conversations**. Models: `Conversation`, `ConversationMember`, `ChatMessage`. Message reactions, threaded replies, read state, typing indicators, per-conversation mute, moderation. Real-time is WebSocket — the SDK connects automatically when `ChatProvider` is mounted (message delivery/read/typing handled for you). Can be space-scoped. The inbox is live too — new conversations, last-message previews, list ordering, and unread counts all update in real time across the whole conversation list, not just the open thread.

### Events *(events bundle)*

Full events surface: create/edit events (inline cover/gallery upload), paginated lists, the **RSVP lifecycle**, and **host/invite** management. Hooks authenticate as the logged-in user (no actor `userId`; the only `userId` args are *targets* on host/invite hooks). Models: `Event`, `EventInvite`, `EventRsvp`. Load one with `EventProvider`/`useEvent`; paginate lists with `useFetchManyEventsWrapper`. Space-scoping needs `spaces`, images need `files-images`, notifications need `notifications`.

### Follows *(follows bundle)*

**Unidirectional** relationship — A follows B, no approval needed. Follower/following counts, status checks, bulk queries. Distinct from Connections (below), which is bidirectional.

### Connections *(connections bundle)*

**Bidirectional, friend-style** relationship — one side sends a request, the other accepts. A separate bundle and model from Follows; don't assume having one implies the other.

## Cross-user systems

Systems that operate across users and content rather than being something a single user "owns."

### App Notifications *(notifications bundle)*

In-app notification records for users, also delivered to your server **via webhook** so you can bridge them downstream (push, email, etc.).

### Push Notifications *(push bundle)*

Native OS push to iOS (APNs), Android (FCM), and Web Push. Clients register devices with `usePushRegistration`; your backend sends with `push.send` (`@sublay/node`). Sublay can also **automatically** push in-app events (comments, mentions, follows, chat messages, …), with per-user type preferences, per-conversation mute, and dashboard-authored copy templates.

### Reputation *(reputation bundle)*

**Space-scoped**, not a single number. Each user accrues reputation into a **general (project-wide) bucket plus one bucket per space**; their `reputation` field is the maintained sum of all of them and is always returned. Missing buckets read as `0`, and scores can go negative. Reputation is **earned** automatically (creating entities/comments, receiving votes and reactions) or **granted** deliberately.

- Read per-space reputation by opting into a **`spaceReputation`** field on any user-returning response — request a specific space, the general bucket, or (on space-context endpoints) the space in context, optionally summing a space's whole subtree.
- **Reputation grants** — the primitive for tipping, bounties, awards, "best answer": a **transfer** (`sourceType: "user"`) debits the sender's bucket and credits the recipient's bucket for the same space (`useCreateReputationGrant`), while a **mint** (`sourceType: "app"`, service/master key only) creates or destroys reputation. A grant can annotate an entity, comment, or chat message (`include: "grants"` returns a per-item summary); the history is listable (`useFetchManyReputationGrantsWrapper`). Your app owns the rules for *why* a grant happens.
- Works with or without `spaces` (without, only the general bucket accrues).

### Moderation *(moderation bundle)*

Report flows for entities, comments, and chat messages (one `POST /reports` endpoint, `useCreateReport`); report **resolution** tooling; **user suspensions** (set from the dashboard or the report-handling flow); space-level moderation actions and ban management. Models include `Report`, `Suspension`, `Block`. Content carries a `moderationStatus` (`approved`/`removed`). **Suspension is enforced for you** (with the `moderation` bundle installed): a suspended user can still sign in but is blocked from all participation-writes — creating/editing content, comments, reactions, chat, uploads, etc., which return `403 user/suspended` — while a few self-limiting actions (change password, leave a space, unfollow) stay open. Enforcement is automatic and not per-action configurable.

**User blocking** also ships here: any user can block another (`useBlockManager`, `useBlockUser`). A block is silent and mutual — it removes follows/connections, freezes 1:1 DMs, stops comments/reactions/mentions/notifications between the two, and hides each from the other's feeds, search, and suggestions (shared group conversations are untouched).

## Platform capabilities

Not per-feature models — infrastructure you build *with*, layered over the models above.

### Custom Tables *(the "remaining 20%" of the data model)*

When app-specific data fits no built-in model, provision **your own tables** and CRUD their rows from the SDKs or dashboard.

- **Logical names, and only your tables**: you name a table `Events` and address it as `Events` — no prefixes to type. The custom-table API can **only** reach tables you created; built-in tables (`Comments`, `Users`, …) are never reachable through it.
- **Two flags you choose at create time** (per table, immutable after): `timestamps` (default **on**) and `paranoid` (default **off**, requires `timestamps`). `paranoid` turns deletes into **soft deletes** — a delete sets `deletedAt`, `restore` clears it, and `includeDeleted` surfaces them; without it, deletes are permanent.
- **Managed columns you never write to**: `id` (uuid PK, always), plus `createdAt`/`updatedAt` (when `timestamps` on) and `deletedAt` (when `paranoid` on). Trying to define a column with one of these names is rejected.
- **9 column types**: `text, integer, float, decimal, boolean, date, timestamp, uuid, json` — each with `nullable` + optional `defaultValue`.
- **Row access everywhere**: `client.table(name)` (node & js SDKs), `useTable(name)` (React/RN). **Management/DDL (create/drop table, add/drop column) is Node-SDK-service-key or dashboard only** — the JS SDK and React hook are row-only (they auth as an end user, hold no service key).
- **Limits**: 100 tables/project, 100 columns/table, 100 rows/bulk call.
- **Security caveat (pre-release):** row CRUD on `/db` is currently **open** — it captures caller identity but enforces no per-row/table authorization. Don't store data needing access control here until the auth gate ships. (DDL *is* gated.)

### Storage / Files & Images *(files-images bundle)*

Upload files/images on behalf of authenticated users; stored in managed cloud storage, served via a proxy URL (credentials never hit the client). Two hooks:

- **`useUploadImage`** — server-side processing: variant generation (resize, format convert, EXIF strip) via a **processing mode** (`exact-dimensions`, `aspect-ratio-width/height-based`, `original-aspect`, `multi-aspect-ratio`).
- **`useUploadFile`** — generic upload, stored as-is, returns a proxy URL.
- 50 MB per upload. Files optionally associate with an entity/comment/space for **cascade deletion**. Also backs avatars & banners.

### Semantic Search & AI *(ai-search bundle + PAID plan + authed user)*

Auto-embeds content on create/update (entities, comments, chat messages, users' profile text, spaces) — no config. Powers:

- **Semantic search** across three domains: content (entities/comments/messages, filterable by `sourceTypes`/space — optionally including child spaces — /conversation), users, spaces. Each result carries a `similarity` (0–1) + hydrated record. Hooks: `useSearchContent`, `useSearchUsers`, `useSearchSpaces`.
- **AI Ask** (`useAskContent`) — retrieval-augmented: streams a synthesized answer grounded in the project's content, along with the source records it drew from. Says so when the content doesn't cover the question.
- **Requires a paid plan AND an authenticated user** (the SDK handles the token/refresh). On a free plan these return `403 project/plan-required`. Embedding runs in the background, so new content becomes searchable shortly after it's written.

### Interest Matching *(interest-matching bundle + ai-search + PAID plan)*

Matches people by **activity-derived interests**, not a static profile field. Activity across all three content types — **entities, comments, and chat messages** (private DMs excluded) — folds into a few decaying **interest facets** per user; `POST /match/users` ranks others against an asker — passively ("who's like me?") or by explicit topic. Facets decay (long half-life) so matches reflect *current* engagement.

- **Ordered prerequisites**: paid plan → install `ai-search` first → install `interest-matching` → turn on the `interestMatching.enabled` project setting. Calls require an authenticated user.
- Matches are **eventually consistent** — activity takes a little while to reflect, and a brand-new or low-activity user is hard to surface until they've been active. Sample content (the text explaining why two people overlap) is opt-in and off by default. Hooks/SDK: `useMatchUsers`, `search.matchUsers`.

## Cross-cutting concepts

- **`foreignId`** — attach Sublay to *existing* content/users without migrating IDs. Address entities/users by your own ID or a static handle.
- **`include` params** — most fetches take an `include` array to hydrate related data (author, space, top comment, saved-state, files) in one round trip instead of N+1.
- **Lists & pagination** — browse/sort/filter hooks over entities (`useEntityList`) and spaces (`useSpaceList`), each keyed by a `listId`; most other lists (events, workspaces, grants, …) use `useFetchMany…Wrapper` hooks.
- **Drafts, soft deletes** — entities support drafts. Deletes are hard by default; soft delete (`deletedAt`, excluded from reads) is opt-in — a paid-plan project setting for entities and events, or the `paranoid` flag on a custom table.
- **Service keys** — server-side (`@sublay/node`) power: bypass user auth, act as any user, do DDL on custom tables, increment views. Never ship a service key to a client.
- **Plan vs bundle are orthogonal** — a feature can need its bundle (provisioning) *and* a paid plan. Paid-only today: semantic search, AI Ask, interest matching, and the entity/event/space deletion settings. A missing bundle is a 403 `database/tables-not-available`; a plan-gated AI call is a 403 `project/plan-required`.
- **Webhooks** — broadcast events after writes (`*.created.complete` / `*.updated.complete`, `notification.created`, `workspace.*`) and **blocking validation** webhooks your server can veto before users, entities, comments, spaces, or chat messages are created/updated.
- **Whitelist domains** in the dashboard for production web apps, or requests are rejected.

## Reaching past this map

This is the menu, not the spec. For anything concrete, go to the Sublay docs — website **https://docs.sublay.io** (v7), or the **public no-auth MCP at `https://docs.sublay.io/mcp`** if a docs MCP is available in your environment (its `search_sublay_documentation` tool) or you can connect it. Where to look:

- exact hook signatures, params, return shapes → the `sdk/*` and `hooks/*` pages
- endpoint paths, request/response bodies → the `api-reference/*` pages
- full field references → the `data-models/*` pages (including supporting models not detailed above — `Block`, `Rule`, `File`, `ConversationMember`, `ReputationGrant`, reputation buckets, and the rest)

Other live surfaces worth knowing: the dashboard is **`https://dash.sublay.io`** (bundles, custom-table editor, project settings, keys) and the API base is **`https://api.sublay.io`**. Prefer docs-sourced detail over memory for any specific claim about params, routes, or field names.
