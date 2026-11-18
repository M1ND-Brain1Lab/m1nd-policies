# CLAUDE.md

Guidance for AI assistants (and humans) working in this repo. Read it in full
before every session; it governs every change.

## What this is

M1ND is a mental-performance app for athletes, starting with football. Teams
and clubs buy a license; players unlock the app with their team's access code
and use guided tools (breathing, relaxation, imagery, focus, self-talk), a
journal and a pre-match plan. Personal data stays on the phone; the backend
only manages licenses and anonymous usage statistics. This repo holds the
Impressum, privacy policy and terms of use, published on GitHub Pages.

Sibling repositories, checked out side by side:

- `m1nd`: the product (app, backend, psychologist panel).
- `m1nd-feedback`: the endpoint that turns in-app feedback into GitHub issues
  in `m1nd`.
- `m1nd-policies`: Impressum, privacy policy and terms of use, which the app,
  the store listings and the website link to.

A change that renames a URL, an endpoint or a contract one repo uses from
another updates every repo involved in the same task.

## Plan first, then execute

Every task that will change a repo starts with a plan, and nothing changes
until the user approves it.

- **Propose before touching anything.** Read the code and docs as much as
  the plan needs. Edit no file, run no command that writes, and make no
  commit, push or merge before approval.
- **The plan names** what changes and where, how it will be checked, and
  **every commit by its exact title**, in order, plus the release-notes
  bullet that goes into the last one. It ends with the pull request into
  `develop`: its title and description.
- **Execute only on a clear yes**, exactly as approved. Approval covers that
  plan only. A follow-up gets a plan of its own.
- **Stop and re-plan** when the plan stops fitting: another file or commit,
  a different commit title, a test that fails for an unforeseen reason, or
  a decision the plan left open.
- A question or an investigation that changes nothing needs no plan.

## Branches

| Branch | Role |
|---|---|
| `main` | Production. **Protected.** Changes only through a pull request from `develop`. |
| `develop` | Integration and **default** branch. Every piece of work lands here first. |
| `feat/<kebab-name>` | A new feature, cut from `develop`. |
| `fix/<kebab-name>` | A bug fix, cut from `develop`. |
| `refactor/<kebab-name>` | A restructuring with no behaviour change, cut from `develop`. |
| `claude/<name>` | A Claude Code session's worktree branch, cut from `develop`. |

- Never commit or push directly to `main`. Never force-push `main` or
  `develop`, and never delete either.
- **One branch per piece of work**, and for Claude Code, one git worktree per
  piece of work (`.claude/worktrees/<name>`). The main checkout stays on
  `develop` and is not edited directly.
- If `develop` moves while the work is in progress, merge `develop` into the
  work branch. Never rebase a pushed branch.
- **When the work is finished:** open a pull request into `develop` and
  squash-merge it, so the work lands as one commit (see "Squash commits").
  GitHub deletes the branch; then remove the worktree. Never merge a branch
  with uncommitted changes, and never leave a finished worktree behind.
- An investigation that ends with nothing committed just removes its
  worktree.


## Commits

Bracket tags, **not** Conventional Commits:

| Tag | Meaning |
|---|---|
| `[+]` | added |
| `[-]` | removed |
| `[*]` | bug fix |
| `[#]` | modification |

- **Title only, one line.** No body and no trailers (`Co-Authored-By` and
  the like). Capitalised and imperative, describing the effect:
  `[+] Add the access code screen`, `[*] Fix audio stopping on screen lock`.
- Optional ticket id in parentheses at the end:
  `[*] Keep the journal after a license renewal ({{TICKET_PREFIX}}-012)`.
- **One concern per commit.** A dependency or package bump gets its own
  commit. Never put unrelated changes in one commit or one squash.
- A database migration is its own commit, named after its effect:
  `[+] Add the licenses table with RLS`.
- A version or build bump is a commit of its own:
  **`[#] Update to X.Y.Z (N)`**, the marketing version and the build number.

### Squash commits

The pull request into `develop` becomes its squash commit: GitHub takes the
PR title as the commit title and the PR description as its message.

- **PR title:** one line in bracket-tag style, summing up the whole piece of
  work rather than repeating one of its commits. Use `[+]` for a feature,
  `[*]` for a fix, and so on. GitHub appends ` (#123)`.
- **PR description:** the branch's commits, one `- ` line each, oldest
  first, titles exactly as written. Leave out merges of `develop`. No
  trailers or attribution lines.

```text
[+] Add code activation against the backend (#12)

- [+] Add the activate Edge Function
- [+] Add the access code screen
- [*] Accept codes typed without dashes
```

## Release notes

In the **last commit of every task**, add one bullet per user-visible change
under `## Unreleased` in `docs/RELEASE_NOTES.md` (create the heading if
missing). Fold the notes into that commit; never make a separate release-notes
commit. At release, the heading becomes `## X.Y.Z — YYYY-MM-DD`.

## Versioning and releases

- Semver. The version and build number live in **one file**
  (`version.json` at the root of `m1nd`). The iOS and Android projects take
  their values from it through `scripts/`, so every target carries the same
  values.
- **Releasing:**
  1. Bump the version (its own commit, `[#] Update to X.Y.Z (N)`) and date
     the release notes.
  2. Open the release PR `develop → main` **as a draft**
     (`gh pr create --draft --base main --head develop`).
  3. Mark it ready (`gh pr ready`) only once `develop` is final: every push
     to `develop` re-runs CI on an open PR.
  4. Merge it with a **merge commit, never a squash**, so `main` stays an
     ancestor of `develop` and the two never need a back-merge.
  5. Tag the merge on `main`: `git tag -a vX.Y.Z -m "M1ND X.Y.Z"`
     and `git push origin vX.Y.Z`.
- Anything that publishes (a TestFlight or Play upload, a store release, a
  production database migration, a production deploy, a push to a `main` that
  deploys) happens only on the user's yes. Ask, never assume.
- iOS builds and uploads are made on a Mac by the team member responsible for
  iOS builds. Never assume a Mac is available.

## Secrets

- No secret is ever committed: tokens, keys, `.env` files, signing material
  (Android keystores, Apple certificates and profiles). Secrets live in the
  host's environment (Supabase, Netlify, Vercel, GitHub Actions secrets) or in
  a git-ignored local file that has a committed `.example` beside it.
- The Supabase `service_role` key and the code-hash pepper exist only on the
  server and in CI. The app and the panel ship only the public `anon` key.
- CI fails if a secrets file is present, so it always checks that a build
  without secrets works.

## CI

GitHub Actions, on Linux runners unless a job needs macOS:

- On every pull request into `develop`: app build and lint, panel build and
  lint, Supabase migrations applied from scratch and `supabase test db`.
- On merge to `develop`: deploy migrations and Edge Functions to the staging
  Supabase project.
- Production deploys of the backend: manual dispatch only, with approval.
- iOS builds stay off CI at first (they are made on a team member's Mac). If a
  macOS job is added later, run it only on pull requests into `main`, on manual
  dispatch and on a weekly schedule, never on every push.


---

## This repo: `m1nd-policies`

The Impressum (DDG § 5), the privacy policy (Datenschutzerklärung,
DSGVO/GDPR) and the terms of use (Nutzungsbedingungen), in German and
English. **The German terms are the binding version.** This repo is the only
copy: the app links here, and so do the App Store and Google Play listings
(both require a public privacy policy URL), the panel and the website footer.

**Layout:** static HTML, no build step, served by GitHub Pages from `main` at
the repo root: <https://m1nd-brain1lab.github.io/m1nd-policies/>.

```text
index.html                        links to every document in both languages
impressum.de.html                 impressum.en.html
datenschutzerklaerung.de.html     datenschutzerklaerung.en.html
nutzungsbedingungen.de.html       nutzungsbedingungen.en.html
style.css
.nojekyll                         Pages serves the files exactly as committed
```

### Rules

- **Claude never writes legal text from scratch.** The pages are skeletons:
  shared header, language switch, `<main>` with the section headings, and
  `{{Paste the text from the generator or lawyer here}}` where the text goes.
  The user generates the text (e.g. Datenschutz-Generator.de for the privacy
  policy, eRecht24 for the Impressum) or has a lawyer write it. Claude fills
  in only facts the user gives.
- The `<!-- Cover: … -->` comments in the privacy policy list what that
  section must cover for M1ND; remove them once the text is in.
- **Keep both languages in step.** A sentence changed in one language changes
  in the other in the same commit, and the sections stay in the same order.
- Update "Stand" / "Last updated" in both privacy-policy files whenever their
  content changes.
- Each page is standalone: a short shared header (charset, viewport,
  `style.css`, language switch) around the document. Edit only the text
  between the language switch and `</main>`.
- If a generator's free licence requires attribution, keep it, and give the
  repo no open-source licence.
- **The product keeps every policy URL in one file**
  (`app/src/config/legal-links.ts` in `m1nd`). Renaming a page or moving the
  site changes that file in the same release, and the store listings' privacy
  URL too.
- When the product starts collecting new data or uses a new processor
  (hosting, analytics, sign-in provider, a new region such as Mumbai or
  China), the privacy policy is updated before that release ships.
- Hindi or other languages may be added later; they follow the same rules,
  with German still binding.
- A push to `main` publishes the site: it happens only on the user's yes.
