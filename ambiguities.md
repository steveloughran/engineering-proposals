# Ambiguities and Issues in `qualifying-an-SDK-upgrade.md`

## Structural Errors

**Wrong section headings — B4 and B5 are mislabelled as B3:**

- Line 510: `##### B3. Long Haul, FIPS, [Object Lock]` — this section covers **B4**
- Line 553: `##### B3. Third-Party Bucket` — this section covers **B5**

**PR readiness checklist duplicates B3 (lines 1655–1663):**

```
[ ] I have run the ITests against B3; no failures were observed.
[ ] If available, I have run the ITests against B3; no failures were observed
```

Both reference B3. One should presumably be B4, the other B5.

---

## Undefined Hadoop Properties

The document uses `${var}` expansion in XML configs, which requires the property `var` to be defined
elsewhere in the Hadoop configuration. The shell env vars (`export B1=s3a://...`) are **not** the same
thing — Hadoop does not inherit shell env vars unless the `${env.VAR}` syntax is used.
The following properties appear in XML but are never defined as Hadoop properties anywhere in the document:

| Property Reference      | Used In   | What It Should Contain                      |
|-------------------------|-----------|---------------------------------------------|
| `${B1}`                 | B1 config | Full S3A URI of bucket 1                    |
| `${B3_Region}`          | B3 config | AWS region for S3 Express bucket            |
| `${B4_REGION}`          | B4 config | AWS region for long-haul bucket             |
| `${ENCRYPTION_KEY_ARN}` | B4 config | ARN of the CSE-KMS encryption key           |
| `${KMSKEY}`             | B1 config | ARN of the SSE-KMS key                      |
| `${STSENDPOINT}`        | B1 config | STS endpoint URL for assumed role           |
| `${REGION}`             | B1 config | AWS region for KMS and STS                  |
| `${ACCESS_POINT_ARN}`   | B2 config | ARN of the access point for B2              |
| `${ROLE_ARN}`           | B1 config | ARN of the IAM role for assumed role        |

The document should either show these being defined as `<property>` entries (e.g. in `auth-keys.xml`),
use `${env.VAR}` syntax, or explicitly tell the reader to substitute literal values.

---

## XML Configuration Issues

| Issue | Location | Detail |
|---|---|---|
| GCS bucket properties in S3 Express (B3) config | Lines 488–498 | `fs.s3a.bucket.stevel-gcs.committer.magic.enabled` and `fs.s3a.bucket.stevel-gcs.checksum.calculation.enabled` are hardcoded to a GCS bucket (`stevel-gcs`) and don't belong here |
| `accesspoint.arn` set twice in B2 config | Lines 417–419, 432–434 | First value is `B2 ACCESS_POINT_ARN` (literal placeholder); second is `${ACCESS_POINT_ARN}` (property ref). Different values, one will silently override the other |
| `test-id` as literal in B1 config | Line 355 | `fs.s3a.bucket.B1.assumed.role.external.id` set to `test-id` — not marked as a placeholder, unclear if this is intentional or needs replacing |
| Global property in B3 config | Line 502 | `fs.s3a.create.storage.class.enabled` has no bucket prefix — applies globally, not just to B3; may interfere with other bucket tests |
| `analytics` stream forced on B2 | Line 412 | B2 is the versioned/path-style bucket; forcing `input.stream.type=analytics` here is unexplained — intentional test of analytics on versioned store, or copy-paste error? |
| B2 XML uses literal `B2` not a property ref | Line 399 | `<value>B2</value>` for `test.fs.s3a.name` — unlike B1 which uses `${B1}`, this is inconsistent and would set the test bucket name to the literal string "B2" |
| `B2_Region` literal in B2 config | Line 402 | `<value>B2_Region</value>` — this is clearly a placeholder but not marked as one |

---

## Broken or Incomplete Commands

**Line 695 — Maven command with no goals:**

```bash
mvn -T 1C
```

No lifecycle phase or goal specified. This would fail immediately. Presumably meant to be
`mvn -T 1C verify` or `mvn -T 1C install`.

**Line 696 — `--pl` should be `-pl`:**

```bash
mvn -T 1C integration-test -Dmaven.plugin.validation=none -Dparallel-tests -DtestsThreadCount=9 -Dscale --pl hadoop-tools/hadoop-aws
```

Maven uses `-pl`, not `--pl`. This command will fail to parse.

---

## Missing Configuration

**B2AP has no XML config block.** The access-point variant of B2 is listed in the bucket table and
env var setup but no XML configuration is provided for it. The reader has no guidance on what
properties differ from the B2 config when using an access point URL directly.

---

## Process Ambiguities

**"Create a notes document" (line 647):** No format, required content, location, or example is given.

**Two pre-upgrade release builds with no explained distinction:**

- Line 707: Build `before-update` release — described as a reference "before the upgrade"
- Line 724: Build `preflight` release — used for CLI testing and cleaning buckets

Both are built before any SDK change is applied, and both use essentially the same
`mvn package -Pdist` command. The distinction between them, and why two separate builds are needed,
is never explained.

**Log review guidance is vague (lines 821–828):** The reader is told to "review every single
`-output.txt` file" but given no specifics on what constitutes a problem beyond "new warning
messages." No baseline comparison strategy is described.

**Cloudstore `bandwidth` command flags are unexplained (lines 1231–1234):**

```bash
time bin/hadoop jar $CLOUDSTORE bandwidth -policy analytics -rename 512M $BUCKET/testfile
time bin/hadoop jar $CLOUDSTORE bandwidth -policy analytics -rename 128M $BUCKET/testfile
```

The `-rename` flag is not explained. The comment on the third invocation says "close during the
download" but no mechanism for triggering a mid-download close from the CLI is described.

**"Stop!" section is self-contradictory (lines 111–122):** It states both "if it is for any feature:
postpone" and "if it is for a critical fix: postpone" — covering all possible cases with the same
answer. It never addresses what to actually do if a critical fix genuinely cannot wait.

**`ILoadTest*` tests have no module specified (line 818):**

```bash
mvn verify -Dtest=skip -Dit.test=ILoadTest\* -Dscale
```

No `-pl hadoop-tools/hadoop-aws` qualifier. Run from the wrong directory this would attempt (and
likely fail) to run against all modules.
