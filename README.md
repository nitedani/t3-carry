# t3-carry: carried T3 nightlies

This repo builds [pingdotgg/t3code](https://github.com/pingdotgg/t3code) nightlies with a few upstream PRs merged in, for nitedani's machines. It holds this README and `.github/workflows/carry.yml`, nothing else; the workflow fetches upstream's source itself.

## Requests

Each machine has one open issue here, opened by its own gh login (`nitedani` or `nitedani-agent`, `REQUESTERS` in carry.yml; other authors' issues are ignored). The machine keeps the number in `~/.local/share/t3-carry/issue`, and its dashboard is the only writer of the body: up to 10 lines, one per PR, nothing else.

```
pick 31488 809f8bb59a0e2363ff284562645ba0f4cebd60c8
pick 37306 8c549d97753af96dba62219c9ae19f61ff7c3431 <res40>
```

The sha must be a commit of that PR; a newer push is built only once the line names it. The optional third field is a resolution (below). The key of a request, computed the same way by the machine:

```bash
picks=$(tr -d '\r' <<<"$body" | grep -E '^pick [1-9][0-9]{0,6} [0-9a-f]{40}( [0-9a-f]{40})?$' | LC_ALL=C sort -k2,2n)
key=$(printf '%s\n' "$picks" | sha256sum | cut -c1-16)
```

## Builds

At 00:50 and 01:50 UTC and by hand (`gh workflow run carry.yml -R nitedani/t3-carry`), the workflow takes the newest upstream nightly `v<V>` and builds every key without a built release for it; when a requester's issue is opened, edited or reopened, only that issue's key. A key is built only when every pick applies (below): the workflow first merges them as the build would, in a blobless clone, and a key with a pick that doesn't apply is not built, since a machine installs no build that leaves a pick out; a Resolve press fixes the pick first. An issue's runs are named `carry #<issue>`, and a newer edit of the issue cancels its running or waiting run; scheduled and manual runs are named `carry #nightly` and wait for each other. It only reads issues.

Per pick, in PR order:
- When upstream merged the PR and its merge commit is in `v<V>`, the pick is `merged` and left out; removing its line is the dashboard's job.
- When the sha is not a commit of the PR, it is `unpinned` and left out.
- Otherwise it is merged into the tree: its resolution if the line names one, else the sha. A pick that conflicts is `doesnt-apply` and left out; the others still go in.

Then upstream's release steps for the Linux x64 CLI archive run: install, typecheck, bundle, build and smoke test, with `dev` telemetry as upstream does outside pingdotgg. A pick merged by its resolution also runs `vp test run` on the test files its PR changes. When any step fails, the nightly alone is built, then in PR order each pick over the picks kept so far: a pick is kept when that builds, and is `doesnt-build` when it doesn't. So a resolution made over an earlier pick is built with that pick. A `doesnt-build` pick is then built over the nightly alone, and over the first one, two, … of the kept picks, until it fails: its `cause` is `alone`, or the kept pick added last. When the nightly alone fails, every pick is `doesnt-build` and `recipe` is `broken`. On a night that builds, none of this runs.

The build gets its own version `<V'>` = `<V>.c<key>` (for example `0.1.11-nightly.20261117.4825.c32b5b524027a2d1b`), which sorts above `<V>` and below the next nightly, so it never shares an install with the official `<V>` or another pick set. The result is the prerelease `c/<key>/v<V'>`, tagged on this repo's `main`. Its body and its `carry.json` asset are the same JSON, schema 3, which says what the build did, not what upstream says of each PR now:

```json
{
  "schema": 3,
  "nightly": "<V>",
  "version": "<V'>",
  "state": "built",
  "recipe": "ok",
  "picks": [
    {"pr": 31488, "sha": "<sha40>", "res": null, "state": "carried", "cause": null}
  ],
  "run": "<the run's URL>"
}
```

- `state`: `built` when at least one pick is `carried`, else `not-built`.
- `recipe`: `ok`, or `broken` when the nightly alone fails to build with these steps.
- Each pick's `state`: `carried`, `doesnt-apply`, `doesnt-build`, `merged` or `unpinned`.
- `cause`: for a `doesnt-build` pick, `alone` or the number of the earlier pick it fails with; else `null`, also when `recipe` is `broken`.

A `built` release also has `t3-<V'>-linux-x64.tar.gz` and `SHA256SUMS`, and the machine installs it with:

```bash
T3CODE_RELEASE_BASE_URL=https://github.com/nitedani/t3-carry/releases/download/c/<key> t3 update --yes <V'>
```

A machine installs a `built` release from the last 36 hours only when every pick it requests is `carried` in it; otherwise it keeps what it runs until a Resolve press, so a release with a `doesnt-build` pick is published only for its `cause`. With no picks, the machine takes the plain nightly (`t3 update --yes --channel nightly`; T3 counts `<V'>` as stable). Each run keeps the newest two releases of each requested key and every release younger than a day, and deletes every other `c/*` release with its tag. Each run also re-enables the workflow through GitHub's API, since GitHub disables a public repo's schedules after 60 days without activity.

## Resolve

A pick that doesn't apply or build is resolved from its machine's dashboard: one Resolve button starts a T3 thread on that machine. For our own open PR it rebases the PR itself and moves the line to its new head. For any other PR it merges the sha onto the nightly (with `--no-ff`, over the earlier picks only when it conflicts with them), resolves it, pushes the commit as branch `t3-carry/<pr>` of the requester's own fork `<login>/t3code`, and adds its sha to the line. The build fetches a resolution only from `https://github.com/<issue author>/t3code`, and uses it only when it holds the pick and its own first-parent line starts with a merge that brings in nothing but the pick and code already in that night's tree; otherwise the pick is `doesnt-apply`. Git's three-way merge then carries the resolution onto later nightlies. The dashboard deletes a `t3-carry/<pr>` branch once no line names it.

When the nightly alone doesn't build (`recipe: broken` in carry.json, so upstream changed how it builds), no pick is at fault: a Resolve press on private, whose login owns this repo, brings the build steps of `carry.yml` in line with upstream's `release.yml` and `release-desktop.yml` at that nightly and restarts the builds. A press on any other machine leaves a note saying so.

Settings: default branch `main`, Issues on, everything else off.
