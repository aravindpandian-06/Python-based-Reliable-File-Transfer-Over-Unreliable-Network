# Reliable File Transfer Over Unreliable Networks (RFTP)

A robust, modular, and fully tested transport-layer protocol implemented in Python over raw UDP sockets (`SOCK_DGRAM`). RFTP provides bulletproof reliability guarantees—packet framing, sequence ordering, acknowledgments, adaptive timeouts, retransmissions, sliding window flow control, and cryptographic integrity verification—on top of an inherently lossy, corrupting, and reordering network channel.

[![Python](https://img.shields.io/badge/Python-3.8%2B-blue.svg)](https://www.python.org/)
[![Protocol](https://img.shields.io/badge/Protocol-UDP%20Transport-green.svg)]()
[![Tests](https://img.shields.io/badge/Tests-58%2F58%20Passed-brightgreen.svg)]()
[![License](https://img.shields.io/badge/License-Academic%20Submission-orange.svg)]()

---

## Table of Contents
1. [Project Overview & Key Features](#project-overview--key-features)
2. [Core Networking Concepts Used](#core-networking-concepts-used)
3. [Binary Packet Specification & Framing Layout](#binary-packet-specification--framing-layout)
4. [Protocol Modes (ARQ Comparison)](#protocol-modes-arq-comparison)
5. [Protocol State Architecture & Sequence](#protocol-state-architecture--sequence)
6. [Interactive CLI & Live Visual Progress](#interactive-cli--live-visual-progress)
7. [Installation & Requirements](#installation--requirements)
8. [How to Run & Example Commands](#how-to-run--example-commands)
9. [Quantitative Performance Benchmark & Sample Results](#quantitative-performance-benchmark--sample-results)
10. [Automated Test Suite (58 Tests)](#automated-test-suite-58-tests)
11. [Known Limitations & Future Enhancements](#known-limitations--future-enhancements)
12. [Project File Summary](#project-file-summary)

---

## Project Overview & Key Features

Modern Internet applications depend on reliable data transfer, yet the underlying network layer (IP/UDP) is fundamentally best-effort: packets can be dropped by congested routers, corrupted by electrical noise, duplicated by multi-path routing, or delayed and reordered in transit. 

RFTP bridges this divide by implementing enterprise-grade ARQ (Automatic Repeat reQuest) transport mechanics from scratch using Python's standard library:

- **11-Byte Big-Endian Binary Framing**: Packed using Python's `struct` module with CRC-32 checksums.
- **Three Selectable Transport Modes**:
  1. **Selective Repeat (SR)**: Out-of-order receive buffering, per-packet timers, surgical retransmissions.
  2. **Go-Back-N (GBN)**: Window-based pipelining with cumulative acknowledgments and zero-buffer receiver.
  3. **Stop-and-Wait (SAW)**: Sequential single-packet lockstep transmission.
- **Pre-Flight Metadata Handshake**: Exchanges filename, file size, total chunks, and SHA-256 hash before data transmission begins.
- **RFC 6298 Adaptive RTT Estimator**: Dynamically measures Round-Trip Time with Jacobson/Karels EWMA smoothing and applies **Karn’s Algorithm** (excluding retransmitted packets from RTT estimation) with exponential backoff.
- **End-to-End Cryptographic Verification**: Automatic post-transfer SHA-256 hashing reporting unambiguous `PASS` or `FAIL`.
- **Integrated Network Impairment Simulator (`simulator.py`)**: Simulates loss, bit corruption, duplication, delay, and reordering with reproducible random seeds.
- **Interactive Terminal UI**: ANSI colored output and live dynamic progress bar tracking percent complete, throughput, and retransmissions in real-time.

---

## Core Networking Concepts Used

| Concept | Implementation in RFTP | Purpose / Value |
|---|---|---|
| **Binary Framing** | Big-Endian format `!BIHI` via `struct` | Ensures cross-platform network byte ordering across heterogeneous architectures. |
| **Error Detection** | 32-bit CRC-32 checksum (`zlib.crc32`) | Detects transmission bit flips and corrupted packets, discarding them immediately. |
| **Packet Sequencing** | 32-bit unsigned sequence numbering | Enables in-order byte stream reconstruction and accurate duplicate packet detection. |
| **Flow Control & Pipelining** | Sliding window sender & receiver | Keeps multiple packets in flight concurrently, dramatically maximizing throughput. |
| **Adaptive RTO** | RFC 6298 Jacobson/Karels algorithm | Dynamically matches timeout intervals to real-world network propagation latency. |
| **Karn's Algorithm** | Retransmission tagging & exclusion | Prevents acknowledgment ambiguity from corrupting the RTT estimator during loss. |
| **Out-of-Order Buffering** | Receiver gap-filling dictionary buffer | Prevents re-sending correctly delivered packets when preceding packets are lost. |
| **Connection Teardown** | 2-way FIN/FIN_ACK with 1.0s lingering | Prevents connection hangs caused by lost teardown packets (Two Generals' Problem). |
| **Cryptographic Integrity** | Streaming SHA-256 digest validation | Confirms bit-for-bit file equality between source and destination files upon transfer. |

---

## Binary Packet Specification & Framing Layout

Every packet transmitted over the wire comprises a fixed **11-byte header** serialized into Big-Endian network byte order, followed by an optional payload.

```text
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Packet Type  |                Sequence Number                |
|    (1 Byte)   |                   (4 Bytes)                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|        Payload Length         |           Checksum            |
|           (2 Bytes)           |           (4 Bytes)           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                           Payload                             |
|                        (0 - 65,535 B)                         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### Struct Field Specifications

| Field Name | Type | Size | Range | Purpose |
|---|---|:---:|---|---|
| **Packet Type** | `B` (unsigned char) | 1 Byte | `0` – `4` | Packet opcode identifier. |
| **Sequence Number** | `I` (unsigned int) | 4 Bytes | `0` – $2^{32}-1$ | Sequential packet index for ordering and deduplication. |
| **Payload Length** | `H` (unsigned short)| 2 Bytes | `0` – $65,535$ | Exact byte length of the payload trailing the header. |
| **Checksum** | `I` (unsigned int) | 4 Bytes | `0` – $2^{32}-1$ | CRC-32 checksum computed across header (with checksum=0) + payload. |
| **Payload** | `bytes` | Variable | Up to MTU/65KB | Application file data or JSON metadata handshake. |

### Packet Type Opcodes

| Opcode | Constant | Description |
|:---:|---|---|
| `0` | `DATA` | Conveys application file chunk data. |
| `1` | `ACK` | Acknowledgment confirming receipt of a packet. |
| `2` | `FIN` | Teardown request signaling sender completion. |
| `3` | `FIN_ACK` | Teardown acknowledgment confirming receiver close. |
| `4` | `METADATA` | Pre-flight handshake containing file size, name, total chunks, and expected SHA-256. |

---

## Protocol Modes (ARQ Comparison)

RFTP allows users to switch between three foundational ARQ strategies via the `--mode` flag:

```text
1. Selective Repeat (SR) [Default]:
   Sender:   [ 0 ][ 1 ][ 2 ][ 3 ][ 4 ]   <-- Window of N in flight concurrently
             Per-packet timers. If pkt 1 drops, ONLY pkt 1 is retransmitted!
   Receiver: [ 0 ]  [GAP]  [ 2 ][ 3 ]   <-- Out-of-order packets buffered in RAM.

2. Go-Back-N (GBN):
   Sender:   [ 0 ][ 1 ][ 2 ][ 3 ][ 4 ]   <-- Window of N in flight concurrently
             Single timer on base. If pkt 1 drops, entire window [1..4] retransmitted!
   Receiver: [ 0 ]  [DISCARD] [DISCARD]  <-- Zero buffering: discards out-of-order packets.

3. Stop-and-Wait (SAW):
   Sender:   [ 0 ] (wait for ACK 0) --> [ 1 ] (wait for ACK 1) ...
   Receiver: One packet at a time.
```

| Dimension | Selective Repeat (`--mode sr`) | Go-Back-N (`--mode gbn`) | Stop-and-Wait (`--mode saw`) |
|---|---|---|---|
| **Sender Window ($W$)** | $W > 1$ (Pipelined) | $W > 1$ (Pipelined) | $W = 1$ (Sequential) |
| **Receiver Window** | $W > 1$ (Buffered) | $1$ (Zero-buffering) | $1$ (Zero-buffering) |
| **Timers** | Per-packet timer | Single timer (`send_base`) | Single timer |
| **Retransmissions** | **Surgical**: Only timed-out packets | **Cascaded**: Retransmits full window | Current packet only |
| **Receiver Memory** | $O(W)$ packet buffer in RAM | $O(1)$ constant memory | $O(1)$ constant memory |
| **Resilience to Loss** | **High**: Efficiency stays >60–95% | **Poor**: High loss causes storm | **Low**: Link sits idle on timeout |

---

## Protocol State Architecture & Sequence

For detailed Mermaid state machine specifications and message sequence diagrams, see [`docs/protocol_spec.md`](docs/protocol_spec.md).

```text
SENDER LIFECYCLE:
[Disconnected] 
      │ (Initiate transfer)
      ▼
[Handshake] ──(Send METADATA, wait for ACK 0 with backoff)──> [Receiver ACKed]
                                                                    │
      ┌─────────────────────────────────────────────────────────────┘
      ▼
[Sliding Window Pipelining]
      ├─ Dispatch packets within window [send_base, send_base + W - 1]
      ├─ Concurrent ACK listener thread processes arrivals & slides base
      ├─ Timer thread checks unACKed packet timeouts & handles retries
      ▼ (All chunks ACKed: send_base == total_chunks)
[Teardown (FIN)] ──(Send FIN, await FIN_ACK)──> [Closed cleanly]

RECEIVER LIFECYCLE:
[Listening on UDP]
      │
      ▼
[Metadata Handshake] ──(Parse JSON, verify disk writable, send ACK 0)
      │
      ▼
[Data Receiving & Buffering]
      ├─ In-order chunk: Write to file, slide rcv_base, send ACK
      ├─ Out-of-order chunk: Buffer in RAM, send individual ACK
      ├─ Duplicate chunk: Re-send ACK, skip file write
      ├─ Corrupted chunk: Discard silently (fail CRC-32)
      ▼ (Receive FIN)
[Lingering Teardown] ──(Close file, send FIN_ACK, linger 1.0s for retransmitted FINs)
      │
      ▼
[SHA-256 Integrity Verification] ──> Print "PASS" or "FAIL" ──> [Closed]
```

---

## Interactive CLI & Live Visual Progress

RFTP features an ANSI-colored console interface and real-time terminal progress bar:

```text
=================================================================
Sender Started (Reliable Transport Protocol)
Protocol Mode  : SELECTIVE-REPEAT (Window Size = 8)
File to Send   : sample.bin (1048576 bytes, 1024 packets)
SHA-256 Digest : 9f86d081884c7d659a2feaa0c55ad015a3bf4f1b2b0b822cd15d6c15b0f00a08
Destination    : UDP 127.0.0.1:12000
Chunk Size     : 1024 bytes
Initial Timeout: 0.5 s | Max Retries: 15
=================================================================

[HANDSHAKE] Sending file metadata to receiver...
[HANDSHAKE] Receiver acknowledged metadata. Ready for data transfer.

[SENDING] [████████████████████░░░░░░░░]  68.4% | 700/1024 pkts | 482.1 KB/s | Retries: 4
[DONE] All file chunks acknowledged.

[FIN] Sending teardown request...
[FIN_ACK] Received teardown confirmation. Connection closed cleanly.

=================================================================
TRANSFER SUMMARY (Sender)
Protocol Mode       : SELECTIVE-REPEAT (Window Size = 8)
Status              : SUCCESS
Total Bytes Sent    : 1048576
Unique Packets      : 1024
Total Packets Sent  : 1032
Retransmissions     : 6
ACKs Received       : 1032
Elapsed Time        : 2.14 s
Throughput          : 478.32 KB/s
Efficiency          : 99.2% (1024/1032)
Final RTT Estimate  : RttEstimator(EstRTT=1.2ms, DevRTT=0.4ms, RTO=50.0ms, samples=1018)
=================================================================
```

---

## Installation & Requirements

### System Requirements
- **Python**: Version 3.8 or higher.
- **Operating System**: Windows 10/11, Linux, or macOS.
- **Dependencies**: The core transport protocols (`sender.py`, `receiver.py`, `packet.py`, `simulator.py`, `rtt.py`, `utils.py`) use **only standard library modules**.
- **Benchmarking & Plotting** (Phase 6): Requires `matplotlib`:
  ```bash
  pip install matplotlib
  ```

---

## How to Run & Example Commands

### 1. Basic Local Transfer (Selective Repeat)

**Terminal 1 — Start the Receiver:**
```bash
python receiver.py received_document.pdf --port 12000 --mode sr --window 8
```

**Terminal 2 — Start the Sender:**
```bash
python sender.py document.pdf --port 12000 --mode sr --window 8
```

---

### 2. Compare Protocol Modes

**Go-Back-N Mode:**
```bash
# Receiver:
python receiver.py out.bin --mode gbn --window 10
# Sender:
python sender.py in.bin --mode gbn --window 10
```

**Stop-and-Wait Mode:**
```bash
# Receiver:
python receiver.py out.bin --mode saw
# Sender:
python sender.py in.bin --mode saw
```

---

### 3. Transfer Under Simulated Adverse Network Conditions

The built-in network simulator allows testing against real-world network impairments:
- `--loss <float>`: Packet drop probability ($0.0 - 1.0$)
- `--corrupt <float>`: Bit inversion probability ($0.0 - 1.0$)
- `--dup <float>`: Packet duplication probability ($0.0 - 1.0$)
- `--delay <min-max>`: Artificial latency in ms (e.g. `50-150`)
- `--reorder <float>`: Packet reordering probability ($0.0 - 1.0$)
- `--seed <int>`: Deterministic seed for reproducible testing

```bash
# Receiver with 10% loss and 5% duplication:
python receiver.py output.bin --loss 0.10 --dup 0.05 --seed 42

# Sender with 10% loss, 5% bit corruption, and 100-200ms delay:
python sender.py input.bin --loss 0.10 --corrupt 0.05 --delay 100-200 --seed 42
```

---

### 4. Running Performance Benchmarks

Run the automated parameter sweep across 45 protocol configurations (5 loss rates $\times$ 4 window sizes) to generate CSV datasets and plots:
```bash
python benchmark.py
```
Generated outputs are stored in [`results/`](results/):
- `results/results.csv`
- `results/throughput_vs_loss.png`
- `results/retransmissions_vs_loss.png`
- `results/throughput_vs_window.png`
- `results/benchmark_summary.png`

---

## Quantitative Performance Benchmark & Sample Results

Transfers of a 100 KB payload (100 DATA packets + 1 METADATA + 1 FIN = 102 total packets) were executed across $W \in \{1, 4, 8, 16\}$ and loss rates $0\%, 5\%, 10\%, 20\%, 30\%$ with 100% SHA-256 PASS rate across all 45 test runs.

### Empirical Performance Summary Table

| Network Loss Rate | Metric | Stop-and-Wait ($W=1$) | Go-Back-N ($W=16$) | Selective Repeat ($W=16$) | Advantage |
| :---: | :--- | :---: | :---: | :---: | :---: |
| **0% Loss** | **Throughput** | 607.06 KB/s | **822.62 KB/s** | 641.85 KB/s | Pipelining saturates socket buffer |
| (Ideal channel) | **Retransmissions** | 0 | 0 | 0 | 100% clean delivery |
| | **Efficiency** | 98.04% | 98.04% | 98.04% | Zero redundant traffic |
| **10% Loss** | **Throughput** | 78.47 KB/s | 97.79 KB/s | **281.35 KB/s** | **SR is 3.58× faster than SAW** |
| (Typical wireless) | **Retransmissions** | 17 | 208 | **9** | **GBN causes 23× more retransmissions** |
| | **Efficiency** | 84.03% | 32.26% | **90.09%** | SR preserves link capacity |
| **30% Loss** | **Throughput** | 18.98 KB/s | 23.46 KB/s | **47.84 KB/s** ($W=4$) | Stop-and-Wait collapses on timeout waits |
| (Severe congestion) | **Retransmissions** | 43 | **653** (Storm!) | **64** | GBN retransmits full window repeatedly |
| | **Efficiency** | 68.97% | 13.25% | **60.24%** | GBN efficiency plummets to 13% |

### Graphical Benchmark Visualizations

| Throughput vs Loss Rate | Retransmissions vs Loss Rate |
| :---: | :---: |
| ![Throughput vs Loss](results/throughput_vs_loss.png) | ![Retransmissions vs Loss](results/retransmissions_vs_loss.png) |
| **Throughput vs Window Size ($W$)** | **Comprehensive Benchmark Dashboard** |
| ![Throughput vs Window](results/throughput_vs_window.png) | ![Benchmark Summary](results/benchmark_summary.png) |

Detailed theoretical and architectural commentary is documented in [`results/analysis.md`](results/analysis.md).

---

## Automated Test Suite (58 Tests)

The test suite covers packet encoding, RTT estimation, impairment simulation, protocol logic, edge cases, metrics, and multi-format end-to-end transfers:

```powershell
python -m unittest discover -s tests -p "test_*.py" -v
```

| Test Module | Tests | Description / Coverage |
|---|:---:|---|
| [`tests/test_packet.py`](tests/test_packet.py) | 21 | Binary packing, CRC-32 validation, header corruption, boundary limits, metadata serialization. |
| [`tests/test_rtt.py`](tests/test_rtt.py) | 5 | RFC 6298 EWMA updates, min/max RTO clamping, exponential backoff, Karn's rule exclusion. |
| [`tests/test_simulator.py`](tests/test_simulator.py) | 6 | Probabilistic dropping, bit inversion, duplication, delay range parsing, seed reproducibility. |
| [`tests/test_stop_and_wait.py`](tests/test_stop_and_wait.py) | 5 | Lockstep transfers, duplicate ACK rejection, timeout retransmission, 20% loss/10% corrupt. |
| [`tests/test_sliding_window.py`](tests/test_sliding_window.py) | 5 | Selective Repeat window advancement, Go-Back-N window retransmissions, out-of-order buffering. |
| [`tests/test_sender_stats.py`](tests/test_sender_stats.py) | 3 | Phase 6 `TransferStats` property validation, dict access, int backward compatibility. |
| [`tests/test_edge_cases.py`](tests/test_edge_cases.py) | 8 | 0-byte empty file, file smaller than 1 chunk, offline receiver, retry limits, lost FIN recovery. |
| [`tests/test_utils.py`](tests/test_utils.py) | 5 | File chunking generator, bit-flip corruption helper, probability drop bounds, SHA-256 computation. |
| [`tests/test_end_to_end.py`](tests/test_end_to_end.py) | 5 | **End-to-End Multi-Format**: Multiline UTF-8 text, PNG image binary, arbitrary firmware blobs under severe compound impairments. |
| [`tests/test_api.py`](tests/test_api.py) | 12 | **FastAPI & WebSocket**: CORS validation, endpoint verification, background transfer orchestration, download, and WebSocket streaming. |
| **Total Test Count** | **70** | **100% Passed (0 Failures, 0 Errors)** |

---

## Phase 8: Web Backend & Real-Time Monitoring (FastAPI + WebSocket)

RFTP provides a production-grade FastAPI web backend designed to power interactive dashboards (such as a React frontend on `http://localhost:5173`).

### 1. Starting the API Server
```bash
# Install dependencies:
pip install -r requirements.txt

# Start FastAPI backend on port 8000:
uvicorn api.main:app --reload --port 8000
```

### 2. Available Endpoints

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `GET` | `/` | Service health check and current transfer state. |
| `POST` | `/transfer/start` | Uploads a file with transfer parameters (`mode`, `window_size`, `loss`, `corruption`, `duplicate`, `delay`, `seed`) and starts background transfer. |
| `POST` | `/transfer/stop` | Cancels any ongoing transfer via thread-safe `stop_event`. |
| `GET` | `/transfer/status` | Returns live progress metrics, percent complete, sliding window state, throughput, and integrity result. |
| `GET` | `/results` | Returns Phase 6 benchmark results and available performance charts. |
| `GET` | `/results/charts/{file}` | Serves benchmark comparison charts as PNG images. |
| `GET` | `/download/{id}` | Downloads the reconstructed and SHA-256 verified file. |
| `WS` | `/ws/progress` | Real-time WebSocket streaming live JSON progress events to the frontend. |

### 3. WebSocket Event Stream (`/ws/progress`)
The WebSocket endpoint streams standardized JSON events in real time:
- `packet_sent`: Packet sequence number, length, and timestamp.
- `ack_received`: Acknowledged sequence number and updated window boundary.
- `packet_dropped`: Simulated loss event with sequence number and packet type.
- `retransmission`: Retransmission event with retry count and sequence number.
- `window_state`: Live sliding window base and next sequence numbers (`base`, `next_seq`).
- `percent_complete`: Percentage of file transmitted and acknowledged.
- `throughput`: Instantaneous transfer speed in KB/s.
- `final_hash_result`: End-to-end integrity check status (`PASS` or `FAIL`) and SHA-256 digests.

---

## Phase 9: Interactive React Frontend (Vite + Tailwind CSS + Framer Motion)

The web dashboard is housed in the [`frontend/`](frontend/) directory, built with **Vite, React 19, Tailwind CSS, Framer Motion, and Recharts**, styled in an ultra-premium **purple-and-white glassmorphic** design system.

### 1. Installation & Running Locally

```bash
# Terminal 1: Start FastAPI backend (Port 8000)
uvicorn api.main:app --reload --port 8000

# Terminal 2: Start Vite React Frontend (Port 5173)
cd frontend
npm install
npm run dev
```

Visit **`http://localhost:5173`** in your browser.

### 2. Frontend Features
- **Design System & Aesthetics**:
  - Curated purple palette: Primary `#7C3AED`, Deep `#5B21B6`, Light Lavender `#EDE9FE`, Soft Violet `#A78BFA`, Dark Purple `#2E1065`.
  - Floating blurred purple orbs in the background.
  - Reusable glassmorphic components (`GlassCard`, `GlassButton`, `GlassInput`, `GlassModal`, `GlassNavbar`).
  - Dark-purple theme toggle with smooth CSS variable transitions.
  - Typography powered by **Google Sans** with clean geometric fallbacks.
- **4-Slide Direction-Aware Wizard (`/transfer`)**:
  - **Slide 1**: Role selection (*Sender*, *Receiver*, *Both Local Demo*) with active purple glow.
  - **Slide 2**: Drag-and-drop file upload zone, queueing, and UDP endpoint inputs.
  - **Slide 3**: Transport protocol pipelining mode (*Selective Repeat*, *Go-Back-N*, *Stop-and-Wait*), sliding window slider, and network impairment sliders (*Loss %*, *Corruption %*, *Duplicate %*, *Delay ms*, *Seed*).
  - **Slide 4**: Real-time transmission controller with:
    - Flowing animated gradient progress bar.
    - Live **SlidingWindowView** displaying in-flight and acknowledged packet boxes.
    - **PacketTimeline** visualizing packet flow and drops between nodes.
    - Color-coded **EventLog**.
    - **ResultCard** featuring SHA-256 verification and confetti particle animation.
- **Benchmark Dashboard (`/dashboard`)**:
  - Live animated metric counters (Total Runs, Peak Throughput, Retransmissions, Success Rate).
  - 4 high-resolution **Recharts** visualizations in purple shades.
  - Interactive transfer history table with live search, sorting, and CSV data export.
  - "Run Benchmark" button triggering real-time parameter sweeps.
- **Interactive Protocol Specs (`/about`)**:
  - 11-byte packet header breakdown diagram.
  - Protocol state machine and pipelining comparison matrix.

---

## Known Limitations & Future Enhancements

1. **Fixed Window Flow Control (No Dynamic Congestion Control)**:
   RFTP uses a statically configured sliding window size ($W$). Future iterations could incorporate TCP Reno/CUBIC congestion control (Slow Start, Congestion Avoidance, Fast Retransmit, Fast Recovery with AIMD) to dynamically adapt window size to link capacity.
2. **Selective Acknowledgment Blocks (SACK)**:
   In GBN mode, acknowledging multiple disjoint blocks is not supported; implementing full SACK TCP options (RFC 2018) would allow GBN to recover faster from multi-packet drops without full window retransmissions.
3. **IPv6 Support**:
   Current implementation defaults to IPv4 (`AF_INET`). Adding dual-stack `AF_INET6` socket binding would allow operation across IPv6 networks.
4. **Path MTU Discovery (PMTUD)**:
   The packet payload size defaults to 1024 bytes. Implementing dynamic PMTUD would allow RFTP to automatically detect and maximize the MTU without IP-level packet fragmentation.

---

## Project File Summary

```text
CN Project/
│
├── config.py                  # Protocol configuration constants (PACKET_SIZE=1024, TIMEOUT=0.5s, WINDOW=5, etc.)
├── packet.py                  # 11-byte binary packet specification, make_packet(), parse_packet(), CRC-32, METADATA
├── rtt.py                     # RFC 6298 adaptive RTT estimator (Jacobson/Karels EWMA + Karn's algorithm)
├── simulator.py               # Network impairment simulator (UnreliableChannel: loss, corrupt, dup, delay, reorder)
├── sender.py                  # Multi-mode sender (Selective Repeat, Go-Back-N, Stop-and-Wait, Handshake, Live UI)
├── receiver.py                # Multi-mode receiver (Out-of-order buffering, SHA-256 verification PASS/FAIL, Lingering)
├── utils.py                   # Utilities: ANSI colors, ProgressBar, SHA-256 hashing, bit-flipping, logging
├── benchmark.py               # Automated parameter sweep benchmark & matplotlib graph generator
├── requirements.txt           # Project dependencies (FastAPI, Uvicorn, Matplotlib, Pytest, WebSockets)
│
├── api/                       # Phase 8 FastAPI web backend
│   ├── __init__.py
│   ├── main.py                # FastAPI routes, CORS configuration, WebSocket /ws/progress, download endpoints
│   ├── manager.py             # Background transfer lifecycle manager, event normalization, WebSocket broadcasting
│   └── storage/               # File storage for uploads and received transfers
│       ├── uploads/
│       └── received/
│
├── frontend/                  # Phase 9 Vite + React Frontend
│   ├── index.html             # HTML entry with Google Sans typography
│   ├── package.json           # Dependencies (react, tailwindcss, framer-motion, recharts, lucide-react)
│   ├── tailwind.config.js     # Custom purple palette, glass shadows, and keyframe animations
│   ├── vite.config.js         # Vite configuration with port 5173
│   └── src/
│       ├── api/apiClient.js   # REST API client & WebSocket connector
│       ├── components/        # GlassCard, GlassButton, GlassNavbar, SlidingWindowView, PacketTimeline, ResultCard
│       ├── context/           # ThemeContext (dark purple toggle) & ToastContext
│       ├── hooks/             # useTransfer, useWebSocket
│       ├── pages/             # Home, Transfer, Dashboard, About
│       └── utils/mockData.js  # Offline preview datasets and fallback simulation
│
├── docs/                      # Architectural documentation & specifications
│   └── protocol_spec.md       # Full protocol specification with Mermaid state diagrams & sequence diagrams
│
├── results/                   # Benchmark figures, datasets, and analysis
│   ├── results.csv            # Tabular metrics for all 45 benchmark configurations
│   ├── throughput_vs_loss.png # Matplotlib chart: Throughput (KB/s) vs Packet Loss Rate
│   ├── retransmissions_vs_loss.png # Matplotlib chart: Retransmissions vs Packet Loss Rate
│   ├── throughput_vs_window.png    # Matplotlib chart: Throughput vs Window Size
│   ├── benchmark_summary.png  # Multi-panel 4-in-1 comparison dashboard
│   └── analysis.md            # In-depth architectural performance report
│
├── tests/                     # 70 Unit & integration tests
│   ├── __init__.py
│   ├── test_packet.py         # Packet creation, parsing, CRC-32 validation, boundary checks
│   ├── test_rtt.py            # RFC 6298 EWMA updates, clamping, backoff, Karn's rule
│   ├── test_simulator.py      # Probabilistic impairment simulation tests & seed reproducibility
│   ├── test_stop_and_wait.py  # Stop-and-Wait lockstep transfers, duplicate ACK rejection
│   ├── test_sliding_window.py # Selective Repeat out-of-order buffering & Go-Back-N window retransmissions
│   ├── test_sender_stats.py   # Phase 6 TransferStats metrics, dictionary access, and typing
│   ├── test_edge_cases.py     # 0-byte file, small files, unreachable receiver, lost FIN recovery
│   ├── test_utils.py          # Chunking generator, SHA-256 calculation, bit flipping
│   ├── test_end_to_end.py     # End-to-end transfers: Text, valid PNG images, and random binary files
│   └── test_api.py            # Phase 8 FastAPI REST endpoints, CORS, background thread transfers, WebSockets
│
└── README.md                  # Comprehensive master project documentation
```
"# Python-based-Reliable-File-Transfer-Over-Unreliable-Network" 
