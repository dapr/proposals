# Sandbox latency summary

GATE PASS (sequential n=500 p99=41.25ms err=0.0% nonpool=0.0%; rate20 n=1800 p99=42.41ms err=0.0% nonpool=0.0% late=0.0%)

## Measurement basis

The gate metric is the server-side timing.totalUs, measured by the caller's daprd.
client_us is the harness's wall clock around the HTTP request. It starts at the actual
send time, not at the scheduled tick time, so client-side queueing (an overloaded
harness) does not show up in either figure. It shows up only as late ticks, which is
why the gate also fails on more than 1% late ticks in rate mode.
A 200 response with timedOut=true is recorded as status 504; a transport error or
request timeout is recorded as status 0. Both count as errors in the gate.

## Input files

- path-a-python-seq.csv: 200 rows; sequential rate 0 x 0 hosts: expected 200, fired 200, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- path-a-rate20.csv: 1200 rows; rate rate 20 x 1 hosts: expected 1200, fired 1200, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- path-a-seq.csv: 500 rows; sequential rate 0 x 0 hosts: expected 500, fired 500, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- path-a-session-import-seq.csv: 200 rows; sequential rate 0 x 0 hosts: expected 200, fired 200, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- path-b-seq.csv: 500 rows; sequential rate 0 x 0 hosts: expected 500, fired 500, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- path-c-seq.csv: 500 rows; sequential rate 0 x 0 hosts: expected 500, fired 500, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- path-d-seq.csv: 500 rows; sequential rate 0 x 0 hosts: expected 500, fired 500, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- sweep-rate10.csv: 300 rows; rate rate 10 x 1 hosts: expected 300, fired 300, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- sweep-rate160.csv: 4800 rows; rate rate 160 x 1 hosts: expected 4800, fired 4800, late 98 (2.0%), max lateness 12 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- sweep-rate20.csv: 600 rows; rate rate 20 x 1 hosts: expected 600, fired 600, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- sweep-rate40.csv: 1200 rows; rate rate 40 x 1 hosts: expected 1200, fired 1200, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- sweep-rate80.csv: 2400 rows; rate rate 80 x 1 hosts: expected 2400, fired 2400, late 5 (0.2%), max lateness 11 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0

WARNING: 46.6% of path (a) echo rows (2623 of 5627) have acquire_path != POOL; they are excluded from the gate

- rows: 12900
- non-200 rows: 5373 (excluded from percentiles, never dropped)
  - HTTP 429: 5373
- path (a) requests with acquire_path != POOL: 2623

## Latency (HTTP 200 rows, ms)

| path | workload | mode | rate | n | non-200 | acquire paths | total p50 | total p99 | client p50 | client p99 |
|---|---|---|---|---|---|---|---|---|---|---|
| a | echo | rate | 10 | 300 | 0 | POOL:300 | 34.68 | 41.88 | 35.04 | 42.17 |
| a | echo | rate | 20 | 1800 | 0 | POOL:1800 | 35.10 | 42.41 | 35.43 | 42.71 |
| a | echo | rate | 40 | 999 | 201 | POOL:153 RESTORE:846 | 1005.00 | 1177.28 | 1005.36 | 1177.56 |
| a | echo | rate | 80 | 1001 | 1399 | POOL:125 RESTORE:876 | 1038.49 | 1549.46 | 1038.90 | 1549.68 |
| a | echo | rate | 160 | 1027 | 3773 | POOL:126 RESTORE:901 | 1055.32 | 1247.03 | 1055.53 | 1247.32 |
| a | echo | sequential | 0 | 500 | 0 | POOL:500 | 34.05 | 41.25 | 34.28 | 41.45 |
| a | python | sequential | 0 | 200 | 0 | POOL:200 | 42.47 | 48.77 | 42.70 | 49.02 |
| a | session-import | sequential | 0 | 200 | 0 | -:200 | 40.43 | 60.65 | 40.69 | 61.09 |
| b | echo | sequential | 0 | 500 | 0 | -:500 | 0.78 | 22.03 | 0.89 | 22.20 |
| c | echo | sequential | 0 | 500 | 0 | RESTORE:500 | 93.12 | 111.71 | 93.37 | 111.97 |
| d | echo | sequential | 0 | 500 | 0 | BOOT:500 | 187.61 | 209.69 | 187.90 | 209.97 |

## Breakdown (HTTP 200 rows, median ms)

| path | workload | mode | rate | host | activate | exec | host other (host - activate - exec) | daprd (total - host) |
|---|---|---|---|---|---|---|---|---|
| a | echo | rate | 10 | 34.63 | 0.15 | 34.48 | 0.01 | 0.05 |
| a | echo | rate | 20 | 35.05 | 0.14 | 34.76 | 0.15 | 0.05 |
| a | echo | rate | 40 | 1004.95 | 968.12 | 35.34 | 1.49 | 0.05 |
| a | echo | rate | 80 | 1038.45 | 1002.49 | 35.34 | 0.62 | 0.04 |
| a | echo | rate | 160 | 1055.29 | 1018.33 | 34.90 | 2.06 | 0.03 |
| a | echo | sequential | 0 | 34.02 | 0.09 | 33.76 | 0.18 | 0.03 |
| a | python | sequential | 0 | 42.44 | 0.09 | 42.24 | 0.11 | 0.03 |
| a | session-import | sequential | 0 | 40.39 | 0.00 | 40.38 | 0.00 | 0.04 |
| b | echo | sequential | 0 | 0.77 | 0.00 | 0.77 | 0.00 | 0.01 |
| c | echo | sequential | 0 | 93.09 | 63.64 | 29.29 | 0.17 | 0.03 |
| d | echo | sequential | 0 | 187.57 | 154.39 | 33.96 | 0.00 | 0.03 |
