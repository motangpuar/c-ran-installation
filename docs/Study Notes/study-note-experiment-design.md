# Experiment Planning


<!-- vim-markdown-toc GFM -->

* [Testing Query](#testing-query)
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
    * [Challenges in CI/CD for O-RAN Testing](#challenges-in-cicd-for-o-ran-testing)
    * [Lavoisier as Worker-RT-01](#lavoisier-as-worker-rt-01)
    * [Experiment Planning and Design](#experiment-planning-and-design)
        * [Research Contributions](#research-contributions)
        * [4.7 Experimental Test Matrix](#47-experimental-test-matrix)
        * [4.6 Contribution Validation Through Experimental Design](#46-contribution-validation-through-experimental-design)
            * [How These Experiments Demonstrate the Contributions?](#how-these-experiments-demonstrate-the-contributions)
            * [How Experiments Were Designed to Achieve These Goals?](#how-experiments-were-designed-to-achieve-these-goals)
        * [4.6 Clarifications](#46-clarifications)
            * [O-RU Selection](#o-ru-selection)
            * [SCS Configuration](#scs-configuration)
            * [PTP Loss Duration](#ptp-loss-duration)
            * [Link Down Duration](#link-down-duration)
            * [Memory Stress](#memory-stress)
            * [CPU Core Offline](#cpu-core-offline-1)

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

# Testing Query

1. User send testing template

```
{
    "test_id": "T01-T03",
    "node_ip": "192.168.8.74",
    "runs_per_variant": 1,
    "artifact_config": {
        "name": "oai-gnb-test",
        "repo_url": "https://github.com/motangpuar/ocloud-helm-templates.git",
        "chart_name": "oai-gnb-fhi-72"
    },
    "fixed_params": {
        "nodeSelector": {
            "kubernetes.io/hostname": "lavoisier"
        },
        "resources": {
            "define": true,
            "limits": {
                "nf": {
                    "memory": "16Gi",
                    "hugepages": "10Gi"
                }
            },
            "requests": {
                "nf": {
                    "memory": "16Gi",
                    "hugepages": "10Gi"
                }
            }
        },
        "config": {
            "amfhost": "open5gs-amf-ngap.5gs-cn"
        },
        "multus": {
            "ruInterface": {
                "create": true,
                "mtu": 9600
            }
        }
    },
    "test_params": {
        "cpu_cores": ["0-3", "0-5", "0-8"],
        "oru_vendor": ["LiteON"]
    }
}
```


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

## Challenges in CI/CD for O-RAN Testing

- **Infrastructure Dependencies**: O-RAN testing demands physical O-RU connectivity (LiteON, Pegatron, Jura), PTP-synchronized timing sources, and real-time kernel configurations. These cannot be replicated in cloud CI runners. The tests must run where the hardware exists.

- **End-to-End Integration Requirements**: O2 interface testing scenarios involving IMS resource queries, DMS deployment operations, and multi-vendor O-RU validation require local infrastructure access. Unlike unit tests that verify isolated components, O-RAN validation requires full stack integration from SMO through O-Cloud to RAN, spanning multiple vendor implementations. After build completion, the CI script needs access to local APIs—SMO, O-RU M-plane, or other devices to validate software on the end-to-end environment.


- **Vendor Heterogeneity**: Each O-RU vendor implements O-RAN specifications differently, requiring parameterized testing across vendor combinations. These performance characteristics vary significantly based on timing window tolerance, bandwidth scaling behavior, and fault recovery capabilities all of which demand physical hardware access.

- **Hybrid CI/CD Architecture**: GitHub Actions handles public repository builds and unit tests, while on-premise Jenkins executes integration tests requiring local facility access. This split introduces pipeline complexity and state management between two systems, but remains necessary for validating software against real O-RU hardware and timing-sensitive fronthaul protocols.

## Lavoisier as Worker-RT-01

- NIC Migrated already and have been tetsed, runs vanilla kubernetes
- Currently runs vanilla kubernetes, considering this server is shared with other student. Vanilla kubernetes can be setup in a way that kubernetes workload only exist as a service. Hence other student can still perform baremetal testing on the same server. Turning this server as StarlingX worker node or OKD's node will prevent sharing utilization with other student.

##  Experiment Planning and Design

### Research Contributions

This work contributes the following to O-RAN deployment practice:

1. **GitOps-Based Test Orchestration for O-RAN**: Demonstrates how NFO and FOCOM modules enable declarative, version-controlled test configuration versus manual deployment procedures. The GitOps-rApp automatically triggers O2-compliant deployment operations based on CI/CD pipeline events, enabling reproducible testing scenarios at scale.

2. **Multi-Vendor O-RU Validation Framework**: Systematic testing methodology across LiteON, Pegatron, and Jura O-RUs reveals significant performance heterogeneity. The framework characterizes vendor-specific timing sensitivities, bandwidth scaling behaviors, and configuration parameter interactions that would be impractical to discover through manual testing.

3. **Automated Resilience Testing via O2 Interface**: Programmatic fault injection (PTP loss, fronthaul link failures, resource stress) demonstrates automated verification of recovery mechanisms. The system quantifies mean-time-to-recovery (MTTR) and validates that declarative GitOps approaches maintain service continuity under infrastructure faults.

4. **ETSI NFV-Compliant O2 Implementation**: NFO and FOCOM modules implement ETSI SOL003 specifications, providing O2-DMS deployment management (Create/Patch/Remove operations) and O2-IMS resource queries.


### 4.7 Experimental Test Matrix

| Experiment                | Category   | Goal                                 | Fault/Remarks                         |
| ------------              | ---------- | ------                               | ---------------                       |
| **P1: CPU Scaling**       | Passive    | Determine minimum CPU allocation     | No interference                       |
| **P2: Timing Windows**    | Passive    | Validate timing margin requirements  | Conservative vs relaxed T1a/Ta4       |
| **P3: Thread Pinning**    | Passive    | Quantify CPU affinity benefits       | Auto vs explicit pinning              |
| **P4: Bandwidth Scaling** | Passive    | Test linear scaling assumptions      | 50MHz, 100MHz with 15/30kHz SCS       |
| **P5: PUSCH Scheduling**  | Passive    | Compare uplink code rate sensitivity | Conservative to aggressive scheduling |
| **A1: PTP Failure**       | Active     | Measure resynchronization behavior   | Stop ptp4l service                    |
| **A2: Link Interruption** | Active     | Validate O-RU reconnection           | Fronthaul interface down              |
| **A3: Memory Pressure**   | Active     | Test QoS resource isolation          | stress-ng 50%/80% memory              |
| **A4: CPU Offline**       | Active     | Characterize capacity degradation    | Disable 1-2 isolated cores (Non L1)            |



### 4.6 Contribution Validation Through Experimental Design

#### How These Experiments Demonstrate the Contributions?

- The passive experiments demonstrate **Contribution #1** (GitOps orchestration) because executing 36 configuration permutations manually would require weeks of repetitive deployment work as each parameter change demands complete gNodeB teardown and redeployment. Automated testing proves the NFO/FOCOM framework enables systematic exploration at operational scale.

- All experiments validate **Contribution #2** (multi-vendor framework) by testing identical scenarios across LiteON, Pegatron, and Jura. Performance differences emerge directly from comparative measurements rather than vendor specifications. This reveals implementation heterogeneity that operators face in real deployments.

- The active experiments prove **Contribution #3** (automated resilience) by measuring system behavior under controlled failures. Without automation, operators manually detect faults and execute recovery procedures. These tests quantify detection time, recovery duration, and service continuity.

- Both experiment categories exercise **Contribution #4** (ETSI compliance) because every test scenario triggers NFO deployment operations (Create/Instantiate/Patch/Terminate). The framework executes hundreds of O2-DMS transactions across all tests, validating that the implementation handles real operational workflows beyond simple proof-of-concept demonstrations.

#### How Experiments Were Designed to Achieve These Goals?

- Passive experiments isolate single parameters through controlled variation. P1 varies only CPU allocation while holding timing, bandwidth, and scheduling constant. P2 varies only timing windows. This factorial approach establishes which parameters significantly impact performance and whether effects differ by vendor. Multiple runs per configuration provide statistical confidence and detect performance variance.

- Active experiments follow a before-during-after measurement pattern. Baseline metrics establish normal operation. Fault injection creates controlled disruption. Recovery monitoring captures system response. This three-phase design quantifies resilience through specific metrics: time-to-detect measures monitoring effectiveness, time-to-recover measures automation speed, throughput-during-fault measures degradation magnitude.

- The multi-vendor structure embeds comparison directly into every experiment. Testing LiteON, Pegatron, and Jura under identical conditions produces parallel datasets. Vendor differences become evident through side-by-side metric comparison rather than requiring separate test campaigns. This design validates that the framework handles heterogeneous implementations systematically.

- All experiments use declarative test templates processed by the GitOps-rApp. Each test case specifies desired configuration as code, triggering automated NFO operations. This design ensures every experiment execution exercises the CI/CD integration, O2 interface calls.

### 4.6 Clarifications

#### O-RU Selection
- LiteON (C3 connection)
- Jura (C3 connection)
- Pegatron (C3 connection)

#### SCS Configuration
- I can test both 15 kHz and 30 kHz tested for 100 MHz bandwidth to compare subcarrier spacing effects.

#### PTP Loss Duration
- 30s tests brief grandmaster switchover. 60s tests prolonged timing drift. Yes this experiment try to see either the GNB and O-RU can still deliver ther service after loss connection to PTP master.

#### Link Down Duration

- 10s tests brief interruption. 30s tests extended outage requiring RRC release. Validates recovery orchestration, not maximum tolerance discovery.

#### Memory Stress
- Side load stress-ng: `stress-ng --vm 4 --vm-bytes X% --timeout 60s`  into the worker node when GNB Runs
- Tests target Guaranteed QoS pods (16Gi memory, 16 CPU). Qos class mentioned refering to k8s qos, basically full allocation means guaranteed, below full means best-effort or burstable (depends on your case) more [official explanation here](https://kubernetes.io/docs/concepts/workloads/pods/pod-qos/)

#### CPU Core Offline

- OAI require minimum 8 core, with 5 of them will be allocated for crucial thread that cant be bothered (l1_tx, l1_rx, system_core, io_core, worker_core). Extra ones are for thread_pool (these are the ones we can try to disable)
- Disable in this context is by removing the cpu allocation on the pod by updating the `/sys/fs/cgroup/cpuset/cpuset.cpus` inside the container

