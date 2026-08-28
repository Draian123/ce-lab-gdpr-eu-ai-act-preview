# Lab M8.07 — Solution: GDPR Data Protection in AWS

**Bucket:** `gdpr-personal-data-1787900583` · **Region:** `eu-west-1` · **Account:** 697345203222
**Completed:** 2026-08-28

Full audit document: [gdpr-evidence.md](gdpr-evidence.md)

> Note: the lab page (unit 8.3.4) supersedes the generic `README.md` shipped in this repo.
> This solution follows the lab page: an S3-based GDPR data store plus a right-to-erasure workflow.

## Verification checklist

| # | Requirement | Result |
|---|---|---|
| 1 | Bucket created in `eu-west-1` | ✅ `"LocationConstraint": "eu-west-1"` |
| 2 | All 4 Block Public Access settings true | ✅ BlockPublicAcls, IgnorePublicAcls, BlockPublicPolicy, RestrictPublicBuckets |
| 3 | Default encryption AES256 + enforce policy | ✅ SSE-S3 with BucketKeyEnabled, plus `DenyUnencryptedUploads` |
| 4 | Versioning enabled | ✅ `"Status": "Enabled"` |
| 5 | CSV uploaded with `DataClassification=PersonalData` | ✅ 4 tags, `ServerSideEncryption: AES256` |
| 6 | Lifecycle rule `gdpr-30-day-retention`, 30-day expiry | ✅ confirmed by `expiry-date` header on upload |
| 7 | U003 (Clara Rossi) removed from current version | ✅ `SUCCESS: U003 record removed.` |
| 8 | Older versions containing U003 deleted | ✅ version `vtvzr9mWVworjo2fYuFbSUUe8L6URpbi` deleted; 1 version remains |
| 9 | `gdpr-evidence.md` complete | ✅ controls table, erasure log, Art. 30 register |

## Steps executed

1. **EU data bucket** — `create-bucket` with `LocationConstraint=eu-west-1`, verified with `get-bucket-location`.
2. **Block Public Access** — all four flags set and verified (Article 32).
3. **Encryption** — default SSE-S3 (AES-256) with S3 Bucket Keys, plus a bucket policy denying any `PutObject` that does not carry `x-amz-server-side-encryption: AES256`.
4. **Versioning** — enabled for audit trail and accidental-deletion protection.
5. **Personal data upload** — 5 simulated EU user records to `users/personal-data.csv`, tagged `DataClassification=PersonalData&GDPRArticle=4&LegalBasis=Consent&RetentionDays=30`.
6. **Lifecycle retention** — tag-filtered rule expiring objects after 30 days and non-current versions after 7 (Article 5(1)(e)).
7. **Right to erasure (Article 17)** for U003 / Clara Rossi:
   - exported her record first (Articles 15 & 20) → `clara-rossi-export.csv`
   - removed the row → `personal-data-cleaned.csv`
   - re-uploaded encrypted and re-tagged
   - **deleted the pre-erasure object version** — the key step: with versioning on, the old version still holds the data, so erasure is incomplete until every version ID is deleted
   - verified with `grep` on the freshly downloaded current object
8. **Evidence** — `gdpr-evidence.md`, four AWS console screenshots (`screenshots/*.png`) and the matching four CLI captures (`screenshots/*.txt`).

## Key takeaway

Versioning and the right to erasure are in direct tension. Enabling versioning is good security practice, but it means a "delete" only hides data behind a new current version. A GDPR erasure request is only satisfied once `list-object-versions` shows no version containing the subject's data — and the same applies to backups, replicas and CloudTrail-adjacent copies in a real system.

## Cleanup

Not yet run — the bucket is left in place so the controls can be inspected. To remove it:

```bash
BUCKET=gdpr-personal-data-1787900583
# delete every version and delete marker
aws s3api list-object-versions --bucket $BUCKET   --query '[Versions,DeleteMarkers][][].{Key:Key,VersionId:VersionId}' --output json | jq -c '{Objects: ., Quiet: true}' | xargs -0 -I{} aws s3api delete-objects --bucket $BUCKET --delete '{}'

aws s3api delete-bucket --bucket $BUCKET --region eu-west-1
```
