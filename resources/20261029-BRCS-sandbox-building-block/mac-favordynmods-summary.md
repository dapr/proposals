# Sandbox latency summary

GATE PASS (sequential n=421 p99=17.79ms err=0.0% nonpool=15.8%; rate20 n=1795 p99=16.25ms err=0.0% nonpool=0.3% late=0.0%)

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
- sweep-rate160.csv: 4800 rows; rate rate 160 x 1 hosts: expected 4800, fired 4800, late 96 (2.0%), max lateness 111 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- sweep-rate20.csv: 600 rows; rate rate 20 x 1 hosts: expected 600, fired 600, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- sweep-rate40.csv: 1200 rows; rate rate 40 x 1 hosts: expected 1200, fired 1200, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- sweep-rate80.csv: 2400 rows; rate rate 80 x 1 hosts: expected 2400, fired 2400, late 9 (0.4%), max lateness 10 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0

WARNING: 47.5% of path (a) echo rows (2605 of 5484) have acquire_path != POOL; they are excluded from the gate

- rows: 12900
- non-200 rows: 5516 (excluded from percentiles, never dropped)
  - HTTP 429: 5516
- path (a) requests with acquire_path != POOL: 2605

## Latency (HTTP 200 rows, ms)

| path | workload | mode | rate | n | non-200 | acquire paths | total p50 | total p99 | client p50 | client p99 |
|---|---|---|---|---|---|---|---|---|---|---|
| a | echo | rate | 10 | 300 | 0 | POOL:300 | 3.05 | 6.42 | 3.41 | 7.08 |
| a | echo | rate | 20 | 1800 | 0 | POOL:1795 RESTORE:5 | 3.05 | 19.04 | 3.40 | 19.49 |
| a | echo | rate | 40 | 964 | 236 | POOL:136 RESTORE:828 | 1029.38 | 1349.70 | 1029.75 | 1350.39 |
| a | echo | rate | 80 | 995 | 1405 | POOL:120 RESTORE:875 | 1071.23 | 1314.80 | 1071.57 | 1315.42 |
| a | echo | rate | 160 | 925 | 3875 | POOL:107 RESTORE:818 | 1109.36 | 2225.24 | 1109.67 | 2225.55 |
| a | echo | sequential | 0 | 500 | 0 | POOL:421 RESTORE:79 | 6.16 | 192.00 | 6.50 | 192.40 |
| a | python | sequential | 0 | 200 | 0 | POOL:182 RESTORE:18 | 17.53 | 175.14 | 17.87 | 175.53 |
| a | session-import | sequential | 0 | 200 | 0 | -:200 | 53.36 | 64.17 | 53.66 | 64.49 |
| b | echo | sequential | 0 | 500 | 0 | -:500 | 1.22 | 2.03 | 1.41 | 2.20 |
| c | echo | sequential | 0 | 500 | 0 | RESTORE:500 | 97.41 | 127.65 | 97.69 | 127.93 |
| d | echo | sequential | 0 | 500 | 0 | BOOT:500 | 216.36 | 264.83 | 216.65 | 265.15 |

## Breakdown (HTTP 200 rows, median ms)

| path | workload | mode | rate | host | activate | exec | host other (host - activate - exec) | daprd (total - host) |
|---|---|---|---|---|---|---|---|---|
| a | echo | rate | 10 | 2.98 | 0.21 | 2.76 | 0.01 | 0.07 |
| a | echo | rate | 20 | 2.99 | 0.20 | 2.79 | 0.01 | 0.06 |
| a | echo | rate | 40 | 1029.30 | 1024.48 | 4.38 | 0.44 | 0.09 |
| a | echo | rate | 80 | 1071.15 | 1066.02 | 5.10 | 0.04 | 0.07 |
| a | echo | rate | 160 | 1109.32 | 1103.46 | 4.53 | 1.32 | 0.05 |
| a | echo | sequential | 0 | 6.12 | 0.11 | 4.20 | 1.81 | 0.04 |
| a | python | sequential | 0 | 17.47 | 0.11 | 16.57 | 0.79 | 0.06 |
| a | session-import | sequential | 0 | 53.29 | 0.00 | 53.28 | 0.01 | 0.06 |
| b | echo | sequential | 0 | 1.19 | 0.00 | 1.19 | 0.00 | 0.03 |
| c | echo | sequential | 0 | 97.38 | 94.48 | 2.78 | 0.13 | 0.03 |
| d | echo | sequential | 0 | 216.31 | 214.50 | 1.79 | 0.02 | 0.04 |
