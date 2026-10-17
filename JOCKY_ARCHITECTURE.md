# JOCKY — Buildable Architecture & Feasibility Spec (Hackathon Edition)

> SIH 26148 · NTRO · Team GOD VALLEY
> Purpose of this file: (1) brutally assess what can actually be built, (2) give an AI builder (or humans) an unambiguous spec for the parts that survive that assessment.
> Where this file conflicts with `JOCKY_Project_Master_Context.md`, **this file wins** (the conflicts are listed in §1.4).

---

## 0. How an AI builder must use this document

1. **Contract-first.** Build in the order of §15. Section 8 (data contracts) and §9 (API) are frozen at Milestone M0. Never change a schema silently; bump `schema_version`, update `fixtures/`, update every consumer.
2. **Walking skeleton first.** By end of M1 a fake collector must travel the whole path: script → compiler → server → agent → evidence → hash → ledger → dashboard. Real collectors are swapped in afterwards.
3. **Simplest thing that satisfies the acceptance test.** If something is ambiguous, pick the simpler option and append one line to `DECISIONS.md`.
4. **No features outside P0 until M4 passes.** P1/P2 items are listed in §1.3 and must not be started early.
5. **Every stub is declared** in `STUBS.md` (file, what is faked, why). A stub that is not declared is a bug.
6. **Determinism.** Analysis output must be byte-identical for identical input (sorted keys, stable ordering, no randomness, no wall-clock in results except explicit `generated_utc`).
7. **Time rules.** All timestamps are ISO-8601 UTC with `Z` and microsecond precision at most (`2026-10-17T09:30:00.123456Z`). Never store local time.
8. **Target-host rule.** The agent is read-only with respect to the target. It writes only inside its own working directory (`<agent_dir>/work/`), deletes work files after upload, and reports its footprint in the evidence manifest.
9. **Every task ends with:** code + unit tests + updated fixtures/schemas + one-paragraph note in `CHANGELOG.md`. CI must be green.
10. **Security-sensitive rule:** no remote shell, no arbitrary command execution, no executing anything that is not a signed Plan (§6).

---

## 1. Feasibility analysis (brutal)

### 1.1 Verdict

| Question | Answer |
|---|---|
| Can the *vision* (full programmable forensic platform) be built for a hackathon? | **No.** Not in any timeframe a hackathon allows. |
| Can a *vertical slice* that demonstrates the vision be built by 6 students? | **Yes, conditionally.** |
| Conditions | (a) scope frozen to §2, (b) walking skeleton by week 3, (c) the hard decisions in §1.4 are accepted, (d) ≥ 9 weeks of part-time work (10 recommended), (e) a rehearsed demo with a recorded fallback. |
| Biggest threats | Integration at the end, Windows C++ API debugging, LLVM rabbit hole, scope creep, one overloaded person, a live demo with no fallback. |

### 1.2 Component scorecard

Rating: 🟢 straightforward · 🟡 doable but has traps · 🔴 will sink you if on the critical path.

| Component | Rating | Brutal assessment |
|---|---|---|
| DSL lexer/parser/AST/semantic checks | 🟢 | Small grammar, classic problem. Keep the language tiny. |
| JOCKY IR → signed Plan (JSON) | 🟢 | Trivial once grammar is fixed. |
| **LLVM codegen on the demo path** | 🔴 | A script that calls `collect_processes()` has nothing for LLVM to optimize. Building LLVM on Windows/Linux, linking it, and shipping JIT into an agent costs weeks and adds *zero* demo value. It would also make the agent huge and AV-suspicious. **Off the critical path** (D1). |
| Agent core (poll, verify, execute, hash, sign, upload) | 🟡 | Straightforward, but C++ networking/TLS/JSON on two OSes eats time. Use libraries (§4). |
| Windows collectors: processes, network, users, files | 🟡 | Documented APIs, but Unicode, elevation, PID reuse, access-denied edge cases and command-line retrieval are traps. |
| Windows event logs | 🟡 | `EvtQuery`/`EvtRender` works; rendering human message text is painful → skip it (emit raw XML + EventData fields). |
| Linux collectors | 🟢/🟡 | `/proc` is easy. Trap: socket→PID mapping, IPv6 hex endianness, races (processes vanish mid-read). |
| Linux logs | 🟡 | Ubuntu may or may not have `/var/log/auth.log`; fall back to journald. Shell out to `journalctl` (acceptable, no shell). |
| Agent↔server TLS | 🟡 | HTTPS with pinned self-signed cert: yes. Full mTLS PKI: no (D3). |
| Evidence repo, hashing, custody chain | 🟢 | Filesystem + SQLite. |
| Ledger (own hash-chained signed service) | 🟢 | ~300 lines. Honest label required (D6). |
| Ledger on Hyperledger Fabric / public chain | 🔴 | Infra heavy, zero demo benefit. Not doing it. |
| Ledger on local EVM (Hardhat/Anvil + 30-line contract) | 🟡 | Feasible as an optional adapter (P1). |
| Normalization, timeline | 🟢 | Agents emit normalized records (D4); timeline is a sort + filter. |
| Evidence graph | 🟡 | Building is easy; making it *readable* is the work. Cap node count, use filters. |
| Correlation rules | 🟡 | 6 deterministic rules, hand-written. Each needs a planted artifact to prove it. |
| Multi-host diff | 🟡 | Set comparison on keys. Cross-OS comparison is semantically tricky → same-OS only (§11.5). |
| "Anomaly detection" (ML) | 🔴 | Not credible with 3–4 hosts. Replaced by rarity + rules (D7). |
| Dashboard | 🟡 | Biggest *volume* of work for one person. Needs help or cuts (§15.3). |
| Network forensics (PCAP analysis) | 🔴 for MVP | Not in P0. P2 stretch only. Be honest that MVP "network" = connection state snapshot + network-related log events. |
| Memory / disk imaging / full artifact parsing | 🔴 | Explicitly out. MVP is **triage collection**, not forensic imaging. |
| AV/EDR "non-interference" | 🟡 | Cannot be guaranteed. Achievable: single signed agent, documented APIs only, read-only, no persistence, declared exclusions in lab (§7.6). |
| Live demo on real infrastructure | 🔴 without fallback | Must have pre-recorded backup + fixtures-driven offline mode. |

### 1.3 Effort budget (person-days, rough, includes tests)

| # | Item | pd | Pri |
|---|---|---|---|
| 1 | Phase 0: language spec, schemas, API contract, fixtures, repo+CI skeleton | 6 | P0 |
| 2 | Compiler front-end (lexer/parser/sema/IR/plan) | 10 | P0 |
| 3 | Plan signing + `jockyc` CLI + tests | 2 | P0 |
| 4 | Agent core (poll, verify plan, executor, hash, sign, upload) | 10 | P0 |
| 5 | Windows collectors: sysinfo, processes, users, network, files | 8 | P0 |
| 6 | Windows event logs | 4 | P0 |
| 7 | Windows persistence | 2 | P1 |
| 8 | Linux collectors: sysinfo, processes, users, network, files | 9 | P0 |
| 9 | Linux logs | 3 | P0 |
| 10 | Linux persistence | 2 | P1 |
| 11 | Server core: REST, DB, jobs/runs, auth, WebSocket | 12 | P0 |
| 12 | Evidence repo + custody chain + verify | 7 | P0 |
| 13 | Ledger service (hash-chained, signed) | 4 | P0 |
| 14 | EVM ledger adapter | 5 | P1 |
| 15 | Normalizers / entity+event builders | 6 | P0 |
| 16 | Timeline | 3 | P0 |
| 17 | Evidence graph | 5 | P0 |
| 18 | Correlation rules (6) | 4 | P0 |
| 19 | Multi-host diff | 4 | P0 |
| 20 | Dashboard core (scaffold, auth, investigations, hosts, runs, WS) | 8 | P0 |
| 21 | Script editor + compile + run | 3 | P0 |
| 22 | Evidence browser + integrity/custody page | 5 | P0 |
| 23 | Timeline view | 3 | P0 |
| 24 | Graph view | 4 | P0 |
| 25 | Compare + findings view | 2 | P0 |
| 26 | Integration, e2e tests, CI hardening | 14 | P0 |
| 27 | Lab VMs, agent packaging, scenario scripts | 5 | P0 |
| 28 | Demo prep, docs, backup video | 6 | P0 |
| 29 | PCAP import → flows/DNS | 4 | P2 |
| 30 | LLVM IR emission (`--emit-llvm`, demo-only) | 6 | P2 |
| | **P0 total** | **147** | |
| | P1 total | 9 | |
| | P2 total | 10 | |

**Capacity math:** 6 people × weeks × ~4 productive days/week × 0.7 (classes, bugs, friction).
- 8 weeks → ~134 pd → **P0 does not fit.**
- 9 weeks → ~151 pd → P0 fits with ~zero slack.
- 10 weeks → ~168 pd → P0 + P1 fits.

AI-assisted coding speeds up boilerplate (schemas, CRUD, React views) but **not** Windows API debugging, cross-machine integration, or demo rehearsal. Do not assume it cuts these numbers in half.

**If you have fewer than 9 weeks — emergency cuts (≈ 17 pd saved):** graph limited to 3 edge types with a fixed layout (−2); correlation limited to R1, R3, R5 (−2); diff limited to processes + users + listening ports (−1); merge Timeline and Compare into table views with no charts (−4); evidence browser = table + JSON viewer (−2); Windows logs limited to Security log and 4 event IDs (−1.5); Linux logs limited to auth only (−1); reduced CI (−3.5). Below ~7 weeks, drop one OS to "basic collectors only" and say so.

### 1.4 Hard decisions (these override the Master Context)

| ID | Decision | Why |
|---|---|---|
| **D1** | **LLVM is not on the critical path.** Compiler is C++17: source → AST → typed IR → signed Plan (JSON). `--emit-llvm` is a P2 demo feature. If P2 isn't done by M4, **remove LLVM claims from slides** ("LLVM backend planned"). | Zero demo value, high cost, hurts agent size/AV profile. |
| **D2** | **Scripts are signed data, not native executables.** One signed agent binary interprets Plans. Agent rejects unsigned/tampered Plans. | Answers "AV flags new binaries", "can scripts do anything?", and "why DSL not Python?" (scripts are restricted, verifiable, read-only by construction). |
| **D3** | **Agent polls outbound over HTTPS.** No inbound ports, no remote shell. TLS with pinned server cert fingerprint + per-agent bearer token + Ed25519 signatures on evidence. mTLS is a stretch, not MVP. | Simplest secure-enough design; avoids firewall pain. |
| **D4** | **Agents emit already-normalized records** (common schema + `ext` for OS-specific fields). Server never parses OS-specific formats. | One contract; collectors and analysis can be built in parallel. |
| **D5** | **Triage collection, not imaging.** Parsed JSONL records only. No raw EVTX, no disk images, no memory. | Scope. State it honestly. |
| **D6** | **Ledger = separate "JOCKY Ledger" service**: append-only, hash-chained, Ed25519-signed blocks, own DB and key, ideally on a different machine. Call it "permissioned, blockchain-style ledger". Optional EVM adapter (P1) behind the same interface. | Real blockchain infra buys nothing here; separation from the evidence repo is what makes tampering detectable. Be honest: tamper-evident, not decentralized. |
| **D7** | **Analysis = deterministic rules + set operations.** No ML. "Anomaly" = rarity across hosts + rule hits. | Credible with 3–4 hosts; explainable to judges. |
| **D8** | **Languages:** C++17 (compiler, agent, shared core). Python 3.11+ (server, ledger, analysis). TypeScript/React (dashboard). | Matches the pitch where it matters; fast elsewhere. |
| **D9** | **SQLite + filesystem.** No Postgres/Kafka/Docker-orchestration requirement. | Zero ops. |
| **D10** | **Time:** store raw agent timestamps and `clock_offset_ms` (server − agent, measured at heartbeat). Analysis uses adjusted time by default, raw time toggle in UI. | Multi-host timelines are wrong without this. |
| **D11** | **Unit of execution = Run** = (immutable script version × selected hosts) → one Job per host. Analysis ops run **server-side** after all jobs of a run reach a terminal state. | Clean model for multi-host. |
| **D12** | **Platforms:** Windows 10/11 x64 and Ubuntu 22.04+/x64 only. Agent runs **elevated, from a terminal**, no service install, no persistence. | Less code, better forensic hygiene. |

### 1.5 What will kill the project (and the countermeasure)

1. **Integration left to the end** → walking skeleton at M1, integration test in CI from week 3, weekly end-to-end run.
2. **Windows C++ rabbit holes** → time-box each collector to its budget; ship partial fields (`null`) rather than block; fall back to WMI/PowerShell-free shell-outs only where listed.
3. **Schema churn** → freeze at M0; changes need a version bump and a team message.
4. **One overloaded person** → see §15.2 load balancing; Ranjith does not own server or dashboard.
5. **Pretty-graph syndrome** → graph is capped, filtered, and tied to findings.
6. **Demo dependency on live infra** → `DEMO_OFFLINE=1` mode replays fixtures through the real server; plus a screen recording.
7. **Claiming too much** → see §1.6.
8. **Planted-artifact scenario built last** → build the lab + scenario scripts in M2, so analysis is developed against real data.

### 1.6 Honest claims (say / don't say)

| Say | Don't say |
|---|---|
| "Programmable triage collection across Windows and Ubuntu from one script." | "Replaces Autopsy/Velociraptor/Volatility." |
| "Permissioned, blockchain-style ledger recording evidence hashes and custody events; tamper-evident." | "Blockchain makes evidence trustworthy / court-ready." |
| "Scripts are signed and restricted, so execution is auditable and read-only by design." | "Undetectable by antivirus." (We aim at *legitimate coordination*, not evasion.) |
| "Validated against native OS tools and a planted-artifact scenario in a controlled lab." | "Forensically certified." |
| "Analysis is reproducible: same evidence + same rules → identical output." | "Collection is reproducible." (Live state changes.) |
| "Closest comparable: Velociraptor (VQL) and osquery; JOCKY adds the evidence-provenance chain and integrated multi-host analysis." | "No tool can do this." |

---

## 2. MVP scope

### 2.1 In scope (P0)
- JOCKY DSL with 8 collect commands + 4 analyze commands (§5).
- Compiler: `jockyc check | build | explain`.
- Windows + Ubuntu agent: enroll, poll, verify Plan, execute, hash, sign, upload.
- Server: investigations, hosts, scripts (versioned), runs/jobs, evidence repo, custody chain, WebSocket progress.
- Ledger service with verification.
- Analysis: timeline, evidence graph, 6 correlation rules, multi-host diff.
- Dashboard: login, investigations, hosts, script editor, run monitor, evidence browser, integrity page, timeline, graph, compare/findings.
- Planted-artifact lab scenario and a tamper-detection demo.

### 2.2 Out of scope (do not build, do not promise)
Disk imaging, memory forensics, raw EVTX/registry hive parsing, file content acquisition, PCAP (P2), macOS/Android, ML, mTLS PKI, Windows service install, multi-tenant, RBAC beyond 2 roles, HA/scaling, report generation beyond JSON/CSV export, real certification.

---

## 3. System architecture

```text
                       ┌─────────────────────────────┐
                       │  Dashboard (React + TS)     │
                       └──────────────┬──────────────┘
                              REST + WebSocket
┌──────────────────────────────────────▼──────────────────────────────────────┐
│ JOCKY Server (Python/FastAPI)                                               │
│  ├─ Auth (operators)           ├─ Runs/Jobs orchestration                   │
│  ├─ Compiler bridge ──► jockyc │  (subprocess; signs Plans)                 │
│  ├─ Evidence service (repo + SQLite + custody chain)                        │
│  ├─ Ledger client ─────────────────────────────┐                            │
│  └─ Analysis package (normalize → timeline → graph → rules → diff)          │
└───────▲──────────────────────────────────────────┼──────────────────────────┘
        │ HTTPS (agent polls outbound)             │ HTTPS
┌───────┴────────┐  ┌────────────────┐     ┌────────▼──────────┐
│ Windows Agent  │  │ Ubuntu Agent   │     │ JOCKY Ledger      │
│ (C++17)        │  │ (C++17)        │     │ (separate process │
│ verify Plan →  │  │ verify Plan →  │     │  own DB + key)    │
│ collect → hash │  │ collect → hash │     └───────────────────┘
│ → sign → upload│  │ → sign → upload│
└────────────────┘  └────────────────┘
```

### 3.1 End-to-end flow (the 14-step demo, made precise)

1. Operator logs in → creates Investigation → selects enrolled hosts.
2. Operator writes/opens a script in the editor (saved as immutable *Script Version*).
3. Dashboard `POST /runs` → server calls `jockyc build` → **signed Plan** → one **Job** per host (state `QUEUED`).
4. Agent long-polls `GET /agent/jobs/next`, receives Plan, **verifies signature** against the pinned server public key.
5. Agent executes ops whose `platforms` match its OS; per op: writes `<op_id>_<command>.jsonl`, computes SHA-256, signs, uploads with manifest; reports progress.
6. Server re-computes SHA-256, verifies agent signature → stores file (content-addressed path), creates `COLLECTED`/`RECEIVED` custody events.
7. Server writes `EVIDENCE_REGISTERED` to the **Ledger**; stores ledger block ref; custody event `REGISTERED`.
8. When all jobs of the run are terminal → server runs analysis ops: normalize → timeline → graph → findings → compare.
9. Dashboard receives WS events and updates live.
10. Operator opens Integrity page → `verify` → 6 independent checks (§10.4) → PASS/FAIL.
11. Demo: tamper with a stored evidence file on the server disk → verify shows **FAIL** on hash and ledger comparison.

### 3.2 Trust boundaries
- **Target host (agent):** untrusted-ish; can be compromised → hash proves post-collection integrity only.
- **Server + evidence repo:** trusted but *not* trusted to be unmodified → hence ledger.
- **Ledger:** separate process/machine/key; holds hashes only.
- **Operator browser:** authenticated session.

---

## 4. Tech stack & repository layout

### 4.1 Stack

| Layer | Choice | Notes |
|---|---|---|
| Build | CMake ≥ 3.20, vcpkg (manifest mode) | MSVC (Windows), GCC/Clang (Linux). x64 only. |
| C++ libs | `nlohmann-json`, `libsodium` (SHA-256 + Ed25519), `cpp-httplib` (+OpenSSL for HTTPS), `doctest` or `Catch2` | Keep dependencies this small. |
| Server | Python 3.11+, FastAPI, uvicorn, SQLite (stdlib `sqlite3` or SQLModel), PyNaCl (Ed25519), `jsonschema`, `bcrypt`, `pyjwt` | |
| Ledger | Python 3.11+, FastAPI, SQLite, PyNaCl | Separate process, separate DB file, separate key. |
| Analysis | Pure Python package `jocky_analysis` | Importable by server and runnable as CLI on fixtures. |
| Dashboard | React + Vite + TypeScript + Tailwind, CodeMirror 6 (editor), Cytoscape.js (graph), `vis-timeline` or virtualized table (timeline) | |
| CI | GitHub Actions: matrix `windows-latest`, `ubuntu-latest` | Build compiler + agent, run unit tests, validate schemas/fixtures, run Python tests, run e2e on Linux with fixture-agent. |

### 4.2 Repo layout

```text
jocky/
├─ README.md  ARCHITECTURE.md (this file)  DECISIONS.md  STUBS.md  CHANGELOG.md
├─ schemas/                 # JSON Schema (single source of truth for contracts)
│  ├─ plan.schema.json  manifest.schema.json  ledger_record.schema.json
│  ├─ records/{system_info,process,account,session,net_conn,file,log_event,persistence}.schema.json
│  ├─ findings.schema.json  graph.schema.json  events.schema.json
│  └─ openapi.yaml          # server + ledger APIs
├─ fixtures/                # realistic sample evidence per OS + expected analysis outputs
│  ├─ windows/*.jsonl  linux/*.jsonl  scenario_expected/*.json
├─ core/                    # C++ shared lib: json helpers, crypto, time, base64, errors
├─ compiler/                # C++17: lexer, parser, ast, sema, ir, plan, sign, cli
│  └─ tests/ (valid/*.jocky, invalid/*.jocky + expected errors)
├─ agent/                   # C++17
│  ├─ src/{main,config,tls_client,enroll,job_loop,plan_verify,executor,evidence_writer,clock}.cpp
│  ├─ src/collectors/common/   # record writers, schema helpers, path utils
│  ├─ src/collectors/win/      # processes, users, network, files, logs, persistence, sysinfo
│  ├─ src/collectors/linux/    # same set
│  └─ tests/
├─ server/                  # FastAPI app
│  └─ app/{main,config,auth,db,models,api/*,services/{runs,compiler_bridge,evidence,custody,ledger_client,verify}.py,ws.py}
├─ ledger/                  # FastAPI ledger service
├─ analysis/                # jocky_analysis package: normalize, timeline, graph, rules/*.py, diff, cli
├─ dashboard/               # React app
├─ lab/                     # VM setup notes, scenario plant/cleanup scripts, oracle comparison scripts
├─ scripts/                 # dev helpers: run_all.sh/.ps1, demo_tamper.sh, gen_keys.py
└─ .github/workflows/ci.yml
```

---

## 5. JOCKY language specification (v0.1)

### 5.1 Design principles
Tiny, declarative, **no loops, no arbitrary expressions, no I/O, no shell, no writes to targets**. A valid script can only call the commands in §5.4. That restriction *is* the safety and auditability feature.

### 5.2 Lexical rules
- Comments: `// …` and `/* … */`.
- Identifiers: `[A-Za-z_][A-Za-z0-9_]*`. Keywords: `investigation meta let on true false`.
- Literals: strings `"…"` (escapes `\\ \" \n \t`), integers, booleans, durations (`INT` followed by `s|m|h|d`, e.g. `24h`), lists `[a, b]`.
- Whitespace-insensitive; statements end with `;`.

### 5.3 Grammar (EBNF)

```ebnf
program     = "investigation" STRING block ;
block       = "{" { stmt } "}" ;
stmt        = meta_stmt | let_stmt | on_stmt | call_stmt ;
meta_stmt   = "meta" IDENT "=" literal ";" ;
let_stmt    = "let" IDENT "=" literal ";" ;
on_stmt     = "on" ( "windows" | "linux" ) "{" { call_stmt } "}" ;   (* no nesting *)
call_stmt   = IDENT "(" [ arg { "," arg } ] ")" ";" ;
arg         = IDENT ":" expr ;
expr        = literal | IDENT ;                                       (* IDENT = let-bound constant *)
literal     = STRING | INT | BOOL | DURATION | "[" [ expr { "," expr } ] "]" ;
```

### 5.4 Commands

**Collect commands (run on agents).**

| Command | Args (name: type = default) | Platforms | Output record type |
|---|---|---|---|
| `collect_system_info()` | — | win, linux | `system_info` |
| `collect_processes(hash_images: bool = true)` | | win, linux | `process` |
| `collect_users()` | | win, linux | `account` + `session` (two evidence items) |
| `collect_network()` | `include_udp: bool = true` | win, linux | `net_conn` |
| `collect_files(paths: [string], depth: int = 2, modified_since: duration = none, hash: bool = false, limit: int = 50000)` | `paths` required | win, linux | `file` |
| `collect_logs(source: string, since: duration = 24h, event_ids: [int] = [], limit: int = 5000)` | `source` required | win: `"Security"`,`"System"`,`"Application"`; linux: `"auth"`,`"syslog"`,`"journal"` | `log_event` |
| `collect_persistence()` | | win, linux | `persistence` |

**Analyze commands (run on the server, per Run, after all jobs terminal).**

| Command | Meaning |
|---|---|
| `analyze_timeline()` | Build unified event timeline from all evidence in the investigation. |
| `analyze_graph()` | Build evidence relationship graph. |
| `analyze_findings()` | Run the correlation rules (§11.4). |
| `compare_hosts()` | Multi-host diff (§11.5). |

Evidence hashing, signing, custody and ledger registration are **implicit** for every collect command; the script cannot disable them.

### 5.5 Semantic rules (compiler errors, each with line:col and message)
1. Unknown command / unknown arg / duplicate arg.
2. Arg type mismatch; required arg missing.
3. Command not supported on the platform of its enclosing `on` block.
4. `let` used before definition; duplicate `let`; `let` shadows keyword.
5. `meta` keys allowed: `case_id`, `examiner`, `description` (strings).
6. At least one `collect_*` call required; `analyze_*`/`compare_hosts` must not appear inside `on` blocks.
7. `collect_logs.source` must be valid for the platform; `since` > 0; `limit` in [1, 100000].
8. Warnings: `collect_files` with `depth` > 6; with `hash: true` and no `modified_since`.

### 5.6 Example script (also `fixtures/scripts/triage.jocky`)

```jocky
investigation "triage-baseline" {
  meta case_id  = "CASE-2026-001";
  meta examiner = "analyst1";

  let window = 24h;

  collect_system_info();
  collect_processes();
  collect_users();
  collect_network();
  collect_persistence();

  on windows {
    collect_logs(source: "Security", since: window, event_ids: [4624, 4625, 4688, 4720, 4732], limit: 5000);
    collect_files(paths: ["C:\\Users"], depth: 3, modified_since: window, hash: true);
  }
  on linux {
    collect_logs(source: "auth", since: window, limit: 5000);
    collect_files(paths: ["/home", "/tmp", "/etc/cron.d"], depth: 3, modified_since: window, hash: true);
  }

  analyze_timeline();
  analyze_graph();
  analyze_findings();
  compare_hosts();
}
```

### 5.7 CLI

```text
jockyc check   <file.jocky>                         # parse + semantic check, exit code 0/1, JSON diagnostics with --json
jockyc build   <file.jocky> -o plan.json --key server_signing.key --investigation-id <id> --run-id <id>
jockyc explain <file.jocky>                         # human-readable execution plan per platform
jockyc build ... --emit-llvm out.ll                 # P2, optional, demo only
```
Exit codes: 0 ok · 1 script error · 2 usage error · 3 internal error. Diagnostics JSON: `[{"severity":"error","line":3,"col":5,"code":"E012","message":"…"}]`.

---

## 6. Plan format & signing

**Envelope** (what travels to the agent):

```json
{
  "envelope_version": 1,
  "signer_key_id": "sha256-of-server-signing-pubkey (hex, first 16)",
  "body_b64": "<base64 of the exact Plan JSON bytes>",
  "signature_b64": "<Ed25519 detached signature over the exact decoded body bytes>"
}
```

**Plan body** (`schemas/plan.schema.json`):

```json
{
  "plan_version": 1,
  "plan_id": "uuid",
  "run_id": "uuid",
  "investigation_id": "uuid",
  "script_name": "triage-baseline",
  "source_sha256": "hex",
  "compiler_version": "0.1.0",
  "compiled_utc": "2026-10-17T09:30:00Z",
  "meta": { "case_id": "CASE-2026-001", "examiner": "analyst1" },
  "ops": [
    { "op_id": "op1", "command": "collect_processes", "args": { "hash_images": true }, "platforms": ["windows","linux"] },
    { "op_id": "op2", "command": "collect_logs", "args": { "source": "Security", "since_seconds": 86400, "event_ids": [4624,4625], "limit": 5000 }, "platforms": ["windows"] }
  ],
  "post_ops": [
    { "op_id": "p1", "command": "analyze_timeline", "args": {} }
  ],
  "limits": { "max_runtime_seconds": 900 }
}
```

Rules:
- Signing over **exact bytes** (no canonicalization needed). The agent verifies before parsing the body.
- Durations are converted to `*_seconds` integers by the compiler.
- Agent skips ops whose `platforms` doesn't include its OS and reports them as `SKIPPED`.
- `post_ops` are ignored by agents; the server executes them.
- Agent rejects: bad signature, unknown `signer_key_id`, unknown command, plan older than 24 h (`compiled_utc`), `plan_version` it doesn't support.
- Key handling: server signing keypair generated by `scripts/gen_keys.py`; private key readable only by the server process; agent receives the **public key** at enrollment (and the operator verifies its fingerprint in the dashboard against what the agent prints).

---

## 7. Agent specification (C++17)

### 7.1 Lifecycle

```text
start (elevated, terminal)
  → load/generate Ed25519 keypair in <agent_dir>/state/
  → enroll if no agent_token (one-time token from dashboard)
  → loop: heartbeat (every 10 s) + long-poll jobs (25 s)
  → on job: verify → execute ops sequentially → upload evidence per op → complete
  → on exit: delete work dir contents; no persistence
```

CLI:
```text
jocky-agent --server https://<host>:8443 --server-fingerprint SHA256:<hex> [--enroll-token <tok>] [--agent-dir <path>] [--insecure-no-pin (dev only)]
```

### 7.2 Enrollment
1. Operator creates a single-use enrollment token in the dashboard (TTL 1 h).
2. `POST /api/agent/enroll` `{token, hostname, os, os_version, arch, agent_version, agent_pubkey_b64}` → `{agent_id, host_id, agent_token, server_pubkey_b64, server_pubkey_id}`.
3. Agent stores `agent_id`, `agent_token`, server pubkey in `state/`. TLS pinned to the fingerprint passed on the command line.

### 7.3 Job loop & executor
- Poll: `GET /api/agent/jobs/next` with `Authorization: Bearer <agent_token>` → 200 `{job_id, envelope}` or 204.
- Verify envelope (§6). If invalid → `POST …/complete` with status `REJECTED` and reason.
- For each op in order: `POST …/progress {op_id, status: RUNNING}` → run collector → write JSONL → hash → sign → upload → `progress {op_id, status: DONE|FAILED|SKIPPED, record_count, error?}`.
- One op failing does not abort the job (`on_error = continue`). Job status: `COMPLETE` (all DONE/SKIPPED), `PARTIAL` (some FAILED), `FAILED` (all FAILED), `REJECTED`.
- Global timeout from `limits.max_runtime_seconds`; ops check a cancellation flag.
- Upload retries: 3× exponential backoff; on persistent failure mark op `FAILED(upload)`.

### 7.4 Evidence file writing
- Output file: JSON Lines, UTF-8, one record per line, one `record_type` per file, validated against `schemas/records/<type>.schema.json` in CI via fixtures (and via `--self-check` flag locally).
- Hash: SHA-256 streaming over the exact file bytes (libsodium).
- Signature: Ed25519 over the ASCII string `"<sha256_hex>|<evidence_id>|<job_id>|<host_id>"`.
- Manifest and file are uploaded in one multipart request (§9.2).
- After successful upload: delete the work file; add the filename + size to the job footprint report.

### 7.5 Collectors

Every collector: returns a list of records of its schema, never throws on a single item failure (set field `null`, add `errors[]` to the record if useful), and stops at `limit` setting `truncated: true` in the manifest.

**Windows (x64, run elevated)**

| Command | APIs / method | Traps |
|---|---|---|
| `system_info` | `GetComputerNameExW`, `RtlGetVersion` (or registry `CurrentVersion`), `GetTimeZoneInformation`, `GetTickCount64` for boot time, `GetAdaptersAddresses` | Don't use `GetVersionEx` (lies). |
| `process` | `CreateToolhelp32Snapshot`/`Process32FirstW`; `OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION)`, `QueryFullProcessImageNameW`, `GetProcessTimes` (start time), `OpenProcessToken` + `LookupAccountSidW` (user); command line via WMI `Win32_Process` (COM) | Access denied on protected processes → leave fields `null`. **Identify processes by (pid, start_time)**, PIDs get reused. Convert FILETIME → UTC. |
| `account` / `session` | `NetUserEnum` (level 2: flags, last_logon), `NetLocalGroupGetMembers("Administrators")`, `WTSEnumerateSessionsW` + `WTSQuerySessionInformationW` | Local accounts only in MVP. |
| `net_conn` | `GetExtendedTcpTable(TCP_TABLE_OWNER_PID_ALL)` for v4/v6, `GetExtendedUdpTable` | Join PID → process name from the process snapshot. |
| `file` | `std::filesystem` recursive iterator + `GetFileAttributesExW`/`GetFileTime`; owner via `GetNamedSecurityInfoW`; SHA-256 if `hash` | Skip reparse points (avoid loops); `\\?\` long paths; permissions errors are normal. |
| `log_event` | `EvtQuery(channel, XPath time+EventID filter)`, `EvtNext`, `EvtRender(EvtRenderEventXml)`; parse `<System>` and `<EventData><Data Name=…>` into `fields{}` | Skip message rendering. Security log requires elevation. Event 4688 only exists if "Audit Process Creation" is enabled in the lab. |
| `persistence` | Registry `Run`/`RunOnce` (HKLM+HKCU), services (registry `HKLM\SYSTEM\CurrentControlSet\Services`: ImagePath, Start), Startup folders, scheduled tasks via `schtasks /query /fo csv /v` (execute without shell) | Tasks are P1-quality; services/run keys are the priority. |

**Ubuntu / Linux (x64, run as root)**

| Command | Method | Traps |
|---|---|---|
| `system_info` | `uname`, `/etc/os-release`, `/proc/uptime` or `/proc/stat btime`, timezone from `/etc/timezone` or `timedatectl show`, interface IPs via `getifaddrs` | |
| `process` | iterate `/proc/<pid>`: `stat` (ppid, starttime tick #22), `status` (Uid), `cmdline` (NUL-separated), `exe` (readlink), `cwd` | Start time = `btime + starttime/CLK_TCK`. Process may vanish mid-read (ENOENT → skip). Kernel threads have empty cmdline/exe. `exe` ending in ` (deleted)` → set `ext.exe_deleted=true`. |
| `account` / `session` | `/etc/passwd`, `/etc/group` (sudo/admin membership), `/etc/shadow` **not read**; sessions from `utmp` via `who`/`last -F` parsing or `utmpx` API; last login from `lastlog` (best effort) | Don't read password hashes. |
| `net_conn` | parse `/proc/net/{tcp,tcp6,udp,udp6}`; map inode → PID by scanning `/proc/*/fd/*` for `socket:[inode]` | IPv4 addresses in these files are little-endian hex; IPv6 is 4 little-endian 32-bit words. |
| `file` | `std::filesystem` + `lstat`; owner via `getpwuid`; times: mtime, atime, ctime (as `changed_utc`); SHA-256 if `hash` | Don't follow symlinks. |
| `log_event` | `auth`: read `/var/log/auth.log` if present, else `journalctl -o json --since … SYSLOG_FACILITY=4 SYSLOG_FACILITY=10` (exec without shell). `syslog`: `/var/log/syslog`. `journal`: `journalctl -o json`. Parse with the regex set in §8.4 | Syslog lines have no year → assume current year, handle year rollover. |
| `persistence` | `/etc/crontab`, `/etc/cron.*/*`, `/var/spool/cron/crontabs/*`, `/etc/systemd/system/*.service` + `~/.config/systemd/user`, `authorized_keys` files, `/etc/rc.local`, shell rc files (`.bashrc`, `.profile`) *existence + mtime only* | |

### 7.6 Agent hygiene (the legitimate "don't get flagged" story)
- One agent binary, **code-signed** in production; in the lab, self-signed with the hash published in the dashboard so test machines can add an explicit allow/exclusion — the same workflow a real SOC would use.
- Documented OS APIs only; read-only; no process injection, no kernel drivers, no persistence, no obfuscation, no self-deletion tricks.
- Behaviour declared up front: the dashboard shows "what this agent does" (the command list) and logs every agent action in custody events.
- Lab README documents the exact Defender/AV exclusion steps used and records what AV did during testing (include in the validation report honestly).

---

## 8. Data contracts

### 8.1 Evidence manifest (`schemas/manifest.schema.json`)

```json
{
  "schema_version": 1,
  "evidence_id": "uuid-v4 (agent generated)",
  "job_id": "uuid", "run_id": "uuid", "plan_id": "uuid", "investigation_id": "uuid",
  "host_id": "uuid", "agent_id": "uuid",
  "op_id": "op3", "command": "collect_processes", "record_type": "process",
  "platform": "windows",
  "collector_version": "0.1.0",
  "collected_start_utc": "2026-10-17T09:30:01.100000Z",
  "collected_end_utc":   "2026-10-17T09:30:02.900000Z",
  "agent_clock_utc": "2026-10-17T09:30:02.901000Z",
  "record_count": 123, "truncated": false,
  "file_name": "op3_collect_processes.jsonl",
  "size_bytes": 48211,
  "sha256": "hex",
  "signature_b64": "Ed25519 over sha256_hex|evidence_id|job_id|host_id",
  "footprint": { "work_files": ["op3_collect_processes.jsonl"], "work_bytes": 48211 }
}
```

### 8.2 Common record envelope
Every JSONL line has:

```json
{ "rt": "process", "sv": 1, "host_id": "uuid", "collected_utc": "…Z", "ext": { } , ...type fields }
```
`ext` holds OS-specific extras. Unknown/unavailable values are `null`, never empty strings.

### 8.3 Record types

**`system_info`**: `hostname, os_name, os_version, arch, boot_utc, timezone, utc_offset_minutes, domain|null, ips[], agent_version, elevated(bool), collected_by_user`

**`process`**: `pid, ppid, name, image_path|null, cmdline|null, user|null, start_utc|null, image_sha256|null` · ext: win `{session_id, wow64}`, linux `{uid, state, exe_deleted}`
Key for identity: `(host_id, pid, start_utc)`.

**`account`**: `username, id (SID|uid), full_name|null, home|null, shell|null, enabled|null, is_admin(bool), groups[], last_logon_utc|null`

**`session`**: `username, session_type (console|rdp|ssh|local|other), source_ip|null, started_utc|null, session_id|null`

**`net_conn`**: `proto (tcp|udp), state (LISTEN|ESTABLISHED|TIME_WAIT|…|null), local_addr, local_port, remote_addr|null, remote_port|null, pid|null, process_name|null`

**`file`**: `path, name, is_dir, size|null, owner|null, mode_or_attrs|null, created_utc|null, modified_utc|null, accessed_utc|null, changed_utc|null (linux ctime), sha256|null`

**`log_event`**: `source ("Security"|"auth"|…), ts_utc, event_id|null, provider|null, level|null, user|null, src_ip|null, pid|null, action|null (normalized, see below), fields{} (Windows EventData / parsed Linux fields), raw (Windows: event XML truncated to 8 KB; Linux: original line)`

**`persistence`**: `kind (run_key|service|scheduled_task|startup_folder|cron|systemd_unit|authorized_keys|rc_local|shell_rc), name, location, target|null (executable/command), user|null, enabled|null, modified_utc|null`

### 8.4 Log normalization (`action` values)
Normalized `action` ∈ `logon_success, logon_failure, logoff, account_created, group_member_added, process_created, service_installed, task_created, sudo_command, other`.

Windows mapping: 4624→`logon_success` · 4625→`logon_failure` · 4634/4647→`logoff` · 4688→`process_created` · 4720→`account_created` · 4732→`group_member_added` · 7045→`service_installed` · 4698→`task_created`.

Linux regex patterns (agent-side; tolerate extra whitespace; each yields `action`, `user`, `src_ip` where present):
- `Failed password for (invalid user )?(?P<user>\S+) from (?P<ip>\S+) port` → `logon_failure`
- `Invalid user (?P<user>\S+) from (?P<ip>\S+)` → `logon_failure`
- `Accepted (password|publickey) for (?P<user>\S+) from (?P<ip>\S+) port` → `logon_success`
- `new user: name=(?P<user>[^,]+)` (useradd) → `account_created`
- `add '(?P<user>[^']+)' to (shadow )?group '(?P<group>[^']+)'` or `usermod.*-aG` → `group_member_added`
- `sudo:\s+(?P<user>\S+) : .* COMMAND=(?P<cmd>.*)` → `sudo_command`
Unmatched lines → `action: "other"` (kept, so the timeline is complete).

### 8.5 Derived: Entities, Events, Findings, Graph (server side)

**Event** (`schemas/events.schema.json`):
```json
{ "event_id": "sha256-based stable id", "host_id": "…", "ts_utc": "…Z", "ts_adjusted_utc": "…Z",
  "kind": "process_start|file_created|file_modified|logon|logon_failure|account_created|group_member_added|service_installed|persistence_modified|last_logon|log_other",
  "summary": "one-line human text", "entities": ["process:…","user:…"], "evidence_id": "…", "record_ref": "line number in evidence file", "severity": "info|low|medium|high" }
```
Derivation: `process.start_utc`→`process_start`; `file.created_utc/modified_utc`→`file_created/file_modified` (accessed time ignored); `session.started_utc`→`logon`; `account.last_logon_utc`→`last_logon`; `log_event`→kind from `action`; `persistence.modified_utc`→`persistence_modified`. `net_conn` has no timestamp: graph only.
`event_id` = first 16 hex of SHA-256(`host_id|kind|ts_utc|evidence_id|record_ref`) → deterministic.

**Finding** (`schemas/findings.schema.json`): `{finding_id, rule_id, title, severity, host_ids[], entity_keys[], evidence_refs[{evidence_id, record_ref}], explanation, attack_tags[]}`.

**Graph** (`schemas/graph.schema.json`): `{nodes:[{id,type,label,host_id,attrs,flagged_by:[finding_id]}], edges:[{id,type,source,target,attrs}]}`.

---

## 9. Server

### 9.1 Data model (SQLite DDL sketch)

```sql
operators(id TEXT PK, username TEXT UNIQUE, pw_hash TEXT, role TEXT CHECK(role IN ('admin','investigator')), created_utc TEXT);
investigations(id TEXT PK, name TEXT, case_ref TEXT, description TEXT, status TEXT, created_by TEXT, created_utc TEXT);
hosts(id TEXT PK, hostname TEXT, os TEXT, os_version TEXT, arch TEXT, first_seen_utc TEXT, last_seen_utc TEXT);
agents(id TEXT PK, host_id TEXT, pubkey_b64 TEXT, token_hash TEXT, version TEXT, status TEXT, last_seen_utc TEXT, clock_offset_ms INTEGER, enrolled_utc TEXT);
enrollment_tokens(token_hash TEXT PK, created_by TEXT, expires_utc TEXT, used_utc TEXT);
investigation_hosts(investigation_id TEXT, host_id TEXT, PRIMARY KEY(investigation_id, host_id));
scripts(id TEXT PK, investigation_id TEXT, name TEXT, created_utc TEXT);
script_versions(id TEXT PK, script_id TEXT, version INTEGER, source TEXT, source_sha256 TEXT, created_by TEXT, created_utc TEXT); -- immutable
runs(id TEXT PK, investigation_id TEXT, script_version_id TEXT, status TEXT, created_by TEXT, created_utc TEXT, finished_utc TEXT);
jobs(id TEXT PK, run_id TEXT, host_id TEXT, agent_id TEXT, plan_id TEXT, plan_envelope_json TEXT, status TEXT, ops_json TEXT, created_utc TEXT, started_utc TEXT, finished_utc TEXT, error TEXT);
evidence(id TEXT PK, investigation_id TEXT, run_id TEXT, job_id TEXT, host_id TEXT, agent_id TEXT, op_id TEXT, command TEXT, record_type TEXT, file_path TEXT, size_bytes INTEGER, sha256 TEXT, record_count INTEGER, truncated INTEGER, manifest_json TEXT, collected_end_utc TEXT, received_utc TEXT, ledger_block_height INTEGER, ledger_block_hash TEXT);
custody_events(id INTEGER PK AUTOINCREMENT, evidence_id TEXT NULL, investigation_id TEXT, seq INTEGER, event_type TEXT, actor_type TEXT, actor_id TEXT, ts_utc TEXT, details_json TEXT, prev_hash TEXT, hash TEXT);
analysis_results(id INTEGER PK AUTOINCREMENT, investigation_id TEXT, kind TEXT CHECK(kind IN ('events','graph','findings','diff')), version INTEGER, input_evidence_ids_json TEXT, result_json TEXT, generated_utc TEXT);
operator_audit(id INTEGER PK AUTOINCREMENT, operator_id TEXT, action TEXT, target TEXT, ts_utc TEXT, details_json TEXT);
```
Events are stored inside `analysis_results(kind='events')` as JSON and mirrored into an `events` table for paged queries if needed (`events(id, investigation_id, host_id, ts_adjusted_utc, kind, severity, summary, entities_json, evidence_id, record_ref)` with indexes on `(investigation_id, ts_adjusted_utc)` and `(investigation_id, host_id)`).

### 9.2 API (REST, JSON; all `/api`; operator endpoints require `Authorization: Bearer <jwt>`)

**Operator**

| Method & path | Purpose |
|---|---|
| `POST /auth/login` | `{username,password}` → `{token, role}` |
| `GET/POST /investigations` · `GET /investigations/{id}` | list/create/get |
| `GET /hosts` · `POST /investigations/{id}/hosts` | list known hosts; attach hosts to investigation |
| `POST /enrollment-tokens` | create single-use agent token (returns token once + server fingerprint info) |
| `GET /agents` | agent status, last seen, clock offset |
| `GET/POST /investigations/{id}/scripts` · `POST /scripts/{id}/versions` | scripts and immutable versions |
| `POST /scripts/check` | `{source}` → compiler diagnostics (for the editor) |
| `POST /investigations/{id}/runs` | `{script_version_id, host_ids[]}` → creates run + jobs |
| `GET /runs/{id}` · `GET /jobs/{id}` | status incl. per-op progress |
| `GET /investigations/{id}/evidence` (filters: host, type, run) · `GET /evidence/{id}` · `GET /evidence/{id}/download` (logged as `DOWNLOADED`) · `GET /evidence/{id}/records?cursor=&limit=&q=` | evidence browser |
| `POST /evidence/{id}/verify` · `POST /investigations/{id}/verify-all` | integrity verification (§10.4) |
| `GET /evidence/{id}/custody` · `GET /investigations/{id}/custody` | custody chain |
| `GET /investigations/{id}/timeline?from=&to=&host=&kind=&q=&cursor=&limit=&time=adjusted\|raw` | timeline |
| `GET /investigations/{id}/graph?host=&focus=<node_id>&depth=` | graph (server-side capped at 500 nodes; `focus` returns neighborhood) |
| `GET /investigations/{id}/findings` · `GET /investigations/{id}/diff` | findings, multi-host diff |
| `POST /investigations/{id}/analyze` | re-run analysis (idempotent) |
| `WS /ws?token=` | events: `agent.status`, `job.status`, `job.progress`, `evidence.new`, `analysis.started/done`, `verify.result` |

**Agent** (`Authorization: Bearer <agent_token>`)

| Method & path | Purpose |
|---|---|
| `POST /agent/enroll` | §7.2 (token in body, no bearer) |
| `POST /agent/heartbeat` | `{agent_clock_utc}` → `{server_utc, clock_offset_ms}`; server stores offset |
| `GET /agent/jobs/next` | long-poll, returns `{job_id, envelope}` or 204 |
| `POST /agent/jobs/{job_id}/progress` | `{op_id,status,record_count?,error?}` |
| `POST /agent/jobs/{job_id}/evidence` | multipart: `manifest` (JSON) + `file` |
| `POST /agent/jobs/{job_id}/complete` | `{status, footprint, summary}` |

Server evidence-ingest algorithm (all steps must pass, else `422` + custody event `REJECTED`):
1. Validate manifest against schema; check `job_id`/`agent_id`/`host_id` coherence and that the job is in an active state.
2. Stream file to a temp path while computing SHA-256; compare to `manifest.sha256`.
3. Verify `signature_b64` with the agent's stored public key.
4. Validate records against `schemas/records/<type>.schema.json` (sampled: first 100 + random 100 with fixed seed; full validation in `strict` mode for tests).
5. Move to `evidence_repo/<investigation_id>/<host_id>/<sha256[:2]>/<sha256>.jsonl` (content-addressed; write-once, file mode read-only).
6. Insert `evidence` row; custody events `COLLECTED` (from manifest) → `RECEIVED` → `STORED`.
7. Register in ledger (§10.3); on success custody `REGISTERED` with block height/hash. If ledger unavailable: evidence status `PENDING_LEDGER`, retried by a background task; dashboard shows a warning (never silently skip).

Job status machine: `QUEUED → CLAIMED → RUNNING → COMPLETE | PARTIAL | FAILED | REJECTED | TIMEOUT` (server marks `TIMEOUT` if no progress for `max_runtime_seconds + 60`). Run status = `RUNNING` until all jobs terminal, then `ANALYZING`, then `COMPLETE` (or `COMPLETE_WITH_ERRORS`).

Errors: JSON `{error:{code,message,details?}}` with proper HTTP codes. Pagination: cursor-based, `limit` ≤ 1000.

---

## 10. Evidence, custody, ledger

### 10.1 Repository rules
Content-addressed, write-once, files stored read-only; DB holds metadata; the **ledger** (not the DB) is the integrity authority.

### 10.2 Custody chain
Per-evidence (and per-investigation for investigation-level events) append-only chain in `custody_events`:
`hash = SHA256( prev_hash | event_type | actor_type | actor_id | ts_utc | canonical_json(details) )` with `prev_hash = "0"*64` for the first event. Canonical JSON in Python: `json.dumps(obj, sort_keys=True, separators=(",", ":"), ensure_ascii=False)`.

Event types (P0): `COLLECTED, RECEIVED, STORED, REGISTERED, VERIFIED, ANALYZED, DOWNLOADED, REJECTED`. Operator actions are also written to `operator_audit`.

### 10.3 JOCKY Ledger service
Separate process, separate SQLite file, separate Ed25519 key, separate bearer token for writers. Intended to run on another machine/VM during the demo.

**Block:** `{height, ts_utc, record, record_sha256, prev_hash, hash, signature_b64}`
- `record_sha256 = SHA256(canonical_json(record))`
- `hash = SHA256( height | prev_hash | record_sha256 | ts_utc )`
- `signature_b64 = Ed25519(ledger_key, hash)`
- Genesis block: height 0, `prev_hash = "0"*64`, record `{"type":"GENESIS"}`.

**Record types:** `EVIDENCE_REGISTERED` `{evidence_id, sha256, size_bytes, investigation_id, host_id, agent_id, plan_id, collected_end_utc, custody_head_hash}` · `VERIFICATION_RESULT` `{evidence_id, result, checked_utc, operator_id}` (P0 only if cheap).

**API:** `POST /records` (writer token) → block · `GET /records/by-evidence/{evidence_id}` · `GET /blocks?from=&limit=` · `GET /head` · `GET /verify-chain` (recomputes hashes + signatures from genesis) · `GET /pubkey`.

**Interface** (`server/app/services/ledger_client.py`): `LedgerBackend.register(record) -> BlockRef`, `get(evidence_id)`, `verify_chain()`. Implementations: `HashChainLedger` (P0), `EvmLedger` (P1; Hardhat/Anvil local node + contract below).

**EVM contract (P1):**
```solidity
contract EvidenceAnchor {
  event Anchored(uint256 indexed id, bytes32 evidenceHash, bytes32 investigationId, bytes32 hostId, uint256 ts);
  mapping(bytes32 => uint256) public anchorOf;   // evidenceHash => id (0 = none)
  uint256 public count;
  function anchor(bytes32 evidenceHash, bytes32 investigationId, bytes32 hostId) external returns (uint256) {
    require(anchorOf[evidenceHash] == 0, "exists");
    uint256 id = ++count; anchorOf[evidenceHash] = id;
    emit Anchored(id, evidenceHash, investigationId, hostId, block.timestamp);
    return id;
  }
}
```

### 10.4 Verification algorithm (`POST /evidence/{id}/verify`)

Returns `{result: PASS|FAIL, checks:[{name, ok, detail}], checked_utc}` and writes custody `VERIFIED` + WS `verify.result`.

| # | Check | Pass condition |
|---|---|---|
| 1 | File hash | SHA-256 of bytes on disk == `evidence.sha256` (DB) |
| 2 | Ledger hash | SHA-256 on disk == hash stored in **ledger record** (fetched from ledger, not DB) |
| 3 | Agent signature | Ed25519 valid with the stored agent public key |
| 4 | Custody chain | Recomputed hash chain matches stored hashes, no gaps in `seq` |
| 5 | Custody head | Custody head at registration time == `custody_head_hash` in ledger record (and later events chain from it) |
| 6 | Ledger chain | Block `hash`/`signature` valid and chain from this block to head verifies |

Demo tamper test: modify one byte in the stored `.jsonl` → checks 1 and 2 fail → overall FAIL, UI highlights which check and why. Second variant: edit the DB `sha256` to match the tampered file → check 2 still fails (ledger disagrees). Third: edit custody rows → check 4/5 fail.

---

## 11. Analysis engine (`jocky_analysis`, pure Python, deterministic)

Pipeline: `load evidence (by investigation) → normalize → events → graph → findings → diff → persist as analysis_results`. CLI: `python -m jocky_analysis run --fixtures fixtures/linux fixtures/windows --out out/` (used by Nazeer from day 1, before real agents exist).

### 11.1 Normalization
- Parse JSONL, validate lightly, attach `evidence_id` + `record_ref`.
- Apply clock offset: `ts_adjusted = ts_utc + agent.clock_offset_ms` (offset sampled nearest to collection time; record stored with the event).
- Build **entity keys**: `process:<host>:<pid>:<start_utc>`, `user:<host>:<username>`, `file:<host>:<path>`, `endpoint:<addr>:<port>`, `persistence:<host>:<kind>:<name>`, `host:<host_id>`.

### 11.2 Timeline
Union of derived events (§8.5), sorted by `(ts_adjusted_utc, host_id, kind, event_id)`. Query-time filters, text search on `summary`. Density histogram endpoint optional (P1). Severity of an event is raised to the max severity of any finding that references its entities (post-findings pass).

### 11.3 Evidence graph

**Node types:** `host, user, process, file, endpoint, persistence, finding_anchor(optional)`.

**Edges (7):**

| Edge | From → To | Rule |
|---|---|---|
| `SPAWNED` | process → process | child.ppid == parent.pid and parent.start_utc ≤ child.start_utc (same host) |
| `RUNS_AS` | process → user | process.user matches account username |
| `CONNECTS_TO` | process → endpoint | `net_conn` with remote addr, via pid |
| `LISTENS_ON` | process → endpoint(local port, host) | `net_conn` state LISTEN |
| `LOGGED_ON` | user → host | `session` or `logon_success` log event |
| `OWNS` | user → file | file.owner == username (only for files retained by filters) |
| `PERSISTS_VIA` | persistence → process/file | persistence.target path equals process.image_path or file.path (case-insensitive on Windows) |

Constraints: cap 500 nodes returned; default view = nodes touched by findings + their 1-hop neighbours; `focus`/`depth` for exploration. Never render all processes by default.

### 11.4 Correlation rules (P0: exactly these 6)

| ID | Name | Logic | Severity | ATT&CK tag (optional) |
|---|---|---|---|---|
| R1 | Brute-force then success | ≥ 5 `logon_failure` for same target user (or same `src_ip`) within 10 min, followed by a `logon_success` for that user within 10 min of the last failure | high | T1110 |
| R2 | New privileged account | `account_created` followed by `group_member_added` (admin/sudo) for the same user within 24 h | high | T1136 |
| R3 | Suspicious-path process with outbound connection | Process image in a user-writable/temp path (`\AppData\Local\Temp\`, `\Windows\Temp\`, `\Users\Public\`, `/tmp/`, `/dev/shm/`, `/var/tmp/`, hidden dir `/.`) **and** has ESTABLISHED connection to non-loopback remote | high | T1071 |
| R4 | Persistence to suspicious path | Persistence `target` in the same path set as R3 | high | T1547 |
| R5 | Masquerading system process | Name in {`svchost.exe`,`lsass.exe`,`services.exe`,`csrss.exe`,`winlogon.exe`,`explorer.exe`, `sshd`, `systemd`, `cron`} but image path outside the expected directory (Windows `C:\Windows\System32\` or `C:\Windows\`; Linux `/usr/sbin/`, `/usr/bin/`, `/lib/systemd/`, `/bin/`, `/sbin/`), **or** `svchost.exe` whose parent is not `services.exe` | high | T1036 |
| R6 | Unique listener | A LISTEN `(proto, port)` present on exactly one host among ≥ 3 same-OS hosts in the investigation | medium | — |

Each finding includes a plain-English `explanation` and exact `evidence_refs` (evidence_id + record_ref) so the dashboard can jump to the source record. Rules are pure functions in `analysis/rules/rNN_*.py`, each with fixtures: one that triggers and one that doesn't.

### 11.5 Multi-host diff
For hosts of the **same OS** in the investigation, per category compute key sets:

| Category | Key |
|---|---|
| processes | `(lower(name), lower(image_path))` |
| accounts | `username` |
| listeners | `(proto, local_port)` |
| persistence | `(kind, lower(name), lower(target))` |
| services (win) / systemd units (linux) | `(name)` from persistence |

Output per host and category: `only_here[]`, `missing_here[]` (present on ≥ 50% of peers), and `rarity = 1 − hosts_with_item / N`. Items with `rarity ≥ 0.66` and N ≥ 3 are marked `notable`. Cross-OS comparison: only `accounts` and `listeners` and clearly labelled "cross-OS". With N = 2 hosts, report differences only (no rarity claims). This is the whole "anomaly" feature.

### 11.6 Reproducibility
Analysis results include `{engine_version, rules_version, input_evidence_sha256_list_hash}`; re-running on the same inputs must produce byte-identical `result_json` except `generated_utc` (excluded from the equality test). CI has a test for this.

---

## 12. Dashboard (React + TypeScript)

### 12.1 Routes & screens

| Route | Content |
|---|---|
| `/login` | username/password |
| `/` | Investigations list + create |
| `/inv/:id` | Layout with tabs below, shows run status bar (live via WS) |
| `/inv/:id/hosts` | Enrolled hosts attached to investigation, agent status (online/offline, last seen, clock offset), "Add enrollment token" modal |
| `/inv/:id/scripts` | Script list; **editor** (CodeMirror with JOCKY syntax highlighting, live `check` diagnostics from `POST /scripts/check`, "Save version", "Run on…" host picker) |
| `/inv/:id/runs` | Runs → jobs → per-op progress (live), errors, footprint |
| `/inv/:id/evidence` | Evidence table (host, type, command, size, hash prefix, ledger height, status), detail drawer with records viewer (virtualized, search) |
| `/inv/:id/integrity` | "Verify all" button; per-evidence result with the 6 checks; custody chain timeline; ledger chain status |
| `/inv/:id/timeline` | Filters (host, kind, severity, time range, text), adjusted/raw toggle, virtualized list, click → record source |
| `/inv/:id/graph` | Cytoscape graph, default = findings-centered; node click → details + "expand neighbours"; filter by host/type; legend |
| `/inv/:id/findings` | Findings list with severity, explanation, jump links to evidence/graph/timeline |
| `/inv/:id/compare` | Host × category diff tables, `notable` highlighted |

### 12.2 UX rules
- Every number is clickable down to the evidence record.
- Show **ledger/verification status badge** on every evidence row (🟢 verified, ⚪ unverified, 🔴 failed).
- Live updates via a single WebSocket hook; reconnect with backoff.
- Dark/light not required. Accessibility: keyboard-navigable tables, sufficient contrast.
- `DEMO_OFFLINE=1` (server env) serves a pre-recorded run built from `fixtures/` through the real pipeline.

---

## 13. Security model (hackathon-grade, honest)

| Area | Implementation |
|---|---|
| Operators | bcrypt passwords, JWT (1 h), 2 roles (`admin` can enroll/revoke agents, `investigator` can run/view), every action in `operator_audit` |
| Agent auth | One-time enrollment token → per-agent bearer token (stored hashed server-side); revocable in dashboard |
| Transport | HTTPS everywhere; agent pins the server cert fingerprint; ledger on HTTPS too |
| Plans | Ed25519-signed, agent verifies; max age 24 h |
| Evidence | SHA-256 + agent signature + ledger registration + custody chain |
| Agent privileges | Runs elevated by operator action; no persistence; no network listener |
| Parsers | All inputs treated as hostile: bounds checks, size caps (`limit`), no `system()` / shell, `execve` with argv arrays only; fuzz the compiler lexer/parser and the log regex parsers in CI (short runs) |
| Secrets | Keys in `./secrets/` with 600 perms, never committed; `.gitignore` enforced; `gen_keys.py` creates server signing key, ledger key, JWT secret, TLS self-signed cert |
| Known gaps (state them) | no mTLS, no HSM, no multi-party ledger consensus, no formal validation/certification, agent compromise undetectable beyond signature/hash checks |

---

## 14. Test lab & demo scenario

### 14.1 Lab topology (minimum)
- `WIN-A` (Windows 10/11 VM) — enable "Audit Process Creation" and "Audit Logon" for events 4688/4624/4625.
- `UBU-A`, `UBU-B` (Ubuntu 22.04+ VMs) — `openssh-server` installed, rsyslog on.
- Optional `WIN-B` for a Windows pair.
- `SRV` (server + dashboard) and `LEDGER` (separate VM or at least separate container/process; separate machine strongly preferred).
- Host requirement: 16 GB RAM minimum for 4 small VMs + server; or use team laptops as hosts. Take VM snapshots: `clean`, `infected`.

### 14.2 Planted-artifact scenario (benign simulation on lab-owned machines)
Scripts live in `lab/scenario/`; each has a matching `cleanup` script.

On **infected** hosts (`WIN-A`, `UBU-A`; `UBU-B` stays clean as baseline):
1. Create local user `svc_backup`; add it to Administrators / `sudo`. → R2
2. Generate ≥ 6 failed logins for `svc_backup` (wrong password via RDP/SSH from another lab machine), then one success. → R1
3. Place a **harmless** long-running program in a temp path, renamed to mimic a system process name (e.g., a copy of `sleep` / a trivial script wrapper on Linux, a trivial test executable on Windows), start it, and have it hold an outbound TCP connection to a lab-controlled listener on `SRV`. → R3, R5
4. Add a persistence entry (Run key / cron entry) pointing at that program. → R4
5. On `UBU-A` also open an unusual listening port. → R6 (with `UBU-B` and a third Ubuntu host or `WIN` pair as appropriate; if only two Ubuntu hosts exist, R6 is shown as "diff-only")

**Expected outputs** committed in `fixtures/scenario_expected/`: the set of finding `rule_id`s per host, graph path `user → process → endpoint`, and the diff items. The e2e test compares against these.

### 14.3 Oracle validation (the "we compared with native tools" evidence)
`lab/oracle/compare_{win,linux}.(ps1|sh)` run native tools and compare to JOCKY output:
- Processes: `tasklist /v` / `ps -eo pid,ppid,user,lstart,cmd` → compare PID set, names (≥ 98 % overlap allowing for churn).
- Network: `netstat -ano` / `ss -tunap` → compare local/remote endpoint sets (≥ 95 %).
- Users: `net user` / `/etc/passwd`.
- Logs: `Get-WinEvent` counts for given event IDs/time window; `journalctl --since` counts.
Results go into `docs/VALIDATION.md` with timestamps and honest deltas.

### 14.4 Demo script (5–7 min)
1. Dashboard → hosts online (3–4 agents, clock offsets visible).
2. Open script editor → show the example script, introduce a typo → live compiler diagnostic → fix.
3. Run on all hosts → live progress.
4. Evidence browser → show hashes + ledger heights.
5. Findings → R1–R5 → click one → graph path → timeline context.
6. Compare tab → infected host stands out.
7. Integrity page → PASS.
8. Tamper a stored file in a terminal → verify → FAIL with exact reason.
9. Close with honest scope slide (what's next: more artifacts, mTLS, validation).
Backup: the same sequence as a screen recording + `DEMO_OFFLINE=1`.

---

## 15. Build plan

### 15.1 Milestones (weeks assume 10; compress per §1.3)

| Milestone | Weeks | Deliverables | Acceptance test |
|---|---|---|---|
| **M0 Contracts** | 1 | Repo, CI skeleton, `schemas/*`, `openapi.yaml`, `fixtures/` for both OSes (hand-made but realistic), language spec frozen, key generation script | CI validates fixtures against schemas; `jockyc` stub parses example script; OpenAPI lints |
| **M1 Walking skeleton** | 2–3 | Compiler builds a signed Plan; server runs/jobs; agent (Linux first) enrolls, polls, verifies, runs a **fake** collector, uploads with hash+signature; evidence stored; ledger registers; dashboard shows host, run, evidence | `scripts/e2e_skeleton.sh` passes in CI; tamper test fails verify |
| **M2 Real collectors + lab** | 3–6 | All P0 collectors on both OSes; lab VMs + scenario scripts; log parsing; oracle scripts; custody + verify complete | Oracle thresholds met; JSONL validates against schemas on real machines |
| **M3 Analysis + views** | 5–8 | Normalize/timeline/graph/rules/diff against real evidence; dashboard timeline/graph/findings/compare/integrity | Scenario expected outputs match; reproducibility test green |
| **M4 Full E2E + freeze** | 8–9 | Whole demo flow on lab; feature freeze for P0; start P1 only if M4 green | `lab/run_demo.sh` end-to-end on real VMs; backup video recorded |
| **M5 Hardening + polish** | 9–10 | Error handling, docs, validation report, fuzz runs, rehearsals ×3, P1 items if time | Demo rehearsed 3× by different presenters; `VALIDATION.md` complete |

Weekly rhythm: Monday plan, Friday integration run of the full pipeline on the lab (not "it works on my machine"), contract changes announced in channel.

### 15.2 Ownership (load-balanced)

| Person | Owns | Approx. pd |
|---|---|---|
| **Ranjith** (tech lead) | Language spec, compiler, Plan signing, `core/` lib, CI, integration & e2e tests | ~32 |
| **Sandeep** | Windows collectors, Windows build/packaging, Windows lab VMs, scenario scripts (Windows side), demo logistics | ~25 |
| **Sankar** | **Agent core** (shared C++), Linux collectors, Linux lab VMs, scenario scripts (Linux side) | ~24 |
| **Archana** | **Server core** (REST/DB/jobs/auth/WS), evidence repo, custody, ledger service + `LedgerBackend`, verification | ~23 (+5 for EVM P1) |
| **Nazeer** | `jocky_analysis`: normalization, timeline, graph, rules, diff, reproducibility tests | ~22 |
| **Lakshana** | Dashboard (all screens), UX, demo mode & recording | ~25 |

This deliberately differs from the Master Context: moving server core to Archana and agent core to Sankar prevents Lakshana (server + dashboard) and Ranjith (compiler + agent + integration) from being the two-person bottleneck. Cross-help rule: when someone finishes, they pull the next-largest unstarted item, not new features.

### 15.3 Parallelization notes
- Nazeer and Lakshana start at M0 using `fixtures/` and a mock server (FastAPI serving fixtures) — no waiting for agents.
- Archana's ingest endpoint and Sankar's agent are developed against each other from M1 using the fake collector.
- Sandeep's Windows collectors can be developed standalone (`jocky-agent --run-collector processes --out file.jsonl`) and validated against schemas before the Windows agent joins the loop.
- If dashboard load is too high: cut per §1.3 emergency list; pair Nazeer on Timeline/Graph views once the analysis API is stable.

---

## 16. Testing & validation

| Layer | Tests |
|---|---|
| Compiler | Table-driven valid/invalid scripts with exact expected diagnostics; golden Plan output for the example script; signature verify/tamper tests; fuzz lexer/parser (short CI run) |
| Agent | Plan verification (good/bad/expired/unknown-key); collector unit tests with recorded `/proc` trees and Windows API mocks where practical; schema validation of every collector output; hash/sign round-trip against Python verifier |
| Server | Ingest algorithm failure cases (bad hash, bad signature, wrong job, bad schema, ledger down); job state machine; API contract tests from `openapi.yaml`; auth tests |
| Ledger | Chain build/verify, tamper detection (edit a record → `verify-chain` fails), restart persistence |
| Analysis | Per-rule trigger/non-trigger fixtures; graph edge rules; diff semantics with N=2/3/4; **byte-identical determinism**; golden outputs for scenario fixtures |
| End-to-end | CI: Linux fake-agent pipeline (every PR). Lab: real VM run (weekly + before demo): scenario expected findings match, oracle thresholds met, tamper tests fail as expected |
| Security | Secrets-in-repo scan; dependency audit; confirm agent refuses unsigned Plans; confirm agent has no listening sockets |

Definition of Done for MVP:
- [ ] Walkthrough in §14.4 runs on real VMs without manual fixes, 3 times in a row.
- [ ] All six rules fire on the scenario and **don't** fire on the clean baseline host (except documented diffs).
- [ ] Verify = PASS on clean evidence, FAIL on each of the 3 tamper variants.
- [ ] Oracle comparison documented in `docs/VALIDATION.md`.
- [ ] Backup video + offline mode present.
- [ ] Slides use only the honest claims (§1.6).

---

## 17. Risk register

| Risk | L | I | Mitigation | Trigger → action |
|---|---|---|---|---|
| Windows collectors slip | H | H | Time-box; partial fields OK; standalone collector CLI; Sandeep pairs with Sankar early on shared helpers | End of week 5 not done → drop `persistence`, `hash_images`, scheduled tasks |
| Integration surprises | H | H | Walking skeleton M1; weekly lab run | Skeleton not green by end of week 3 → stop feature work, all-hands fix |
| Dashboard overload | M | H | Fixtures + mock server from week 1; emergency cuts | Behind at M3 → table views instead of charts |
| Schema change mid-way | M | H | Freeze at M0; version bump + fixture update rule | Any change → announce + update all consumers same day |
| AV flags the agent in lab/demo | M | H | Sign; documented exclusions; test early (week 3) on fresh Windows | Flagged on demo machine → pre-agreed exclusion, show as "SOC allowlisting workflow" |
| Ledger unavailable during demo | L | H | Ledger on separate stable box; `PENDING_LEDGER` retry; recorded backup | — |
| LLVM scope creep | M | M | D1; P2 only after M4 | Anyone starts LLVM before M4 → stop |
| Clock skew breaks timelines | M | M | Offset capture (D10); VMs on NTP; show offsets in UI | — |
| Overclaiming to judges | M | H | §1.6 | — |
| Team burnout / exams | M | H | 10-week plan with slack; P1 is explicitly optional | — |

---

## 18. Appendix

### 18.1 `DECISIONS.md` entry format
```
YYYY-MM-DD · D-<n> · <short title>
Context: <1-2 lines>   Decision: <1 line>   Alternatives rejected: <list>   Owner: <name>
```

### 18.2 `STUBS.md` entry format
```
<path> · <what is faked> · <reason> · <removal milestone>
```

### 18.3 Glossary
- **Plan:** signed, compiled form of a JOCKY script, executed by the agent.
- **Run:** one script version dispatched to N hosts; produces N Jobs.
- **Evidence item:** one JSONL file + manifest from one op on one host.
- **Custody chain:** hash-linked sequence of events about an evidence item.
- **Ledger:** separate append-only hash-chained, signed record store for evidence hashes and key custody events.
- **Triage collection:** targeted live collection of selected artifacts (not imaging).
