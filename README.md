# FOSecure

A Dynamics 365 Finance & Operations project collecting security hardening changes raised by
penetration tests and audit findings. Every feature stands on its own, ships disabled, and is turned
on from a standard parameter page.

## Features

| Feature | What it does | Docs |
|---|---|---|
| Attachment security | Checks an attachment's magic bytes against its extension, blocks files carrying JavaScript, macros or other content that runs when the file is opened, and scans them with Defender for Storage. | [docs/attachment-security.md](docs/attachment-security.md) |

## Install

Deploy the package produced by the GitHub Actions build
([`.github/workflows/build.yml`](.github/workflows/build.yml), FSC-PS, target 10.0.49).

## Licence

MIT — see [LICENSE](LICENSE).
