# Experiment Planning


<!-- vim-markdown-toc GFM -->

* [Passive Testing](#passive-testing)
    * [1 Parmeters: CPU Increase Scenario](#1-parmeters-cpu-increase-scenario)
    * [Timing Window Tolerance](#timing-window-tolerance)
    * [Thread Pinning Effectiveness](#thread-pinning-effectiveness)
    * [Bandwidth Scaling](#bandwidth-scaling)
    * [PUSCH Scheduling Aggressiveness](#pusch-scheduling-aggressiveness)
* [Active Testing](#active-testing)
    * [PTP Grandmaster Failure Recovery](#ptp-grandmaster-failure-recovery)
    * [Fronthaul Link Interruption](#fronthaul-link-interruption)
    * [QOS Protection Under Memory Preassure](#qos-protection-under-memory-preassure)
    * [CPU Core Offline](#cpu-core-offline)

<!-- vim-markdown-toc -->

```
        ┌──────────────┐              ┌──────────────┐              ┌──────────────┐
        │              │              │              │              │              │
        │    INPUT     │─────────────▶│   TESTING    │─────────────▶│   OUTPUT     │
        │ PARAMETERS   │              │   PROCESS    │              │   METRICS    │
        │              │              │              │              │              │
        └──────────────┘              └──────────────┘              └──────────────┘
              ▲                             │                               │
              │                             │                               │
              │                             ▼                               │
              │                      ┌──────────────┐                       │
              │                      │   PASSIVE    │                       │
              │                      │    TESTS     │                       │
              │                      └──────────────┘                       │
              │                             │                               │
              │                             ▼                               │
              │                      ┌──────────────┐                       │
              │                      │   ACTIVE     │                       │
              │                      │    TESTS     │                       │
              │                      └──────────────┘                       │
              │                                                             │
              │                                                             │
              │                      ┌──────────────┐                       │
              │                      │   ANALYSIS   │                       │
              └──────────────────────│   & UPDATE   │◀──────────────────────┘
                                     │              │
                                     └──────────────┘
```

> Assume:
> - 3 O-RU
> - 1 GNB Vendors
>  How many permutations of test can be performed automatically on End-to-end


# Passive Testing

> Passive testing means the environment will be triggered as it is.

## 1 Parmeters: CPU Increase Scenario

> Data Source: iperf3, top, perf stat -e context-switches

| Test ID | O-RU Vendor | CPU Isolated Cores | Runs | DL (Mbps) | UL (Mbps) | PTP RMS (ns) | CPU Platform (%) | CPU App (%) |
|---------|-------------|-------------------|------|-----------|-----------|--------------|------------------|-------------|
| T01     | LiteON      | 0-7 (8 cores)     | 10   |           |           |              |                  |             |
| T02     | LiteON      | 0-11 (12 cores)   | 10   |           |           |              |                  |             |
| T03     | LiteON      | 0-15 (16 cores)   | 10   |           |           |              |                  |             |
| T04     | Pegatron    | 0-7 (8 cores)     | 10   |           |           |              |                  |             |
| T05     | Pegatron    | 0-11 (12 cores)   | 10   |           |           |              |                  |             |
| T06     | Pegatron    | 0-15 (16 cores)   | 10   |           |           |              |                  |             |
| T07     | Jura        | 0-7 (8 cores)     | 10   |           |           |              |                  |             |
| T08     | Jura        | 0-11 (12 cores)   | 10   |           |           |              |                  |             |
| T09     | Jura        | 0-15 (16 cores)   | 10   |           |           |              |                  |             |


## Timing Window Tolerance

| Test ID | O-RU     | T1a_cp_dl (ns) | Ta4 (ns)   | Runs | DL (Mbps) | PTP RMS (ns) | FH Connected |
|---------|----------|----------------|------------|------|-----------|--------------|--------------|
| P28     | LiteON   | (285, 470)     | (110, 180) | 10   |           |              |              |
| P29     | LiteON   | (285, 550)     | (110, 280) | 10   |           |              |              |
| P30     | Pegatron | (285, 470)     | (110, 180) | 10   |           |              |              |
| P31     | Pegatron | (285, 550)     | (110, 280) | 10   |           |              |              |
| P32     | Jura     | (285, 470)     | (110, 180) | 10   |           |              |              |
| P33     | Jura     | (285, 550)     | (110, 280) | 10   |           |              |              |



## Thread Pinning Effectiveness
> Goal: Validate real benefits of explicit thread assignment benefits per vendor\
> Data Source: iperf3, perf stat, top

| Test ID | O-RU     | Thread Pool | CPU Manager | Runs | PUSCH Variance (%) | Thread Migration | L1 Cache Miss (%) | DL (Mbps) |
|---------|----------|-------------|-------------|------|--------------------|------------------|-------------------|-----------|
| P13     | LiteON   | auto        | none        | 10   |                    |                  |                   |           |
| P14     | LiteON   | auto        | static      | 10   |                    |                  |                   |           |
| P15     | LiteON   | 8-13        | static      | 10   |                    |                  |                   |           |
| P16     | Pegatron | auto        | none        | 10   |                    |                  |                   |           |
| P17     | Pegatron | auto        | static      | 10   |                    |                  |                   |           |
| P18     | Pegatron | 8-13        | static      | 10   |                    |                  |                   |           |
| P19     | Jura     | auto        | none        | 10   |                    |                  |                   |           |
| P20     | Jura     | auto        | static      | 10   |                    |                  |                   |           |
| P21     | Jura     | 8-13        | static      | 10   |                    |                  |                   |           |

## Bandwidth Scaling

> Compare different vendors performance on different bandwidth\
> Source: iperf3, top

| Test ID | O-RU     | Bandwidth | SCS    | Runs | DL (Mbps) | CPU App (%) |
|---------|----------|-----------|--------|------|-----------|-------------|
| P34     | LiteON   | 50 MHz    | 30 kHz | 10   |           |             |
| P35     | LiteON   | 100 MHz   | 30 kHz | 10   |           |             |
| P36     | Pegatron | 50 MHz    | 30 kHz | 10   |           |             |
| P37     | Pegatron | 100 MHz   | 30 kHz | 10   |           |             |
| P38     | Jura     | 50 MHz    | 30 kHz | 10   |           |             |
| P39     | Jura     | 100 MHz   | 30 kHz | 10   |           |             |
| P40     | LiteON   | 100 MHz   | 15 kHz | 10   |           |             |
| P41     | Pegatron | 100 MHz   | 15 kHz | 10   |           |             |
| P42     | Jura     | 100 MHz   | 15 kHz | 10   |           |             |

## PUSCH Scheduling Aggressiveness

> Compare different vendors on using differnt puschTargetCodeRate\
> Source: iperf3, top

| Test ID | O-RU     | puschTargetCodeRate | Runs | UL (Mbps) | CPU App (%) |
|---------|----------|---------------------|------|-----------|-------------|
| P43     | LiteON   | 193                 | 10   |           |             |
| P44     | LiteON   | 308                 | 10   |           |             |
| P45     | LiteON   | 517                 | 10   |           |             |
| P46     | Pegatron | 193                 | 10   |           |             |
| P47     | Pegatron | 308                 | 10   |           |             |
| P48     | Pegatron | 517                 | 10   |           |             |
| P49     | Jura     | 193                 | 10   |           |             |
| P50     | Jura     | 308                 | 10   |           |             |
| P51     | Jura     | 517                 | 10   |           |             |



# Active Testing

> Active testing means during testing observer will perform actions that can affect the testing result\
> i.e. to make sure qos class allocated by kubernets really protect the process, observer will trigger extra workload on the cluster to occupy 50% memory from the whole cluster.

## PTP Grandmaster Failure Recovery

> Stop ptp4l service on targeted cluster for N second

| Test ID | O-RU     | PTP Loss (s) | Runs | Time to Detect (s) | Time to Resync (s) | DL During (Mbps) | DL After (Mbps) |
|---------|----------|--------------|------|--------------------|--------------------|------------------|-----------------|
| A01     | LiteON   | 30           | 10   |                    |                    |                  |                 |
| A02     | Pegatron | 30           | 10   |                    |                    |                  |                 |
| A03     | Jura     | 30           | 10   |                    |                    |                  |                 |
| A04     | LiteON   | 60           | 10   |                    |                    |                  |                 |
| A05     | Pegatron | 60           | 10   |                    |                    |                  |                 |
| A06     | Jura     | 60           | 10   |                    |                    |                  |                 |

## Fronthaul Link Interruption

> Trigger link down of fronthaul interface

| Test ID | O-RU     | Link Down (s) | Runs | DL During (Mbps) | Time to Reconnect (s) | DL After (Mbps) | UE Still Attached |
|---------|----------|---------------|------|------------------|-----------------------|-----------------|-------------------|
| A07     | LiteON   | 10            | 10   |                  |                       |                 |                   |
| A08     | Pegatron | 10            | 10   |                  |                       |                 |                   |
| A09     | Jura     | 10            | 10   |                  |                       |                 |                   |
| A10     | LiteON   | 30            | 10   |                  |                       |                 |                   |
| A11     | Pegatron | 30            | 10   |                  |                       |                 |                   |
| A12     | Jura     | 30            | 10   |                  |                       |                 |                   |


## QOS Protection Under Memory Preassure

> Run memory stress test
> i.e. `stress-ng`
>  Watch what happened to service

| Test ID | O-RU     | QoS Class  | Memory Stress | Runs | DL Mean (Mbps) | DL Std Dev | Pod Restarts | OOM Events |
|---------|----------|------------|---------------|------|----------------|------------|--------------|------------|
| A13     | LiteON   | BestEffort | 50%           | 10   |                |            |              |            |
| A14     | LiteON   | Guaranteed | 50%           | 10   |                |            |              |            |
| A15     | Pegatron | BestEffort | 50%           | 10   |                |            |              |            |
| A16     | Pegatron | Guaranteed | 50%           | 10   |                |            |              |            |
| A17     | Jura     | BestEffort | 50%           | 10   |                |            |              |            |
| A18     | Jura     | Guaranteed | 50%           | 10   |                |            |              |            |
| A19     | LiteON   | BestEffort | 80%           | 10   |                |            |              |            |
| A20     | LiteON   | Guaranteed | 80%           | 10   |                |            |              |            |
| A21     | Pegatron | BestEffort | 80%           | 10   |                |            |              |            |
| A22     | Pegatron | Guaranteed | 80%           | 10   |                |            |              |            |
| A23     | Jura     | BestEffort | 80%           | 10   |                |            |              |            |
| A24     | Jura     | Guaranteed | 80%           | 10   |                |            |              |            |


## CPU Core Offline

> Toggle online/offline state of currently used CPU
> `echo 0 > /sys/devices/system/cpu/cpu12/online`

| Test ID | O-RU     | Cores Offline | Runs | DL Before (Mbps) | DL After (Mbps) | CPU App After (%) | Pod Status |
|---------|----------|---------------|------|------------------|-----------------|-------------------|------------|
| A25     | LiteON   | 1 core        | 10   |                  |                 |                   |            |
| A26     | LiteON   | 2 cores       | 10   |                  |                 |                   |            |
| A27     | Pegatron | 1 core        | 10   |                  |                 |                   |            |
| A28     | Pegatron | 2 cores       | 10   |                  |                 |                   |            |
| A29     | Jura     | 1 core        | 10   |                  |                 |                   |            |
| A30     | Jura     | 2 cores       | 10   |                  |                 |                   |            |
