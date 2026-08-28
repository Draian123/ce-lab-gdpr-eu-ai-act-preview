# GDPR Compliance Evidence

**Date:** 2026-08-28T07:06:26Z
**Engineer:** Dennis Beitel
**Bucket:** `gdpr-personal-data-1787900583`
**Region:** eu-west-1 (Ireland)
**AWS Account:** 697345203222

---

## Technical Controls Implemented

| Control | GDPR Basis | Status | Evidence |
|---|---|---|---|
| Data residency: eu-west-1 | Article 44-46 — Data transfers | ✅ Compliant | `get-bucket-location` → `"LocationConstraint": "eu-west-1"` |
| S3 Block Public Access | Article 32 — Security measures | ✅ Enabled | All 4 settings = `true` |
| Encryption at rest (AES-256) | Article 32(1)(a) — Encryption | ✅ Enforced | Default SSE-S3 + `DenyUnencryptedUploads` bucket policy |
| Encryption in transit (HTTPS) | Article 32(1)(a) — Encryption | ✅ Enforced | S3 API endpoint accessed over TLS |
| Object versioning (audit trail) | Article 5(2) — Accountability | ✅ Enabled | `get-bucket-versioning` → `"Status": "Enabled"` |
| Data classification tags | Article 5(2) — Accountability | ✅ Applied | `DataClassification=PersonalData`, `GDPRArticle=4`, `LegalBasis=Consent`, `RetentionDays=30` |
| 30-day auto-deletion | Article 5(1)(e) — Storage limitation | ✅ Active | Lifecycle rule `gdpr-30-day-retention`; S3 returned `expiry-date="Mon, 28 Sep 2026 00:00:00 GMT"` on upload |
| Non-current version expiry | Article 5(1)(e) — Storage limitation | ✅ Active | `NoncurrentVersionExpiration: 7 days` |

### Enforced encryption policy

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyUnencryptedUploads",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:PutObject",
    "Resource": "arn:aws:s3:::gdpr-personal-data-1787900583/*",
    "Condition": {
      "StringNotEquals": { "s3:x-amz-server-side-encryption": "AES256" }
    }
  }]
}
```

---

## Right to Erasure (Article 17)

**Data subject:** Clara Rossi (`user_id: U003`)
**Request received:** 2026-08-28
**Completed:** 2026-08-28 — same day, well within the one-month deadline of Article 12(3)

**Actions taken:**

1. **Export before deletion** (Articles 15 & 20) — the record held for U003 was extracted and provided to the data subject as `clara-rossi-export.csv`:
   ```
   user_id,name,email,ip_address,country,signup_date
   U003,Clara Rossi,clara.rossi@example.com,172.16.0.8,IT,2024-03-10
   ```
2. **Erasure from the active dataset** — the U003 row was removed, producing `personal-data-cleaned.csv` (4 records remaining: U001, U002, U004, U005).
3. **Re-upload** — the cleaned dataset was written back to `s3://gdpr-personal-data-1787900583/users/personal-data.csv` with `--server-side-encryption AES256` and the full GDPR tag set. New version: `OA8BertmV3YYFWJYtneT5dQ9vHvGEU12`.
4. **Purge of historic versions** — versioning was enabled, so the pre-erasure version still contained U003's data. Version `vtvzr9mWVworjo2fYuFbSUUe8L6URpbi` was explicitly deleted with `s3api delete-object --version-id`. Erasure is not complete until every version is removed.
5. **Verification** — the current object was downloaded and searched.

**Verification result:**

```
$ grep "U003" verify.csv && echo "ERROR: Record still exists!" || echo "SUCCESS: U003 record removed."
SUCCESS: U003 record removed.
```

`list-object-versions` now returns a single version (the post-erasure one). No copy of U003's personal data remains in the bucket.

---

## Data Processing Activity (Article 30 register)

| Field | Value |
|---|---|
| Data category | Name, email address, IP address, country of residence, signup date |
| Special categories (Art. 9) | None processed |
| Data subjects | EU-resident SaaS platform users (FR, DE, IT) |
| Legal basis | Consent — Article 6(1)(a) |
| Purpose | User account management and service delivery |
| Retention period | 30 days from object creation, enforced automatically by S3 lifecycle rule `gdpr-30-day-retention` |
| Storage location | Amazon S3, `eu-west-1` (Ireland) — no transfer outside the EEA |
| Processor | AWS Europe (EU Data Processing Addendum in force) |
| Access control | IAM least-privilege; no public access (all 4 Block Public Access settings enabled) |
| Encryption | SSE-S3 / AES-256 at rest, TLS in transit |

### Note on IP addresses

IP addresses are personal data under GDPR (CJEU, *Breyer v Germany*, C-582/14) because they are identifiable in combination with other data. They are therefore covered by the same classification tag, encryption and retention rule as names and email addresses.

---

## Breach Notification Process

1. **Detection** — GuardDuty and Security Hub findings, CloudTrail data events on the bucket.
2. **Supervisory authority** — notify the lead DPA within 72 hours of becoming aware (Article 33).
3. **Data subjects** — notify without undue delay where the breach is likely to result in a high risk to rights and freedoms (Article 34). Encryption at rest is a mitigating factor under Article 34(3)(a).

---

## Evidence Files

| File | Contents |
|---|---|
| `screenshots/bucket-encryption.png` | S3 console → Properties: default encryption = SSE-S3 (AES-256), Bucket Key enabled, SSE-C blocked |
| `screenshots/public-access-block.png` | S3 console → Permissions: "Block *all* public access: On" with all four individual settings ticked, and the `DenyUnencryptedUploads` bucket policy |
| `screenshots/lifecycle-rule.png` | S3 console → Management: lifecycle rule `gdpr-30-day-retention`, Status Enabled, Scope Filtered, Action Expires + permanently delete noncurrent versions |
| `screenshots/erasure-verification.png` | S3 console → Objects with **Show versions** enabled: a single version (`OA8BertmV3YYF…`) of `users/personal-data.csv` remains — the pre-erasure version holding U003's data is gone |
| `screenshots/bucket-encryption.txt` | CLI: bucket region, default encryption, deny-unencrypted policy, versioning |
| `screenshots/public-access-block.txt` | CLI: all four Block Public Access settings = true |
| `screenshots/lifecycle-rule.txt` | CLI: lifecycle rule `gdpr-30-day-retention` and object GDPR tags |
| `screenshots/erasure-verification.txt` | CLI: version list after purge and the U003 removal verification |
| `personal-data.csv` | Dataset as originally uploaded (before erasure) |
| `personal-data-cleaned.csv` | Dataset after erasure (no U003) |
| `clara-rossi-export.csv` | Data export supplied to the data subject |

Each control is evidenced twice: an AWS Management Console screenshot showing the configured state, and
the corresponding AWS CLI API response. The console screenshots were taken in the `eu-west-1` (Europe,
Ireland) console for account 697345203222, which is itself corroborating evidence of the data-residency
control.

---

## Data Protection Officer

- Contact: dpo@example.com
