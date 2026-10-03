# Week 4 — Mediroza General Hospital: Black-box Penetration Test

**Program:** NetworkWalks Cybersecurity & Ethical Hacking, Batch 83
**Target:** `https://medirozahospital.com` (authorized training target provided by NetworkWalks)
**Type:** Black-box web application pentest
**Milestones:** M1 Initial Access · M2 Data Extraction · M3 Critical Data Exposure · M4 Reporting

> Written authorization for this engagement was granted as part of the NetworkWalks training brief. Testing was limited to the target domain — no social engineering, no DoS, no testing outside scope.

---

## TL;DR

A SQL injection flaw in the patient portal login allowed a full authentication bypass, giving access to three password-protected patient lab reports. All three passwords were cracked with Hashcat in seconds using `rockyou.txt`. Metadata inside one of the PDFs led to a legacy directory with a fully exposed internal database backup containing staff salaries, national ID numbers, and shareholder ownership data.

---

## M1 — Initial Access

Recon started with WHOIS, DNS, and a `robots.txt` check:

```
User-agent: *
Disallow: /patient/
Disallow: /staff/
Disallow: /old/
```

Both `/staff/` and `/old/` turned out to have directory listing enabled (misconfigured LiteSpeed server). `/staff/login.php` resisted SQL injection, but the **patient portal** login did not:

```
Username: admin' -- 
Password: (blank)
```

This bypassed authentication entirely and returned a "My lab reports" page with three password-protected PDFs belonging to patients unrelated to the injected username — confirming a broken auth check, not a real account.

## M2 — Data Extraction

All three PDFs were password-protected (PDF 1.4, RC4 128-bit). Hashes were extracted with `pdf2john` and cracked with **Hashcat** (mode `10500`) against `rockyou.txt`:

| File | Password | Cracking effort |
|---|---|---|
| patient_report_1.pdf | `123456` | Immediate |
| patient_report_2.pdf | `password` | Immediate |
| patient_report_3.pdf | `!@#$%^&` | ~1 second (73k attempts) |

## M3 — Critical Data Exposure

`exiftool` on the decrypted files revealed an internal comment left in `patient_report_3.pdf`:

```
Author:    j.malik
Comments:  DB backup moved to /old before site migration, do not delete
```

This confirmed the `/old/` lead from `robots.txt`. The directory listing exposed `mediroza_db_backup_2019.sql` — a full HR database dump (staff salaries, national IDs, shareholder records), downloadable with no authentication.

## Key takeaway

No single flaw here was advanced. It was the combination — SQL injection, a misconfigured server, weak passwords, and a forgotten backup — that turned into a critical exposure. That's usually how it goes in practice.

## Tools used

`whois` · `dig`/`nslookup` · `gobuster` · manual SQL injection · `pdf2john` · Hashcat · `qpdf` · `exiftool`

## Full report

See [`reports/Mediroza_Penetration_Testing_Report.docx`](./reports/Mediroza_Penetration_Testing_Report.docx) for the full writeup (Executive Summary, Risk Ratings, Recommendations).

---

*Evidence (screenshots, cracked hashes, the downloaded SQL dump) stored under `weeks/week4/evidence/` — not committed in full here to avoid publishing synthetic-but-realistic PII (names, national ID numbers, salaries) outside the training environment.*
