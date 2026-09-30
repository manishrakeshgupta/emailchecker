EMAIL VERIFIER — README2
=========================

PURPOSE
-------

This document records the local CLI fix, the controlled verification benchmark,
and the recommended operating logic for large-scale email hygiene.

The benchmark figures below are observations from a controlled test only.
They must not be interpreted as population-wide accuracy rates.


1. BUILD FIX — CLI FLAG COLLISION
---------------------------------

The original CLI contains a short-flag collision:

Global/root command:
-d = debug

Bulk command:
-d = delay

As a result, the bulk command fails during CLI initialization with a flag
redefinition error before any email verification begins.

The local source fix removes the short "-d" alias from the bulk delay flag.

Original:

    bulkCmd.Flags().Float64VarP(&bulkDelay, "delay", "d", 2.0, ...)

Fixed:

    bulkCmd.Flags().Float64Var(&bulkDelay, "delay", 2.0, ...)

Therefore:

    --delay

remains available for bulk verification, while:

    -d
    -dd
    -ddd

remain available for the global debug setting.

The corrected source was rebuilt successfully on Windows and the resulting
executable successfully launched and processed the controlled test dataset.

This fix is intentionally minimal: no SMTP verification logic was changed by
this CLI correction.


2. CONTROLLED 50-RECORD BENCHMARK
---------------------------------

The controlled test contained 50 records with independently known negative
delivery outcomes.

First-pass verifier result:

- VALID:    5
- INVALID: 34
- ERROR:   11
- UNKNOWN:  0
- RISKY:    0

Observed result distribution:

- INVALID: 68%
- ERROR:   22%
- VALID:   10%

ERROR results represent verification/connection failures and must not be
automatically classified as either valid or invalid.


3. CATCH-ALL FOLLOW-UP
----------------------

The five records initially returned as VALID were tested again with catch-all
detection enabled.

Second run:

- VALID:    3
- INVALID:  2
- ERROR:    0
- UNKNOWN:  0
- RISKY:    0

Catch-all checking removed two of the five initial false-positive VALID
results.

Three records still returned VALID despite having independently known
negative delivery outcomes.

Observed residual false-positive proportion in this controlled benchmark:

3 / 50 = 6%

IMPORTANT:

This 6% figure is a benchmark observation only.

It must NOT be treated as the expected error rate for other datasets.


4. CRITICAL INTERPRETATION
--------------------------

A successful SMTP recipient acceptance response does NOT guarantee future
delivery.

Therefore:

VALID != GUARANTEED DELIVERABLE

The verifier should be treated as an evidence-producing hygiene layer, not
as an absolute mailbox-existence oracle.

Result categories:

VALID
-----
The receiving server accepted the verification request.

INVALID
-------
The receiving server explicitly rejected the recipient.

ERROR / UNKNOWN
---------------
The verifier could not establish a reliable result because of connection,
DNS, timeout, server policy, rate limiting, EOF, reputation, or similar
conditions.

RISKY / CATCH-ALL
-----------------
The receiving environment accepts recipients in a way that makes individual
mailbox verification less certain.


5. RECOMMENDED LARGE-SCALE WORKFLOW
-----------------------------------

The first verification pass should not automatically become the final
deliverability decision.

Recommended pipeline:

RAW DATA
   |
   v
NORMALIZE / DEDUPE
   |
   v
SYNTAX / DOMAIN / MX CHECKS
   |
   v
SMTP VERIFICATION
   |
   +---- INVALID ----------> REMOVE / QUARANTINE
   |
   +---- ERROR / UNKNOWN --> HOLD FOR RETRY / REVIEW
   |
   +---- VALID ------------> SECOND-PASS VERIFICATION
   |
   +---- RISKY ------------> SECOND-PASS / REVIEW
                                  |
                                  v
                           FINAL WORKING SET


6. SECOND-PASS VERIFICATION
---------------------------

The VALID population from the first pass should be eligible for an additional
verification layer.

The second-pass system should be selected and benchmarked independently.

The benchmark should specifically test whether the second-pass system can
identify addresses that:

- accept an SMTP recipient verification request;
- subsequently fail delivery;
- behave as accept-all/catch-all environments; or
- otherwise produce misleading positive SMTP responses.

The same controlled test population should be used when comparing candidate
second-pass tools.

Do not select a second-pass tool solely because it reports a high percentage
of VALID results.


7. LARGE DATABASE STRATEGY
--------------------------

Do not immediately spend money verifying the entire database.

First-pass objective:

1. Normalize and deduplicate.
2. Apply inexpensive syntax and domain checks.
3. Perform MX checks.
4. Run SMTP verification.
5. Preserve VALID / INVALID / ERROR / UNKNOWN / RISKY separately.
6. Remove or quarantine clear INVALID records.
7. Preserve ERROR/UNKNOWN records for later handling.
8. Send the VALID/RISKY population through the second verification layer.
9. Build the final working dataset from the combined evidence.

Illustrative example ONLY:

LARGE RAW DATABASE
    |
    v
FIRST-PASS FILTERING
    |
    v
SMALLER WORKING POPULATION
    |
    v
SMTP VERIFICATION
    |
    v
VALID POPULATION
    |
    v
SECOND-PASS VERIFICATION
    |
    v
FINAL WORKING POPULATION

Any numbers used in planning examples are placeholders and must not be
interpreted as forecasts or actual database counts.


8. DATA PRESERVATION
--------------------

Never overwrite the original source dataset.

Every processed record should retain its original identifiers and source
fields.

Verification output should be written to separate files or tables.

At minimum preserve:

- original record ID / UID
- original email
- normalized email
- domain
- verification status
- verification reason
- verification timestamp
- verification engine/version

Original source data must remain recoverable.


9. BATCHING
-----------

Large-scale processing should use controlled batches.

Determine the maximum stable batch size empirically using progressively larger
tests.

A 100,000-record batch may be used as a conservative operational ceiling
after the environment has demonstrated stable performance.

After a large batch, use a controlled pause before beginning the next batch.

The pause is intended to reduce the risk of:

- DNS throttling
- SMTP connection instability
- local resource exhaustion
- remote-server rate limiting
- network/reputation issues

Do not assume that a larger batch is automatically faster or safer.


10. BENCHMARKING
----------------

Before moving to large-scale processing, record:

- total records processed
- processing duration
- records per minute
- VALID count
- INVALID count
- ERROR count
- UNKNOWN count
- RISKY count
- connection failures
- DNS failures
- timeout count
- incomplete output count

A batch is not considered successful merely because the process exits without
an error.

The output must also be complete and internally consistent.


11. ACCURACY EXPECTATIONS
-------------------------

Do not claim:

- 100% deliverability
- 100% mailbox existence
- zero bounce rate
- guaranteed delivery
- guaranteed mailbox-level accuracy

SMTP verification provides useful evidence but has known limitations.

Receiving servers can accept recipient commands while later rejecting or
otherwise failing delivery.

The purpose of the second verification layer is to reduce this uncertainty.


12. CURRENT CONTROLLED BENCHMARK
--------------------------------

Initial test:

50 records

First pass:

- VALID:    5
- INVALID: 34
- ERROR:   11

Catch-all rerun of the initial VALID population:

- VALID:    3
- INVALID:  2
- ERROR:    0

Observed benchmark false-positive VALID population:

3 records

Observed benchmark false-positive proportion:

6% of the original 50-record test population.

This figure is retained solely as a benchmark for evaluating subsequent
verification methods.


13. NEXT ACTIONS
----------------

A. Preserve the corrected source and benchmark artifacts.

B. Run one additional random 100-record operational test using catch-all
   checking.

C. Measure runtime, throughput, result distribution, connection failures,
   and output completeness.

D. Investigate and benchmark a second-pass verification method specifically
   against the controlled false-VALID cases.

E. Once the second-pass method has been benchmarked, apply the two-stage
   workflow to larger datasets.

F. Keep original source data untouched and merge only verified results into
   a separate working dataset.

G. Establish the maximum stable production batch size from measured
   performance rather than assuming one in advance.


CURRENT STATUS
--------------

CLI flag collision: FIXED

Corrected executable: BUILT AND TESTED

50-record first pass: COMPLETE

- 34 INVALID
- 11 ERROR
- 5 VALID

Catch-all rerun of the 5 VALID:

- 2 INVALID
- 3 VALID

Observed residual false-VALID benchmark:

3 / 50 = 6%

Next technical test:

RANDOM 100-RECORD OPERATIONAL TEST

The 6% figure is a benchmark observation, not a claimed universal error rate.
