# Reproduction: the nix manager puts the GitHub token in the process command line

Minimal reproduction for the Renovate pull request
**"fix(nix): pass the GitHub token in the environment, not on the command line"**.

PR: <!-- replace with the PR link once it is open -->

## Current behavior

When the nix manager updates `flake.lock`, it appends the GitHub token to the
command it runs:

```ts
// lib/modules/manager/nix/artifacts.ts
cmd += `--extra-access-tokens github.com=${quote(token)} `;
```

`/proc/<pid>/cmdline` is world-readable on Linux, so **any local user can read the
token with `ps` for as long as `nix flake update` runs**. On a self-hosted Renovate
this includes every other account on the host.

## Expected behavior

The token should reach nix through the environment instead. `/proc/<pid>/environ` is
mode `0400`, readable only by the process owner, and nix accepts the same setting
from `NIX_CONFIG`.

## No secret is needed to reproduce this

Renovate does not validate the token before putting it on the command line: the
code only tests it for truthiness. Any string is enough, so this repository
contains no credentials and you can reproduce it with an obviously fake one.
Renovate's own unit test demonstrates the same thing with the literal token
`token`.

## How to observe it

1. Clone this repository.
2. Run Renovate against it with a fake token, for example:

   ```sh
   GITHUB_COM_TOKEN=ghp_FAKE_NOT_A_REAL_TOKEN \
     renovate --platform=local
   ```

3. While it runs, from another shell:

   ```sh
   ps -ef | grep -- --extra-access-tokens
   ```

   or watch the process directly:

   ```sh
   grep -a extra-access-tokens /proc/*/cmdline
   ```

You will see the token in the command line of the `nix` process.

The same thing without Renovate in the way, which isolates the mechanism to nix and
`/proc` rather than to Renovate's own code:

```console
$ nix --extra-experimental-features 'nix-command flakes' \
      --extra-access-tokens github.com=ghp_FAKE_NOT_A_REAL_TOKEN flake update &
$ grep -a extra-access-tokens /proc/*/cmdline
nix --extra-experimental-features nix-command flakes --extra-access-tokens github.com=ghp_FAKE_NOT_A_REAL_TOKEN flake update
$ stat -c '%A %n' /proc/self/cmdline /proc/self/environ
-r--r--r-- /proc/self/cmdline
-r-------- /proc/self/environ
```

## Why this repository looks the way it does

- `flake.nix` declares one `github:` input, which is the least that makes the nix
  manager act.
- `flake.lock` must exist, or `updateArtifacts` returns early and the command never
  runs.
- `renovate.json` enables the nix manager, which ships `enabled: false`, and turns on
  `lockFileMaintenance` so there is always an update to perform rather than having to
  wait for `nixos-unstable` to move.

## Observed on

- Renovate `main` at commit `b97f88fb6`
- nix 2.34.9 on x86_64-linux
