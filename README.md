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

[To be filled in as environment is set up locally]

### Steps to Reproduce

1. Open any book in Stump
2. Navigate to the BookOverviewScene
3. Observe that there is no rating or review UI present

### Reproduction Evidence

- **Commit showing reproduction:** [To be added]
- **Screenshots/logs:** [To be added]
- **My findings:** [To be added]

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

HOW TO RUN
==========

The app requires two terminals: one for the backend, one for the frontend.

1) Start the backend
   cd /Users/jonayet/Desktop/ai301/stump
   source .devenv.sh
   cargo run -p stump_server

2) Start the frontend (in a separate terminal)
   cd /Users/jonayet/Desktop/ai301/stump
   source .devenv.sh
   yarn web dev
