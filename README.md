# Multi-Operator 5G Walk-Test Campaign: Midtown Manhattan

Raw modem telemetry, latency probes and server-side logs from a nine-day
multi-operator 5G walk-test campaign along a fixed 6 km loop in Midtown
Manhattan, plus a two-day supplementary pilot covering a fourth carrier.

This is the companion dataset for:

> Wonyul Choi and Ahan Kak. 2026. *Three Phones, Nine Afternoons: An
> Experience Report on Multi-Operator 5G Walk-Testing in Manhattan.* In 20th
> ACM Workshop on Wireless Network Testbeds, Experimental evaluation &
> Characterization (WiNTECH '26), October 26–30, 2026, Austin, TX, USA.
> https://doi.org/10.1145/3831662.3844164

The paper's central argument is that the hard part of such campaigns is not
collecting the data but trusting it. This dataset is published in that
spirit: it includes the failed walks, the mislabelled phases and the
telemetry artifacts, not just the clean sessions. Read
[Known pitfalls](#known-pitfalls) before you compute anything.

---

## Contents

- [Read this first: anonymization](#read-this-first-anonymization)
- [Getting the data](#getting-the-data)
- [Quick start](#quick-start)
- [How the campaign worked](#how-the-campaign-worked)
- [Repository layout](#repository-layout)
- [Naming conventions](#naming-conventions)
- [Pairing operators](#pairing-operators)
- [Session inventory](#session-inventory)
- [File format: NSG JSON](#file-format-nsg-json)
- [File format: TWAMP CSV](#file-format-twamp-csv)
- [File format: iperf3 server logs](#file-format-iperf3-server-logs)
- [Field reference](#field-reference)
- [Known pitfalls](#known-pitfalls)
- [Worked examples](#worked-examples)
- [What is not in the dataset](#what-is-not-in-the-dataset)
- [Citing](#citing)

---

## Read this first: anonymization

Operators are labelled **Carrier A, B, C and D**, matching the paper. The
mapping to real operator names is not published.

This is a de-identified derivative of the raw capture. Operator names,
PLMNs, APNs, subscriber and handset identities, IMS/SIP signalling, server
addresses and globally-unique cell identities have been removed or replaced
with consistent pseudonyms. Radio measurements, throughput, bands, physical
cell identities, GPS and timestamps are byte-identical to the raw capture.

The short version of what this means in practice:

- Cell identifiers (`cellIdentity`, `ECI`, `CellID`, `NR Global Cell
  Identity`, `TAC`, `LAC`) are synthetic. Distinct cells stay distinct, so
  cell **counts** are exact — but you cannot look them up in OpenCelliD, and
  you cannot recover site grouping by bit-shifting them.
- Physical cell identities (`PCI`, `physCellId`) are **real**.
- IP addresses are in RFC 5737 documentation ranges. `192.0.2.0/24` is our
  own measurement hosts; `198.51.100.0/24` and `203.0.113.0/24` are other
  public addresses.
- `PCAPPacket` (raw PDU hex) is absent from every record. The decoded RRC
  and NAS structures that NSG produced from those PDUs are retained.

---

## Getting the data

Each walk day ships as its own gzipped tar archive at the repository root,
so you can fetch only the days you need. Together they are about 623 MB;
unpacked the dataset is 7.1 GB.

| Archive | Compressed | Unpacked | Carriers |
| --- | --- | --- | --- |
| `20251005.tar.gz` | 45.2 MB | 507 MB | A, B, C, D (pilot) |
| `20251207.tar.gz` | 61.7 MB | 731 MB | A, B, C, D (pilot) |
| `20260301.tar.gz` | 62.1 MB | 715 MB | A, B, C |
| `20260322.tar.gz` | 49.6 MB | 611 MB | A, B, C |
| `20260329.tar.gz` | 72.1 MB | 844 MB | A, B, C |
| `20260412.tar.gz` | 39.1 MB | 453 MB | A, B, C |
| `20260419.tar.gz` | 73.2 MB | 844 MB | A, B, C |
| `20260425.tar.gz` | 45.5 MB | 538 MB | A, B, C |
| `20260426.tar.gz` | 48.1 MB | 593 MB | A, B, C |
| `20260502.tar.gz` | 68.2 MB | 739 MB | A, B, C |
| `20260510.tar.gz` | 57.7 MB | 649 MB | A, B, C |

`README.md`, `sessions.csv` and the paper are stored uncompressed so the repository 
stays browsable without downloading any data.

Every archive expands to a directory named after its day, and no two
archives overlap, so extracting any subset into the repository root
reproduces the tree documented in [Repository layout](#repository-layout).
**All paths elsewhere in this README assume you have extracted the relevant
day.**

```bash
tar -xzf 20260426.tar.gz          # one day
for f in 2*.tar.gz; do tar -xzf "$f"; done   # everything
```

`tar` also ships with Windows 10 and later, so the same command works in
PowerShell and `cmd`.

You do not have to unpack at all if you only want a few sessions — Python
reads them straight out of an archive:

```python
import tarfile, json

want = "carrier-C/2 Download - 1000M/test_04261316.UE1.json"
with tarfile.open("20260426.tar.gz") as tf:
    member = next(m for m in tf if m.name.endswith(want))
    log = json.load(tf.extractfile(member))

print(len(log["data"]), "records")
```

Reading a member this way streams it, so memory stays proportional to the
one session rather than the whole archive. `sessions.csv` lists every
session's path, so you can decide what to extract without opening an
archive at all.

## Quick start

Extract a day first (see [Getting the data](#getting-the-data)):

```bash
tar -xzf 20260426.tar.gz
```

Every measurement file is a single JSON object. No special tooling is needed
and the files are valid UTF-8.

```python
import json

path = "20260426/20260426/carrier-C/2 Download - 1000M/test_04261316.UE1.json"
with open(path, encoding="utf-8") as fh:
    log = json.load(fh)

print(log["device"])      # {'index': 1, 'name': 'motorola motorola edge 2024', ...}
print(log["starttime"], "->", log["endtime"])
print(len(log["data"]), "records")

# each record is one ~1 s sample, with whichever sub-blocks the modem reported
for rec in log["data"][:5]:
    loc = rec.get("Location")
    nr = (rec.get("NR5G") or {}).get("Data_Performance") or {}
    print(rec["Timestamp"],
          (loc["Latitude"], loc["Longitude"]) if loc else None,
          nr.get("PDCP_Throughput_DL"))
```

Files are large (median 56 MB, max 126 MB) but shallow: a full
`json.load` of the largest file peaks around 0.5 GB of memory.

`sessions.csv` in the repository root is a manifest of all 158 recordings
with day, carrier, phase, start/end time, duration and size — start there
rather than walking the tree yourself.

---

## How the campaign worked

Three Motorola Edge 2024 handsets (Snapdragon X62) were carried in a
three-phone chest harness so all three traversed an identical trajectory at
the same instant, each carrying the SIM of one operator. The walker moved at
roughly 1.3 m/s around a fixed 6 km loop through Midtown Manhattan, spanning
approximately 42nd to 59th Streets. Operator-to-harness-slot assignment was
rotated between walk days to balance position bias.

Each handset ran **Network Signal Guru (NSG)**, an Android modem diagnostic
tool that records and decodes the full protocol stack from physical-layer
measurements through Layer 2 to Layer 3 RRC/NAS signalling, and exports
native JSON. That JSON is what this dataset primarily contains.

Traffic was generated with `iperf3` over UDP against a fixed, publicly
reachable host per operator, all three hosts in the same datacenter and
deliberately sited far from the test corridor so latency reflects a
realistic wide-area round trip. The observed RTT floor is 35 ms. Latency was
probed separately with TWAMP against a reflector on the same hosts.

**Three traffic regimes**, each interrogating a different operating point:

| Regime | Target rate | Bottleneck | Days |
| --- | --- | --- | --- |
| Line-rate | 1000–1500 Mbps UDP | the wireless channel; heavy loss expected and informative | 20260301 – 20260426 |
| Rate-capped | 20–50 Mbps UDP | the configured cap; near-zero loss expected | 20260502, 20260510 |
| Latency | TWAMP probes | n/a | phases named `Latency` |

**Framework evolution.** The setup was not fixed in advance; it changed as
the field demanded. By 20260322 it had settled into the campaign spine of a
line-rate DL phase, a line-rate UL phase and a TWAMP latency phase.
Server-side `iperf3` logging was added from 20260425 onward. This is why
early days have more, shorter sessions and inconsistent phase naming.

---

## Repository layout

The repository root holds the docs plus one archive per day:

```
.
├── README.md                  ← this file
├── sessions.csv               ← manifest of all 158 recordings
├── wintech2026-final5.pdf     ← the paper
├── 20251005.tar.gz            ← one archive per walk day
├── 20251207.tar.gz
│   ...
└── 20260510.tar.gz
```

Each archive expands to the structure below (shown here as if every day had
been extracted):

```
.
│
├── 20251005/20251005/         ← supplementary pilot day 1 (4 carriers)
│   ├── carrier-A/
│   │   ├── 01 Measurement with Uplink Traffic/
│   │   │   ├── test_10051514.UE1.json
│   │   │   └── test_10051608.UE1.json
│   │   └── 02 Measurement with Downlink Traffic-not enough data/
│   ├── carrier-B/ ...
│   ├── carrier-C/ ...
│   └── carrier-D/ ...
│
├── 20251207/20251207/         ← supplementary pilot day 2 (4 carriers, no phase dirs)
│   ├── carrier-A/test_12071505.UE1.json ...
│   └── ...
│
├── 20260301/20260301/         ← nine-day campaign begins (3 carriers)
│ ...
└── 20260510/measurement-20260510/
    ├── carrier-A/
    │   ├── 1 Upload - Continous 30Mbps/
    │   ├── 2 Download - Continous 50Mbps - iperf stopped at some time point .../
    │   ├── 3 Latency/
    │   │   ├── test_05101652.UE1.json
    │   │   └── twamp_test_20260510_165215.csv
    │   └── 20260510_server_carrier-A.log
    └── ...
```

Two structural quirks to be aware of, both preserved deliberately so the
release maps one-to-one onto the original capture:

- **The date directory is doubled** (`20260301/20260301/...`), an artifact of
  how the daily archives were packaged. The inner directory is named
  `measurement-20260510` on the last day rather than repeating the date.
- **20251207 has no phase subdirectories**; its JSON files sit directly in
  the carrier directory.

Phase directory names are free text written in the field and are **part of
the provenance**. Names like `2 Download - 1000Mbps - maybe traffic not
generated correctly - detour at some point`, `3 Latency - no data saved` and
`3 Latency - twamp, several crash so many missing data` are warnings from
the people who collected the data. Read them.

---

## Naming conventions

| Element | Pattern | Notes |
| --- | --- | --- |
| Day directory | `YYYYMMDD` | Local date of the walk (America/New_York) |
| Carrier directory | `carrier-A` … `carrier-D` | Matches the paper's Carrier A/B/C/D |
| Phase directory | `<n> <description>` | Ordering prefix plus free-text field note |
| NSG recording | `test_MMDDHHMM.UE1.json` | Month, day, hour, minute of recording start |
| TWAMP results | `twamp_test_YYYYMMDD_HHMMSS.csv` | |
| Server log | `server_carrier-X_YYYYMMDD.log` or `YYYYMMDD_server_carrier-X.log` | Both spellings occur; the convention changed mid-campaign |

`UE1` in every filename is the NSG device index on that handset, which is
always 1. **It is not a carrier discriminator** — the carrier is given by
the directory. All three handsets produce files named `...UE1.json`.

The `MMDDHHMM` stamp in a recording's filename is its **start time**, and
carries no year.

---

## Pairing operators

This is the dataset's most important structural property and the basis for
every cross-operator claim in the paper.

Because the three handsets shared a harness and traversed the route
together, any comparison within a simultaneous session is **paired**: the
operators experienced the same instant, weather, walker and urban-canyon
geometry. Pairing removes the time-of-day, route and environmental variation
that plagues sequential single-operator measurement.

Simultaneous recordings are identified by **the same filename under
different carrier directories on the same day**:

```
20260426/20260426/carrier-A/2 Download - 1000M/test_04261316.UE1.json
20260426/20260426/carrier-B/2 Download - 1000M/test_04261316.UE1.json
20260426/20260426/carrier-C/2 Download - 1000M/test_04261316.UE1.json
```

**Do not rely on exact filename equality.** Each handset's NSG instance was
started by hand, so start times can differ by a second or two and the
filename stamp differs with them. Real examples in this dataset:

| Day | Phase | Carrier A | Carrier B | Carrier C |
| --- | --- | --- | --- | --- |
| 20260502 | `03 Download - 30Mbps1hr` | `test_05022013` | `test_05022014` | `test_05022013` |
| 20260425 | `3 Latency` | *(not saved)* | `test_04251400` | `test_04251401` |

Match on **(day, phase, nearest start time)** with a tolerance of a few
seconds, not on the filename string. Days where filenames align exactly for
all carriers: 20251005, 20260322, 20260412, 20260419, 20260426, 20260510.

One provenance oddity: `20260425/.../carrier-A/1 Upload - .../test_04191907.UE1.json`
carries an **April 19** stamp inside the April 25 directory. Trust the
directory for the walk day and the file's own `starttime` field for the
actual recording time.

---

## Session inventory

158 NSG recordings in total. Not all are analysis-grade: the campaign
includes aborted starts and very short recordings, with durations from
0.2 to 80.6 minutes.

The paper analyses **83 walk sessions**, which is reproduced exactly by
taking the nine campaign days (excluding the two 2025 pilot days) and
keeping recordings of **at least 30 minutes and at least 5 MB**:

| Day | Sessions passing filter | Total recordings | Regime | Server log | TWAMP |
| --- | --- | --- | --- | --- | --- |
| 20251005 | 8 | 16 | pilot, 4 carriers | – | – |
| 20251207 | 8 | 19 | pilot, 4 carriers | – | – |
| 20260301 | 9 | 28 | line-rate | – | – |
| 20260322 | 9 | 21 | line-rate | – | 3 |
| 20260329 | 11 | 17 | line-rate | – | 2 |
| 20260412 | 6 | 9 | line-rate | – | – |
| 20260419 | 12 | 12 | line-rate | – | 3 |
| 20260425 | 9 | 9 | line-rate | 3 | 2 |
| 20260426 | 9 | 9 | line-rate | 3 | 3 |
| 20260502 | 9 | 9 | rate-capped | 3 | – |
| 20260510 | 9 | 9 | rate-capped | 3 | 4 |
| **nine-day total** | **83** | **123** | | **12** | **17** |

Per carrier across the whole release: Carrier A 50 recordings, Carrier B 50,
Carrier C 49, Carrier D 9.

`sessions.csv` has one row per recording with `day, carrier, phase, file,
start, end, duration_min, size_mb, device, passes_paper_filter`, so the
filter above is a one-line query.

**Carrier D appears only on the two 2025 pilot days.** It is the
"fourth carrier" of the paper's Appendix A, captured once on its native 5G
standalone network and once after its transition to a shared RAN. Do not mix
it into the nine-day cross-operator comparison.

---

## File format: NSG JSON

### Top level

```json
{
  "version": 3,
  "library": "5.8.7",
  "log":    { "beginTime": "...", "creationTime": "...", "devices": [0, 1],
              "endTime": "...", "lastModifiedTime": "...",
              "payloadBeginAt": 7247, "payloadLength": 27316952,
              "rotateFrom": 0, "rotateTo": 0, "uuid": "...", "version": 84412190 },
  "device": { "index": 1, "name": "motorola motorola edge 2024",
              "subscription": 1, "type": "Qualcomm" },
  "starttime": "2025-10-05T14:49:43.618212-04:00",
  "endtime":   "2025-10-05T15:14:36.966979-04:00",
  "PCAPHeader": "a1b2c3d4...",
  "data": [ ... ]
}
```

`starttime` / `endtime` are ISO 8601 with UTC offset and are the authoritative
recording bounds. `PCAPHeader` is a vestigial libpcap global header; the
packets it described have been removed by anonymization. `log.uuid` is a
synthetic session identifier.

### The `data` array

Each element is one sample, nominally at 1 s cadence, carrying whichever
sub-blocks the modem reported in that interval. **Every key except
`Timestamp` is optional** — always use `.get()`. Representative fill rates
from one 1493-record file:

| Key | Type | Present | Meaning |
| --- | --- | --- | --- |
| `Timestamp` | string | always | Host (phone OS) clock, ISO 8601 with offset |
| `EquipmentTimestamp` | string | most | **Modem** clock — see [pitfalls](#known-pitfalls) |
| `Location` | object | most | GPS fix |
| `LTE` | object | most | LTE radio, throughput, cells, NAS |
| `NR5G` | object | most | NR radio, throughput, cells |
| `Common` | object | most | Technology mode, device identity |
| `WCDMA` | object | rare | NAS-only stub; see pitfalls |
| `messages` | array | some | Decoded L3 RRC/NAS messages |
| `events` | array | some | Named modem events |

### `Location`

```json
{ "Accuracy": 15.06, "Altitude": 17.56, "Latitude": 40.76612711,
  "Longitude": -73.98354609, "Speed": 0, "Bearing": 148.3 }
```

`Bearing` is present in roughly a third of fixes. All coordinates lie inside
latitude 40.7530–40.7693 and longitude −73.9905 to −73.9729, the fixed loop.

### `Common`

`Data_Technology_Mode` and `Radio_Technology_Mode` are vendor enums (values
such as 48–52 and 256/4096/4352) whose encoding is not documented by the
tool. **Prefer `NR5G.DedicatedRadioLink.Connectivity_Mode` to determine
NSA vs SA** (see the field reference). `Device_IMEI` / `Device_IMSI` appear
once per recording and are synthetic in this release.

### `LTE` and `NR5G`

Both are containers of sub-blocks. The ones you are most likely to want:

| Sub-block | Holds |
| --- | --- |
| `LTE.PrimaryCell`, `LTE.PrimaryCell_Radio` | PCell band, EARFCN, PCI, RSRP, RSRQ, RSSI, SINR |
| `LTE.PrimaryCell_MIMO` | Per-layer RSRP/RSRQ/RSSI/SINR (array, one entry per receive branch) |
| `LTE.SecondaryCell_1`, `_2` (+ `_Radio`, `_MIMO`) | Carrier-aggregation SCells |
| `LTE.LTE_Data_KPI` | Aggregate LTE throughput, MCS, BLER, RB, `ServingCarriers` |
| `LTE.ServingCells`, `LTE.NeighborCells` | Cell lists with band, EARFCN, PCI, RSRP, RSRQ |
| `LTE.DownlinkMeasurements`, `UplinkMeasurements` | CQI, modulation usage, grants |
| `LTE.RACH` | Random-access attempts, results, timing advance |
| `LTE.NAS` | Decoded EMM/ESM state (identity fields pseudonymised) |
| `NR5G.Data_Performance` | NR throughput at PHY/MAC/RLC/PDCP, plus `MRDC_*` variants |
| `NR5G.PrimaryCell_Radio_DL` / `_UL` | CSI/SSB RSRP, RSRQ, SINR, CQI, MCS, BLER, RB, rank, pathloss |
| `NR5G.NRCellTable_Serving`, `_Neighbor` | NR cell lists with ARFCN, PCI, beam index, RSRP, RSRQ |
| `NR5G.DedicatedRadioLink` | `Connectivity_Mode`, `RRC_State`, `ServingCellType`, `DuplexMode` |
| `NR5G.TDDConfigurations` | TDD slot/symbol pattern |
| `NR5G.SecondaryCell`, `NR5G.SecondaryCell_Radio_DL` | NR carrier-aggregation SCell, present while aggregating |
| `NR5G.DownlinkMeasurements`, `UplinkMeasurements` | CQI, scheduled counts, CCE usage, PDCCH DCI format, Tx power |
| `NR5G.RACH` | NR random access |
| `NR5G.NAS` | Decoded 5GMM/5GSM state (identity fields pseudonymised) |
| `NR5G.VoNR` | Voice-over-NR indication (appears in a handful of records) |

### `messages`

Decoded Layer-3 signalling. Each entry:

```json
{ "Category": "LTE", "Title": "Measurement Report",
  "Direction": "up", "Detail": { ... },
  "Timestamp": "...", "EquipmentTimestamp": "..." }
```

`Category` ∈ `LTE` (328,224), `NR` (212,811), `EMM` (10,924), `MM5G`
(6,271), `ESM` (2,773), `SM5G` (33) across the release. `Title` names the
procedure (`SIB1`, `RRC Connection Reconfiguration`, `Measurement Report`,
`NR_RACH_Attempt`, `Tracking Area Update Accept`, …). `Detail` is the fully
decoded ASN.1/NAS structure with 3GPP field names (`measResults`,
`rrcReconfiguration`, `reconfigurationWithSync`, `plmn-IdentityList`, …).

Records with `Category: "IMS"` have been **removed** by anonymization.

Some `Detail` leaves read `"-- Decode on next scan --"`. That is an NSG
placeholder meaning the tool did not decode that container, not a data
error and not a redaction.

### `events`

```json
{ "Title": "NR_SCG_Addition_Success", "Description": "Reason: HANDOVER",
  "Timestamp": "...", "EquipmentTimestamp": "..." }
```

`Description` is free text carrying event parameters, commonly measurement
thresholds in the form `"rsrp: -122, h: 1.0, t: 640 ms, Id: 3"`
(threshold, hysteresis, time-to-trigger, measurement ID), beam transitions
`"BeamID 1 -> 5"`, or a reason code. `Description` is often an empty string.

---

## File format: TWAMP CSV

Round-trip latency from the Android TWAMP client against the fixed
reflector. Each file has three blocks separated by blank lines.

```
TWAMP Test Results
Test Date,2026-05-10 15:55:26
Destination,192.0.2.1          ← pseudonymised reflector address
Port,862
Total Packets,3600
Interval (ms),1000
Payload (bytes),1017259832     ← see warning below
Light Mode,Yes

Summary Statistics
Metric,Value,Unit
Packet Loss,0.00,%
Min Latency,0.000,ms           ← see warning below
Max Latency,0.000,ms
Average Latency,0.000,ms
Average Jitter (StdDev),0.000,ms

Individual Packet Details
Packet Number,Send Timestamp (us),Receive Timestamp (us),Round-Trip Time (us),Latency (ms),Status
1,1778442927854000,1778442928048000,194000,193.910,Success
2,1778442928853000,1778442928922000,69000,69.006,Success
```

The per-packet block is the authoritative data. `Status` is `Success`
(35,314 rows across the release) or `Lost` (22,286 rows). Timestamps are
microseconds since the Unix epoch. `Latency (ms)` is the RTT in
milliseconds; `Round-Trip Time (us)` is the same quantity, coarser.

> **Two fields in the header are unreliable.**
>
> 1. **The Summary Statistics block is zeroed in 14 of the 17 files.** Only
>    the three 20260322 files carry correct summary values. Everywhere else
>    `Min/Max/Average Latency` read `0.000` while the per-packet rows hold
>    perfectly valid RTTs. **Always compute summary statistics yourself from
>    the per-packet rows.**
> 2. **`Payload (bytes)` is garbage** — it holds large, often negative
>    integers (e.g. `-1539844296`) that are not payload sizes. Ignore it.
>
> `twamp_test_20260510_160102.csv` contains **zero successful packets**; it
> is the crashed run its directory name warns about. Keep it for provenance,
> exclude it from analysis.

The 20260322 files are also much shorter than the rest (179–226 successful
packets versus roughly 3,400 on later days).

---

## File format: iperf3 server logs

Plain `iperf3 --logfile` output from the server side, one file per operator
per day, available for the four days from 20260425 onward. These provide the
**server-side ground truth** that distinguishes a radio-side anomaly from a
server-side one — the basis of the paper's NR-SCG silent-failure case study.

```
Sat Apr 25 08:44:36 2026 Accepted connection from 198.51.100.3, port 59144
Sat Apr 25 08:44:36 2026 [ 10] local 192.0.2.1 port 5201 connected to 198.51.100.3 port 59145
Sat Apr 25 08:44:37 2026 [ ID] Interval           Transfer     Bitrate         Jitter    Lost/Total Datagrams
Sat Apr 25 08:44:37 2026 [ 10]   0.00-1.00   sec   335 KBytes  2.74 Mbits/sec  12.396 ms  1274/1523 (84%)
...
Sat Apr 25 08:44:47 2026 [ 10]   0.00-10.24  sec  1.95 MBytes  1.59 Mbits/sec  8.922 ms  23068/24549 (94%)  receiver
Sat Apr 25 08:44:47 2026 -----------------------------------------------------------
Sat Apr 25 08:44:47 2026 Server listening on 5201
```

Every line is prefixed with the server's local wall-clock time, which
`iperf3` does not normally emit — it was added by the logging wrapper and is
what lets you align server records with UE records.

- `Accepted connection from <ip>` marks a new session; the address is the
  UE's pseudonymised public address.
- `local <ip> port 5201` is the pseudonymised measurement host.
- Interval rows are per-second; the row ending `receiver` is the session
  summary.
- Loss percentages above 90% are **expected** in the line-rate regime, where
  the offered rate deliberately exceeds channel capacity.
- Negative loss counts and percentages appear occasionally (an `iperf3`
  accounting quirk under heavy reordering); treat them as missing.
- `[SUM]  ... datagrams received out-of-order` lines appear in some logs.

To join a server session to a UE recording, match the `Accepted connection`
wall-clock time against the UE record's **`Timestamp`** (host clock), not
`EquipmentTimestamp`. The paper aligns 603 connection accepts one-to-one
with UE records this way.

---

## Field reference

Quantities you are most likely to want, and where they actually live.

| Quantity | Path |
| --- | --- |
| GPS position | `Location.Latitude`, `.Longitude`, `.Altitude`, `.Accuracy`, `.Speed`, `.Bearing` |
| NSA vs SA, per record | `NR5G.DedicatedRadioLink.Connectivity_Mode` — **1 = NSA, 2 = SA** |
| NR cell role | `NR5G.DedicatedRadioLink.ServingCellType` — `MCG0-0` under SA, `SCG1-x` under NSA |
| NR serving-cell RSRP / RSRQ / SINR | `NR5G.PrimaryCell_Radio_DL.PCell_SS_RSRP`, `.PCell_SS_RSRQ`, `.PCell_SS_SINR` |
| NR CSI-based RSRP / RSRQ / SINR | `NR5G.PrimaryCell_Radio_DL.PCell_CSI_RSRP`, `.PCell_CSI_RSRQ`, `.PCell_CSI_SINR` |
| NR PHY throughput DL | `NR5G.Data_Performance.Physical_Throughput_DL` (SA) / `.MRDC_Physical_Throughput_DL` (NSA) |
| NR PDCP throughput | `NR5G.Data_Performance.PDCP_Throughput_DL`, `.PDCP_Throughput_UL` |
| NR MCS / BLER / rank | `NR5G.PrimaryCell_Radio_DL.PCell_MCS_DL`, `.PCell_MAC_BLER_DL`, `.PCell_RankIndicator_DL` |
| NR resource blocks | `NR5G.PrimaryCell_Radio_DL.PCell_RB_Num_Average_DL` — **per-carrier mean, see pitfalls** |
| NR serving NR carrier count | `len(NR5G.NRCellTable_Serving)` — **there is no scalar field**; each list entry is one serving NR carrier |
| NR band / frequency | `NR5G.PrimaryCell.PCell_Band`, `NR5G.NRCellTable_Serving[].NR_ARFCN` |
| LTE anchor RSRP / RSRQ | `LTE.PrimaryCell_Radio.PCell_RSRP`, `.PCell_RSRQ` |
| LTE throughput | `LTE.LTE_Data_KPI.PDCP_Throughput_DL`, `.Agg_Throughput_DL` |
| LTE carrier count | `LTE.LTE_Data_KPI.ServingCarriers` |
| LTE band / EARFCN / PCI | `LTE.ServingCell.Band`, `.EARFCN_DL`, `.PCI` |
| Neighbour cells | `LTE.NeighborCells[]`, `NR5G.NRCellTable_Neighbor[]` |
| Handover / reconfiguration | `messages[]` with `Title` containing `Reconfiguration`, `Detail.…reconfigurationWithSync` |
| RACH | `LTE.RACH`, `NR5G.RACH`, and `events[]` titled `*_RACH_*` |

Units: RSRP/RSRQ/RSSI/SINR in dB or dBm; throughput in **Mbps**; latency in
ms unless the column name says µs.

---

## Known pitfalls

The paper catalogues fourteen telemetry artifacts. These are the ones that
will change your code. Values below were measured on this released data.

### 1. The modem clock is not the host clock

Each record can carry two timestamps: `Timestamp` (host/OS) and
`EquipmentTimestamp` (modem). For one operator the modem clock runs on **GPS
time**, which is ahead of UTC by the accumulated leap-second offset. Median
`EquipmentTimestamp − Timestamp` measured per carrier:

| Carrier | 20251005 | 20260322 | 20260425 | 20260510 |
| --- | --- | --- | --- | --- |
| A | +2.85 s | −2.22 s | −1.57 s | +1.01 s |
| B | **+17.66 s** | **+17.12 s** | **+19.29 s** | **+18.73 s** |
| C | +3.46 s | −0.58 s | −0.30 s | −0.60 s |

**Use `Timestamp` for every cross-tool and cross-operator join.** Use
`EquipmentTimestamp` only for modem-internal relative timing, such as
measuring RRC interruption by differencing a reconfiguration against its
completion within one recording.

### 2. The first record claims 1980

Every recording contains 1–3 records whose `EquipmentTimestamp` falls in
January 1980 — the GPS epoch, emitted before the modem achieves time lock.
Drop records with a 1980 `EquipmentTimestamp` at parse time.

### 3. Throughput fields differ by connectivity mode

In MR-DC (NSA) the combined NR+LTE figure lives in `MRDC_*_Throughput`; in
SA the figure lives in plain `Physical_Throughput` / `MAC_Throughput`. Both
families are present in the files, but their fill rates invert with mode.
From one download phase on 20260426:

| Carrier | Mode | `MRDC_Physical_Throughput_DL` | `Physical_Throughput_DL` |
| --- | --- | --- | --- |
| B | NSA | 389 records | 282 records |
| C | SA | 137 records | 2151 records |

Reading either field uniformly across operators silently zeroes one of them.
**Branch per record on `Connectivity_Mode`**, not per carrier — Carrier C is
predominantly SA but has records in NSA mode too.

### 4. Recordings include idle gaps — gate on activity

A recording spans the whole phase, including the gaps between consecutive
`iperf3` sessions and the walk to the start point. During those gaps the
radio stays connected and reports **small but non-zero** throughput, so
filtering on `> 0` does not isolate active transfer. Computing a median over
raw samples therefore returns approximately zero even on a healthy
high-throughput walk.

Measured on the 20260426 download phase, PHY-layer DL throughput in Mbps:

| Gate | Carrier B (NSA) | Carrier C (SA) |
| --- | --- | --- |
| no gate | 74.7 | **0.0** |
| `PCell_Scheduled_Count_DL > 0` | 157.3 | **0.0** |
| `PCell_RB_Num_Average_DL > 0` | 157.3 | **0.0** |
| **throughput > 1 Mbps** | **171.3** | **349.8** |

Only the minimum-rate gate gives sensible values, and those values land in
the ranges the paper reports (Carrier B DL median 182 Mbps, Carrier C
333 Mbps). Note that a non-zero scheduling count or RB count is **not**
sufficient evidence of an active transfer.

Pick the gate that suits your question, state it, and apply it identically
across operators.

### 5. `RB_Num_Average_DL` is a per-carrier mean, not a total

`PCell_RB_Num_Average_DL` is averaged over serving carriers, not summed
across them. A throughput model must form an effective RB count as

```
effective_RB = PCell_RB_Num_Average_DL × (number of serving NR carriers)
```

or it under-credits carrier-aggregating operators.

There is no scalar carrier-count field: derive it from
`len(NR5G.NRCellTable_Serving)`. On the 20260426 Carrier C download phase
that list holds 1, 2 or 3 entries, averaging **2.32** carriers once the
activity gate of pitfall 4 is applied — which is the figure the paper
reports for that operator under active scheduling.

### 6. NR `PCell_*` is the PSCell under NSA

NSG labels the NR serving cell with a `PCell_` prefix in both modes. Under
NSA that cell is the **PSCell** (secondary cell group); under SA it is the
true PCell. `ServingCellType` disambiguates: `SCG1-x` versus `MCG0-0`. The
paper uses the neutral term "NR serving-cell" for this reason.

### 7. "Handover" mixes distinct procedures

An aggregate handover count conflates NSA PSCell changes with SA full PCell
handovers, which are different 3GPP procedures with different interruption
times. Classify each `reconfigurationWithSync` by its PDU contents and
report per class.

### 8. One RSRQ field is derived, not measured

Within `PrimaryCell_Radio_DL`, one RSRQ field equals the SSB RSRQ and
another sits exactly 3 dB below it. Treat the offset one as redundant and
report the measured SSB value.

### 9. `WCDMA` appears although no WCDMA network exists

A small number of records carry a `WCDMA` block. It contains only a `NAS`
sub-block, with no PCI or radio measurements. Compare only fields that all
operators populate.

### 10. Not every recording is analysis-grade

Durations run from 0.2 to 80.6 minutes. Apply the ≥30 min and ≥5 MB filter
(precomputed in `sessions.csv`) and honour the field notes in the phase
directory names.

### 11. Some walks record a genuine failure, not a measurement

Specific sessions capture silent failure modes and must be excluded from
typical-performance claims:

- **Carrier A, 20260425 download** — NR secondary cell group silently
  stopped carrying data. The UE reports normal NSA attachment while the NR
  serving-cell sub-block is absent from virtually all records (0.0% on
  20260425, 0.3% on 20260426, against a 22–34% baseline) and NR PDCP
  throughput is zero throughout. The server log for that day shows the
  server emitting at the full target rate, which is what localises the fault
  to the UE.
- **Carrier B, 20260426 download** — smaller partial NR primary-cell drop.
- **Carrier C, 20260329 latency** — returned no TWAMP responses.

A useful screen is the per-walk NR serving-cell presence rate: compute the
fraction of records containing `NR5G.NRCellTable_Serving` and flag walks far
below the cohort baseline.

### 12. GPS does not retrace the route identically

The route is fixed but GPS does not trace it identically on every walk. The
paper reports 1.6% of samples (4,859 of 297,578) falling more than 5 m off
the canonical footprint, concentrated on a handful of days. Stationary test
clusters also occur — `20260425`'s upload phase directory notes that the
first three iperf measurements were taken at the same location. Flag and
exclude such clusters from per-location statistics while keeping them for
per-walk statistics.

---

## Worked examples

### Extract a throughput and position time series

```python
import json, statistics

with open("20260426/20260426/carrier-C/2 Download - 1000M/"
          "test_04261316.UE1.json", encoding="utf-8") as fh:
    log = json.load(fh)

ACTIVE_MBPS = 1.0          # activity gate, see pitfall 4

series = []
mode = None                # DedicatedRadioLink is reported only on change
for rec in log["data"]:
    if rec.get("EquipmentTimestamp", "").startswith("1980"):
        continue                                  # pitfall 2
    nr = rec.get("NR5G") or {}
    drl = nr.get("DedicatedRadioLink") or {}
    if "Connectivity_Mode" in drl:
        mode = drl["Connectivity_Mode"]           # carry forward
    perf = nr.get("Data_Performance") or {}

    if mode == 2:                                 # SA
        tput = perf.get("Physical_Throughput_DL")
    elif mode == 1:                               # NSA (MR-DC)
        tput = perf.get("MRDC_Physical_Throughput_DL")
    else:
        tput = None                               # mode not yet known
    if tput is None or tput <= ACTIVE_MBPS:       # pitfall 4
        continue

    loc = rec.get("Location")
    series.append((rec["Timestamp"], tput,
                   loc["Latitude"] if loc else None,
                   loc["Longitude"] if loc else None))

print(len(series), "active samples")
print("median", round(statistics.median(s[1] for s in series), 1), "Mbps")
# -> 618 active samples
# -> median 349.8 Mbps
```

Two things in that loop are not optional. `DedicatedRadioLink` is emitted
only when it changes, so the connectivity mode must be **carried forward**
rather than treated as unknown whenever it is absent. And without the
activity gate the median of this same walk is 0.0 Mbps (pitfall 4).

### Screen a walk for the NR-SCG silent failure

```python
def nr_serving_presence(log):
    n = len(log["data"])
    have = sum(1 for r in log["data"]
               if (r.get("NR5G") or {}).get("NRCellTable_Serving"))
    return have / n if n else 0.0

rate = nr_serving_presence(log)
print(f"NR serving-cell presence: {rate:.1%}")
if rate < 0.05:
    print("SUSPECT: near-zero NR presence — check against the server log "
          "before treating this walk as typical performance")
```

### Pair the three operators for one phase

```python
import os, glob, json, datetime

def start_of(path):
    head = open(path, encoding="utf-8").read(4096)
    import re
    return datetime.datetime.fromisoformat(
        re.search(r'"starttime":\s*"([^"]+)"', head).group(1))

def paired_sessions(day_dir, phase_glob, tol_s=5):
    """Group recordings across carriers by start time within tol_s seconds."""
    found = []
    for c in sorted(os.listdir(day_dir)):
        if not c.startswith("carrier-"):
            continue
        for p in glob.glob(os.path.join(day_dir, c, phase_glob, "*.json")):
            found.append((start_of(p), c, p))
    found.sort()
    groups = []
    for ts, c, p in found:
        for g in groups:
            if abs((ts - g[0][0]).total_seconds()) <= tol_s:
                g.append((ts, c, p)); break
        else:
            groups.append([(ts, c, p)])
    return groups

for g in paired_sessions("20260502/20260502", "03 Download*"):
    print({c: os.path.basename(p) for _, c, p in g})
# -> {'carrier-A': 'test_05022013.UE1.json',
#     'carrier-B': 'test_05022014.UE1.json',
#     'carrier-C': 'test_05022013.UE1.json'}
```

### Read a TWAMP file correctly

```python
import csv, statistics

def twamp(path):
    rows, in_packets = [], False
    with open(path, encoding="utf-8") as fh:
        for row in csv.reader(fh):
            if not row:
                continue
            if row[0] == "Packet Number":
                in_packets = True; continue
            if in_packets and len(row) >= 6:
                rows.append(row)
    ok = [float(r[4]) for r in rows if r[5] == "Success"]
    loss = 1 - len(ok) / len(rows) if rows else float("nan")
    # compute statistics from the rows: the summary block is zeroed
    # in 14 of 17 files (see pitfalls)
    return {
        "n": len(rows), "loss": loss,
        "median_ms": statistics.median(ok) if ok else None,
        "p95_ms": sorted(ok)[int(len(ok) * 0.95)] if ok else None,
        "p99_ms": sorted(ok)[int(len(ok) * 0.99)] if ok else None,
    }

print(twamp("20260510/measurement-20260510/carrier-A/3 Latency/"
            "twamp_test_20260510_165215.csv"))
```

### Join a server log to UE records

```python
import re, datetime

ACCEPT = re.compile(
    r"^(?P<ts>\w{3} \w{3} +\d+ \d+:\d+:\d+ \d{4}) Accepted connection from (?P<ip>\S+), port (?P<port>\d+)")

def accepts(path, year_tz):
    out = []
    for line in open(path, encoding="utf-8"):
        m = ACCEPT.match(line)
        if m:
            t = datetime.datetime.strptime(m.group("ts"), "%a %b %d %H:%M:%S %Y")
            out.append(t.replace(tzinfo=year_tz))
    return out

# match each accept against UE record Timestamp (host clock, pitfall 1)
```

---

## What is not in the dataset

**Removed by anonymization**: raw `PCAPPacket` PDU
hex, all IMS/SIP signalling, subscriber phone number, encrypted NAS
containers, and operator-provisioned emergency number lists.

**Never captured**, and therefore not a gap you can close from these files:

- Modem-internal measurement logs. NSG exports what the diagnostic interface
  exposes, which is why the paper's case study stops at "the UE never
  achieved an NR measurement good enough to cross the reporting threshold"
  rather than distinguishing *measured but below threshold* from *never
  measured*.
- FR2 / mmWave. The handsets are non-FR2 devices.
- Application-layer or TCP measurements. Traffic is UDP `iperf3` throughout.
- Any operator-side or RAN-side view.
- Client-side `iperf3` logs. Only the server side was logged, and only from
  20260425 onward.

**Scope caveat.** These measurements reflect the experience of a single
subscriber on one device model over nine weekend afternoons along a fixed
route in one city. They are not generalisable carrier-level performance
claims, and the paper is explicit about this.

---

## Citing

```bibtex
@inproceedings{choi2026threephones,
  author    = {Choi, Wonyul and Kak, Ahan},
  title     = {Three Phones, Nine Afternoons: An Experience Report on
               Multi-Operator 5G Walk-Testing in Manhattan},
  booktitle = {Proceedings of the 20th ACM Workshop on Wireless Network
               Testbeds, Experimental evaluation \& Characterization
               (WiNTECH '26)},
  year      = {2026},
  address   = {Austin, TX, USA},
  publisher = {ACM},
  doi       = {10.1145/3831662.3844164}
}
```

The paper is distributed under CC BY-NC-ND 4.0.

## Acknowledgements

Measurements were collected with [Network Signal Guru](https://www.qtrun.com) (Qtrun Technologies). 
Latency used [twamp-gui](https://github.com/demirten/twamp-gui) as client and reflector.
Throughput used [`iperf3`](https://github.com/esnet/iperf).