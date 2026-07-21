# FreeRTOS Mastery Path

A free, structured, module-wise FreeRTOS training program — entry level to
master level, covering real-time operating system concepts with real
compilable C/C++ code in every module and a hands-on project at the end of
each level. Level 1 uses the ESP32 Arduino framework (which ships FreeRTOS
built in), so every example runs in the free
[Wokwi online simulator](https://wokwi.com/) — no hardware required to start.
The FreeRTOS POSIX/Linux simulation port and QEMU are covered as pure-C
alternatives.

**Live site:** https://sigilipelli.github.io/freertos-mastery-path/

## Contents

- **Level 1 · Entry** — why an RTOS, creating tasks, the scheduler, queues, semaphores & mutexes, software timers, task notifications & event groups, interrupts, memory & debugging, capstone (multi-task smart sensor device)
- **Level 2 · Intermediate** — stream & message buffers, static allocation, tickless idle & low power, ESP-IDF FreeRTOS specifics, POSIX/Linux port, queue sets, FreeRTOSConfig.h, gatekeeper patterns, watchdogs
- **Level 3 · Advanced** — kernel internals (lists, ready lists), context switching, SMP scheduling, MPU support, tracing (Tracealyzer/SystemView), porting, ISR latency design, performance tuning, FreeRTOS-Plus-TCP
- **Level 4 · Master** — safety-certified FreeRTOS (SAFERTOS), formal timing analysis (RMA/WCET), production firmware architecture, certification workflows, TrustZone, OTA, fault tolerance, multi-core AMP

## Local development

```bash
python3 -m venv .venv
.venv/bin/pip install mkdocs-material
.venv/bin/python -m mkdocs serve
```

## Related

- [Embedded Systems Mastery Path](https://sigilipelli.github.io/embedded-mastery-path/)
- [Python Mastery Path](https://sigilipelli.github.io/python-mastery-path/)
- [Java Mastery Path](https://sigilipelli.github.io/java-mastery-path/)
- [JavaScript Mastery Path](https://sigilipelli.github.io/javascript-mastery-path/)
- [Shell Mastery Path](https://sigilipelli.github.io/shell-mastery-path/)
- [C Mastery Path](https://sigilipelli.github.io/c-mastery-path/)
- [C++ Mastery Path](https://sigilipelli.github.io/cpp-mastery-path/)
- [Go Mastery Path](https://sigilipelli.github.io/go-mastery-path/)
- [SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/)
- [Rust Mastery Path](https://sigilipelli.github.io/rust-mastery-path/)
- [TypeScript Mastery Path](https://sigilipelli.github.io/typescript-mastery-path/)
- [Ruby Mastery Path](https://sigilipelli.github.io/ruby-mastery-path/)
- [PHP Mastery Path](https://sigilipelli.github.io/php-mastery-path/)
- [Kotlin Mastery Path](https://sigilipelli.github.io/kotlin-mastery-path/)
- [Swift Mastery Path](https://sigilipelli.github.io/swift-mastery-path/)
- [Scala Mastery Path](https://sigilipelli.github.io/scala-mastery-path/)
- [R Mastery Path](https://sigilipelli.github.io/r-mastery-path/)
- [Dart Mastery Path](https://sigilipelli.github.io/dart-mastery-path/)
- [PowerShell Mastery Path](https://sigilipelli.github.io/powershell-mastery-path/)
- [AI & Machine Learning Mastery Path](https://sigilipelli.github.io/ai-ml-mastery-path/)
- [LLM Development Mastery Path](https://sigilipelli.github.io/llm-dev-mastery-path/)
- [RAG Mastery Path](https://sigilipelli.github.io/rag-mastery-path/)
- [AWS Mastery Path](https://sigilipelli.github.io/aws-mastery-path/)
- [Azure Mastery Path](https://sigilipelli.github.io/azure-mastery-path/)
- [Google Cloud Mastery Path](https://sigilipelli.github.io/gcp-mastery-path/)
- [IBM Cloud Mastery Path](https://sigilipelli.github.io/ibm-cloud-mastery-path/)
- [Adobe Creative Cloud Mastery Path](https://sigilipelli.github.io/adobe-mastery-path/)
