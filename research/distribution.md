# Distributing a small open-source macOS app in 2026

Research for issue #5 (map: #2). Question: what does it take, on macOS 26 (Apple Silicon),
to get a small open-source app to non-technical SCX-4623FW owners through (a) Developer ID
signing plus notarization, (b) an unsigned or ad-hoc-signed build with instructions, or
(c) a Homebrew cask in a personal tap? The comparison covers cost, first-launch Gatekeeper
behaviour, Homebrew policy and CI. Sources checked 2026-09-23.

## Summary

| | (a) Developer ID + notarization | (b) Ad-hoc / unsigned + instructions | (c) Cask in a personal tap |
|---|---|---|---|
| Money | $99/year Apple Developer Program | $0 | $0 (a GitHub repo `homebrew-tap`) |
| Identity | Legal name, 2FA, address verification | none | none |
| First launch | Normal "downloaded from the Internet" dialog, one click on Open | Blocked. User must go to System Settings > Privacy & Security > Open Anyway, then confirm | Blocked like (b), unless the cask strips quarantine, and then it opens with no dialog at all |
| Homebrew | Qualifies for the official `homebrew/cask` if it meets the other rules | Not eligible for `homebrew/cask` | Allowed. Homebrew's Gatekeeper rule covers only the official tap |
| CI (GitHub Actions) | `macos-26` runner, free for public repos. Needs the certificate `.p12` and notary credentials as secrets | Same runner, no secrets | Same, plus a job that bumps the cask's `version`/`sha256` |

Short version: only (a) gives a normal first launch, and it costs $99 a year plus identity
verification. (b) is free, but since macOS 15 the Control-click shortcut is gone, so every
user has to go through System Settings. (c) is free and fits this project, because users
already need Homebrew for `sane-backends`. On its own, though, it inherits (b)'s Gatekeeper
block. The only way around that is a cask step that removes the quarantine attribute, which
works today but is a deliberate bypass that Homebrew discourages.

## (a) Developer ID signing + notarization

**Cost and enrollment.** The Apple Developer Program costs "$99 USD per membership year"
(local prices vary). Fee waivers exist only for nonprofits, accredited schools and
government entities, so an individual open-source maintainer pays. Individuals need an
Apple Account with two-factor authentication, their legal name, and a verified address.
Organizations also need a D-U-N-S number and a website.
Source: https://developer.apple.com/programs/enroll/

Developer ID certificates and "Mac software notarization" come only with the paid
membership. The free account tier does not include them.
Source: https://developer.apple.com/support/compare-memberships/

**What notarization requires.** Every executable must be signed with a *Developer ID
Application* certificate ("Don't use a Mac Distribution, ad hoc, Apple Developer, or local
development certificate"). The requirements also include Hardened Runtime, a secure
timestamp, no `get-task-allow` entitlement, and submission with `notarytool` (`altool` has
been rejected since 2023-11-01). The notary service is automated, not App Review. It
returns a ticket that you staple to the app, and it also publishes that ticket online for
Gatekeeper.
Source: https://developer.apple.com/documentation/security/notarizing-macos-software-before-distribution

**Consequence for this app.** Hardened Runtime turns on *library validation*, which
"prevents a program from loading frameworks, plug-ins, or libraries unless they're either
signed by Apple or signed with the same Team ID as the main executable." If the app links
Homebrew's `libsane` directly, it needs the
`com.apple.security.cs.disable-library-validation` entitlement. Driving `scanimage` as a
subprocess, the way `bin/scx-scan` does, avoids the issue because the child is a separate
process.
Source: https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.cs.disable-library-validation

**Gatekeeper.** Once the app is notarized, Gatekeeper finds the ticket and shows the usual
first-launch dialog with descriptive information. The user clicks Open once. (Same Apple
notarization page.)

## (b) Unsigned / ad-hoc-signed build with instructions

**Signing floor.** On Apple Silicon, "any executable must be signed before it's allowed to
run. There isn't a specific identity requirement for this signature: a simple ad-hoc
signature is sufficient." The linker applies one automatically, and `codesign -s -` does it
by hand. A truly unsigned arm64 binary does not run at all, so in practice (b) always means
ad-hoc signed.
Source: https://developer.apple.com/documentation/macos-release-notes/macos-big-sur-11_0_1-universal-apps-release-notes

**Gatekeeper since macOS 15.** "In macOS Sequoia, users will no longer be able to
Control-click to override Gatekeeper when opening software that isn't signed correctly or
notarized. They'll need to visit System Settings > Privacy & Security" (Apple, 2024-08-06).
Source: https://developer.apple.com/news/?id=saqachfa

The current user procedure has five steps:

1. Try to open the app.
2. Open System Settings.
3. Go to Privacy & Security and scroll down.
4. Click **Open Anyway**.
5. Confirm with **Open** in the next prompt.

After that the app is saved as an exception and opens normally. Apple's page no longer
mentions Control-click.
Source: https://support.apple.com/en-us/102445

The alternative for technical users is `xattr -dr com.apple.quarantine /Applications/App.app`
in Terminal. It works because Gatekeeper assesses only quarantined files, but it is not
something to ask of non-technical users.

**Cost:** $0. **CI:** any macOS runner builds and ad-hoc-signs with no secrets.

## (c) Homebrew cask in a personal tap

**Official tap policy.** `homebrew/cask` requires that apps "must pass Homebrew's Gatekeeper
checks and must not require System Integrity Protection or Gatekeeper to be disabled or
bypassed."
Source: https://docs.brew.sh/Acceptable-Casks

Homebrew 5.0.0 (2025-11-12) deprecated casks without codesigning and announced: "We will
disable all Homebrew/homebrew-cask casks that fail Gatekeeper checks in September 2026". It
also deprecated `--no-quarantine`/`--quarantine` because Homebrew "does not wish to easily
provide circumvention to macOS security features". 6.0.0 (2026-06-11) confirmed the plan.
Sources: https://brew.sh/2025/11/12/homebrew-5.0.0/ , https://brew.sh/2026/06/11/homebrew-6.0.0/

The disable has taken effect. The official tap now carries
`disable! date: "2026-09-01", because: :fails_gatekeeper_check` in about 600 casks, for
example `Casks/s/sioyek.rb`. Checked through the GitHub code search at
https://github.com/Homebrew/homebrew-cask.

**Personal taps are allowed.** Homebrew maintainers point unsigned-app authors to "a
personal tap, which is very easy to do". They also say they will not bless such taps as
alternatives.
Source: https://github.com/orgs/Homebrew/discussions/7050

**What the user types (Homebrew 7.0, current release 7.0.6 of 2026-09-21).** Third-party
taps now have to be explicitly trusted: "Use `brew trust` for each non-official tap,
formula, cask or command." So the install becomes `brew tap filippolmt/tap`, then
`brew trust` on that tap, then `brew install --cask <name>`.
Sources: https://brew.sh/2026/09/13/homebrew-7.0.0/ , https://brew.sh/7.0.0-migration-guide/ , https://docs.brew.sh/Manpage

**Gatekeeper.** Homebrew sets `com.apple.quarantine` on cask downloads (see
`Library/Homebrew/cask/quarantine.rb` in https://github.com/Homebrew/brew), so an ad-hoc
app from a tap hits the same block as (b). Taps that ship unsigned apps get around this
with a `postflight_steps` block that `run`s `xattr -dr com.apple.quarantine` on the
installed app. The structured `*_steps` stanzas, including `run`, are the supported form.
The legacy Ruby `postflight` blocks "remain available temporarily for third-party tap
compatibility", are rejected in official taps, and are deprecated in 7.0.0 with removal
set for 2027-12-11.
Sources: https://docs.brew.sh/Cask-Cookbook , https://brew.sh/2026/09/13/homebrew-7.0.0/ ,
for the pattern in the wild: https://github.com/warrenseine/homebrew-opossum-desktop/issues/1

Caveat: stripping quarantine is exactly the circumvention Homebrew no longer wants to offer
through a flag. It works in a third-party tap today, but the project could restrict it
later.

**Unverified variant.** A tap can also hold a *formula* that builds the app from source on
the user's machine. Files built locally are not quarantined, so Gatekeeper never assesses
them. The cost is a build toolchain on every user's Mac (Xcode or the Command Line Tools
for Swift) and a longer install. This idea has not been tested.

**Cost:** $0. **CI:** a release workflow builds the `.zip`/`.dmg`, attaches it to a GitHub
Release, and opens a commit or PR on the tap that updates `version` and `sha256`.

## CI on GitHub Actions (all paths)

- Runners: `macos-26` (arm64), which is also `macos-latest`, plus `macos-15` and Intel
  `-intel`/`-large` variants.
  Source: https://github.com/actions/runner-images
- Price: "GitHub Actions usage is free for standard GitHub-hosted runners in public
  repositories."
  Source: https://docs.github.com/en/actions/concepts/billing-and-usage
- Signing on a runner, needed for (a) only: store the Developer ID `.p12` as base64 plus its
  password as secrets, create and unlock a temporary keychain, `security import`, then
  `security set-key-partition-list`. Notarization additionally needs `notarytool`
  credentials, either an App Store Connect API key or an Apple ID with an app-specific
  password, stored as secrets.
  Source: https://docs.github.com/en/actions/how-tos/deploy/deploy-to-third-party-platforms/sign-xcode-applications

## Implications for the map (#2)

- The audience already installs Homebrew for `sane-backends`, so the extra effort of (c)
  over (b) is small for users. The tap also gives them `brew upgrade`.
- (c) by itself is not "no Gatekeeper friction". It either needs quarantine stripping
  (which works but is discouraged) or signing.
- (a) and (c) combine well: a notarized app in a personal tap needs no quarantine hack. It
  could also go to `homebrew/cask` later, subject to that tap's notability rules.
- Whatever path is chosen, prefer running `scanimage` as a subprocess over linking
  `libsane` so that Hardened Runtime needs no library-validation exception.
