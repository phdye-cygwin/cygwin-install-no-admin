# cygwin-install-no-admin

Groundwork for adding a no-admin option to
[cygwin/cygwin-install-action](https://github.com/cygwin/cygwin-install-action),
so Cygwin can be installed on runners whose account is not a local
administrator.

Nothing is implemented yet. This repository records what has been
established, so the work can be picked up without rediscovering it.

## The problem

`cygwin-install-action` cannot install Cygwin on a non-admin runner.

Without `--no-admin`, `setup.exe` tries to elevate itself. From the 2013
patch by Shaddy Baddah that introduced the flag:

> `main.cc (NoAdminOption)`: Add new CLI options `-B`/`--no-admin`. This
> option allows the user to suppress privilege elevation (in tandem with
> `asInvoker` `requestedExecutionLevel` changes to exe manifests).
> `(WinMain)`: check if setup run with Administrator privilege and if the
> `NoAdminOption` has not been specified, attempt to elevate privilege to
> an Administrator via WINAPI `ShellExecuteEx()`.

A UAC prompt cannot be answered on a non-interactive runner, so the
install does not merely degrade, it stops. This is a capability gap
rather than a convenience, and that distinction is the case for the
change.

## Evidence the need is general

Several downstream installers have each had to solve this separately:

- [rvm/rvm#3762](https://github.com/rvm/rvm/issues/3762): under a Windows
  group policy setting, the installer window reports the program is not
  allowed by administrators. The report notes the RVM installer "is just
  missing the `-no-admin` flag".
- [chocolatey-community/chocolatey-packages#457](https://github.com/chocolatey-community/chocolatey-packages/issues/457):
  a `/NoAdmin` package parameter was added for the Cygwin package.
- [protz/ocaml-installer#30](https://github.com/protz/ocaml-installer/issues/30):
  without privileges the Cygwin part silently does not install; the
  answer given is to run setup with `--no-admin`.

The constituency is enterprises that mandate non-admin accounts, which
commonly run self-hosted Actions runners under constrained service
accounts. For them this is not regression testing, it is whether the
action works at all.

## What the upstream action looks like

Read from `action.yml` at `master`, 2026-08-18. It is a composite action,
82 lines, and every input is passed as an environment variable to
`src/install.ps1`, which does the real work.

Existing inputs: `platform`, `packages`, `install-dir`, `check-sig`,
`pubkeys`, `site`, `add-to-path`, `allow-test-packages`, `check-hash`,
`check-installer-sig`, `work-vol`.

Outputs: `setup`, `root`, `package-cache`.

There is no `no-admin` input.

Because the plumbing is uniform, the change is small: one `inputs:` block
in `action.yml`, one `inputs_no_admin:` line in the `env:` map, and a
conditional `--no-admin` on the setup invocation in `src/install.ps1`.

That smallness is the argument for upstreaming rather than forking
permanently. Reimplementing the action would mean owning installer hash
checking, Authenticode verification, `setup.ini` signature checking,
extra pubkeys, `work-vol`/`install-dir` resolution, PATH handling, and
platform selection, all of it security relevant, and tracking upstream
`setup.exe` changes forever. That is a large permanent cost to avoid one
flag.

## Proving it, rather than asserting it

The strongest available evidence is a test on a GitHub-hosted runner,
which is admin and can therefore create a standard local user and run
work as that user. That demonstrates both halves:

- as a non-admin user, the default path fails
- as the same user, `--no-admin` succeeds

That is an empirical demonstration of the capability gap, which is more
persuasive than the precedents above.

Mechanics that work non-interactively:

- Create the account with `New-LocalUser` or ADSI. Not `net user`, which
  has been observed to hang and block the calling script.
- Run as that user with `Start-Process -Credential` and a `PSCredential`
  built from a per-run generated password, or with a scheduled task via
  `schtasks /RU /RP`.
- Redirect the child's output to a file and poll it. A process launched
  under another account does not hand stdout back to the caller.

Expect these to bite:

- Workspace ACLs. A newly created local user has no access to the
  workspace under `D:\a\...`. It needs an explicit `icacls` grant, and
  the install directory has to be somewhere that user can write.
- Password complexity. Generate per run, satisfy the Windows policy, and
  never echo it.
- `-LoadUserProfile`, or `$HOME`-dependent behaviour gets strange.

## Two honest caveats

A standard local user is not the same constraint as a managed-elevation
corporate environment. The test user is Medium integrity with ordinary
privileges. A managed-elevation account can be High integrity while still
withholding the Administrators group and most privileges. Both are "not
admin", and both would fail the `ShellExecuteEx` elevation, but they are
different environments and the test should not be described as
reproducing the second.

A no-admin install is not friction-free once it succeeds. The Cygwin
mailing list records postinstall scripts failing under a non-elevated
user, for example `procmail.sh` exiting 1 because a non-admin user cannot
`chown` to SYSTEM. Being able to install at all is the prerequisite, but
a proposal should not oversell what follows.

## Plan

1. Open an issue upstream before writing code. State the capability gap,
   cite the `ShellExecuteEx` behaviour and the downstream precedents, and
   ask whether a `no-admin` input would be welcome. Cheap, and the
   maintainers may prefer a different shape or name than a guess.
2. If the response is positive, fork and submit the patch plus the test
   above.
3. If it stalls or is declined, keep the fork as a maintained no-admin
   variant. Rebasing an 82-line composite action and one script is cheap.

Fork under `phdye-cygwin`, which already holds the Cygwin work and is
public like the upstream. This repository is deliberately not named
`cygwin-install-action`, so that name stays free for the fork.

## Related work in this org

`phdye-cygwin/cyg-rhel-8.10` is the immediate consumer. It carries:

- `action.yml`, a composite action wrapping `cygwin-install-action` with
  a pinned 2019-08-01 Time Machine snapshot and `check-sig: false`. It
  currently installs with admin. When a `no-admin` input exists, that
  action should gain a matching input.
- `install-rhel810-noadmin.sh`, which is the no-admin install path today,
  driven directly rather than through any action. It has been validated
  by a real `--no-admin` install on a machine with managed elevation.

Note also that `site` plus `check-sig: false` is the documented Time
Machine case in the upstream action's own documentation, so that project
already anticipates unusual install sources. A no-admin input is a
comparable accommodation.

## Status

Nothing here is built. Next step is the upstream issue.
