---
layout: post
title:  "Moving Unreleased Apps to a Different Apple Developer Account"
date:   2026-07-26 07:39:21 -0400
categories: tech
tags: [ci-cd, macos, code-signing, fastlane, github-actions, app-store-connect]
description: "Two TestFlight-only apps needed to live in a different Apple developer account. Deleting and recreating them is cheap, but the tooling around signing, store metadata and CI has opinions about it."
comments: false
permalink: /moving-apps-between-apple-accounts
---

I had two apps sitting in TestFlight under one Apple developer account that needed to belong to a
different account. Neither had ever been released, so there were no reviews, no ratings, no public
installs, and nobody outside the family had ever downloaded them.

Apple has an official app transfer flow, but it refuses apps that have never been released. That
sounds like bad news and is actually good news: an unreleased app is nearly free to delete and
recreate. The store record is a shell. The interesting part is not the store, it is everything
downstream that quietly assumed the old identity.

## TL;DR

- **You lose the bundle ID forever.** Once a build has been uploaded under it, Apple reserves it
  permanently. Deleting the app record does not free it, and no other account can ever claim it.
  Everything else is recoverable.
- **Rename the old app before creating the new one**, otherwise the app name is still reserved and
  the new record cannot use it.
- **Rotate your CI credentials before running any automation** against the new account, or your
  tooling will authenticate as the old account and cheerfully fail to find records that exist.
- `fastlane deliver` **fails on a brand new record** with `[!] No data` until the App Review contact
  info exists. Once per record, not once per release.
- **`match` will not share one storage branch between two teams.** One branch per team fixes it.
  `match nuke` is the wrong tool here and will delete your old material for no benefit.
- **Do not trust "success"**. One lane reported that it applied the privacy policy URL, and it had
  silently skipped it. Read the store state back through the API.
- **Verify the whole record, not the field you changed.** A later run of the same lane cleared a Kids
  Category age band that had already been set and verified, while still reporting success.
- **A new bundle ID means a new UserDefaults domain.** Every user's saved progress is gone, and
  depending on your UI that can look exactly like you shipped an old build.

The three scripts I leaned on are in [this
gist](https://gist.github.com/janpreet/be9f37d013a3f810ee5195328bf65868): check which account you are
authenticated as, seed the App Review contact details, and read store state back after a deploy.
Plain Ruby, stdlib only, so they run on a stock macOS Ruby with no Gemfile.

## What survives and what does not

| Thing | Survives? |
| --- | --- |
| Bundle ID | No, and it is burned permanently |
| App name | Yes, if you free it before reusing it |
| App Store numeric ID, SKU | No, new record gets new ones |
| Signing certificate and profiles | No, they belong to the old team |
| Old TestFlight builds and testers | No, and old builds expire on their own |
| Source, CI config, store metadata answers | Yes, they are just data |
| Users' in-app progress | No, see the UserDefaults note below |

## The order that actually works

Sequence matters more than any individual step here, because several steps quietly depend on
earlier ones.

1. **Confirm the app was never released.** If it has a live store page, stop and use the real
   transfer flow instead. Losing actual users is not worth the convenience. In App Store Connect,
   platforms reading `1.0 Prepare for Submission` are a good sign that nothing ever shipped.
2. **Enrol the new account** and let it finish. Accept the Free Apps agreement. Create an App Store
   Connect API key with the App Manager role, and note that the issuer ID differs per account, so it
   is not just the key that changes.
3. **Change the identity in the repo.** All of it, on a branch, before touching the store.
4. **Rename the old app records** to free the names.
5. **Create the new records** in the new account.
6. **Rotate CI secrets.** Team ID, key ID, issuer ID, key contents.
7. **Signing**, then **merge**, then **deploy manually** for the first build.
8. Delete the old records whenever you feel like it. There is no rush and no benefit.

Step 3 before step 5 is deliberate. I have a small script that prints every store form field from
the app's manifest, and if the manifest still holds the old identity when you sit down to fill in
the New App dialog, you will type the old bundle ID into the new account.

## Gotcha 1: the app name is reserved per localization

Renaming the old app frees the name, but the name lives on the localization, not the app. If your
record has listings in two languages, both hold a reservation, and the one you did not rename is the
one that blocks you.

Check the localization dropdown next to the name field rather than assuming there is only one. Also,
the save takes effect immediately, while the reservation clearing does not always feel immediate. The
only real confirmation is typing the name into the new account's New App dialog and seeing it
accepted.

## Gotcha 2: rotate credentials before you automate anything

I had a workflow that applies the codified compliance answers (age rating, privacy URL) to a store
record. I ran it right after creating the new records and it failed. Of course it did: it reads four
secrets, and all four still pointed at the old account. It authenticated perfectly, as the wrong
identity, and then could not see records that plainly existed.

Worth checking your credentials independently before blaming anything else. A JWT and one request to
`/v1/apps` tells you which account you are actually talking to:

```
HTTP 200
  My First App   |  com.newowner.firstapp   |  id=REDACTED
  My Second App  |  com.newowner.secondapp  |  id=REDACTED
```

Two apps listed, both the new ones, no old ones. That is the whole answer, and it took less time
than reading the failing log. That check is
[`asc_whoami.rb`](https://gist.github.com/janpreet/be9f37d013a3f810ee5195328bf65868#file-asc_whoami-rb)
in the gist.

One detail if you script this: check how your Fastfile consumes the key. Mine passes
`is_key_content_base64: true`, so the secret has to hold base64 of the `.p8` and not the PEM itself.

## Gotcha 3: the first `deliver` on a new record dies with "No data"

This one is a known fastlane issue and it will find you on every brand new app record:

```
[!] No data (RuntimeError)
  .../spaceship/lib/spaceship/connect_api/model.rb:82:in `parse'
Uploading the very first version of my app yields `No data`
```

Note that fastlane itself prints the hint. The cause is that `deliver` reads the version's review
details before writing, and on a never-touched version Apple returns an empty payload rather than an
empty object, which the parser will not accept.

The fix is to make the record exist by putting real values in App Review Information: contact name,
phone, email. In current App Store Connect this lives on each platform's version page, not on the
sidebar item confusingly named App Review, which is the submission queue. If you have API access it
is faster to POST an `appStoreReviewDetails` record per version and skip the clicking:

```
OK   My First App  IOS    1.0 (PREPARE_FOR_SUBMISSION) -> HTTP 201
OK   My First App  TV_OS  1.0 (PREPARE_FOR_SUBMISSION) -> HTTP 201
OK   My Second App IOS    1.0 (PREPARE_FOR_SUBMISSION) -> HTTP 201
OK   My Second App TV_OS  1.0 (PREPARE_FOR_SUBMISSION) -> HTTP 201
```

After that the API behaves normally forever. It is once per record, so on a migration you pay it
once per app rather than once per release. That is
[`asc_seed_review_details.rb`](https://gist.github.com/janpreet/be9f37d013a3f810ee5195328bf65868#file-asc_seed_review_details-rb),
which takes a `--dry-run` first if you would rather look before writing.

## Gotcha 4: a green lane that silently did nothing

This is the one I am glad I caught. After the fix above, the bootstrap lane ran clean and printed
that it had applied the age rating and the privacy policy URL. I read the store state back through
the API anyway, and got this:

```
My First App: ageRating="FOUR_PLUS"  kidsAgeBand="FIVE_AND_UNDER"
  en-US: privacyPolicyUrl=nil
```

The age rating landed. The privacy URL did not, on either app, despite an explicit success message.

The cause was in my own code, and it is a subtle one. The lane wrote `privacy_url.txt` into a
`default/` metadata directory, on the reasonable assumption that `default` means all locales. That
holds for version level metadata. The privacy URL is not version level, it lives on App Info
localizations, and `deliver` only diffs those against real locale directories. So `default`
contributed nothing, `deliver` compared empty against empty, and skipped:

```
Loading '.../default/privacy_url.txt'...
Detected languages: ["default"]
No changes to localized App Info detected. Skipping upload.
Will begin uploading metadata for '1.0' on App Store Connect
Applied: age rating (all NONE, Kids <=5), privacy URL.
```

Everything needed to diagnose it is in those five lines, and the last line is a lie my own lane told
me. Writing to `en-US/` instead fixed it, and this time the read-back agreed.

The general lesson is boring and keeps being true: a store deploy's exit code tells you the tool
finished, not that the store changed. If a field matters, read it back. That is what
[`asc_verify_store_state.rb`](https://gist.github.com/janpreet/be9f37d013a3f810ee5195328bf65868#file-asc_verify_store_state-rb)
is for, and it earned its keep twice.

## Gotcha 5: verify every field, not just the one you changed

The second time it earned its keep was worse, and I nearly missed it.

Later on I changed one app's privacy policy URL and re-ran the same lane. It succeeded, I read the
state back, and the URL was correct. Except I had read back only the URL, because that was the field
I had changed. When I ran the full check some time later, for an unrelated reason:

```
My First App:  kidsAgeBand=nil            ageRatingOverride="NONE"
My Second App: kidsAgeBand="FIVE_AND_UNDER" ageRatingOverride="NONE"
```

The Kids Category age band on the app I re-ran was gone. It had been set correctly, I had verified it
with my own eyes, and then a later run of the same lane cleared it while reporting success. The other
app, which I had not re-run, still had it.

For a kids app that field is not cosmetic, it is the thing that puts you in the Kids Category. If I
had not stumbled onto it, the next thing to notice would have been an App Review rejection weeks
later.

Re-running the lane once more set it back, and this time I checked both fields together. I do not
have a satisfying explanation for the mechanism, and I am wary of inventing one. What I take from it
is narrower and more useful: **verify the whole record, not the diff you think you made.** Idempotent
set-to-value automation can still clear a neighbouring field, and "I checked that field last time" is
not the same as "that field is correct now".

## Gotcha 6: `match` will not share a storage branch across teams

I reasoned about this in advance and got it wrong, which is worth admitting because the wrong
conclusion sounded convincing. My thinking was: `match` files certificates under their own
certificate ID and profiles under their bundle ID, so old and new material cannot collide by name,
so they can coexist in one storage repository.

They cannot:

```
Verifying that the certificate and profile are still valid on the Dev Portal...
Certificate 'REDACTED' (stored in your storage) is not available on the Developer Portal
for the user
Make sure to use the same user and team every time you run 'match' for this Git repository.
[!] To reset the certificates of your Apple account, you can use the `fastlane match nuke` feature
```

Filenames were never the constraint. `match` validates the certificate it finds in storage against
the team it authenticates as, and refuses to continue when they disagree.

**Do not take the suggestion in that error message.** `nuke` acts on the team you are authenticated
as, which is the new one, and the new team has nothing to revoke. But it still clears the storage
repository, so you would delete your old encrypted material while the old certificate stays live and
unrevoked in the old account. All cost, no benefit.

The right answer is `match`'s own mechanism for exactly this, one branch per team:

```ruby
git_url(ENV["MATCH_GIT_URL"])
# One branch per Apple team: match refuses to run when the cert in storage
# belongs to a different team than the one it authenticates as.
git_branch(ENV["MATCH_GIT_BRANCH"] || "master")
storage_mode("git")
type("appstore")
```

Set the branch per team, pass it to CI as a secret, and the two teams stay out of each other's way.
Old material stays on `master` untouched, which also means you can revoke the old certificate later,
on your own schedule, once you have confirmed it does not sign anything else.

## Gotcha 7: your own guardrails hardcode the old identity

I had a CI check enforcing that bundle IDs only appear in the app manifests and not scattered through
workflows and Fastfiles. Good rule. It was written like this:

```bash
if grep -rn 'com\.olddev\.' .github/workflows; then
  echo "::error::workflow files must not hardcode bundle ids"; exit 1
fi
for f in ...; do
  n=$(grep -c 'com\.olddev\.' "$f")
  ...
done
```

The guard against hardcoded identity had the identity hardcoded in it. Once the prefix changed, the
grep matched nothing, `grep -c` exited nonzero on zero matches, the step ran under `bash -e`, and the
job failed. The check did not catch a problem, it became one.

It now matches any organization prefix for the project's app names, which is what the rule always
meant:

```bash
ID_RE='com\.[a-z0-9-]\+\.\(firstapp\|secondapp\)'
n=$(grep -c "$ID_RE" "$f" || true)
```

Worth grepping your repo for the old identity in places that are *about* identity: compliance
checks, lint rules, helper scripts, skill definitions. They are the last places you think to look and
the first ones that break.

## Gotcha 8: things that are not the bundle ID but travel with it

A couple of edits I did not plan for and would rather have found before CI did than after:

- The bundle ID was the key in a bundle-to-language map used at runtime. Leaving one platform on the
  old ID would have dropped it into an unintended fallback rather than failing loudly. Identity used
  as a lookup key is easy to miss because it does not look like configuration.
- On Android, `namespace` and `applicationId` both moved, and the Kotlin package had to move with the
  namespace, because the manifest resolves `.MainActivity` relative to it.
- Skipping the auto deploy on merge needed the marker in the **merge commit** message. Putting it in
  individual commits does nothing if your workflow reads the merge commit, though it does work if you
  put it in the pull request title, since that gets folded into the merge commit body.

## Gotcha 9: everyone's saved progress disappears

This is the one your actual users notice, and it is not fixable.

A new bundle ID is a new UserDefaults domain. Stars, scores, resume positions, last opened screen,
all empty on first launch. There is no migration path between two bundle IDs.

The part that fooled me for a minute: one of these apps has a Continue card on its home screen that
only renders when there is saved progress.

```dart
if (lastMode.isNotEmpty && lastTheme != null) ...[
```

A fresh install therefore looks like a build with a missing feature. I got a screenshot of the new
TestFlight build with no Continue card and the reasonable question of why an old version had been
deployed. It had not: five deploy runs all built the same commit, the tip of the default branch. The
tell was in the screenshot all along, a star counter reading zero.

If your app has any progress-gated UI, expect this question after a bundle ID change, and check the
progress state before you go looking at build numbers.

## Result

Both apps now live in the new account, on both platforms, signed with the new team's certificate:

| App | Platform | Version | Build | State |
| --- | --- | --- | --- | --- |
| First app | tvOS | 1.0 | 2 | VALID |
| First app | iOS | 1.0.0 | 1 | VALID |
| Second app | tvOS | 1.0 | 2 | VALID |
| Second app | iOS | 1.0.0 | 1 | VALID |

Age ratings and privacy URLs verified by reading them back from the API rather than trusting a lane's
output. Signing confirmed by the certificate name in the CI log rather than by the job going green.

Total elapsed time was an evening, and roughly none of it was the part I expected. Changing the
identity in the repository was about twenty files and mechanical, and both test suites assert the
bundle ID, so a missed literal failed loudly instead of shipping quietly. Everything slow was the
tooling around the edges: credentials pointing at the old account, a lane that lied about success, a
signing tool with a reasonable objection I had talked myself out of in advance, and a compliance
check that had the thing it was checking baked into it.

## If I did it again

- Read the store state back through the API after every automated store change, and read back the
  whole record rather than the field I touched. This is the single highest value habit here.
- Rotate credentials before running anything that talks to the new account, not after.
- Grep for the old identity in checks, guards and helper scripts, not just source and config.
- Expect one branch per team in signing storage from the start, rather than discovering it.
- Tell your users their progress will reset, before they find out themselves.
