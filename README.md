# Contribution 1: [FEATURE] Book reviews and ratings

**Contribution Number:** 1
**Student:** Jonayet Lavin
**Issue:** [https://github.com/stumpapp/stump/issues/119](https://github.com/stumpapp/stump/issues/119)
**Status:** Phase I In Progress

---

## Why I Chose This Issue

Stump is a self-hosted media server built with a modern, production-grade stack — TypeScript on the frontend and Rust on the backend. Issue #119 asks for a book ratings and reviews system, and what drew me to it is how clearly the maintainer scoped it out. There's a concrete checklist of exactly what the feature needs to do, which makes it easy to understand what a finished contribution actually looks like — especially important when you're navigating an unfamiliar codebase for the first time.

I also chose this because the feature touches both the UI and the data layer, which is the kind of end-to-end work I want more experience with. Building a ratings system means thinking about how data gets stored, how it moves through an API, and how it finally shows up for a user — all within one issue. That full-stack scope, combined with a TypeScript frontend I can work in comfortably, made this the right pick for a first open source contribution.

---

## Understanding the Issue

### Problem Description

Stump currently has no way for users to rate or review books in their library. There is no rating input, no text review field, and no place in the UI where this information is displayed.

### Expected Behavior

Users should be able to assign a 1–5 star rating to a book and optionally write a text review alongside it. Server admins should be able to toggle these features on or off per user via basic role-based access control. Ratings should support both private and public visibility. The average rating and any reviews should be displayed on the `BookOverviewScene` page.

### Current Behavior

No ratings or reviews functionality exists anywhere in the app — no UI, no API endpoints, and no database support for storing or retrieving this data.

### Affected Components

- The `BookOverviewScene` frontend component (TypeScript/React)
- The backend API layer handling book data (Rust)
- The database schema (new tables/fields needed for ratings and reviews)
- Permission/RBAC logic for toggling features per user

---

## Reproduction Process

### Environment Setup

The development environment was set up on macOS (Apple Silicon, arm64) following the [Stump developer guide](https://github.com/stumpapp/stump/blob/main/docs/content/docs/developer/contributing.mdx). Per the [contributing guidelines](https://github.com/stumpapp/stump/blob/main/.github/CONTRIBUTING.md), all work is branched off `nightly` (not `main`).

**Repository setup:**

```bash
# Fork cloned, upstream added
git remote add upstream https://github.com/stumpapp/stump.git
git fetch upstream nightly
git checkout -b feat/book-reviews-backend upstream/nightly
```

**Toolchain installed:**

| Tool | Version | Notes |
|------|---------|-------|
| Rust | 1.92.0 | Pinned by `rust-toolchain.toml`; installed via `rustup` (includes `rustfmt` 1.8.0, `clippy` 0.1.92) |
| Node | 22.22.3 | Installed via `nvm`. **Node ≥ 22.12.0 is required** — Node 20 fails because the `@tanstack/react-start` dependency demands it |
| Yarn | 1.22.21 | Enabled via `corepack` (project pins this version in `package.json`) |

**Install dependencies and build:**

```bash
yarn install                        # JS dependencies (+ patch-package patches)
cargo build -p stump_server         # First compile of the Rust backend (slow; ~131 MB binary)
```

**Running the app** (two terminals — the repo's `yarn dev:web` shortcut relies on `bacon`, which I ran without by starting the processes separately):

```bash
# Terminal 1 — backend API, serves on http://localhost:10801
cargo run -p stump_server

# Terminal 2 — web frontend (Vite), serves on http://localhost:3000
yarn web dev
```

In development the web client points itself at the API on port `10801` automatically (`apps/web/src/App.tsx:6`). On first run the server creates its config, logs, and SQLite database under `~/.stump/`.

**Test library:** A library was created in the UI pointing at a local folder containing three sample books copied from the repo's own fixtures (`core/integration-tests/data/`): `leaves.epub`, `science_comics_001.cbz`, and `rust_book.pdf` — covering the EPUB, CBZ, and PDF formats Stump supports (`core/src/filesystem/content_type.rs:74-79`).

---

### Steps to Reproduce

Because this is a **feature request**, "reproducing the issue" means demonstrating that the feature is absent across the UI, the API, and the data layer.

1. Start the backend and frontend, log in, and create a library with at least one book.
2. Open any book to reach the **`BookOverviewScene`** (`packages/browser/src/scenes/book/BookOverviewScene.tsx`).
3. Observe the page top-to-bottom: there is no star-rating widget, no review list, no "write a review" input, and no average rating.
4. Open the GraphQL playground at `http://localhost:10801/api/graphql` and confirm the API exposes no review fields (queries below).

---

### Reproduction Evidence

The absence of the feature was confirmed at **all three layers** of the stack.

**1. UI layer — `BookOverviewScene` has no ratings/reviews**

Opening a book (e.g. *Leaves of Grass*) shows the overview scene with header metadata, "Next in series," a Metadata table, and File Information — but **nothing for ratings or reviews** anywhere on the full page.

> 📷 *Screenshots:* `docs/screenshots/repro-library.png` (library with imported books) and `docs/screenshots/repro-book-overview-*.png` (full `BookOverviewScene`, scrolled top to bottom).

> Note: the `ageRating` field that can appear in the Metadata section is *publisher content-maturity metadata* — it is not a user rating or review, which is what issue #119 requests.

**2. API layer — the GraphQL schema has no review fields**

Run against the playground at `http://localhost:10801/api/graphql` (authenticated via the logged-in session):

*Introspection — the `Media` type's field list contains no `reviews` or `averageRating`:*

```graphql
{
  __type(name: "Media") {
    fields { name }
  }
}
```

*Attempting to query the fields returns a schema validation error:*

```graphql
{
  mediaById(id: "anything") {
    id
    averageRating
    reviews { id rating }
  }
}
```

Response:

```
Cannot query field "averageRating" on type "Media".
Cannot query field "reviews" on type "Media".
```

> 📷 *Screenshots:* `docs/screenshots/repro-graphql-introspection.png` and `docs/screenshots/repro-graphql-error.png`.

**3. Data layer — table exists, but nothing reads or writes it**

Interestingly, the database scaffolding is already present but unused:

- A `reviews` entity is defined at `crates/models/src/entity/review.rs` (`rating`, `content`, `is_private`, `media_id`, `user_id`).
- The `reviews` table is created by the init migration (`crates/migrations/src/m20250807_202824_init.rs`).

But there is **no code that exposes or populates it**:

- No `CreatePublicReview` permission in the `UserPermission` enum (`crates/models/src/shared/enums.rs`).
- No review mutation in `crates/graphql/src/mutation/media.rs`, and no dedicated `review.rs` mutation.
- No `reviews` or `average_rating` field resolver on the media object (`crates/graphql/src/object/media.rs`).

**Summary of findings:** The data model anticipates reviews, but the feature is entirely unimplemented above the database — confirming exactly the gap described in issue #119, and matching the maintainer's note that the work is primarily a matter of adding a permission, a mutation, and two field resolvers.


---

## Solution Approach

### Analysis

Exploring the codebase revealed that the data layer for reviews **already exists but is completely unused above the database**:

- A `reviews` entity is defined at `crates/models/src/entity/review.rs` (`id`, `rating`, `content`, `is_private`, `media_id`, `user_id`).
- The `reviews` table is created by the init migration (`crates/migrations/src/m20250807_202824_init.rs`), with foreign keys to `media` and `users`.

What was missing was everything that exposes or writes to that table:

- **No permission** to gate the feature — the `UserPermission` enum (`crates/models/src/shared/enums.rs`) had no review-related variant.
- **No API surface** — no GraphQL mutation to create a review, and no field resolvers to read them.
- **No display** — the `BookOverviewScene` had no UI for ratings or reviews.

Stump's backend is built with **Rust + async-graphql + SeaORM**. GraphQL types follow a consistent pattern: SeaORM entity `Model`s are wrapped in `SimpleObject` structs (e.g. `object/tag.rs`, `object/bookmark.rs`), input types implement an `into_active_model` helper (e.g. `input/reading_list.rs`), mutations are grouped into `MergedObject`s (`mutation/mod.rs`), and relational data is exposed as field resolvers on the parent object via `#[ComplexObject]` (e.g. `Media::tags`).

The maintainer scoped the work explicitly: add a `CreatePublicReview` permission (gating only *public* reviews — private reviews are a personal record and need no gate), a create mutation, and two field resolvers on the `Media` object (`reviews` and `averageRating`). The UI is to be a separate, isolated contribution.

### Proposed Solution

A **backend-only** pull request that exposes the existing `reviews` table through GraphQL, scoped to match the maintainer's direction. The frontend (`BookOverviewScene` UI) is intentionally deferred to a separate PR to keep each change reviewable.

The backend change consists of:

1. A new `CreatePublicReview` permission, gating only public reviews.
2. A `Review` GraphQL object type wrapping the existing entity.
3. A `ReviewInput` type with built-in 1–5 rating validation.
4. A `createReview` mutation that enforces the permission **only when the review is public**.
5. `reviews` and `averageRating` field resolvers on the `Media` object, with privacy-aware visibility.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Stump needs a ratings and reviews feature for books, with RBAC controls, public/private visibility, and display on the BookOverviewScene. The database scaffolding already exists; the gap is the permission, the API surface, and the UI.

**Match:** Mirrored existing Stump conventions rather than inventing new ones:
- Permission added to the `UserPermission` enum in the same doc-commented style as the `improve-permissioning` branch.
- `Review` object modeled on `object/tag.rs` / `object/bookmark.rs` (a flattened `SimpleObject` over the entity `Model`).
- `ReviewInput` modeled on `input/reading_list.rs`, including an `into_active_model` helper and unit tests.
- `createReview` mutation modeled on `ReadingListMutation`, grouped into the existing `ContentMutations` merged object.
- `reviews` / `averageRating` resolvers modeled on the `Media::tags` field resolver.
- Permission enforcement reused the existing `AuthContext::enforce_permissions` helper (`data.rs`).

**Plan** (revised after exploration — the DB layer turned out to already exist, and the work splits cleanly into a backend PR and a separate UI PR):

1. ~~Add database schema fields/tables for ratings and reviews~~ — **already present** (entity + migration). No change needed.
2. Add a `CreatePublicReview` permission (RBAC toggle), gating only public reviews.
3. Add a `Review` GraphQL object type and a `ReviewInput` input type (with 1–5 validation).
4. Implement a `createReview` mutation with a runtime public-review permission check.
5. Implement `reviews` and `averageRating` field resolvers on the `Media` object, with public/private visibility.
6. Regenerate the GraphQL schema (`cargo codegen`).
7. Write tests (unit + integration).
8. *(Separate PR)* Build the frontend UI: rating input, markdown/spoiler review rendering, and display of average rating + reviews on `BookOverviewScene`.

**Implement:** Branch `feat/book-reviews-backend` (based on `nightly`, per the contributing guidelines). Backend implementation complete; schema regeneration and integration tests in progress.

**Review:** Self-review checklist:
- [x] Permission gates only public reviews; private reviews are ungated.
- [x] Review author is taken from the authenticated session, never client input (no forging reviews as another user).
- [x] Private reviews are never returned to other users by the `reviews` resolver.
- [x] Private ratings are excluded from `averageRating` (no leakage via the aggregate).
- [x] Rating constrained to 1–5 at the schema level.
- [x] Each crate compiles cleanly (`cargo build -p models`, `cargo build -p graphql`).
- [ ] `cargo fmt` and `cargo clippy` pass.
- [ ] Integration tests covering visibility and permission enforcement pass.

**Evaluate:** Verified by (a) per-crate compilation after each step, (b) unit tests on the input's `into_active_model`, and (c) planned integration tests that exercise the create mutation and both resolvers end-to-end against a test database, asserting the visibility and permission rules.

---

## Testing Strategy

### Unit Tests

- [x] Test case 1: `ReviewInput::into_active_model` maps fields correctly (rating, content, privacy, author).
- [x] Test case 2: Review text (`content`) is correctly associated, including the `None` (rating-only) case.
- [ ] Test case 3: RBAC toggle correctly restricts public-review creation per user.

### Integration Tests

- [ ] A review can be created and retrieved for a book via the GraphQL API.
- [ ] Public/private visibility is respected: a user's private review is visible to them but hidden from other users; public reviews are visible to all.
- [ ] `averageRating` returns the correct mean over public reviews and `null` when there are none.
- [ ] Creating a public review without `CreatePublicReview` is rejected; creating a private review is always allowed; a server owner bypasses the check.

### Manual Testing

Reproduction of the feature's absence was performed manually before implementation (see *Reproduction Evidence*): the running app showed no rating/review UI on `BookOverviewScene`, and the GraphQL schema rejected `reviews` / `averageRating` queries on the `Media` type. Post-implementation manual verification (via the GraphiQL playground at `/api/graphql`) is planned to confirm the new mutation and resolvers behave as expected.

---

## Implementation Notes

### Week 1 Progress

Set up the local development environment on macOS (Rust 1.92.0, Node 22, Yarn 1.22.21), built and ran the Stump server + web client, and reproduced the issue. Confirmed the feature's absence across all three layers (UI, GraphQL API, and unused DB table). Branched `feat/book-reviews-backend` off `nightly` per the contributing guidelines.

### Week 2 Progress

Implemented the backend feature, mirroring existing codebase conventions:
- Added the `CreatePublicReview` permission.
- Added the `Review` object type and `ReviewInput` (with 1–5 validation and unit tests).
- Implemented the `createReview` mutation with a runtime public-review permission check.
- Implemented the `reviews` and `averageRating` field resolvers on `Media`, with privacy-aware visibility.

Each step was verified to compile. Remaining: regenerate the GraphQL schema, add integration tests, run `fmt`/`clippy`, and open the PR.

### Week 3 Progress

[To be filled in]

### Code Changes

- **Files modified:**
  - `crates/models/src/shared/enums.rs` — added `CreatePublicReview` permission.
  - `crates/models/src/entity/review.rs` — derived `SimpleObject` on the entity model.
  - `crates/graphql/src/object/review.rs` *(new)* — `Review` object type.
  - `crates/graphql/src/object/mod.rs` — registered the `review` object module.
  - `crates/graphql/src/input/review.rs` *(new)* — `ReviewInput` + unit tests.
  - `crates/graphql/src/input/mod.rs` — registered the `review` input module.
  - `crates/graphql/src/mutation/review.rs` *(new)* — `ReviewMutation` / `createReview`.
  - `crates/graphql/src/mutation/mod.rs` — wired `ReviewMutation` into `ContentMutations`.
  - `crates/graphql/src/object/media.rs` — added `reviews` and `averageRating` resolvers.
- **Key commits:** [To be added when committed]
- **Approach decisions:**
  - **Reused the existing DB layer.** The `reviews` entity and table already existed, so no migration was needed.
  - **Gated only public reviews.** Because the gate depends on the request's `is_private` value, it is a *runtime* check inside the mutation (`enforce_permissions`) rather than a static `#[graphql(guard)]` attribute. Private reviews are intentionally ungated.
  - **Privacy defaults to safe.** `is_private` defaults to `true`, so a client must explicitly opt into a public review (which then requires the permission).
  - **Author from session, not input.** The reviewing user is set from `AuthContext`, preventing forged authorship.
  - **`averageRating` excludes private reviews** to avoid leaking a user's private rating through the public aggregate; returns `null` (not `0`) when there are no public reviews.
  - **Direct queries, not DataLoaders.** Suitable for the single-book `BookOverviewScene`; noted as a future change if reviews ever render across lists of books (to avoid N+1 queries).


## Pull Request

**PR Link:** [To be added]

**PR Description:** [To be added]

**Maintainer Feedback:**
- [Date]: [To be added]

**Status:** In Progress

---

## Learnings & Reflections

### Technical Skills Gained

[To be filled in]

### Challenges Overcome

[To be filled in]

### What I'd Do Differently Next Time

[To be filled in]

---

## Resources Used

- [Stump contributing guide](https://github.com/stumpapp/stump/blob/develop/CONTRIBUTING.md)
- [GitHub issue #119](https://github.com/stumpapp/stump/issues/119)
