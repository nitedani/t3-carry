# nitedani/t3code: carried T3 nightlies

This fork builds [pingdotgg/t3code](https://github.com/pingdotgg/t3code) nightlies with a few upstream PRs merged in, for nitedani's machines. The only branch is `carry`: this README, `.github/workflows/carry.yml` and `fixes/`. The workflow fetches upstream's source itself.

## Requests

Each machine has one open issue here, opened by its own gh login (`nitedani` or `nitedani-agent`, listed in carry.yml; other authors are ignored). The machine keeps the number in `~/.local/share/t3-carry/issue`. The body is up to 10 lines, nothing else:

```
pick 31488 809f8bb59a0e2363ff284562645ba0f4cebd60c8
```

The sha must be a commit of that PR; a newer push is built only once the line names it. The key of a request, computed the same way by the machine:

```bash
picks=$(tr -d '\r' <<<"$body" | grep -E '^pick [1-9][0-9]{0,6} [0-9a-f]{40}$' | LC_ALL=C sort -k2,2n)
key=$(printf '%s\n' "$picks" | sha256sum | cut -c1-16)
```

When upstream merged a pick's PR and its merge commit is in the newest nightly, the workflow removes its line and comments `dropped #N: merged upstream in <V>` (as github-actions[bot], one line).

## Builds

At 00:50 and 01:50 UTC, on a requester's issue change, on a push to `fixes/`, and by hand (`gh workflow run carry.yml -R nitedani/t3code`), the workflow takes the newest upstream nightly `v<V>`. For every key without a built release for it, it applies each pick (its fix patch if one exists, else a merge), then runs upstream's release steps for the Linux x64 CLI archive: install, typecheck, bundle, build and smoke test, with `dev` telemetry as upstream does outside pingdotgg.

The build gets its own version `<V'>` = `<V>.c<key>` (for example `0.1.11-nightly.20261117.4825.c32b5b524027a2d1b`), which sorts above `<V>` and below the next nightly, so it never shares an install with the official `<V>` or another pick set. The result is the prerelease `c/<key>/v<V'>`, tagged on `carry`. Its body and its `carry.json` asset are the same JSON: `nightly` (`<V>`), `version` (`<V'>`), `state` (`built`, `not-applied` or `failed`), the failed `step`, the conflicting `files`, the run URL, and each pick with its own `state`: `carried`, the set's state for the pick it failed on, or `blocked` for the others. A built release also has `t3-<V'>-linux-x64.tar.gz` and `SHA256SUMS`, and the machine installs it with:

```bash
T3CODE_RELEASE_BASE_URL=https://github.com/nitedani/t3code/releases/download/c/<key> t3 update --yes --channel nightly --version <V'>
```

On anything but a `built` release from the last 36 hours, or when that update fails, the machine takes the plain nightly (`t3 update --yes --channel nightly`; T3 counts `<V'>` as stable). Each run keeps the newest two releases of each requested key and every release younger than a day, and deletes every other `c/*` release with its tag.


Settings: default branch `carry`, Issues on, everything else GitHub's defaults.
