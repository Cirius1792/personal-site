# Homepage Resume Enhancement Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enhance the personal site landing page with stronger hero copy and a narrative "About me" section, using only config.toml and standard Hugo content — no theme modifications.

**Architecture:** The LoveIt theme renders the homepage hero block from config keys (`subtitle`, `disclaimer`, etc.) and also renders `.Content` from `content/_index.md` below the hero. We add only configuration and one markdown file.

**Tech Stack:** Hugo v0.162.0 (extended), LoveIt theme, Firebase Hosting

**Design spec:** `docs/superpowers/specs/2026-05-30-homepage-resume-design.md`

---

### Task 1: Update hero copy in config.toml

**Files:**
- Modify: `config.toml` (subtitle and disclaimer lines)

- [ ] **Step 1: Update `subtitle` with the role line**

  Change:
  ```toml
  subtitle = "Software Engineer"
  ```
  To:
  ```toml
  subtitle = "Engineering Manager at Core Reply | Software Architect"
  ```

- [ ] **Step 2: Update `disclaimer` with the supporting sentence**

  Change:
  ```toml
  disclaimer = ""
  ```
  To:
  ```toml
  disclaimer = "I lead engineering teams, shape software architecture, and stay close to business needs while delivering great quality software for modern architectures."
  ```

  *(Note: the exact text is an open decision — confirm with user before finalizing)*

- [ ] **Step 3: Verify by rebuilding**

  Run:
  ```bash
  rm -rf public/ resources/_gen/
  hugo
  ```
  Expected: site builds without errors

- [ ] **Step 4: Commit**

  ```bash
  git add config.toml
  git commit -m "feat: update hero copy with role line and supporting sentence"
  ```

---

### Task 2: Create content/_index.md with narrative About section

**Files:**
- Create: `content/_index.md`

- [ ] **Step 1: Write the homepage content**

  Create `content/_index.md` with the following structure:

  ```markdown
  ## About me

  I'm an Engineering Manager at Core Reply, where I lead engineering teams, shape software architecture, and drive delivery across complex projects. My background spans backend engineering, distributed systems, and platform architecture — with a focus on building reliable, scalable systems that serve real business needs.

  Over the years I've worked extensively in financial platforms and enterprise environments, balancing hands-on technical involvement with team leadership. I care deeply about connecting technical execution with team growth, and I believe great software comes from clear thinking, honest collaboration, and disciplined engineering.

  **Leadership · System Architecture · Backend Engineering · Distributed Systems**
  ```

  *(Note: the exact text is an open decision — confirm with user before finalizing)*

- [ ] **Step 2: Verify by rebuilding**

  Run:
  ```bash
  rm -rf public/ resources/_gen/
  hugo
  ```
  Expected: site builds without errors; `public/index.html` contains the About section content

- [ ] **Step 3: Commit**

  ```bash
  git add content/_index.md
  git commit -m "feat: add narrative About me section on homepage"
  ```

---

### Task 3: Update AGENTS.md to reflect new content structure

**Files:**
- Modify: `AGENTS.md`

- [ ] **Step 1: Update architecture diagram**

  Add `content/` to the architecture diagram:
  ```
  content/             ← homepage narrative content (_index.md)
  ```

- [ ] **Step 2: Update "Critical" note**

  Change:
  ```
  **Critical:** There are **no** `content/` directories or `.md` content files.
  ```
  To:
  ```
  **Critical:** The site has one content file: `content/_index.md` for the homepage narrative.
  ```

- [ ] **Step 3: Update "What NOT to do"**

  The first bullet says:
  ```
  ❌ Don't create `content/` pages
  ```
  This is now intentional and should be removed or updated to reflect that `content/_index.md` exists and is expected.

- [ ] **Step 4: Add note about homepage content**

  In the "Common edits" section, add an entry:
  ```
  | Homepage narrative text | Edit `content/_index.md` |
  ```

- [ ] **Step 5: Commit**

  ```bash
  git add AGENTS.md
  git commit -m "docs: update AGENTS.md to reflect content/ directory"
  ```

---

### Task 4: Full build verification

- [ ] **Step 1: Full clean build**

  ```bash
  rm -rf public/ resources/_gen/
  hugo
  ```
  Expected: exits 0, no errors

- [ ] **Step 2: Verify output**

  ```bash
  grep -c "Engineering Manager" public/index.html
  grep -c "About me" public/index.html
  ```
  Expected: both return > 0

- [ ] **Step 3: Dev server smoke test (optional)**

  ```bash
  hugo server &
  sleep 3
  curl -s http://localhost:1313 | grep -c "About me"
  kill %1 2>/dev/null
  ```

---

## Open decisions (needs user input)

Before executing this plan, confirm with the user:

1. **Hero supporting sentence** — exact text for `disclaimer` field
2. **About section copy** — exact 2-paragraph narrative + optional strengths line
3. **Whether to include a strengths line** at the end of the About section

The plan uses the design spec's draft copy as defaults, but these should be finalized before execution.
