# Sandbox latency summary

GATE PASS (sequential n=482 p99=16.44ms err=0.0% nonpool=3.6%; rate20 n=3600 p99=11.36ms err=0.0% nonpool=0.0% late=0.0%)

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
- path-a-rate20.csv: 2400 rows; rate rate 20 x 2 hosts: expected 2400, fired 2400, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- path-a-seq.csv: 500 rows; sequential rate 0 x 0 hosts: expected 500, fired 500, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- path-a-session-import-seq.csv: 200 rows; sequential rate 0 x 0 hosts: expected 200, fired 200, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- path-b-seq.csv: 500 rows; sequential rate 0 x 0 hosts: expected 500, fired 500, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- path-c-seq.csv: 500 rows; sequential rate 0 x 0 hosts: expected 500, fired 500, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- path-d-seq.csv: 500 rows; sequential rate 0 x 0 hosts: expected 500, fired 500, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- sweep-rate10.csv: 600 rows; rate rate 10 x 2 hosts: expected 600, fired 600, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- sweep-rate160.csv: 9600 rows; rate rate 160 x 2 hosts: expected 9600, fired 9600, late 1171 (12.2%), max lateness 5 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- sweep-rate20.csv: 1200 rows; rate rate 20 x 2 hosts: expected 1200, fired 1200, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- sweep-rate40.csv: 2400 rows; rate rate 40 x 2 hosts: expected 2400, fired 2400, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- sweep-rate80.csv: 4800 rows; rate rate 80 x 2 hosts: expected 4800, fired 4800, late 6 (0.1%), max lateness 4 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0

WARNING: 39.6% of path (a) echo rows (4724 of 11915) have acquire_path != POOL; they are excluded from the gate

- rows: 23400
- non-200 rows: 9585 (excluded from percentiles, never dropped)
  - HTTP 429: 9585
- path (a) requests with acquire_path != POOL: 4724

## Latency (HTTP 200 rows, ms)

| path | workload | mode | rate | n | non-200 | acquire paths | total p50 | total p99 | client p50 | client p99 |
|---|---|---|---|---|---|---|---|---|---|---|
| a | echo | rate | 10 | 600 | 0 | POOL:600 | 6.59 | 7.90 | 6.92 | 8.29 |
| a | echo | rate | 20 | 3600 | 0 | POOL:3600 | 7.18 | 11.36 | 7.49 | 11.68 |
| a | echo | rate | 40 | 2400 | 0 | POOL:1928 RESTORE:472 | 10.73 | 208.60 | 11.16 | 209.11 |
| a | echo | rate | 80 | 2466 | 2334 | POOL:302 RESTORE:2164 | 822.77 | 1033.37 | 823.10 | 1033.63 |
| a | echo | rate | 160 | 2349 | 7251 | POOL:279 RESTORE:2070 | 902.37 | 1147.92 | 902.74 | 1148.18 |
| a | echo | sequential | 0 | 500 | 0 | POOL:482 RESTORE:18 | 8.30 | 103.87 | 8.68 | 104.30 |
| a | python | sequential | 0 | 200 | 0 | POOL:200 | 35.86 | 39.52 | 36.19 | 39.85 |
| a | session-import | sequential | 0 | 200 | 0 | -:200 | 66.25 | 72.27 | 66.53 | 72.61 |
| b | echo | sequential | 0 | 500 | 0 | -:500 | 1.64 | 2.37 | 1.80 | 2.54 |
| c | echo | sequential | 0 | 500 | 0 | RESTORE:500 | 81.98 | 92.16 | 82.28 | 92.44 |
| d | echo | sequential | 0 | 500 | 0 | BOOT:500 | 227.84 | 244.74 | 228.13 | 245.04 |

## Breakdown (HTTP 200 rows, median ms)

| path | workload | mode | rate | host | activate | exec | host other (host - activate - exec) | daprd (total - host) |
|---|---|---|---|---|---|---|---|---|
| a | echo | rate | 10 | 6.05 | 0.33 | 5.71 | 0.02 | 0.54 |
| a | echo | rate | 20 | 6.64 | 0.32 | 6.29 | 0.03 | 0.54 |
| a | echo | rate | 40 | 9.97 | 0.56 | 7.87 | 1.54 | 0.76 |
| a | echo | rate | 80 | 821.95 | 812.51 | 8.50 | 0.94 | 0.83 |
| a | echo | rate | 160 | 901.65 | 891.92 | 8.76 | 0.97 | 0.71 |
| a | echo | sequential | 0 | 7.67 | 0.36 | 7.19 | 0.12 | 0.62 |
| a | python | sequential | 0 | 35.32 | 0.29 | 34.99 | 0.04 | 0.54 |
| a | session-import | sequential | 0 | 65.81 | 0.00 | 65.80 | 0.01 | 0.44 |
| b | echo | sequential | 0 | 1.39 | 0.00 | 1.39 | 0.00 | 0.25 |
| c | echo | sequential | 0 | 81.47 | 75.84 | 5.58 | 0.05 | 0.51 |
| d | echo | sequential | 0 | 227.36 | 225.57 | 1.81 | 0.00 | 0.47 |
