# gocan
Standardized CAN bus logging, database handling, and format adapters for raw and structured CAN data.

# gocan

**gocan** is a lightweight framework for **standardizing CAN bus logs, databases, and raw data handling**, with adapters for converting between common CAN formats.

Modern automotive and embedded development relies heavily on CAN data, but logging formats are fragmented across different tools and ecosystems (CSV, JSON, BLF, MDF4, custom binaries, etc.).
`gocan` provides a **clean, consistent, and tool-independent foundation** for recording, processing, and converting CAN data.

The project focuses on **simplicity, interoperability, and performance**.

---

# Motivation

CAN logs are produced by many different tools, each with its own format:

* Vector tools → **BLF**
* Measurement tools → **MDF4**
* Custom scripts → **CSV / JSON**
* Embedded systems → **binary logs**

This fragmentation causes several issues:

* Difficult data sharing between teams
* Vendor tool lock-in
* Hard to build reusable data pipelines
* Inefficient conversion between formats

`gocan` solves this by introducing a **standard internal representation of CAN data** and a **modular adapter architecture**.

---

# Core Goals

* Provide a **standard structure for CAN logs**
* Support **raw CAN frame logging**
* Enable **format conversion between common CAN formats**
* Support **CAN database (DBC) integration**
* Provide a **simple developer-friendly API**
* Enable **high-performance logging pipelines**

---

# Architecture

`gocan` is organized into several modular components:

```
gocan
│
├── log/
│   Standardized CAN log structures
│
├── raw/
│   Raw CAN frame representation
│
├── db/
│   CAN database handling (DBC support)
│
├── adapters/
│   Format conversion adapters
│   ├── csv
│   ├── json
│   ├── blf
│   └── mdf4
│
└── tools/
    CLI utilities and helper tools
```

Each adapter converts external formats into the **canonical internal CAN frame model**.

---

# Canonical CAN Frame Model

All input formats are internally mapped to a simple frame structure:

```
type CANFrame struct {
    Timestamp uint64
    ID        uint32
    DLC       uint8
    Data      [8]byte
    Flags     uint8
}
```

This structure acts as the **intermediate representation** for all adapters.

---

# Supported Formats

| Format     | Status  | Notes                    |
| ---------- | ------- | ------------------------ |
| CSV        | ✓       | Human-readable CAN logs  |
| JSON       | ✓       | Structured pipelines     |
| Raw Binary | ✓       | High-performance logging |
| BLF        | Planned | Vector logging format    |
| MDF4       | Planned | Measurement data format  |

---

# Example Usage

### Reading a CSV CAN Log

```
reader := csv.NewReader("log.csv")

for reader.Next() {
    frame := reader.Frame()
    fmt.Println(frame.ID, frame.Data)
}
```

---

### Writing Raw CAN Frames

```
writer := raw.NewWriter("capture.gocan")

writer.Write(frame)
```

---

# CLI Tools (Planned)

Future releases will include CLI utilities:

```
gocan convert log.blf log.csv
gocan inspect capture.gocan
gocan decode log.gocan dbc.dbc
```

These tools will allow easy conversion, inspection, and decoding of CAN logs.

---

# Use Cases

`gocan` can be used in many scenarios:

* Automotive CAN logging
* Embedded system telemetry
* CAN IDS research datasets
* Data conversion between automotive tools
* Vehicle diagnostics and analytics
* Large-scale CAN dataset preprocessing

---

# Design Principles

The project follows several design principles:

* **Minimal core**
* **Clear and explicit data structures**
* **Vendor-neutral tooling**
* **High performance**
* **Extensible adapter system**

---

# Roadmap

Planned development milestones:

* [ ] Core CAN frame abstraction
* [ ] CSV adapter
* [ ] JSON adapter
* [ ] Raw binary logger
* [ ] DBC decoding support
* [ ] BLF reader
* [ ] MDF4 reader
* [ ] CLI utilities
* [ ] Streaming pipeline support
* [ ] IDS dataset tooling

---

# Contributing

Contributions are welcome.

Possible areas for contribution:

* New format adapters
* Performance improvements
* Dataset tooling
* Documentation improvements

If you want to contribute:

1. Fork the repository
2. Create a feature branch
3. Submit a pull request

---

# License

MIT License

---

# Vision

The long-term goal of `gocan` is to become a **simple, open, and standardized foundation for CAN data logging and processing**, enabling easier development, research, and data exchange across automotive and embedded ecosystems.
