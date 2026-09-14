# Supply-chain pins

This fork exists to make the released `.ipa` reproducible from a fixed set of inputs.
Upstream builds correctly but references several of its build inputs by moving tag
(`@v1`, `@v4`, `releases/latest`). Any of those can change under the build without a
commit landing in this repository. The changes here remove that class of drift.

## What is pinned

| Input | Pinned to | Why |
|---|---|---|
| `maxim-lobanov/setup-xcode` | `ed7a3b1fda3918c0306d1b724322adc0b8cc0a90` | tag `v1` is mutable |
| `actions/checkout` | `a37ce9120846195fa4ece8f58b268e6043cb2f26` (v3), `11d5960a326750d5838078e36cf38b85af677262` (v4) | tags are mutable |
| `actions/upload-artifact` | `ea165f8d65b6e75b540449e92b4886f43607fa02` | tag `v4` is mutable |
| `actions/download-artifact` | `d3f86a106a0bac45b974a628896c90dbdf5c8093` | tag `v4` is mutable |
| `andelf/nightly-release` | `c5ed4bdb7c1da04a4fa1e40bc5e67306f682563b` | tag `v1` is mutable; this action holds `contents: write` |
| `ldid` | `v2.1.5-procursus7`, sha256 `4b8862b2fefa2cd7fa8f88cb0310779619aeaa8c72d6aff22f019b470f2fa99a` | was `releases/latest` with no checksum; this binary signs the shipped app |
| `ParsecSDK.framework` | submodule commit `bb336699ee7acd7a4eb0c460410a65c23d7f3879`, binary sha256 `2ef504fa2aad5e23a615c5f0d2d440a3102fe5dd34b471c700e08acf492e4603` | closed-source blob, verified at build time before it is linked |

## ParsecSDK provenance

Parsec removed their public SDK from GitHub after the Unity acquisition, so the iOS
framework now only survives in third-party copies. The submodule points at an orphan
commit in `MalfoyJW/parsec-sdk` that is not reachable from that repository's `master`,
which is not a trustworthy pin on its own.

The blob was corroborated against an unrelated copy in `mbroemme/parsec-sdk` at
`sdk/ios/ParsecSDK.framework/ParsecSDK`. Both are byte-identical:

```
git blob sha1  7793be4c777ae9d64ad6cef0e3568e4c117af060
file sha256    2ef504fa2aad5e23a615c5f0d2d440a3102fe5dd34b471c700e08acf492e4603
size           5671424
```

Two independent uploaders holding identical bytes is reasonable evidence the framework
is the genuine Parsec SDK rather than a modified one. Static analysis of the binary found
a single network host, `kessel-ws.parsecgaming.com`, plus standard UPnP/NAT-PMP traversal
strings. No other endpoints.

The workflow now fails the build if either the submodule commit or the binary hash moves,
rather than silently compiling whatever is there.

## Rotating a pin

Update the SHA in `.github/workflows/build.yml`, and for `ldid` or `ParsecSDK` update the
matching checksum in the same step. A pin bump should be its own commit so it is reviewable.

## AltStore source

`update_json.py` derives the repository from `GITHUB_REPOSITORY` instead of hardcoding
upstream. Without this, a fork's generated `altstore.json` points AltStore at the upstream
`.ipa`, so the hardened build gets produced and then never installed.
