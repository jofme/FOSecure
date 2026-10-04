# Attachment security

F&O trusts whatever extension a user puts on an attachment, and there is no way to run an
anti-malware agent over the attachment store. This adds three checks.

| Check | What it does | Needs Azure |
|---|---|---|
| **Magic bytes** | Reads the file signature and rejects the file if it contradicts the extension. A `.exe` renamed to `.pdf`. | No |
| **Active content** | Rejects files carrying something that runs when the file is opened. JavaScript in a PDF, a macro in a `.docx`, script in an SVG, a polyglot valid as two formats at once. | No |
| **Malware scan** | Copies the attachment to a quarantine container scanned by Microsoft Defender for Storage and records the verdict. | Yes |

All three run from `Docu.delegateScanDocument`, the extension point Microsoft documents for
[scanning attachments for viruses and malicious code](https://learn.microsoft.com/en-us/dynamics365/fin-ops-core/dev-itpro/organization-administration/configure-document-management#scanning-attachments-for-viruses-and-malicious-code),
so they cover upload, preview and download. F&O ships no scanner of its own.

## Prerequisites

Malware scanning only.

- An Azure Storage account with **Defender for Storage malware scanning** enabled.
- A blob container used for **nothing else**
- A **container-scoped SAS** with `Create`, `Write`, `Delete` and `Tag`, stored in Key Vault as a
  **Manual** secret. Manual is the only type the platform can read a value from.
- That Key Vault registered on *System administration > Setup > Key Vault parameters*.

## Configuration

*Organization administration > Document management > Document management parameters > **Security***

| Field | Default | Effect |
|---|---|---|
| **Security** | | |
| Validate attachment content against file name extension | No | Rejects a file whose signature contradicts its extension. Undeterminable types are allowed. |
| Block attachments that contain active content | No | Rejects on a *Blocking* finding. When off, findings are still recorded. |
| Maximum bytes to inspect per attachment | 4 MB | How much of each file is read. Clamped to 64 KB – 16 MB. |
| **Malware scanning** | | |
| Scan attachments for malware | No | Copies each attachment to the quarantine container. Scanning is asynchronous, so the file is accepted before its verdict is known. |
| Block attachments when a scan result is unavailable | No | No preview or download until Defender reports the file clean, including while the scan is still pending. |
| Quarantine storage account / container | — | Where attachments are copied. |
| Key Vault / Key Vault secret for the container SAS | — | Where the container SAS is read from. |
| Maximum file size to scan (bytes) | 256 MB | Larger files are recorded *Not scanned*. |
| Seconds to wait for a scan result on upload | 10 (max 60) | Longer scans are resolved by the batch job. |

Scanning stays off unless the account, container and a readable SAS secret are all present.

## Monitoring

*Organization administration > Document management > Attachment security*

- **Attachment malware scans**: one row per submitted file, with state, detail, timestamps, size,
  SHA-256 and poll count. The SHA-256 correlates a row with its Defender for Cloud alert.
  - **Scan a file** uploads a file to prove the container, SAS and Defender all work. An EICAR test
    file should come back *Malicious*.
  - **Update scan results** polls outstanding rows now.
- **Attachment content inspections**: one row per inspected file, with detected type, extension check
  outcome, severity, findings, bytes inspected and whether inspection was truncated.

**Update scan results** is also a batch job (`FOSecureMalwareScanStatusService`). Schedule it with a
short recurrence. It resolves rows left *Pending*, *Scan error* or *Unavailable*, and marks rows still
unresolved 6 hours after submission as *Timed out*. Without it scheduled, rows stay *Pending*.

## What blocks

| | Rejected when |
|---|---|
| Extension check | Outcome is *Contradicts the extension*. Not checked, unrecognized, identified and matches all pass. |
| Active content | Severity is *Blocking* and *Block active content* is on. *Informational*, *Suspicious* and *Could not be assessed* never block. |
| Malware scan | State is *Malicious*, which blocks whenever scanning is enabled. The other non-clean states (*Pending*, *Scan error*, *Not scanned*, *Unavailable*, *Timed out*) block only when *Block when unavailable* is on. |

Severity of a finding:

| Finding | Severity |
|---|---|
| Polyglot, valid as more than one format | Blocking |
| Dynamic media, or launching another application | Blocking |
| Script | Blocking. Not for `.html`, `.htm`, `.xhtml`, `.mht` and `.mhtml`, where script is expected and the extension restriction list is the right control. SVG gets no exemption, because it is served inline, so script in one runs. |
| Macro | Blocking when the extension forbids one (`.docx`, `.xlsx`, `.pptx`, `.dotx`, `.xltx`, `.potx`). Suspicious where the format allows one, such as `.xlsm`. |
| Automatic action, external reference, external entity | Suspicious |
| Embedded file, interactive form | Informational. A PDF/A-3 e-invoice embeds XML and a tax form is interactive. |

## Security

Role **Attachment security administrator** holds the duty *Maintain attachment security scanning*,
which grants *Maintain attachment malware scanning* (the scan page and the batch job) and *View
attachment content inspections*.
