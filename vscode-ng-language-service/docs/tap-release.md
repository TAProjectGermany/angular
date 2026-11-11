# TAP: Angular Language Service release (.vsix)

Our fork only changes the VS Code extension (`vscode-ng-language-service`).
The `.vsix` is built locally; Angular npm packages are not published.

## Branch model

- `upstream` is the original `angular/angular`, `origin` is our fork.
- One branch per release line: `tap/21.2.x`, `tap/22.2.x`, ...
  Each branch = an upstream tag (e.g. `v22.2.5`) + our commits on top.
- **Never merge** branches of different versions into each other. Use cherry-pick / rebase only.

Our patches:

| Commit | Purpose |
| --- | --- |
| `fix: completely disable the Angular language service for TypeScript files` | removes `{scheme: 'file', language: 'typescript'}` from `documentSelector` in `client/src/client.ts` |
| `release: bump VSCode extension version to X.Y.Z-tap.N` | version in `vscode-ng-language-service/package.json` |
| `docs: add TAP release guide` | this file |

To list the patches on a branch:

```bash
git log --oneline v22.2.5..tap/22.2.x
```

## Build (in WSL)

Build in WSL (Ubuntu), not on Windows. Bazel builds of this repo break on Windows
(`MODULE.bazel.lock` mismatch caused by CRLF, `--config=release` stamping crashes in `ng-dev`).
Keep the clone on the Linux filesystem (`~/...`), not under `/mnt/c` or `/mnt/e`: it is much
faster and has LF line endings.

Prerequisites in WSL: `git`, `node` (see `.nvmrc`), `pnpm`.

### First time: clone

```bash
git clone https://github.com/TAProjectGermany/angular.git ~/angular
```

### Build

Push the branch from Windows first (`git push origin tap/22.2.x`), then in WSL:

```bash
cd ~/angular
git fetch origin
git checkout -f -B tap/22.2.x origin/tap/22.2.x
pnpm install --frozen-lockfile
pnpm bazel build //vscode-ng-language-service:vsix --config=release
```

To build unpushed commits, fetch directly from the Windows checkout instead of `origin`:

```bash
git -c safe.directory='*' fetch /mnt/e/TAPGitHub/angular tap/22.2.x
git checkout -f -B tap/22.2.x FETCH_HEAD
```

Output: `~/angular/dist/bin/vscode-ng-language-service/ng-template.vsix`.
The first build is slow (the compiler is built from source); subsequent builds use the Bazel cache.

### Copy to Windows and install

```bash
VERSION=$(node -p "require('./vscode-ng-language-service/package.json').version")
cp dist/bin/vscode-ng-language-service/ng-template.vsix /mnt/c/Users/<windows-user>/Downloads/ng-template-$VERSION.vsix
```

Then in Windows (PowerShell), adjust the path/version:

```powershell
code --install-extension $HOME\Downloads\ng-template-22.2.0-tap.1.vsix --force
```

Do not run `code --install-extension` inside WSL: that installs the extension into the WSL
remote, not into Windows VS Code.

## Updating to a new upstream patch (e.g. 22.2.5 → 22.2.6)

```bash
git fetch upstream --tags
git rebase --onto v22.2.6 v22.2.5 tap/22.2.x
git push --force-with-lease origin tap/22.2.x
```

If upstream changed the extension version in `package.json`, set it to `<new version>-tap.1`.

## Moving to a new major/minor (e.g. 22.2.x → 23.0.x)

```bash
git fetch upstream --tags
git tag -l 'v23.0.*' --sort=-v:refname        # find the latest tag (tags are prefixed with v!)
git switch -c tap/23.0.x v23.0.3
git cherry-pick <fix-commit> <docs-commit>    # do NOT carry over the version commit
```

Then set the version to `<extension version>-tap.1` in
`vscode-ng-language-service/package.json` (use the current `"version"` value there; the
extension has its own versioning, which does not match the framework tag) and commit it
with the `release:` type.

If the cherry-pick conflicts in `client.ts`, just comment out the line
`{scheme: 'file', language: 'typescript'},` in `documentSelector`.

## Tips

- `git config rerere.enabled true` makes git remember conflict resolutions.
- The commit hook rejects the `chore` type; use `fix`, `docs` or `release`.
