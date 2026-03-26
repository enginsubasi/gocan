# GoCAN

**Standardized CAN Bus Logging, Database Handling, and Format Adapters for Raw and Structured CAN Data**

*Version 1.0 | March 2026*

---

**GoCAN** is a lightweight, open-source framework for **standardizing CAN bus logs, databases, and raw data handling**, with adapters for converting between common CAN formats and a three-tier CSV data structure optimized for AI/ML intrusion detection research.

Modern automotive and embedded development relies heavily on CAN data, but logging formats are fragmented across different tools and ecosystems (CSV, JSON, BLF, MDF4, custom binaries, etc.). GoCAN provides a **clean, consistent, and tool-independent foundation** for recording, processing, converting, and analyzing CAN data.

---

## 1. Motivation

CAN logs are produced by many different tools, each with its own format:

- Vector tools → **BLF**
- Measurement tools → **MDF4**
- Custom scripts → **CSV / JSON**
- Embedded systems → **binary logs**

This fragmentation causes several issues: difficult data sharing between teams, vendor tool lock-in, hard-to-build reusable data pipelines, and inefficient conversion between formats. Beyond format differences, public CAN bus datasets used in IDS research (HCRL, SynCAN, ROAD/ORNL, X-CANIDS, CAN-MIRGU, can-train-and-test, and others) each use different column names, data types, encoding conventions, and labeling schemes — forcing researchers to write one-off parsers for every dataset and making cross-dataset benchmarking and reproducibility difficult.

GoCAN solves both problems by introducing a **standard internal representation of CAN data**, a **three-tier CSV schema** for AI/ML consumption, and a **modular adapter architecture** for format conversion.

## 2. Design Principles

- **ML-first data types:** All columns consumed by models use native numeric types (integers, floats, booleans). Hex strings are relegated to optional display columns, never to primary fields.
- **Three-tier derivation:** Each tier is derived from the one below it. Raw is the ground truth; Standard adds computed features; Enhanced adds decoded signals. A user can always regenerate a higher tier from a lower one.
- **DBC-free ML operation:** The Standard tier is deliberately independent of any DBC file, making it suitable for cross-vehicle and cross-manufacturer studies where signal definitions are unavailable or proprietary.
- **CSV as the canonical format:** Plain CSV with UTF-8 encoding ensures readability, tool compatibility, and version-control friendliness. No binary formats or proprietary containers.
- **Minimal core:** The framework stays small and focused — clear data structures, vendor-neutral tooling, and an extensible adapter system.
- **High performance:** Raw binary logging and streaming pipeline support for high-throughput capture scenarios.
- **Companion metadata:** Every dataset release includes a structured JSON sidecar file for provenance, reproducibility, and automated cataloging.

## 3. Architecture

GoCAN is organized into several modular components:

```
gocan/
│
├── log/                    Standardized CAN log structures (three-tier schema)
│
├── raw/                    Raw CAN frame representation
│
├── db/                     CAN database handling (DBC support)
│
├── adapters/               Format conversion adapters
│   ├── csv                     CSV ↔ GoCAN
│   ├── json                    JSON ↔ GoCAN
│   ├── blf                     BLF → GoCAN (Vector)
│   ├── mdf4                    MDF4 → GoCAN (Measurement)
│   ├── candump                 candump → GoCAN (Linux SocketCAN)
│   ├── asc                     ASC → GoCAN (Vector ASCII)
│   └── pcap                    PCAP → GoCAN (Wireshark)
│
├── pipeline/               Tier derivation pipeline (Raw → Standard → Enhanced)
│
└── tools/                  CLI utilities and helper tools
```

Each adapter converts external formats into the **canonical internal CAN frame model**, which then maps directly to the GoCAN Raw tier.

### Canonical CAN Frame Model

All input formats are internally mapped to a simple frame structure:

```
type CANFrame struct {
    Timestamp   float64     // Epoch seconds, µs precision
    Channel     uint8       // Bus channel index
    ID          uint32      // Arbitration ID (integer)
    IsExtended  bool        // 29-bit extended frame flag
    IsRemote    bool        // RTR flag
    DLC         uint8       // Data Length Code
    Data        []byte      // Payload bytes
    IsFD        bool        // CAN FD flag
    IsError     bool        // Error frame flag
    Label       string      // Attack/anomaly label (see Section 5)
}
```

This structure acts as the **intermediate representation** for all adapters and maps 1:1 to the Raw tier CSV columns.

## 4. Three-Tier Data Format

The core of GoCAN is a three-tier CSV structure where each tier is derived from the one below it. This enables both raw data preservation and ML-ready feature engineering in a single, consistent schema.

```
Source Format ──▶ Raw ──▶ Standard ──▶ Enhanced
  (candump,       │        │              │
   BLF, ASC,      │        │              │
   PCAP, MF4,     │        │              ▼
   CSV, JSON)     │        │         Requires DBC
                   │        ▼
                   │   Computed features only
                   ▼   (no DBC needed)
              Type normalization,
              column mapping
```

### 4.1 Tier 1 — Raw

The Raw tier is the closest representation to what comes off the CAN bus or a logging tool. It undergoes minimal transformation: only type normalization (e.g., hex ID to integer) and column renaming. No derived features are computed. Any CAN logging format can be mapped into this tier via the adapter system.

| Column | Type | Description |
|---|---|---|
| `timestamp` | float64 | Epoch time in seconds with microsecond precision |
| `channel` | uint8 | Bus channel index (0, 1, …) |
| `arbitration_id` | uint32 | Integer CAN arbitration ID (e.g., 193) |
| `arbitration_id_hex` | string | Hex display form (e.g., 0x0C1) — optional convenience column |
| `is_extended` | bool | Extended frame flag (29-bit ID if true, 11-bit if false) |
| `is_remote` | bool | Remote Transmission Request (RTR) frame flag |
| `dlc` | uint8 | Data Length Code: 0–8 for CAN 2.0, 0–64 for CAN FD |
| `data` | uint8[] | Comma-separated byte array (e.g., `0,255,18,171,0,0,0,0`) |
| `is_fd` | bool | CAN FD frame flag |
| `is_error` | bool | Error frame flag |
| `label` | string | Attack/anomaly label: `normal`, or attack type. Empty if unlabeled |

**Filename convention:** `<dataset_id>_raw.csv`

> **Design note:** The `arbitration_id` is stored as uint32 (integer), not hex string, to eliminate hex parsing overhead in every ML pipeline. The optional `arbitration_id_hex` column provides human readability for debugging without penalizing model training.

### 4.2 Tier 2 — Standard

The Standard tier is derived from Raw by computing protocol-level and statistical features that are universally applicable across vehicles and bus configurations. No DBC file or domain-specific signal knowledge is required. This tier is the primary input for most unsupervised and transfer-learning-based intrusion detection models.

The Standard tier **retains all Raw columns** and appends the following:

| Column | Type | Description |
|---|---|---|
| `delta_t` | float64 | Inter-arrival time: seconds since previous message with the same `arbitration_id` |
| `delta_t_global` | float64 | Seconds since the previous message on the bus (any ID) |
| `payload_int` | uint64 | Full payload as a single big-endian unsigned integer |
| `hamming_prev` | uint8 | Bitwise Hamming distance from previous payload of the same ID |
| `byte_entropy` | float32 | Shannon entropy across the DLC payload bytes |
| `msg_freq_window` | float32 | Message frequency of this ID over a rolling window (default 1 s) |
| `seq_index` | uint32 | Global monotonic message counter (0-based) |
| `id_seq_index` | uint32 | Per-ID monotonic message counter (0-based) |

**Filename convention:** `<dataset_id>_standard.csv`

> **Design note:** The Standard tier is intentionally DBC-free. This is critical for cross-vehicle domain adaptation research, where the same model must generalize across vehicles with different DBC mappings. Features like `delta_t`, `hamming_prev`, and `byte_entropy` capture behavioral patterns without requiring signal semantics.

### 4.3 Tier 3 — Enhanced

The Enhanced tier is derived from Standard by decoding CAN payloads using a DBC (or equivalent signal database) file. This tier is **one row per signal**, not one row per frame, since a single CAN frame can carry multiple multiplexed signals. A `frame_ref` column back-references to the originating frame's `seq_index`.

The Enhanced tier **retains all Standard columns** (at the frame level, repeated per signal row) and appends the following:

| Column | Type | Description |
|---|---|---|
| `frame_ref` | uint32 | Back-reference to `seq_index` of the originating frame |
| `signal_name` | string | Decoded signal name (e.g., `VehicleSpeed`) |
| `signal_value` | float64 | Physical value after DBC scaling and offset |
| `signal_unit` | string | Engineering unit (e.g., km/h, rpm, °C) |
| `signal_min` | float64 | DBC-defined minimum valid value |
| `signal_max` | float64 | DBC-defined maximum valid value |
| `out_of_range` | bool | True if `signal_value` falls outside [`signal_min`, `signal_max`] |
| `ecu_source` | string | Transmitting ECU name from DBC |
| `message_name` | string | DBC message name (e.g., `EMS_Status`) |
| `dbc_version` | string | DBC file identifier or SHA-256 hash for traceability |

**Filename convention:** `<dataset_id>_enhanced.csv`

> **Design note:** The row expansion (one row per signal) is intentional. It enables signal-level anomaly detection and makes SHAP-based explainability directly attributable to named physical signals. The `dbc_version` field ensures that re-decoding with an updated DBC is traceable.

### 4.4 Tier Derivation Rules

- **Source Format → Raw:** Type normalization, column mapping, hex-to-integer conversion. Source-specific adapters handle candump, BLF, ASC, PCAP, MF4, and custom CSV variants.
- **Raw → Standard:** Computed features (`delta_t`, `hamming_prev`, `byte_entropy`, `msg_freq_window`, counters). Requires only the Raw CSV and a configuration specifying the rolling window size (default: 1 second).
- **Standard → Enhanced:** DBC-based signal decoding with row expansion. Requires the Standard CSV and a valid DBC file. The DBC file hash is recorded in `dbc_version` for traceability.

The derivation is one-directional: Enhanced cannot be reverse-engineered to Standard (signal decoding is lossy in the general case due to rounding), but Standard can always be regenerated from Raw, and Enhanced can always be regenerated from Standard plus DBC.

## 5. Label Taxonomy

The `label` column uses a controlled vocabulary to ensure consistent labeling across datasets, tools, and research groups. Every frame must carry exactly one label value.

### 5.1 Benign Label

| Label | Description |
|---|---|
| `normal` | Legitimate CAN traffic with no known anomaly or attack |

### 5.2 Attack Labels — Injection Category

Attacks where the adversary **injects** additional frames onto the bus.

| Label | Description | Typical Signature |
|---|---|---|
| `dos` | Denial of Service — high-frequency flooding of one or more IDs | Abnormally short `delta_t`, burst of a single ID |
| `fuzzing` | Random ID and/or payload injection | Previously unseen `arbitration_id` values, high `byte_entropy` |
| `replay` | Replayed recording of legitimate traffic | Valid payloads but anomalous timing (`delta_t` deviation) |
| `spoofing_drive` | Spoofed driving-related signals (e.g., speed, steering, throttle) | Valid ID with manipulated payload, out-of-range `signal_value` |
| `spoofing_rpm` | Spoofed engine RPM signal | Targeted single-ID injection with abnormal physical values |
| `spoofing_gear` | Spoofed gear/transmission signal | Targeted single-ID injection with abnormal physical values |
| `spoofing_generic` | Spoofed signal not covered by a specific sub-label | Catch-all for targeted spoofing on any ID |

### 5.3 Attack Labels — Suppression Category

Attacks where the adversary **suppresses or manipulates** existing frames.

| Label | Description | Typical Signature |
|---|---|---|
| `suspension` | Targeted suppression of a specific ID (bus-off attack, ECU disconnect) | Missing expected messages, increased `delta_t` for the target ID |
| `drop` | Selective frame dropping (partial suppression) | Intermittent gaps in `id_seq_index` continuity |
| `masquerade` | Attacker disables legitimate ECU and impersonates it | Subtle timing and payload deviations on an existing ID |

### 5.4 Attack Labels — Diagnostic / Protocol Category

Attacks exploiting CAN diagnostic or protocol-level features.

| Label | Description | Typical Signature |
|---|---|---|
| `diagnostic` | Unauthorized UDS/OBD-II diagnostic requests (e.g., 0x7DF, 0x7E0–0x7E7) | Diagnostic arbitration IDs appearing outside a service context |
| `exploit` | Known CVE or ECU firmware exploit payload delivery | Specific byte patterns targeting known vulnerabilities |

### 5.5 Special Labels

| Label | Description |
|---|---|
| *(empty string)* | Unlabeled frame — ground truth is unknown. Used for unsupervised / real-world captures without annotation |
| `unknown` | Frame flagged as anomalous by a detection system but not yet classified into a specific attack type |

### 5.6 Label Rules

- Labels are **lowercase**, **snake_case**, and **ASCII-only**.
- Each frame carries exactly **one** label. If a frame is affected by multiple simultaneous attacks, use the most specific applicable label.
- The `attack_types` array in the metadata sidecar must list every distinct label value (excluding `normal` and empty) present in the dataset.
- When mapping from an existing dataset whose labels don't match the taxonomy above, use the closest GoCAN label and document the mapping in a `label_mapping.json` file shipped alongside the dataset.
- New attack labels may be proposed via pull request. They must include a description, typical signature, and at least one public dataset or reference where the attack is demonstrated.

### 5.7 Cross-Dataset Label Mapping Reference

The table below shows how labels from major public datasets map to GoCAN labels:

| Source Dataset | Original Label | GoCAN Label |
|---|---|---|
| HCRL | DoS | `dos` |
| HCRL | Fuzzy | `fuzzing` |
| HCRL | Spoofing (RPM) | `spoofing_rpm` |
| HCRL | Spoofing (Gear) | `spoofing_gear` |
| SynCAN | plateau, continuous, playback | `spoofing_generic`, `spoofing_generic`, `replay` |
| ROAD / ORNL | flooding | `dos` |
| ROAD / ORNL | targeted ID | `spoofing_generic` |
| ROAD / ORNL | masquerade | `masquerade` |
| X-CANIDS | Normal, Attack | `normal`, *(use specific sub-label based on attack description)* |
| CAN-MIRGU | Normal, Attack | `normal`, *(use specific sub-label based on attack description)* |
| can-train-and-test | DoS, Fuzzy, Impersonation | `dos`, `fuzzing`, `masquerade` |

## 6. Metadata Sidecar

Every GoCAN dataset release must include a companion JSON metadata file named `<dataset_id>_meta.json`. This file provides provenance, capture context, and integrity verification. It enables automated dataset catalogs and reproducibility.

| Field | Type | Description |
|---|---|---|
| `dataset_id` | string | Unique identifier (UUID or slug) |
| `vehicle_make` | string | Manufacturer (e.g., Hyundai, Toyota) |
| `vehicle_model` | string | Model name |
| `vehicle_year` | uint16 | Model year |
| `bus_type` | string | `CAN 2.0A` \| `CAN 2.0B` \| `CAN FD` |
| `bus_speed` | string | Bus bitrate (e.g., 500 kbps, 2 Mbps for FD) |
| `capture_tool` | string | Logging hardware/software (e.g., PCAN-USB, candump) |
| `capture_duration_s` | float64 | Total capture duration in seconds |
| `total_frames` | uint64 | Total number of CAN frames in the dataset |
| `unique_ids` | uint32 | Count of distinct arbitration IDs observed |
| `has_dbc` | bool | Whether a DBC file is available for Enhanced tier |
| `attack_types` | string[] | List of attack/anomaly types present (empty if benign-only) |
| `tier` | string | `raw` \| `standard` \| `enhanced` |
| `license` | string | SPDX license identifier (e.g., `CC-BY-4.0`) |
| `sha256_checksum` | string | SHA-256 hash of the CSV file for integrity verification |

### Example Metadata File

```json
{
  "dataset_id": "hcrl_sonata_2023",
  "vehicle_make": "Hyundai",
  "vehicle_model": "Sonata",
  "vehicle_year": 2017,
  "bus_type": "CAN 2.0B",
  "bus_speed": "500 kbps",
  "capture_tool": "PCAN-USB",
  "capture_duration_s": 300.0,
  "total_frames": 1523847,
  "unique_ids": 42,
  "has_dbc": false,
  "attack_types": ["dos", "fuzzing", "spoofing_rpm", "spoofing_gear"],
  "tier": "standard",
  "license": "CC-BY-4.0",
  "sha256_checksum": "a1b2c3d4e5f6..."
}
```

## 7. Supported Formats

| Format | Direction | Notes |
|---|---|---|
| CSV | ↔ | Human-readable CAN logs |
| JSON | ↔ | Structured pipelines |
| Raw Binary | ↔ | High-performance logging (`.gocan`) |
| candump | → GoCAN | Linux SocketCAN text logs |
| ASC | → GoCAN | Vector ASCII trace format |
| BLF | → GoCAN | Vector binary logging format |
| MDF4 | → GoCAN | Measurement data format |
| PCAP | → GoCAN | Wireshark CAN captures |

## 8. Dataset Compatibility Matrix

The following table maps well-known public CAN bus datasets to the GoCAN tiers they can support:

| Dataset | Raw | Standard | Enhanced |
|---|:---:|:---:|---|
| HCRL (KR) | ✅ | ✅ | ❌ No public DBC |
| SynCAN | ✅ | ✅ | ⚠️ Partial (synthetic signals provided) |
| ROAD / ORNL | ✅ | ✅ | ⚠️ Partial (partial DBC available) |
| X-CANIDS | ✅ | ✅ | ❌ No public DBC |
| CAN-MIRGU | ✅ | ✅ | ❌ No public DBC |
| can-train-and-test | ✅ | ✅ | ❌ No public DBC |

## 9. Usage

### Reading a CSV CAN Log

```go
reader := csv.NewReader("log.csv")

for reader.Next() {
    frame := reader.Frame()
    fmt.Println(frame.ID, frame.Data)
}
```

### Writing Raw CAN Frames

```go
writer := raw.NewWriter("capture.gocan")

writer.Write(frame)
```

### CLI Tools

```bash
gocan convert log.blf log.csv       # Format conversion
gocan inspect capture.gocan          # Log inspection
gocan decode log.gocan dbc.dbc       # DBC-based decoding
gocan derive raw.csv standard.csv    # Tier derivation (Raw → Standard)
gocan label-map hcrl raw.csv         # Apply label mapping from known dataset
```

## 10. Use Cases

- **CAN IDS research datasets** — Standardize heterogeneous public datasets into a single schema for cross-dataset benchmarking
- **Cross-vehicle transfer learning** — Use the Standard tier from multiple vehicles with the shared DBC-free feature space
- **Explainable AI / SHAP analysis** — Use the Enhanced tier for signal-level attribution
- **Automotive CAN logging** — Capture and store CAN traffic in a clean, portable format
- **Embedded system telemetry** — Log CAN data from ECUs and test benches
- **Data conversion** — Bridge between Vector, measurement, and open-source tooling
- **Vehicle diagnostics and analytics** — Decode and analyze CAN traffic with DBC support
- **Large-scale dataset preprocessing** — Pipeline from raw captures to ML-ready features

## 11. ML Research Guidelines

- **Unsupervised / anomaly-detection models** (autoencoders, GANs, isolation forests): Use the **Standard** tier. The DBC-free feature set ensures that models trained on one vehicle can be evaluated on another without feature re-engineering.
- **Supervised models with known attack labels:** Use the **Standard** tier with the `label` column. The label field follows the controlled vocabulary defined in [Section 5 — Label Taxonomy](#5-label-taxonomy). Use the cross-dataset mapping table (Section 5.7) when converting existing public datasets.
- **Explainable AI / SHAP-based analysis:** Use the **Enhanced** tier. Signal-level features allow attribution maps to reference physical quantities (e.g., `VehicleSpeed`, `EngineRPM`) rather than raw byte positions.
- **Cross-vehicle transfer learning / domain adaptation:** Use the **Standard** tier from multiple vehicles. The shared feature space (timing, entropy, Hamming distance) provides the basis for domain-invariant representations without requiring matched DBC files.

## 12. Versioning and Governance

GoCAN follows semantic versioning (`MAJOR.MINOR.PATCH`):

- **MAJOR** — Breaking schema changes (column removal, type change)
- **MINOR** — Backward-compatible additions (new optional columns)
- **PATCH** — Documentation or metadata corrections only

The current specification version is **1.0.0**.

## License

MIT License

## Contributing

Contributions are welcome. Possible areas for contribution:

- New format adapters (BLF, MDF4, ASC, PCAP)
- Performance improvements
- IDS dataset converters and tooling
- Label taxonomy extensions (new attack types via PR)
- Documentation improvements
- Python/Rust bindings

To contribute: fork the repository, create a feature branch, and submit a pull request on the [gocan](https://github.com/YOUR_USERNAME/gocan) repository.