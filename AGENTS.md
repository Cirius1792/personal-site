# AGENTS.md — Personal Site

## Project at a glance

- **What:** A static one-page personal landing site (name, title, bio, social links). No blog, no posts, no content pages.
- **Stack:** Hugo → LoveIt theme → Firebase Hosting → GitHub Actions
- **Live:** https://ciroluciotecce.it
- **Firebase project:** `clt-personal-site`

## Architecture

```
config.toml          ← single source of truth (profile, bio, socials, baseURL)
content/             ← homepage narrative content (_index.md)
.pi/agents/          ← project-local Pi agent definitions (worker agent, etc.)
static/              ← served as-is (avatar images, profile pics)
themes/LoveIt/       ← git submodule (the Hugo theme)
public/              ← built output (committed, deployed by Firebase)
.github/workflows/   ← deploy pipelines
firebase.json        ← tells Firebase to serve public/
```

**Critical:** The site has one content file: `content/_index.md` for the homepage narrative. Everything else is configured in `config.toml`. The `archetypes/` directory exists but is unused.

## Hugo specifics

- **Theme:** LoveIt (https://github.com/dillonzq/LoveIt) — managed as a git submodule at `themes/LoveIt`
- **Hugo version:** extended v0.121+ (footer in `public/index.html` shows the version used)
- **Build output:** `hugo` writes to `public/`
- **Dev server:** `hugo server` (hot-reload works for config.toml and static/ changes)

### Theme submodule

```bash
# Initialize/update after cloning
git submodule update --init --recursive

# To pull latest LoveIt
cd themes/LoveIt && git pull && cd ../..
```

## Configuration — config.toml

This is the only file you'll edit for content changes. Structure:

```toml
baseURL = 'https://ciroluciotecce.it'          # ← change if migrating domains
theme = 'LoveIt'

[params]
  [params.header.title]
    name = "Ciro Lucio Tecce"

  [params.home]
    [params.home.profile]
      enable = true
      gravatarEmail = ""
      avatarURL = "job-profile-pic.jpg"         # ← file in static/
      title = "Ciro Lucio Tecce"
      subtitle = "..."                          # ← bio text
      typeit = false
      social = true
      disclaimer = ""

  [params.social]
    GitHub = "Cirius1792"
    Linkedin = "ciro-lucio-tecce-3bb628175/"
    Instagram = "clt92/"
    Steam = ""
    Googlescholar = ""
    Email = "cirolucio.tecce@gmail.com"
```

### Common edits

| Want to change | Where |
|---|---|
| Name / title | `[params.header.title].name` and `[params.home.profile].title` |
| Bio / description | `[params.home.profile].subtitle` |
| Homepage narrative text | Edit `content/_index.md` |
| Avatar photo | Replace file in `static/`, update `avatarURL` |
| Social links | `[params.social]` section |
| Domain | `baseURL` |
| Subtitle animation | `typeit = true` |

## File conventions

- **Images:** Put in `static/`. Reference by filename only (e.g., `avatarURL = "photo.jpg"`). Hugo serves them at the root.
- **Built files:** `public/` is committed. It's the deployment artifact. Don't edit files in `public/` directly — always edit `config.toml` or `static/` and rebuild.
- **Pi project-local agents:** `.pi/agents/` contains Pi agent definitions for this repository. Currently includes `worker.md` — a delegated implementation worker that uses the `llama-cpp-cltec/Qwen3.6-35B-A3B-thinking` model. See the design spec at `docs/superpowers/specs/2026-05-30-worker-agent-design.md` before making prompt changes.
- **Resources cache:** `resources/_gen/` contains Hugo's SCSS compilation cache. Safe to delete if you have build issues.
- **`.hugo_build.lock`:** Create artifact from `hugo server`. Safe to delete.

## Deploy

### Automatic (push to main)
GitHub Action `firebase-hosting-merge.yml` builds and deploys on every push to `main`. No manual steps.

### Manual
```bash
hugo
firebase deploy --only hosting
```

### PR preview
`firebase-hosting-pull-request.yml` deploys a preview URL on PRs from the repo (not forks).

### Required secret
`FIREBASE_SERVICE_ACCOUNT_CLT_PERSONAL_SITE` — Firebase service account JSON. Set in repo Settings → Secrets.

## What NOT to do

- ❌ Don't add extra `content/` pages beyond `_index.md` — the site is designed for a single profile page
- ❌ Don't edit files in `public/` — they get overwritten on every build
- ❌ Don't delete `themes/LoveIt/` — it's a submodule and the site won't build without it
- ❌ Don't remove `public/` from git — it's the deployment artifact Firebase serves

## Common tasks

### Update avatar/profile picture
1. Replace the image in `static/`
2. Update `avatarURL` in `config.toml`
3. Commit and push

### Change bio text
1. Edit `subtitle` in `config.toml` under `[params.home.profile]`
2. Commit and push

### Change a social link
1. Edit the relevant key in `[params.social]` in `config.toml`
2. Commit and push

### Rebuild after config changes
```bash
rm -rf public/ resources/_gen/
hugo
```

### Switch Hugo theme
1. Update the submodule URL in `.gitmodules`
2. `git submodule update --init --recursive`
3. Update `theme = '...'` in `config.toml`
4. Adjust config keys if the new theme uses different settings

### Reference the Pi worker agent from another agent
1. Set `agent: worker` and include the full task payload using the input contract documented in the design spec
2. Include context, constraints/acceptance criteria, and working directory
3. The worker returns a structured report with one of: `DONE`, `DONE_WITH_CONCERNS`, `BLOCKED`, `NEEDS_CONTEXT`

## Keeping AGENTS.md up to date

This file is a living document. **Update it whenever the project structure or conventions change.** Specifically, after any of these events:

| Trigger | What to update |
|---|---|
| Adding content pages (blog posts, etc.) | Add a "Content structure" section; remove "no content pages" from the architecture and "What NOT to do" |
| Changing the Hugo theme | Update theme name, submodule URL, version requirements, and config key references |
| Adding new static assets or directories | Update the file structure diagram and conventions |
| Adding or changing project-local Pi agents | Update the file structure diagram, file conventions, common tasks, and reference the relevant design spec in `docs/superpowers/specs/` |
| Changing the hosting/deploy setup | Update the Deploy section (new CLI flags, new secrets, new workflows) |
| Adding new configuration keys | Document them in the `config.toml` walkthrough |
| New or removed GitHub Actions | Update the Deploy section and secrets list |
| Introducing a new tech dependency | Add it to the Tech stack table and Architecture |
| User gives new instructions about the project | Fold them into the relevant section |

### How to update

1. **Scan the repo** — run `find` or `ls` to verify the actual state of the filesystem
2. **Compare** against what AGENTS.md says
3. **Edit** AGENTS.md to match reality
4. **Commit** the updated AGENTS.md alongside the other changes

### Self-audit checklist

Periodically (e.g. before committing any change to this repo), ask:

- [ ] Does the architecture diagram match the actual directory structure?
- [ ] Are all file/directory references in AGENTS.md still accurate?
- [ ] Is the Hugo version still correct (check `public/index.html` footer or `hugo version`)?
- [ ] Are the config.toml walkthrough keys still valid for the current theme?
- [ ] Does the "What NOT to do" section still apply?
- [ ] Are all deploy instructions still valid (check `.github/workflows/`)?
- [ ] Are all secrets listed still required?

If *anything* has drifted, fix AGENTS.md. It is the source of truth for future agents working on this project.
