- Feature Name: `keyring_credential_default`
- Start Date: 2026-07-02
- RFC PR: [rust-lang/rfcs#3981](https://github.com/rust-lang/rfcs/pull/3981)
- Rust Issue: [rust-lang/rust#0000](https://github.com/rust-lang/rust/issues/0000)

# Summary
[summary]: #summary

Cargo should stop writing registry tokens to a plaintext file by default. Instead, `cargo login` should put the token in the operating system's credential store: Keychain on macOS, Credential Manager on Windows, Secret Service on Linux and the BSDs. Any command that reads a token, `cargo publish` included, should migrate an existing plaintext token into the store when one is found. Cargo should fall back to the current file-based storage only when no such store is available.

# Motivation
[motivation]: #motivation

Today, `cargo login` writes your token to `~/.cargo/credentials.toml` in plaintext. This file gets swept up in dotfile repos, backups, and misconfigured shared home directories, and it's a well-known target for token-stealing malware. A stolen crates.io token means someone can publish under your name. And this class of attack is getting cheaper by the month: with AI-assisted tooling, writing and deploying a credential-harvesting script that knows exactly where every ecosystem keeps its plaintext tokens is now a trivial exercise, so "it's just a file in the home directory" is a worse bet than it was even a couple of years ago.

The Rust Maintainers actually solved most of this problem already. Since 1.74, Cargo has pluggable credential providers, and it even ships secure ones: `cargo:macos-keychain`, `cargo:wincred`, and `cargo:libsecret`. The problem is that they're opt-in: a user only gets secure storage if they know the feature exists and edit their config to enable it, which is exactly the population least at risk in the first place. The plaintext file remains what every `cargo login` produces out of the box. Meanwhile `gh`, Docker, and pip all moved to OS credential stores by default years ago, and the sky did not fall.

This RFC proposes flipping the default.

# Guide-level explanation
[guide-level-explanation]: #guide-level-explanation

For most users, nothing visibly changes. You run `cargo login`, paste your token, and publish as usual. The difference is where the token lives: it goes into your OS keychain rather than a file. On macOS you might see a one-time Keychain prompt. On Linux, whatever implements the Secret Service API on your desktop (GNOME Keyring, KWallet, or `pass` via the `pass-secret-service` bridge) holds it. `cargo logout` removes it.

If you already have a plaintext token from before the upgrade, you don't need to log in again to get the benefit. The next time any command needs your credential, whether that's `cargo login`, `cargo publish`, `cargo yank`, or `cargo owner`, Cargo migrates the token into the keyring, deletes it from `credentials.toml`, and prints a notice saying it did so. In practice this means most users are moved to secure storage the first time they publish after upgrading.

If you're on a headless server or in CI, nothing changes at all. `CARGO_REGISTRY_TOKEN` still takes precedence and is never migrated anywhere, and if there's no credential store to talk to, Cargo quietly falls back to the file just like today. Nobody's build breaks because D-Bus isn't running.

# Reference-level explanation
[reference-level-explanation]: #reference-level-explanation

Rather than maintaining three separate built-in providers, Cargo's default provider would be a single `cargo:keyring` provider built on the `keyring` crate, which already abstracts over the macOS Security Framework, Windows Credential Manager, and Secret Service (with a kernel keyutils fallback on Linux, and Secret Service coverage on FreeBSD and OpenBSD). The default provider list becomes `["cargo:keyring", "cargo:token"]`: try the keyring, fall back to the file.

Migration happens in the credential-resolution path, not just in `login`. When resolution falls through to the file provider and finds a token, and the keyring provider reported itself available, Cargo writes the token to the keyring, verifies the write by reading it back, and only then removes it from `credentials.toml`. If the keyring write or read-back fails, the file is left untouched and the token is used as before. This ordering means an interrupted migration can duplicate a credential but never lose one. Tokens supplied via `CARGO_REGISTRY_TOKEN` or `--token` bypass migration entirely, since they were never Cargo's to store.

Entries are keyed by service `cargo-registry:<index-url>` so alternative registries each get their own credential. On Linux the Secret Service connection is made over D-Bus at runtime, not linked at build time, so a missing D-Bus daemon just means the provider reports that no store is available and the chain moves on to the file provider.

The existing three per-platform providers stay for a deprecation period so nobody's explicit config breaks, but they'd eventually become aliases for `cargo:keyring`.

# Drawbacks
[drawbacks]: #drawbacks

Keychain access can prompt the user, which is surprising the first time it happens mid-script, though the env-var and file fallbacks mean scripted environments never hit this in practice. Migrating during `publish` makes that first prompt more likely to appear in the middle of a workflow the user thinks of as non-interactive, which is why the notice needs to be loud about what just happened. It also adds the `keyring` crate (and its platform dependencies) to Cargo's dependency tree, which is real supply-chain surface for a security-sensitive code path and would need the usual vetting.

# Rationale and alternatives
[rationale-and-alternatives]: #rationale-and-alternatives

The obvious alternative is the status quo: the secure providers exist, just turn them on yourself. But a security measure that requires discovering a config key does not protect the default install, and the default install is the threat model here. Defaults matter precisely because most users never change them, and that cuts both ways. Right now it cuts toward plaintext.

Migrating only on `cargo login` would be simpler, but it leaves every existing user in plaintext indefinitely, since a working token means there's no reason to ever run `login` again. Publish-time migration is what actually moves the installed base.

The Rust Maintainers could also keep the three separate per-platform providers and just change the default list per platform. That works, but it's three implementations to keep behaviorally in sync where one crate already does the job, and it leaves the BSDs out.

# Prior art
[prior-art]: #prior-art

Cargo's own credential-process RFC ([RFC 2730](https://rust-lang.github.io/rfcs/2730-cargo-token-from-process.html)) and the 1.74 stabilization laid all the groundwork here. This RFC is really just about the default. Outside Rust: `docker-credential-helpers`, `gh auth`, and Python's `keyring` package all made the same move, and their experience (including the headless-fallback problem) is directly applicable.

# Unresolved questions
[unresolved-questions]: #unresolved-questions

Whether the deprecation period for the old provider names needs a formal timeline or can just ride the normal edition cadence. And whether publish-time migration should ask for confirmation on its first run or just proceed with a notice, given that a confirmation prompt in a previously non-interactive flow is its own kind of breakage.

# Future possibilities
[future-possibilities]: #future-possibilities

Once tokens live behind a provider interface by default, the same path could carry asymmetric tokens ([RFC 3231](https://rust-lang.github.io/rfcs/3231-cargo-asymmetric-tokens.html)) without users ever handling a bearer secret directly.
