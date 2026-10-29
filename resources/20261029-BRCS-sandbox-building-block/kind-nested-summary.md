# Sandbox latency summary

GATE FAIL (sequential n=500 p99=886.88ms err=0.0% nonpool=0.0%; rate20 n=4 p99=25064.91ms err=98.9% nonpool=89.5% late=52.8%): sequential p99 886.88ms >= 100ms; rate20 p99 25064.91ms >= 100ms; rate20 err 98.9% > 1% (non-200 incl. status 0); rate20 fewer than 100 POOL rows (4); rate20 late 52.8% > 1%

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
- path-a-rate20.csv: 2400 rows; rate rate 20 x 2 hosts: expected 2400, fired 2400, late 1255 (52.3%), max lateness 231 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- path-a-seq.csv: 500 rows; sequential rate 0 x 0 hosts: expected 500, fired 500, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- path-a-session-import-seq.csv: 200 rows; sequential rate 0 x 0 hosts: expected 200, fired 200, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- path-b-seq.csv: 500 rows; sequential rate 0 x 0 hosts: expected 500, fired 500, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- path-c-seq.csv: 100 rows; sequential rate 0 x 0 hosts: expected 100, fired 100, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- path-d-seq.csv: 100 rows; sequential rate 0 x 0 hosts: expected 100, fired 100, late 0 (0.0%), max lateness 0 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- sweep-rate10.csv: 600 rows; rate rate 10 x 2 hosts: expected 600, fired 600, late 68 (11.3%), max lateness 147 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- sweep-rate160.csv: 9600 rows; rate rate 160 x 2 hosts: expected 9600, fired 9600, late 9202 (95.9%), max lateness 979 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- sweep-rate20.csv: 1200 rows; rate rate 20 x 2 hosts: expected 1200, fired 1200, late 645 (53.8%), max lateness 244 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- sweep-rate40.csv: 2400 rows; rate rate 40 x 2 hosts: expected 2400, fired 2400, late 1868 (77.8%), max lateness 272 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0
- sweep-rate80.csv: 4800 rows; rate rate 80 x 2 hosts: expected 4800, fired 4800, late 4658 (97.0%), max lateness 1355 ms, request timeout 30000 ms, aborted requests 0, aborted cleanups 0, malformed timing values 0

WARNING: 14.1% of path (a) echo rows (85 of 604) have acquire_path != POOL; they are excluded from the gate

- rows: 22600
- non-200 rows: 20897 (excluded from percentiles, never dropped)
  - HTTP 0: 164
  - HTTP 429: 19943
  - HTTP 500: 789
  - HTTP 504: 1
- path (a) requests with acquire_path != POOL: 85

## Latency (HTTP 200 rows, ms)

| path | workload | mode | rate | n | non-200 | acquire paths | total p50 | total p99 | client p50 | client p99 |
|---|---|---|---|---|---|---|---|---|---|---|
| a | echo | rate | 10 | 14 | 586 | POOL:4 RESTORE:10 | 20338.52 | 29351.97 | 20349.01 | 29360.39 |
| a | echo | rate | 20 | 38 | 3562 | POOL:4 RESTORE:34 | 20113.83 | 29706.00 | 20121.22 | 29756.67 |
| a | echo | rate | 40 | 20 | 2380 | POOL:7 RESTORE:13 | 22664.35 | 28053.85 | 22710.59 | 28063.33 |
| a | echo | rate | 80 | 17 | 4783 | POOL:2 RESTORE:15 | 17121.11 | 23072.58 | 18084.28 | 23457.91 |
| a | echo | rate | 160 | 15 | 9585 | POOL:2 RESTORE:13 | 21513.83 | 28364.94 | 25528.15 | 29657.24 |
| a | echo | sequential | 0 | 500 | 0 | POOL:500 | 244.05 | 886.88 | 244.55 | 887.45 |
| a | python | sequential | 0 | 200 | 0 | POOL:200 | 923.64 | 8200.68 | 924.03 | 8201.35 |
| a | session-import | sequential | 0 | 199 | 1 | -:199 | 1072.32 | 4271.64 | 1072.81 | 4272.27 |
| b | echo | sequential | 0 | 500 | 0 | -:500 | 4.23 | 24.08 | 4.74 | 26.30 |
| c | echo | sequential | 0 | 100 | 0 | RESTORE:100 | 290.07 | 538.94 | 290.56 | 539.52 |
| d | echo | sequential | 0 | 100 | 0 | BOOT:100 | 2533.34 | 3205.87 | 2533.76 | 3206.35 |

## Breakdown (HTTP 200 rows, median ms)

| path | workload | mode | rate | host | activate | exec | host other (host - activate - exec) | daprd (total - host) |
|---|---|---|---|---|---|---|---|---|
| a | echo | rate | 10 | 20334.51 | 4416.94 | 8544.73 | 7372.84 | 4.01 |
| a | echo | rate | 20 | 20089.87 | 10329.93 | 9274.67 | 485.27 | 23.96 |
| a | echo | rate | 40 | 22223.22 | 2497.72 | 14215.07 | 5510.44 | 441.13 |
| a | echo | rate | 80 | 16418.03 | 10920.39 | 6339.34 | 0.00 | 703.08 |
| a | echo | rate | 160 | 19205.63 | 9325.23 | 9455.24 | 425.17 | 2308.19 |
| a | echo | sequential | 0 | 243.40 | 0.43 | 242.69 | 0.28 | 0.65 |
| a | python | sequential | 0 | 923.11 | 0.40 | 922.72 | 0.00 | 0.54 |
| a | session-import | sequential | 0 | 1071.77 | 0.00 | 1071.76 | 0.01 | 0.55 |
| b | echo | sequential | 0 | 3.65 | 0.00 | 3.65 | 0.01 | 0.58 |
| c | echo | sequential | 0 | 289.41 | 138.06 | 149.92 | 1.44 | 0.66 |
| d | echo | sequential | 0 | 2532.77 | 2476.24 | 51.50 | 5.04 | 0.56 |
