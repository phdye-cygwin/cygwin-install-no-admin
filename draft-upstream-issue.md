# Draft: upstream issue text

To post at https://github.com/cygwin/cygwin-install-action/issues
Not yet posted. Check the existing issues and pull requests first, in
case this is already proposed.

---

**Title:** Cannot install on a runner whose account is not an administrator

**Body:**

On a runner whose account is not a local administrator, this action
cannot install Cygwin.

`setup.exe` attempts to elevate itself unless `--no-admin` is passed.
From the patch that introduced the option:

> `(WinMain)`: check if setup run with Administrator privilege and if the
> `NoAdminOption` has not been specified, attempt to elevate privilege to
> an Administrator via WINAPI `ShellExecuteEx()`.

There is nobody to answer a UAC prompt on a non-interactive runner, so
this is not a degraded install, it is no install.

This affects enterprises that mandate non-admin accounts and run
self-hosted runners under constrained service accounts. Several
downstream installers have each hit the same thing and added their own
flag for it:

- rvm/rvm#3762, where a Windows group policy blocks the installer and the
  report notes the missing `-no-admin` flag
- chocolatey-community/chocolatey-packages#457, which added a `/NoAdmin`
  package parameter
- protz/ocaml-installer#30, where the answer given is to run setup with
  `--no-admin`

Would a `no-admin` input be welcome? The change looks small: an input, an
entry in the `env:` map, and a conditional flag on the setup invocation
in `src/install.ps1`.

I can also contribute a test. A hosted runner is admin, so it can create
a standard local user and run the install as that user, showing that the
default path fails and `--no-admin` succeeds. Two caveats I would rather
state up front: a standard local user is not identical to a
managed-elevation corporate account, and a no-admin install still has
known rough edges, such as postinstall scripts that `chown` to SYSTEM
failing under a non-elevated user.

Happy to send a pull request if the direction is agreeable.
