# Mediroza General Hospital — Penetration Testing Report

**Networkwalks Cybersecurity Program | Batch B083 | Week 4**

> ⚠️ **Educational Engagement Notice**
> This assessment was carried out against `medirozahospital.com`, a training target set up for the Networkwalks Batch B083 program, under explicit written authorisation from Networkwalks for the sole purpose of this coursework. Every technique described below was used in that controlled, authorised context only. None of this should be attempted against any system you don't own or have written permission to test — doing so is illegal in most jurisdictions.

---

## Project Overview

| | |
|---|---|
| **Client** | Mediroza General Hospital |
| **Engagement Type** | Black-box Penetration Test |
| **Target** | `https://medirozahospital.com` |
| **Duration** | 5 days |
| **Tester** | *[Your name / handle]* |
| **Program** | Networkwalks — Batch B083, Week 4 |
| **Authorisation** | Written permission granted by Networkwalks for this engagement |

The brief for this engagement was simple to state and more involved to execute: get into the application as an outsider would, pull three confidential patient lab reports, break whatever was protecting them, and see how far that access actually went. It turned into a pretty clean example of how a handful of small, "low severity on their own" misconfigurations stack into a serious data breach when nobody's connecting the dots between them.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Scope & Rules of Engagement](#2-scope--rules-of-engagement)
3. [Methodology & Tools](#3-methodology--tools)
4. [Findings & Proof of Exploitation](#4-findings--proof-of-exploitation)
   - [Finding 1 — Information Disclosure via robots.txt / Broken Access Control](#finding-1--information-disclosure-via-robotstxt--broken-access-control)
   - [Finding 2 — Weak PDF Encryption (Crackable Password Protection)](#finding-2--weak-pdf-encryption-crackable-password-protection)
   - [Finding 3 — Exposed Database Backup File](#finding-3--exposed-database-backup-file)
5. [Risk Rating Summary](#5-risk-rating-summary)
6. [Recommendations & Remediation](#6-recommendations--remediation)
7. [Evidence](#7-evidence)
8. [Tools Used](#8-tools-used)
9. [Lessons Learned](#9-lessons-learned)
10. [Disclaimer](#10-disclaimer)

---

## 1. Executive Summary

Mediroza General Hospital's public-facing web application exposed sensitive data at multiple points along the chain, from initial recon all the way through to a full database backup sitting in an unauthenticated directory. No part of this engagement required anything exotic — there was no zero-day, no custom exploit development, no social engineering. Every step relied on the same pattern: the server told you where not to look, and nothing actually stopped you from looking anyway.

In short, over the course of this assessment I was able to:

- Enumerate the application and identify directories the client clearly intended to keep private, straight from `robots.txt`.
- Retrieve three confidential, password-protected patient pathology reports from one of those restricted paths with no authentication required.
- Crack the password protection on all three PDFs offline and recover the full patient data inside them.
- Use information surfaced during that process to locate a forgotten backup directory, which contained a full MySQL dump of the hospital's internal database — including staff salaries, national ID numbers, and shareholder equity records.

Taken together, these issues amount to a **critical-risk** exposure of both patient health information (PHI) and internal financial/personnel data, for an organisation whose entire reason for existing is to be trusted with exactly that kind of information.

---

## 2. Scope & Rules of Engagement

| Item | Detail |
|---|---|
| **Scope** | `medirozahospital.com` and its web infrastructure only |
| **Test type** | Black-box — no prior credentials, source code, or internal documentation provided |
| **Allowed** | Reconnaissance, vulnerability identification, exploitation to demonstrate real-world impact, use of AI tooling for analysis/reporting |
| **Not allowed** | Social engineering, denial-of-service testing, any testing outside the agreed domain |
| **Authorisation** | Written permission confirmed prior to testing, issued through the Networkwalks program |

The goals were broken into four milestones by the brief, which I've used as the spine for the findings section below:

| Milestone | Objective |
|---|---|
| M1 | Gain initial access and retrieve the 3 confidential patient PDF reports |
| M2 | Crack the encryption protecting those 3 files |
| M3 | Use what's recovered to find a deeper exposure — staff salaries & shareholder data |
| M4 | Document everything in a professional report (this document) |

---

## 3. Methodology & Tools

The approach followed a fairly standard black-box web app methodology:

1. **Passive & active reconnaissance** — fingerprint the stack, pull `robots.txt`, brute-force the directory structure.
2. **Access restricted content** — follow up on anything recon flagged as "hidden" rather than actually protected.
3. **Analyse what's retrieved** — don't just read the files, look at how they're protected and what metadata they carry.
4. **Offline attacks** — where content is encrypted, extract a crackable hash and go after it with a wordlist/brute-force attack rather than hammering the live server.
5. **Pivot on findings** — treat every recovered artifact as a potential lead to something else, not just a deliverable to check off.

### Tools used

| Tool | Purpose |
|---|---|
| `whatweb` | Technology/stack fingerprinting |
| `gobuster` | Directory & file brute-forcing |
| `curl` / browser | Manual verification of discovered paths |
| `pdf2john` | Extracting a crackable hash from password-protected PDFs |
| `John the Ripper` | Offline password cracking against the PDF hashes |
| `exiftool` / `pdfinfo` | Inspecting PDF metadata and encryption parameters |
| `pdftotext` | Reading recovered PDF contents once decrypted |

---

## 4. Findings & Proof of Exploitation

### Finding 1 — Information Disclosure via robots.txt / Broken Access Control

**Where it started:** `robots.txt` is meant to tell search engine crawlers which paths *not* to index — it is not, and has never been, an access control mechanism. Mediroza's `robots.txt` read:

```
User-agent: *
Disallow: /patient/
Disallow: /staff/
Disallow: /old/

Sitemap: https://medirozahospital.com/sitemap.xml
```

That's effectively a signed confession of which three directories the client considered sensitive — which made them the first three things I checked.

A `gobuster` sweep of the site (extension list included `.php`, `.txt`, `.bak`, `.pdf`) corroborated two of them independently: `/staff/` and `/old/` both came back with a `301` redirect rather than the `403` that properly access-controlled directories returned elsewhere on the site. `/patient/` wasn't in the brute-force wordlist, but since `robots.txt` had already named it directly, there was no need to guess — I just requested it.

**The problem:** none of these three directories enforced any authentication. Whatever process restricted other parts of the site (most paths returned `403 Forbidden`, including some fairly aggressive attempts at `.htaccess`/`.git` disclosure) simply wasn't applied here. Requesting `/patient/` directly returned the three confidential pathology lab reports:

- `patient_report_1.pdf`
- `patient_report_2.pdf`
- `patient_report_3.pdf`

**Impact:** anyone — no login, no session, no special tooling — who thought to check `robots.txt` could pull confidential patient lab reports straight off the live server. This satisfies Milestone 1 in full: proof of access plus the three retrieved files.

> 📸 *Screenshot placeholder: `robots.txt` contents, gobuster output showing `/staff/` and `/old/` as 301s, and the browser request retrieving the 3 PDFs from `/patient/`.*

---

### Finding 2 — Weak PDF Encryption (Crackable Password Protection)

Each of the three recovered PDFs was password-protected — opening any of them without a password returned an `Incorrect password` error from standard PDF tooling. That's the right instinct on the client's part, but the implementation didn't hold up.

**Approach:**

1. Ran `pdf2john` against each file to extract a crackable hash in John the Ripper's PDF hash format, e.g.:

   ```
   $pdf$2*3*128*-4*1*32*<salt>*32*<u-value>*32*<o-value>
   ```

   All three reports came back as **RC4, 128-bit, revision 3** encryption — an older algorithm that's been considered weak for password-recovery purposes for a long time, since it doesn't apply any meaningful work-factor (no iterated key derivation) between password attempts, making it very fast to brute-force offline.

2. Fed each hash into John the Ripper against a wordlist. Because the hashes shared the same algorithm, the cracking method didn't need to change between files — but the passwords themselves weren't identical, so each file still had to be attacked individually rather than assuming success on one meant success on all three.

3. Once a password was recovered for a given file, it was used to strip the encryption (`patient_report_X_decrypted.pdf`), which could then be opened normally.

**Impact:** all three files yielded their full contents — patient name, report date, patient ID, and the pathology findings themselves. In other words, the password protection added effort for a legitimate user trying to retrieve their own report, but essentially no meaningful protection against an attacker who already had the file. Milestone 2 complete: recovered contents of all three, decrypted and readable.

> 📸 *Screenshot placeholder: `pdf2john` hash extraction, John the Ripper cracking session (status/cracked output), and one decrypted report opened showing the (redact patient details in your screenshot) report layout.*

> **Note on the report itself:** the actual decrypted contents contain realistic-looking patient names and medical data generated for this training environment. When you add your screenshots, consider blurring/redacting the specific patient identifiers even though this is a lab — it's the habit that matters, and it's exactly what you'd be required to do in a real client-facing report.

---

### Finding 3 — Exposed Database Backup File

This is where Milestone 3's hint — *"look beyond the obvious content"* — actually paid off. `robots.txt` and the gobuster scan had already flagged `/old/` as a directory nobody wanted indexed, same as `/patient/` and `/staff/`. Having already learned that "disallowed" didn't mean "protected," `/old/` was the obvious next stop.

Browsing to `/old/` directly led to an unauthenticated listing containing a leftover database export: **`mediroza_db_backup_2019.sql`**.

Opening the file revealed a full MySQL dump, header included:

```sql
-- mediroza_db_backup_2019.sql
-- Mediroza General Hospital - internal database backup
-- Generated by Mediroza CMS module
-- WARNING: contains confidential staff and shareholder records
```

...which was not an exaggeration. The dump contained two tables in particular:

- **`staff`** — full name, job title, department, email, phone, national ID number, monthly salary, and date joined for roughly 30 hospital employees, from pathology lab staff up through the IT and HR departments.
- **`shareholders`** — shareholder name, share percent, shares held, and share class for the hospital's ownership structure.

**Impact:** this single file turned a "we leaked some lab results" incident into a full internal data breach. National ID numbers and salary data for every staff member, plus the hospital's cap table, were sitting in a publicly reachable directory with zero authentication — no cracking, no brute-forcing, just a direct request. This satisfies Milestone 3's tasks: both staff salaries and shareholder details were recovered in full.

> 📸 *Screenshot placeholder: directory listing at `/old/`, and the opened `.sql` file showing the `staff` and `shareholders` table structure/contents (redact real-looking salary/ID values in the screenshot the same way as above).*

---

## 5. Risk Rating Summary

| # | Finding | Risk Rating | Justification |
|---|---|---|---|
| 1 | Information disclosure via `robots.txt` + missing access control on `/patient/`, `/staff/`, `/old/` | **Critical** | Zero-authentication access to confidential patient health records; also the root cause that enabled Findings 2 & 3 |
| 2 | Weak/crackable PDF encryption (RC4-128) protecting patient reports | **High** | Protection is trivially defeated offline once the file is obtained; no rate limiting possible since cracking happens off the server |
| 3 | Exposed database backup (`/old/` directory) containing staff & shareholder PII/financial data | **Critical** | Full compromise of employee PII (incl. national ID numbers) and confidential corporate ownership data, with no exploitation effort beyond a direct request |

**Overall engagement risk: Critical.** The combination of a predictable directory-disclosure pattern and a complete absence of authentication on anything under those paths means the real root cause isn't "three separate bugs" — it's a systemic lack of access control on non-public content.

---

## 6. Recommendations & Remediation

| Finding | Recommendation |
|---|---|
| **robots.txt / Broken Access Control** | Stop relying on `robots.txt` for confidentiality — it is a crawler courtesy convention, not a security control. Every path serving patient, staff, or administrative content must require proper server-side authentication and authorisation (e.g., session-based auth with role checks), not just an entry in a disallow list. Return `403`/`401` consistently for any unauthenticated request to these paths, and verify directory listing is disabled everywhere. |
| **Weak PDF encryption** | Stop depending on PDF "password protect" as a security boundary for PHI. If PDFs must be distributed, use strong, modern encryption (AES-256 at minimum) and, more importantly, don't let them be reachable by unauthenticated users in the first place — the file should never be fetchable without an authenticated session tied to the specific patient/record. |
| **Exposed backup file** | Remove all backup, archive, and "old" content from any web-accessible directory, full stop. Backups belong in storage that isn't served by the web server at all (e.g., encrypted off-site storage with access logging). Add this to a pre-deployment checklist, and periodically re-scan the production web root for forgotten files. |
| **General** | Run a recurring external attack-surface review (recon + content discovery) against the live site, since this entire chain was found using only publicly available, non-invasive techniques. Any of these three findings alone would have been bad; together, they represent exactly the kind of "small misconfigurations chain into a breach" scenario that periodic reviews are meant to catch before someone with less good intentions does. |

---

## 7. Evidence

Screenshots and supporting evidence files are stored in [`/evidence`](./evidence) *(update this path once uploaded)* and referenced inline above. Evidence included for this engagement:

- [ ] `robots.txt` contents
- [ ] `gobuster` scan output
- [ ] `whatweb` fingerprint output
- [ ] Browser request retrieving the 3 patient PDFs from `/patient/`
- [ ] `pdf2john` hash extraction output (all 3 files)
- [ ] John the Ripper cracking session output
- [ ] Decrypted PDF contents (redacted)
- [ ] `/old/` directory listing
- [ ] `mediroza_db_backup_2019.sql` contents (redacted)

---

## 8. Tools Used

`whatweb` · `gobuster` · `curl` · `pdf2john` · `John the Ripper` · `exiftool` · `pdfinfo` · `pdftotext`

---

## 9. Lessons Learned

The part of this project that stuck with me wasn't any single technique — none of them were advanced on their own. It was how much the three findings depended on each other. The `robots.txt` disclosure wouldn't have mattered much if `/old/`, `/staff/`, and `/patient/` had actually been access-controlled. The weak PDF encryption wouldn't have mattered if the files hadn't been reachable without authentication in the first place. And the database backup would never have been found if the first finding hadn't established the "disallowed paths are actually just unlisted, not protected" pattern for this specific application.

That's really the takeaway for the write-up: a report that scores each bug in isolation would undersell this engagement. The real finding is the pattern connecting them.

---

## 10. Disclaimer

This assessment was conducted in a controlled, educational environment as part of the Networkwalks Batch B083 training program, against a target explicitly provisioned and authorised for this purpose. All data referenced (patient records, staff details, shareholder information) is synthetic/training data generated for this exercise. These techniques and findings must never be applied to any system without explicit, written, prior authorisation from its owner.
