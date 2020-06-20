- Feature Name: posix-namespace (fill me in with a unique ident, my_awesome_feature)
- Start Date: 2020-06-19 (fill me in with today's date, YYYY-MM-DD)
- RFC PR: (leave this empty)
- Redox Issue: (leave this empty)

# Summary
[summary]: #summary

_One para explanation of the feature._

The proposed schemes and namespace would, while maintaining compatibility with existing software, put the user in complete control of their new home directory: `file:/`. A namespace would wrap existing software, mapping paths such as `/bin`, `/tmp`, `~/.config`, and `~/.cache` to new schemes. 

# Motivation
[motivation]: #motivation

_Why are we doing this? What use cases does it support? What is the expected outcome?_

A Unix-style filesystem contains file with various purposes:
- documents (user-written code, documents, images)
- assets (e.g. icons)
- secrets (e.g. auth tokens stored in `~/.config`)
- customization (dotfiles)
- cache files (e.g. `~/.cache`)
- environment (e.g. globally-installed libraries, `/bin`)
- temporary files (e.g. `/tmp`)

Redox is built upon "everything is a URL," allowing us to use schemes to separate different kinds of "files" (e.g `/proc`, `/sys`, `/dev/random`). However, the above kinds of files are still mixed, all stored under `file:`. This can be seen by running `ls /` and `ls -a ~` on Redox.

Because of this mixing, the user does know which files are important and why. This makes the filesystem feel unfamiliar. It also makes backups difficult.

Documents need to be user-wide. This is why there's one home directory per user. Secrets, cached data, and customization are often stored in this home directory.

If the user wants multiple versions of the same software installed, the versions will stomp upon each other's config and cache files, causing problems. This is typically solved by creating new users or temporarily changing variables such as `$XDG_CONFIG_HOME`, which are supported by only some software.

A lot of existing software expects dependencies at absolute, global paths such as `/bin` or `/usr/local`. This causes messy dependency conflicts. These global paths, in addition to the aforementioned paths in the user directory, makes the filesystem unfamiliar. The user is presented with files they did not create and don't know the importance of (e.g. does anything depend on `/bin/sha384sum`?). This, in addition to making the user feel slightly uneasy, make backups difficult. It also makes uninstalling software difficult; generated configs, cached data, and downloaded dependencies are often untracked and left behind.

All these issues are fixable without breaking existing software.

# Detailed design
[design]: #detailed-design

_This is the bulk of the RFC. Explain the design in enough detail for somebody familiar with the language to understand, and for somebody familiar with the compiler to implement._

_This should get into specifics and corner-cases, and include examples of how the feature is used._

As far as I know, solving the above problems does not require modifying the kernel. The solution should involve mereley wrapping existing software and writing new software differently.

First, we'll examine Redox from the perspective of new, well-behaving software. Then, we'll examine how we can create a namespace to maintain compatibility with existing software.

## New organization

The proposed organization of Redox userspace puts the user in complete control of their filesystem. The user's new home directory is `file:/`, and that directory should contain only _documents_, files created by the user such as text files, code, images, etc.

Other kinds of files will be located under their own scheme:
- `env:` (formerly `/bin`, `/usr/bin/env`, and `$PATH`) - a mapping from the name of software (e.g. "firefox") to its executable.
- `asset:` (formerly `/usr/share`) - assets used by software, such as icons, jar files, and static dependencies. See note #1 in [Unresolved questions](#unresolved-questions).
- `config:` (formerly `~/.config`) - user configuration.
- `secret:` - passwords, tokens. See the [Auth Scheme Interface Proposal](https://github.com/redox-os/rfcs/pull/7)
- `cache:` (formerly `~/.cache`) - files that can be deleted, with a performance penalty
- `state:` - shell history, like `/tmp` but long-term


Most processes will have their own namespace.

Typically, a package's namespace behaviour will be defined by the person who packaged the software for Redox. `secrets:/` could be mapped to `secrets:/$process_name/`. A similar mapping could be used for `config:`.

In addition to cleaning up `file:`, adding these schemes would allow for some interesting functionality:
- `env:` could be process-specific. The user no longer needs to see executables that they did not directly install
- `config:` could share user configuration across machines
- `secret:` could be stored on an encrypted, external thumb drive
- `env:`, `asset:`, and `config:` could be declaratively generated


## Compatibility layer

```
posix:/
├─ bin ➜ env:
├─ usr/bin/env ➜ env:
├─ share ➜ assets:
└─ home/$USER
   ├─ .config ➜ config:
   ├─ .cache ➜ cache:
   ├─ .local/share ➜ assets:
   └─ * ➜ file:/
```

## Packaging software

[Nix](https://nixos.org/nix) is a build system similar to the Redox cookbook. Nix would be helpful for declaratively generating custom namespace wrappers.

However, packaging should still be possible via shell scripts similar to the existing cookbook recipes.

## Future extensions

The idea of most processes having their own namespace is a step towards capability-based security.

# Drawbacks
[drawbacks]: #drawbacks

_Why should we *not* do this?_

This proposal pollutes the scheme namespace. It also adds a layer between processes and resources, which could be a performance issue.

# Alternatives
[alternatives]: #alternatives

_What other designs have been considered? What is the impact of not doing this?_

Currently, application data can be somewhat controlled via XDG environment variables. Not all software supports these variables.

# Unresolved questions
[unresolved]: #unresolved-questions

_What parts of the design are still TBD?_

1. Perhaps `asset:` could be inspired by [Nix](https://nixos.org/nix)'s `/nix/store`, full of directories with names based on the hash of their contents. This would allow multiple versions of software to be installed. Perhaps binaries could be stored under assets, maybe hard-linking to other asset directories. In this case, `env:` could simply redirect paths to specific binaries in the `asset:` scheme.

2. Should a `tmp:` scheme be created?

3. Is "POSIX" the correct term?
