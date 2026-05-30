# Homepage Resume Enhancement Design

## Summary

Enhance the personal site's landing page so it feels more substantial than a digital business card, while still staying lightweight and fully aligned with the existing LoveIt theme.

The chosen direction is **Variant A — narrative intro**:

- keep the current LoveIt homepage hero/profile block
- strengthen the hero copy using existing configuration fields
- add a short narrative "About me" section below the hero using standard Hugo homepage content
- **do not modify the theme's layout, templates, or styling for the final implementation**

This design intentionally avoids custom theme work. It relies only on configuration and supported content features already provided by Hugo + LoveIt.

## Goals

- Present a stronger professional identity on the homepage
- Reflect the current role more accurately: leadership + architecture + hands-on backend
- Add a concise professional summary without turning the homepage into a full CV
- Reuse LoveIt's built-in capabilities instead of customizing the theme
- Keep the future `/resume` page separate and more detailed

## Non-goals

- Do not build the full resume page yet
- Do not add a job timeline to the homepage
- Do not introduce custom homepage sections via theme template overrides
- Do not add custom CSS/JS for the final implementation
- Do not modify `themes/LoveIt/` for visual behavior or layout changes

## Constraints

- The site currently uses Hugo with the LoveIt theme as a git submodule
- The project is currently documented as a config-only homepage with no `content/` files
- The final implementation should stay feasible **without theme customization**
- The homepage should remain short, readable, and professional
- If this design is approved, adding `content/_index.md` becomes an intentional project-level change and `AGENTS.md` must be updated to reflect that new reality

## Existing theme capabilities to reuse

Based on LoveIt documentation and local theme inspection, the final design can reuse these supported features:

1. `params.home.profile.title`
   - controls the homepage title
2. `params.home.profile.subtitle`
   - supports HTML, but the final implementation should use plain text only
   - suitable for the current role line
3. `params.home.profile.disclaimer`
   - supports HTML, but the final implementation should use plain text only
   - suitable for one concise supporting line
4. homepage body content via `content/_index.md`
   - the current LoveIt homepage template renders `.Content` below the profile hero on the homepage
   - this is standard Hugo/LoveIt behavior and does not require theme modification
   - suitable for a short narrative intro section

These are sufficient for the chosen design. No theme override is required.

## Proposed homepage structure

### 1. Existing hero block, with improved copy

Keep the existing LoveIt profile block exactly as the theme renders it today.

Use it for:

- **Name**: `Ciro Lucio Tecce`
- **Role line**: `Engineering Manager at Core Reply | Software Architect`
- **Short supporting line**: one sentence summarizing leadership, architecture, and backend focus
- **Social links**: unchanged, still visible in the hero

### 2. Narrative introduction below the hero

Use standard homepage content in `content/_index.md` for a short "About me" section.

Recommended structure:

- heading: `About me`
- 2 short paragraphs
- optional final line with a few highlighted strengths written as plain text or a compact list

This section should read like a curated introduction, not like a resume entry.

## Recommended copy model

### Hero subtitle

```text
Engineering Manager at Core Reply | Software Architect
```

### Hero supporting line

Draft direction:

```text
I lead engineering teams, shape software architecture, and stay close to business needs while delivering great quality software for modern architectures.
```

This line should remain short and scannable.

### About section tone

The homepage summary should be:

- professional but approachable
- narrative rather than list-heavy
- balanced across leadership, architecture, and implementation
- selective rather than exhaustive

### About section content focus

The narrative should emphasize:

- current leadership role
- hands-on architectural involvement
- backend and distributed systems background
- experience in financial platforms / enterprise systems
- interest in connecting technical execution with team growth and delivery

## Content boundaries

To keep the homepage focused, exclude the following from the final landing page:

- full work history
- detailed technologies list
- education details
- publication details
- business unit / P&L detail unless it naturally fits later copy revisions

Those belong on the future `/resume` page.

## Implementation outline

The final implementation should be limited to project-level content/config changes:

### `config.toml`
Update only:

- `[params.home.profile].subtitle`
- `[params.home.profile].disclaimer`

Optional only if separately justified by existing site metadata needs:

- `[params.author].name`
- `[params.author].email`
- `[params.author].link`

### `content/_index.md`
Add the homepage narrative content:

- `## About me`
- 2 short paragraphs
- optional short strengths line

## No-theme-customization rule

For this phase, the implementation must **not**:

- edit `themes/LoveIt/layouts/...` to change homepage markup
- add custom layout overrides in local `layouts/` to alter the homepage structure
- add custom CSS solely to reshape the homepage layout
- introduce prototype switchers or variant logic into the final site

Allowed changes are only:

- content
- config
- standard Hugo homepage markdown

## Acceptance criteria

The design is correctly implemented when all of the following are true:

- the homepage hero remains structurally the same as LoveIt renders it by default
- the role line is updated through `config.toml`
- the supporting sentence is updated through `config.toml`
- the homepage includes one `About me` section rendered from `content/_index.md`
- there are no edits under `themes/LoveIt/` for homepage layout or styling
- there are no local layout overrides used to reshape the homepage
- there is no custom CSS or JS added for the final homepage implementation
- `AGENTS.md` is updated to reflect that the site is no longer strictly config-only if `content/_index.md` is adopted

## Why this approach fits

This approach works because it gives the homepage more substance while preserving the strengths of the current site:

- still simple
- still fast
- still theme-native
- easy to maintain
- easy to evolve later with a dedicated `/resume` page

It also avoids prematurely committing to custom layout work before validating whether the stronger copy alone is enough.

## Future phase

Once the homepage copy is finalized, a second phase can introduce a dedicated `/resume` page using standard Hugo content pages and navigation, again preferably without modifying the theme unless a clear limitation is reached.

## Open decisions for implementation

Before implementation, finalize:

1. the exact one-line supporting sentence in the hero
2. the final 2-paragraph homepage summary text
3. whether to include a short strengths line at the end of the About section

## Decision

Proceed with a **theme-native homepage enhancement** using:

- `config.toml` for hero copy
- `content/_index.md` for the narrative introduction
- no modifications to LoveIt's templates or styles for the final landing page implementation
