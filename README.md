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

[To be filled in after exploring the codebase]

### Proposed Solution

[To be filled in]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Stump needs a ratings and reviews feature for books, with RBAC controls, public/private visibility, and display on the BookOverviewScene.

**Match:** [To be filled in after reviewing similar patterns in the codebase]

**Plan:**
1. Add database schema fields/tables for ratings and reviews
2. Implement backend API endpoints to create, read, and manage ratings/reviews
3. Build the frontend UI components for rating input and review text
4. Add RBAC toggle logic for admins
5. Display average rating and reviews on `BookOverviewScene`
6. Write tests

**Implement:** [Link to branch/commits to be added as work progresses]

**Review:** [Self-review checklist to be completed]

**Evaluate:** [Verification approach to be added]

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: Rating can be created and retrieved for a book
- [ ] Test case 2: Review text is correctly associated with a rating
- [ ] Test case 3: RBAC toggle correctly restricts rating/review access per user

### Integration Tests

- [ ] Rating and review data flows correctly from backend to BookOverviewScene
- [ ] Public/private visibility settings are respected across users

### Manual Testing

[To be filled in]

---

## Implementation Notes

### Week 1 Progress

[To be filled in]

### Week 2 Progress

[To be filled in]

### Week 3 Progress

[To be filled in]

### Code Changes

- **Files modified:** [To be filled in]
- **Key commits:** [To be filled in]
- **Approach decisions:** [To be filled in]

---

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
