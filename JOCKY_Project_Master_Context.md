# JOCKY — Project Master Context

## Project Identity
- Project: JOCKY
- SIH Problem Statement: 26148
- Organization: NTRO
- Theme: Blockchain & Cybersecurity
- Team: GOD VALLEY
- College: Dr. M.G.R. Educational and Research Institute
- Project: Cross-platform computer and network forensic programming language + framework

## Core Problem
Forensic investigators need repeatable, systematic computer/network forensic analysis across different operating systems and machines. Existing forensic capabilities are powerful but fragmented across tools, operating systems, artifact types, and workflows.

JOCKY's core proposition:
> A programmable, cross-platform forensic framework that lets investigators define and execute systematic forensic workflows through a purpose-built domain-specific language.

JOCKY is NOT positioned as an immediate replacement for the entire forensic ecosystem.

## Core Architecture

```text
Investigator
     ↓
JOCKY Script
     ↓
JOCKY Compiler (C++ + LLVM)
     ↓
JOCKY AST / IR
     ↓
Forensic Execution Framework
     ↓
 ┌───────────────┬───────────────┐
 ↓               ↓
Windows         Ubuntu/Linux
Backend         Backend
 ↓               ↓
 └────── Evidence Collection ──┘
                 ↓
        Evidence Repository
                 ↓
      Hash + Chain of Custody
                 ↓
          Blockchain Ledger
                 ↓
        Forensic Analysis
        ├── Timeline
        ├── Evidence Graph
        └── Multi-Host Analysis
                 ↓
          Web Dashboard
```

## LLVM Clarification
LLVM is compiler infrastructure, not the forensic framework.

- C++: implementation language for compiler/system components.
- LLVM: IR, optimization and target-specific code generation.
- JOCKY: high-level domain-specific forensic language.
- OS-specific backends: actual Windows/Linux forensic execution.

Correct wording:
> One JOCKY compiler uses LLVM target support, while OS-specific forensic backends provide platform-specific execution.

Do not say that LLVM itself makes the forensic framework cross-platform.

## JOCKY DSL
JOCKY is a high-level domain-specific language for digital forensics.

Example conceptual commands:

```text
collect_processes()
collect_users()
collect_files()
collect_network()
collect_logs()
hash_evidence()
```

The same command can have platform-specific backend implementations.

## How We Build the Framework
We do not need another forensic company to give us its internal architecture.

We build from:
1. Official OS documentation — Windows APIs/interfaces, Event Log/EVTX, Linux `/proc`, journald/syslog, networking interfaces, filesystem/OS documentation.
2. Digital-forensics research and standards — NIST, academic research, public artifact specifications and established methodologies.
3. Existing open-source forensic projects — for concepts, expected artifacts, benchmarking and validation; they are not required dependencies.
4. Controlled experiments — generate known artifacts on our own Windows/Ubuntu machines and verify JOCKY's results.

Important:
> We are not reverse-engineering proprietary products. We are implementing forensic functionality from publicly documented operating-system mechanisms, forensic research and controlled testing.

## Existing Forensic Tools
Relevant tools include:
- Autopsy / The Sleuth Kit — disk/filesystem forensics
- Velociraptor — endpoint collection/investigation
- Volatility 3 — memory forensics
- Plaso/log2timeline — forensic timelines
- KAPE — Windows artifact collection
- Eric Zimmerman tools — Windows artifact analysis
- RegRipper — Windows Registry
- Chainsaw — Windows Event Logs
- Hayabusa — Windows event-log analysis
- Wireshark — network analysis
- Zeek — network metadata/analysis
- YARA — pattern/signature scanning
- Sigma — detection rules
- Hashdeep — hashing
- ExifTool — metadata

Do not claim JOCKY is the first forensic framework or that existing tools cannot do these things.

JOCKY's differentiation is the combination of:
- Purpose-built forensic DSL
- Cross-platform forensic abstraction
- Programmable workflows
- Own execution framework
- Centralized investigation
- Evidence timeline
- Evidence Relationship Graph
- Multi-host analysis
- Blockchain-backed evidence provenance

## Why JOCKY If Existing Tools Exist?
Preferred answer:
> Existing forensic tools are powerful, but JOCKY approaches the problem from a different angle: making forensic investigation itself programmable.

JOCKY can eventually coexist with or benchmark against existing tools, but its core framework is independently built.

## Primary Users
- Digital forensic investigators
- Cybersecurity / SOC teams
- Government and law-enforcement investigation teams
- Enterprise incident-response teams

These users already exist and already use forensic tools. JOCKY gives them a programmable way to perform repeatable cross-platform investigations.

## Trust Model
A real forensic product must establish trust through:
- Evidence integrity
- Cryptographic hashing
- Chain of custody
- Auditability
- Reproducibility
- Validation against known forensic datasets
- Comparison with established forensic tools
- Controlled deployment
- Security testing
- Independent validation/certification where appropriate

Example:

```text
Evidence collected
      ↓
SHA-256 hash
      ↓
Evidence metadata
      ↓
Blockchain record
      ↓
Evidence retrieved/processed
      ↓
Hash calculated again
      ↓
MATCH → integrity verified
MISMATCH → evidence changed
```

Blockchain does NOT automatically make JOCKY trustworthy. It provides a tamper-evident record of evidence provenance and custody events.

## Blockchain Role
Blockchain is NOT used to:
- collect logs
- process logs
- analyze logs
- replace the evidence repository

Instead:
> JOCKY collects and processes evidence. Blockchain stores proof of the evidence's integrity and provenance.

Potential record:
- Evidence ID
- Evidence Hash
- Timestamp
- Source Machine
- Investigation ID
- Collector/Agent ID
- Previous Record Hash

The actual evidence file remains in the evidence repository rather than being stored directly on-chain.

Other uses:
- Evidence integrity
- Chain of custody
- Evidence transfer verification
- Investigation audit trail
- Multi-machine evidence provenance

Simple explanation:
> Blockchain = trusted record of evidence and its history.

## Core Differentiators

### 1. Forensic Timeline
Reconstructs system events in chronological order.
Answers: What happened and in what order?

### 2. Evidence Relationship Graph
Connects users, files, processes, network activity and other evidence.
Answers: How are these pieces of evidence related?

### 3. Multi-Host Analysis
Identifies differences and anomalies across multiple systems.
Answers: Which machine is different and why?

### 4. Programmable Forensics
Enables investigators to create custom investigation workflows using JOCKY scripts.

## SIH Scope
Feasible for an SIH working application:
- JOCKY DSL
- Lexer/parser
- AST/IR
- C++ compiler components
- LLVM integration
- Windows backend
- Ubuntu/Linux backend
- Process collection
- User/account information
- File metadata
- Network information
- Log/event collection
- Evidence repository
- SHA-256 hashing
- Chain-of-custody metadata
- Blockchain evidence records
- Timeline
- Evidence graph
- Basic multi-host comparison
- Web dashboard

Not realistic as initial SIH scope:
- Complete replacement for Autopsy
- Complete replacement for Volatility
- Every Windows artifact parser
- Full disk recovery
- Complete memory forensics
- Every network-forensics capability
- Production-grade validation across thousands of cases
- Universally trusted law-enforcement platform

The correct goal:
> Build a complete vertical slice with a deliberately limited forensic capability set.

## Full Working Application
For SIH, "prototype" should mean a working vertical slice, not mock screens.

Demo workflow:
1. Investigator opens dashboard.
2. Selects authorized test machine.
3. Creates/selects JOCKY script.
4. JOCKY compiler compiles it.
5. Agent executes forensic commands.
6. Real evidence is collected.
7. Evidence is hashed.
8. Hash + custody information recorded.
9. Evidence reaches repository.
10. Evidence is analyzed.
11. Timeline generated.
12. Evidence relationships generated.
13. Blockchain record verified.
14. Investigator sees results in dashboard.

Use controlled machines that the team owns/controls. The evidence should be real evidence generated by the test environment.

## Development Phases

### Phase 1 — Foundation & Architecture
- Finalize JOCKY syntax
- Define forensic command set
- Design compiler architecture
- Define AST/IR
- Define agent/server communication
- Define evidence format/metadata
- Define blockchain record model
- Git/CI/CD/coding standards

Output: architecture + language specification + project skeleton.

### Phase 2 — JOCKY Language & Compiler
```text
JOCKY Script → Lexer → Parser → AST → JOCKY IR → LLVM
```
Build lexer, parser, semantic checking, AST, IR, LLVM integration and basic commands.

Output: executable JOCKY program.

### Phase 3 — Forensic Execution Framework
Build:
- JOCKY execution engine
- Windows agent/backend
- Ubuntu/Linux agent/backend
- Process collection
- User/account collection
- File metadata
- Network information
- Log/event collection
- Secure communication

Output: JOCKY script → compiler → agent → real evidence.

### Phase 4 — Evidence Management & Integrity
Build:
- Evidence repository
- Metadata
- SHA-256
- Evidence IDs
- Timestamps
- Source machine information
- Chain of custody
- Evidence transfer
- Integrity verification
- Blockchain ledger

Output: evidence can be stored, tracked and verified.

### Phase 5 — Forensic Analysis Engine
Build:
- Evidence normalization
- Timeline engine
- Event correlation
- Rule engine
- Evidence Relationship Graph
- Multi-host comparison
- Basic anomaly/deviation detection

Output: raw evidence becomes investigation results.

### Phase 6 — Central Investigation Platform
Build:
- Web dashboard
- Investigation creation
- Machine/agent management
- Script management
- Job execution
- Real-time status
- Evidence browser
- Timeline visualization
- Evidence graph visualization
- Multi-host comparison
- Integrity verification

Output: complete usable application.

### Phase 7 — Integration & End-to-End System
Connect:
```text
Investigator
 → JOCKY Script
 → Compiler (C++ + LLVM)
 → Execution Engine
 → Windows/Ubuntu Agents
 → Evidence
 → Hash + Chain of Custody
 → Blockchain
 → Analysis
 → Timeline + Evidence Graph
 → Dashboard
```

Output: one complete working investigation workflow.

### Phase 8 — Validation, Security & Hardening
- Controlled forensic datasets
- Compare results with established tools
- Windows/Ubuntu compatibility
- Evidence integrity
- Blockchain verification
- Multi-host operation
- Authentication/authorization
- Secure agent communication
- Error handling
- Performance testing
- Reproducibility testing
- Documentation

Output: defensible release candidate.

## Realistic Timeline
- SIH MVP: approximately 2–4 months
- Serious prototype: 6–12 months
- Usable v1: 12–24 months
- Mature forensic platform: 2–4+ years for a small dedicated team

The long-term effort is mostly broad artifact coverage, OS-specific behavior, validation, security, scalability, reproducibility and production reliability.

## Six-Member Team Split

### Ranjith — Architecture + Compiler
- JOCKY language design
- Lexer/parser
- AST/IR
- LLVM integration
- Compiler architecture
- System integration

### Sandeep — Windows Forensic Backend
- Windows agent
- Process collection
- User information
- File information
- Network collection
- Windows logs/events
- Windows-specific artifacts

### Sankar — Ubuntu/Linux Forensic Backend
- Linux/Ubuntu agent
- `/proc`
- Users
- Files
- Network
- Logs
- Services
- Linux-specific artifacts

### Archana — Evidence + Blockchain
- Evidence repository
- SHA-256 hashing
- Evidence metadata
- Chain of custody
- Blockchain ledger
- Integrity verification

### Nazeer — Analysis Engine
- Timeline engine
- Evidence Relationship Graph
- Multi-host comparison
- Correlation
- Rule engine
- Basic anomaly/deviation analysis

### Lakshana — Central Platform + Dashboard
- REST/WebSocket server
- Dashboard
- Investigation management
- Agent/machine management
- Job execution
- Evidence visualization
- Timeline visualization
- Evidence graph visualization

## Team Integration Rule
Do not build six isolated projects. Everything must integrate into:

```text
JOCKY Script
 → Compiler
 → Windows/Ubuntu Agent
 → Evidence
 → Evidence + Blockchain
 → Analysis
 → Dashboard
```

## Jury Answers

### How will you build all this?
> We are building JOCKY as a layered framework. First we build the DSL and compiler using C++ and LLVM. Then we implement OS-specific forensic backends for Windows and Ubuntu. Each JOCKY forensic command maps to a backend implementation that obtains the required artifacts directly from the operating system. We validate the implementation using official OS documentation, forensic standards, research, controlled test machines and known datasets.

### Are you building everything from scratch?
> We build the JOCKY framework and its core forensic capabilities ourselves. We don't need to reinvent every low-level OS mechanism. We use documented Windows and Linux interfaces as the foundation and build our own abstraction, collection, processing and investigation layers.

### A large company took years. How can students build this?
> We are not trying to reproduce an entire mature forensic suite. Our innovation is the JOCKY framework. We start with a focused set of core forensic operations, build the cross-platform execution and evidence architecture around them, and expand the modules incrementally.

### Why would existing forensic companies support you?
> They don't need to. JOCKY is designed independently using publicly documented operating-system interfaces, forensic standards and research. Existing tools are used for validation and benchmarking, not as dependencies on proprietary architecture.

### Why should investigators trust JOCKY?
> Trust is established through reproducibility, evidence hashing, chain of custody, auditability, validation against known forensic datasets and comparison with established forensic tools. Blockchain provides an additional tamper-evident record of evidence provenance.

### Why use blockchain?
> Blockchain is used for evidence provenance and integrity. JOCKY collects and processes the evidence; blockchain records its hash and chain-of-custody information so later changes can be detected.

### Why not Python?
> Python is general-purpose. JOCKY is a domain-specific language designed around forensic operations, allowing investigators to express reusable workflows without dealing directly with platform-specific implementation details.

### Why LLVM?
> LLVM provides mature intermediate representation, optimization and target-specific code generation, allowing us to focus on the JOCKY language and forensic framework instead of building a complete compiler backend from scratch.

### Can one JOCKY compiler support Windows and Ubuntu?
> Yes. We use one JOCKY compiler with LLVM target support and separate OS-specific forensic backends. The same JOCKY forensic command can have platform-specific implementations.

### What happens if Windows and Ubuntu expose different information?
> The common JOCKY command defines the investigation-level operation, while the backend maps that operation to artifacts and interfaces available on each operating system.

### What will the SIH MVP demonstrate?
> A working JOCKY compiler with a focused set of forensic commands, execution on Windows and Ubuntu, real evidence collection from controlled machines, centralized evidence storage, integrity verification, blockchain provenance, timeline/evidence-graph analysis and a web dashboard.

## Spoken Introduction
Good afternoon, respected judges.

The problem we are addressing is that forensic investigators need to analyze computers and networks across different operating systems using different tools and procedures, while legitimate forensic activities can also be unnecessarily flagged by security solutions.

Our solution to this problem is JOCKY — a cross-platform forensic programming language and framework that allows investigators to create, execute, and centrally manage systematic computer and network forensic investigations.

With JOCKY, investigators can write forensic workflows using a simple, purpose-built language, collect evidence across Windows and Ubuntu systems, analyze and correlate that evidence, and manage the entire investigation through a centralized platform.

In short, we are not just building another forensic tool. We are building a programmable forensic investigation platform.

## One-Line Problem
> Forensic investigators need a standardized way to perform computer and network forensic analysis across different systems while minimizing unnecessary interference from security solutions.

## One-Line Solution
> JOCKY — a cross-platform forensic programming language and framework for creating, executing and centrally managing systematic forensic investigations.

## Reference / Research
1. NIST — Digital Investigation Techniques: A NIST Scientific Foundation Review
   https://www.nist.gov/publications/digital-investigation-techniques-nist-scientific-foundation-review

2. NIST SP 800-86 — Guide to Integrating Forensic Techniques into Incident Response
   https://doi.org/10.6028/NIST.SP.800-86

3. LLVM Official Documentation
   https://llvm.org/docs/

## Core Principles
1. JOCKY is its own framework.
2. JOCKY is a high-level forensic DSL.
3. C++ implements compiler/system components.
4. LLVM provides compiler infrastructure.
5. Windows and Ubuntu use OS-specific forensic backends.
6. JOCKY collects and processes evidence; blockchain does not.
7. Blockchain provides evidence provenance/integrity/audit records.
8. Existing forensic tools are references/benchmarks, not required dependencies.
9. Do not claim JOCKY replaces the entire forensic ecosystem.
10. SIH prototype should be a complete working vertical slice with limited forensic breadth.
11. Use controlled machines for demonstrations.
12. Do not claim production-grade forensic certification before it exists.
13. Focus on programmable forensics as the main differentiator.
14. Core differentiators: Timeline, Evidence Graph, Multi-Host Analysis, Programmable Forensics.
15. Long-term product can expand through additional OS/artifact modules.

## Current Recommended Product Positioning
> JOCKY is a cross-platform, programmable digital-forensics framework built around a purpose-specific forensic language. It provides a common investigation layer across Windows and Ubuntu, executes modular forensic operations through OS-specific backends, centrally correlates collected evidence, and maintains verifiable evidence provenance using cryptographic hashing and blockchain-backed audit records.
