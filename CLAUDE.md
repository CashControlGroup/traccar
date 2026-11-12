# CLAUDE.md

Guidance for Claude Code sessions in `CashControlGroup/traccar`.

## What this is

Our fork of [Traccar](https://github.com/traccar/traccar), the Java/Netty GPS tracking server,
deployed white-labelled as **Tracking Shop**. The React front-end is the `traccar-web` git
**submodule**, itself our fork `CashControlGroup/traccar-web`. Infrastructure, deploy workflow
and runbooks live in `CashControlGroup/traccar-infra`, which has its own CLAUDE.md.

> **Global standards do not apply here.** Jamie's global standards (C#/.NET, DDD layering,
> Blazor/MAUI) describe our greenfield work. This is upstream Java and React: match upstream's
> patterns in the file you are editing, and keep our changes small so upstream merges stay cheap.

## Branches

- `master` mirrors `traccar/traccar` master. Never commit to it.
- `custom/main` is what we build and deploy. Our customisations on it are **additive files
  only**: `.github/workflows/{build-and-push,custom-ci,upstream-sync}.yml`, `docker/Dockerfile`,
  `branding/web/`, `.gitattributes` and this file. Do not edit upstream Java source unless the
  change is unavoidable; prefer configuration (`traccar.xml` on the VM, server attributes).
- **Front-end changes are made in `traccar-web`**, not here. Its `custom/main` carries our UI
  work and its own CLAUDE.md. This repo only records which web commit to ship.

## Build

- Server: `./gradlew assemble` (JDK 21+). Tests: `./gradlew test`; checkstyle runs in `check`.
- Container: `docker build -f docker/Dockerfile .` from a recursive checkout. It builds the web
  bundle (Node 22), overlays `branding/web/` onto `traccar-web/public/`, builds the server jar and
  layers both onto the official `traccar/traccar:6-alpine` runtime.
- On Windows, `.gitattributes` keeps `gradlew` and `*.sh` LF; without it the Linux build fails
  with `./gradlew: not found`.

## Deploy chain

1. Push to `custom/main` → `build-and-push` builds `docker/Dockerfile`, pushes to ACR, then
   repository-dispatches `deploy-image` to `CashControlGroup/traccar-infra`.
2. traccar-infra's `deploy` workflow rolls the tag onto the VM (`az vm run-command`, compose up)
   and health-checks `/api/server`.
3. Front-end: merging a PR into `traccar-web` `custom/main` runs that repo's `ship-to-server`
   workflow, which moves the `traccar-web` submodule here to the merged commit and pushes to
   `custom/main` — step 1 follows. To ship by hand, do the same here:
   `git update-index --cacheinfo 160000,<web sha>,traccar-web`, then commit and push.

`.gitmodules` (upstream's) sets `ignore = all` on the submodule, so `git status`, `git diff` and
`git commit -a` never show a pointer change. Check it with `git rev-parse HEAD:traccar-web` (or
`git ls-tree HEAD traccar-web`) and stage it with `update-index` as above.

Repository-dispatch, workflow-dispatch and schedule triggers only run on the **default branch**
(`master` here), so workflows on `custom/main` respond to push and pull_request only. That is
why the weekly `upstream-sync` schedule never fires: take releases by hand as below.

## Taking an upstream release

1. In `traccar-web`: fast-forward `master` to the commit the new upstream tag's submodule points
   at (`git ls-tree <tag> traccar-web`), then merge that commit into `custom/main` (PR, merge
   commit). Check every file in `branding/web/` still exists in `traccar-web/public/`.
2. Here: branch `sync/upstream-YYYYMMDD` from `custom/main`, `git merge <tag>`. If the only
   conflict is the `traccar-web` pointer, set it to the head of traccar-web `custom/main`
   (`update-index`, as above), then confirm with `git ls-tree HEAD traccar-web`.
3. `git diff <tag> HEAD --stat` must list only our additive files. Open a PR into `custom/main`;
   `custom-ci` builds the full image (without pushing) on it.
4. Merge with a **merge commit, never squash** — squashing flattens upstream history and breaks
   later merge-bases. Merging deploys to production.
