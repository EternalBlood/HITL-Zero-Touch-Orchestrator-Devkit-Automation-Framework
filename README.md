# ⚙️ HITL Zero-Touch Orchestrator: Devkit Automation Framework

![.NET](https://img.shields.io/badge/.NET-5C2D91?style=for-the-badge&logo=.net&logoColor=white)
![C#](https://img.shields.io/badge/C%23-239120?style=for-the-badge&logo=c-sharp&logoColor=white)
![Asynchronous Architecture](https://img.shields.io/badge/Architecture-Async_Multi--Threading-blue?style=for-the-badge)
![AI Workflow Ready](https://img.shields.io/badge/AI_Ready-Pipeline_Orchestration-FF9900?style=for-the-badge&logo=openai&logoColor=white)

> ⚠️ LEGAL DISCLAIMER
> Due to strict Non-Disclosure Agreements (NDAs) regarding proprietary first-party SDKs (Sony, Microsoft, Nintendo) and unreleased AAA intellectual property, the true source code for this project cannot be made public at all. 
> 
> This repository serves as a System Architecture Showcase. Its purpose is to demonstrate enterprise-grade engineering practices, asynchronous workflow design, Hardware-in-the-Loop (HITL) orchestration, and readiness for AI-assisted QA pipelines.

---

## 📖 1. The Engineering Thesis

In high-volume AAA game testing, manual console provisioning is a massive bottleneck. QA engineers lose countless hours dealing with SDK command-line interfaces, network timeouts, ghost processes blocking disk I/O, and unpredictable hardware states. 

This framework treats bare-metal game consoles as Finite State Machines (FSM). It abstracts highly complex, multi-step CLI routines into single-click, mathematically predictable state transitions. By injecting CI/CD deployment principles locally, it reduces devkit configuration time by over 95% while eliminating human error.

---

## 🏗️ 2. Architectural Highlights & Solutions

Building a robust bridge between a Windows host and proprietary devkits introduces severe multi-threading and I/O challenges. This tool was built to survive the chaos of physical QA labs.

### 🛡️ Cascading Abort Protocol (The "Zero-Zombie" Matrix)
Aborting a massive 80GB build extraction or an SDK payload mid-flight traditionally leaves "zombie" threads consuming RAM and locking files. 
* The Solution: A strictly managed CancellationTokenSource matrix. Triggering a global abort immediately broadcasts a cancellation exception through the entire logical tree—halting HttpClient streams, nested Task.Delay loops, and OS-level SDK executions in under 1 second, restoring host stability with zero memory leaks.

### 🚦 Pre-Flight State Synchronization
Sending format or install commands while a target console is silently running a background process causes kernel-level access violations.
* The Solution: The framework queries the target's active memory pool before executing destructive commands. If a process is detected, it forces a safe RAM dump (suspend) followed by a strict OS terminate, guaranteeing 100% clean disk I/O.

### 💾 Safe Asynchronous Unpacking
Decompressing large .xvc or .pkg builds asynchronously can silently corrupt host hard drives if space runs out mid-extraction.
* The Solution: An early HTTP Header inspection module calculates the required decompression footprint (3x multiplier of the ContentLength) and pre-validates host disk availability before opening any data streams.

---

## 💻 3. Code Showcase: Resilient Hardware Polling

To demonstrate the underlying C# architecture without breaching NDAs, below is a sanitized snippet of the Hardware Orchestrator module. 

This specific routine ensures target hardware is physically awake, responsive at the kernel level, and accessible via TCP before injecting deployments. It showcases advanced System.Threading concepts, including Linked Tokens, Competitive Asynchronous Tasks, and Fire-and-Forget Re-injections to handle unstable local networks.

<details>
<summary><b>👨‍💻 Click to expand C# Snippet (HardwareOrchestrator.cs)</b></summary>

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
namespace QAMasterSuite.Core.Orchestration
{
    public class HardwareOrchestrator
    {
        private readonly ISdkCommandBridge _sdkBridge;
        private readonly INetworkTelemetry _networkScanner;

        public HardwareOrchestrator(ISdkCommandBridge sdkBridge, INetworkTelemetry networkScanner)
        {
            _sdkBridge = sdkBridge;
            _networkScanner = networkScanner;
        }

        /// <summary>
        /// Ensures target hardware is physically awake and accessible before deployment.
        /// </summary>
        public async Task<bool> EnsureHardwareIsReadyAsync(string targetIp, string macAddress, CancellationToken masterToken)
        {
            // 1. FAST-PATH: Check via lightweight TCP Ping
            bool isTcpOpen = await _networkScanner.PingPortAsync(targetIp, targetPort: 11443, timeoutMs: 1000);
            masterToken.ThrowIfCancellationRequested();

            if (isTcpOpen)
            {
                bool isKernelReady = await CheckKernelReadyPulseAsync(targetIp, masterToken);
                masterToken.ThrowIfCancellationRequested();

                if (isKernelReady) return true;
            }

            // 2. WAKE-ON-LAN (WOL) INJECTION
            if (!string.IsNullOrEmpty(macAddress))
            {
                using (var wolCts = new CancellationTokenSource(20000))
                using (var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(wolCts.Token, masterToken))
                {
                    await _sdkBridge.ExecuteTargetCommandAsync(SdkCommandType.WakeOnLan, targetIp, macAddress, linkedCts.Token);
                }
                await Task.Delay(2000, masterToken); 
            }

            // 3. PERSISTENT HANDSHAKE LOOP (The "Defibrillator")
            int handshakeResult = -1;
            using (var timeoutCts = new CancellationTokenSource(TimeSpan.FromMinutes(6)))
            using (var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(timeoutCts.Token, masterToken))
            {
                var handshakeTask = _sdkBridge.ExecuteTargetCommandAsync(SdkCommandType.EstablishPersistentConnection, targetIp, string.Empty, linkedCts.Token);

                int timeRemainingSeconds = 360;
                int wolSpamCounter = 0;

                while (!handshakeTask.IsCompleted && timeRemainingSeconds > 0)
                {
                    masterToken.ThrowIfCancellationRequested();

                    // FIRE-AND-FORGET RE-INJECTION: Safely re-sends packets in unstable network conditions
                    if (wolSpamCounter >= 15 && !string.IsNullOrEmpty(macAddress))
                    {
                        wolSpamCounter = 0;
                        _ = Task.Run(async () =>
                        {
                            using (var spamCts = new CancellationTokenSource(20000))
                            using (var linkedSpamCts = CancellationTokenSource.CreateLinkedTokenSource(spamCts.Token, masterToken))
                            {
                                await _sdkBridge.ExecuteTargetCommandAsync(SdkCommandType.WakeOnLan, targetIp, macAddress, linkedSpamCts.Token);
                            }
                        }, masterToken);
                    }
                    wolSpamCounter++;

                    var delayTask = Task.Delay(1000, masterToken);
                    await Task.WhenAny(handshakeTask, delayTask);

                    if (handshakeTask.IsCompleted) break;
                    timeRemainingSeconds--;
                }
                handshakeResult = await handshakeTask;
            }

            if (handshakeResult != 0) return false;

            // 4. STABILIZATION VERIFICATION
            for (int i = 0; i < 10; i++)
            {
                masterToken.ThrowIfCancellationRequested();
                if (await CheckKernelReadyPulseAsync(targetIp, masterToken)) return true;
                await Task.Delay(3000, masterToken);
            }

            return false;
        }
        private async Task<bool> CheckKernelReadyPulseAsync(string targetIp, CancellationToken token)
        {
            int exitCode = -1;
            using (var pulseCts = new CancellationTokenSource(3000))
            using (var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(pulseCts.Token, token))
            {
                exitCode = await _sdkBridge.ExecuteTargetCommandAsync(SdkCommandType.QuerySystemReadyTime, targetIp, string.Empty, linkedCts.Token);
            }
            return exitCode == 0;
        }
    }
}
```
</details>

---

## 🚀 4. The Road Ahead: AI-Powered Orchestration
While the current framework excels at deterministic deployment and SDK abstraction, the architectural foundation was built specifically to support Generative AI workflows for game testing.

Upcoming Iterations:
1.  AI Telemetry Observers: Piping the orchestrator's real-time `stdout` wrapper directly into LLM endpoints to autonomously flag abnormal hardware temperatures, memory spikes, or failed network handshakes during overnight deployment batches.
2.  Smart Test-Suite Execution: Allowing AI agents to trigger specific test builds and regression scripts based on Jira Webhooks, completely automating the build-verification pipeline across a fleet of physical devkits.
3.  Automated Environment Reporting: Upon detecting a crash, the framework will package the FSM state, engine logs, and hardware telemetry into a contextualized, LLM-filtered bug report, drastically reducing manual data entry for QA Leads.

---

## 👨‍💻 Author
Víctor Espíndola
* SDET | AI-Assisted QA Automation & Console Tooling Engineer
* Specializing in Hardware-in-the-Loop Orchestration & Intelligent QA Pipelines.
* [LinkedIn Profile](https://www.linkedin.com/in/v%C3%ADctor-esp%C3%ADndola-a8332140a)

*"Bugs are not just defects; they are logical contradictions in the state machine."*
