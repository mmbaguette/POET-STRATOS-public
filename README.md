# POET-STRATOS — Camera Control and Space-to-Ground Data Link

Flight software for a short-wave infrared (SWIR) astronomy payload flown on a stratospheric balloon from the Canadian Space Agency's balloon base in Timmins, Ontario, in **[MONTH] 2025**, in partnership with the French space agency (CNES).

The payload was a technology demonstration for [POET](https://exoplanetes.umontreal.ca/en/a-new-canadian-telescope-to-study-exoplanets/), a proposed Canadian exoplanet space telescope. The goal was to show that a **commercial off-the-shelf infrared camera** — rather than a custom space-qualified sensor — can produce usable astronomical images in a near-space environment, which would substantially lower the cost of the eventual satellite mission.

The balloon carried the payload to roughly **33 km** altitude. Everything on board had to run unattended, recover from its own errors, and be controllable from the ground over a slow, lossy radio link.

> **My role:** I was the engineering student responsible for camera control and the ground-to-payload communications system. I wrote the C++ camera interface layer, the Python telecommand and data-downlink system, and the payload operations documentation. Team credits are at the bottom.

**This repository contains documentation only.** The flight source code is not published here.

---

## Table of contents

- [What the payload had to do](#what-the-payload-had-to-do)
- [System overview](#system-overview)
- [Camera control layer (C++)](#camera-control-layer-c)
- [Data downlink (Python)](#data-downlink-python)
- [Ground station and telecommands](#ground-station-and-telecommands)
- [Hardware integration problems](#hardware-integration-problems)
- [Results](#results)
- [What I would do differently](#what-i-would-do-differently)
- [Skills demonstrated](#skills-demonstrated)
- [Credits](#credits)

---

## What the payload had to do

Four requirements drove every design decision:

1. **Take long-exposure infrared images** of specific stellar targets, at exposure times set from the ground, with the sensor actively cooled to a commanded temperature.
2. **Get science data to the ground** across a link with high latency, sustained packet loss, and periods of total interruption — on a small allocated bandwidth.
3. **Stay alive without anyone touching it.** Once the balloon launched, nobody could reach the hardware for the duration of the flight. Any unhandled exception that killed the program ended the science return.
4. **Meet agency interface requirements.** Physical, electrical, and network interfaces to the gondola were specified by CSA and CNES documentation and had to be verified before flight. *(See [PASTIS telemetry integration guide](https://cnes.fr/sites/default/files/2024-07/9_cnes_ballons_-_pastis_telemetry_integration_guide_-_2019_en.pdf) — the 2025 campaign used a revision of this document.)*

---

## System overview

```
                      ~33 km altitude
  ┌──────────────────────────────────────────────┐
  │  GONDOLA / PAYLOAD                           │
  │                                              │
  │  Owl 1280 SWIR camera                        │
  │        │ Camera Link + RS-232 serial         │
  │        ▼                                     │
  │  PIXCI frame grabber ── rugged PC            │
  │        │                                     │
  │        ▼                                     │
  │  C++ camera control  ──►  FITS files         │
  │        │ (DLL, called from Python)           │
  │        ▼                                     │
  │  Python flight program                       │
  │    ├── on-board photometry / lightcurves     │
  │    ├── image compression + packetisation     │
  │    └── telecommand server                    │
  └───────────────┬──────────────────────────────┘
                  │  PASTIS Wi-Fi link
                  │  (gondola → relay balloon → ground antenna)
                  ▼
  ┌──────────────────────────────────────────────┐
  │  GROUND STATION (laptops)                    │
  │    ├── receiver  — downloads science data    │
  │    └── telecommand client — operator console │
  └──────────────────────────────────────────────┘
```

Two separate UDP channels were used: one for the science data downlink and one for telecommands, so that a stalled file transfer could never block an operator's ability to reach the payload.

---

## Camera control layer (C++)

The camera was a **Raptor Photonics Owl 1280**, a 12-bit SWIR camera, connected over Camera Link to an **EPIX PIXCI mf2280** frame grabber and driven through the EPIX XCLIB SDK.

Capture works in two stages: the camera exposes and pushes pixel data to the frame grabber, and the program then reads the bitmap off the grabber and either displays it or writes it out as a FITS file (the standard astronomy image format, which stores instrument metadata alongside the pixels). Images were saved with temperature and exposure metadata written into the FITS headers so that imaging performance could later be correlated against thermal conditions during flight.

### Serial control

Camera settings that XCLIB does not expose are set over an RS-232 serial connection using the Owl's own command set — sequences of hexadecimal bytes, each terminated with an `ETX` acknowledgement byte. Setting a single parameter often means writing to two or more registers in sequence, one byte at a time, and checking for an acknowledgement after each.

I implemented the following:

| Function | What it does |
|---|---|
| Exposure time | Converts milliseconds to counts of the camera's 70 MHz clock, splits the result across four consecutive registers, and writes them in order |
| Digital gain | Writes a 16-bit gain value as high/low bytes, then **reads it back and compares** to confirm the write took effect |
| Manufacturer data | Reads the camera's non-volatile memory for serial number, build date, and factory temperature calibration points |
| Calibration | Derives ADC and DAC slope/offset from the factory calibration points, so raw sensor values can be converted to degrees Celsius |
| TEC setpoint | Converts a target temperature to a 12-bit DAC value using those calibration constants and commands the thermoelectric cooler |
| Temperature reads | Sensor temperature, cooler setpoint, and internal board temperature, including sign-extension of the 12-bit signed format the camera returns |

Every write is verified where the camera allows a read-back. Silent failure was the thing I most wanted to avoid, because on a balloon there is no way to notice and no way to intervene.

The C++ layer was compiled as a DLL and called from the main Python flight program, so the higher-level logic could stay in Python while the timing-sensitive camera interaction stayed in C++.

### A firmware bug worth knowing about

Certain exposure times caused the camera to return frames with a large block missing from the top of the image — a firmware fault, not a capture or transfer error. **1000 ms and 30 000 ms both reproduced it.** Because there was no fix available, I characterised the behaviour empirically and restricted the flight to exposure times verified to return complete frames:

> 965, 1500, 2000, 3000, 4000, 10 000 and 29 990 ms

Note that 965 and 29 990 ms are deliberate near-misses of the two bad values — close enough to the intended exposure to be scientifically equivalent, far enough to avoid the fault.

I also found that enabling the camera's auto-exposure and disabling AGC, both of which seemed like reasonable things to do for long exposures, were unnecessary and could themselves cause capture failures.

---

## Data downlink (Python)

This was the hardest part of the project and took considerably longer than planned. Moving large files across a link with long interruptions, high loss, and high latency is a different problem from moving them across a network that mostly works.

The downlink sends three kinds of data, each on its own independently configurable interval:

- **Photometry / flux data** — measurements extracted on board from the captured frames
- **Compressed preview images** — a downsized, JPEG-compressed view of the most recent capture, for real-time situational awareness on the ground
- **Full FITS files** — complete science frames, sent far less frequently

Each transfer is compressed, split into packets sized to the currently allocated bandwidth, and sent with a configurable delay between packets so the payload never floods the link. After the last packet, the payload waits for the ground station to request any packets that did not arrive, retransmits them, and repeats until the ground stops asking.

### Compression under a deadline

Preview images are governed by a **maximum acceptable transmission time** rather than a fixed quality setting. If an image would take longer than that budget to send at the current bandwidth, it is progressively downscaled and recompressed until it fits. This keeps the ground station's view of the sky current even when the link degrades, instead of blocking on one oversized image.

### Testing a space link on the ground

On a normal network almost every packet arrives, so the retransmission logic never runs and the protocol is never actually exercised. To test it, degraded network conditions were simulated with traffic-shaping tools (NetLimiter on Windows, Network Link Conditioner on macOS) to force packet loss and bandwidth limits that matched the expected flight link.

### Failure handling

The flight program is written to survive rather than to be correct. Exceptions are caught broadly, logged verbosely to disk, reported to the ground station, and execution continues. A payload that keeps running in a degraded state returns more data than one that exits cleanly on an unexpected error.

---

## Ground station and telecommands

Operators on the ground had a command console that could reach the payload throughout the flight. Available commands included:

- Retrieve logs and debug information
- Request a screenshot of the payload computer's desktop
- Pause and resume continuous capture
- Set exposure time, cooler setpoint, and digital gain
- Request a specific piece of data on demand
- Edit payload parameters in flight

**Tunable in flight:** downlink bandwidth, inter-packet delay, image resize factor, JPEG quality, the preview-image time budget, and the send interval for each data type independently (including disabling any of them entirely).

That is a lot of flexibility for what is usually a fixed configuration. The reasoning: once the balloon is up, these parameters are the only control you have, and you cannot predict which one you will wish you could change.

The console also exposed the on-board photometry parameters — star detection thresholds, aperture and annulus radii, centroid tolerance, frame chunk size, and master sky frame management — so the science processing could be retuned against real flight data rather than being frozen at launch.

---

## Hardware integration problems

### PCIe link negotiation and frame grabber overflow

XCAP logged a persistent advisory that the frame grabber might not keep up with a high-bandwidth camera, and capture intermittently failed with **PCI FIFO Overflow**, requiring a camera power cycle to recover.

The frame grabber is designed for a PCIe x4 Gen2 slot. The rugged PC's expansion cassette advertised an x4 connector but negotiated only **x1 Gen2** — a quarter of the expected lanes. The BIOS listed four separate PCI Express root ports where the specification implied one, which is what pointed at the cause.

I documented the behaviour and escalated to the frame grabber manufacturer with the configuration details, BIOS observations, and device-manager topology. Confirmed findings:

- The card and the bus negotiate the best mutually supported configuration; if the host only offers x1 Gen2, that is what you get, and the card cannot span multiple slots to compensate.
- x1 Gen2 is nevertheless sufficient for this camera.
- **Bit-packing** the 12-bit pixel data transfers each pixel as 1.5 bytes instead of 2, cutting capture bandwidth by **25%** at the cost of extra CPU work when displaying or saving — a direct mitigation for the overflow.
- Loose or defective Camera Link cabling produces both image noise and FIFO overflow, and is worth ruling out first.

The lesson I took from this: a warning in a log is a measurement. Chasing it to root cause with the vendor turned an intermittent, flight-threatening failure into a known, bounded condition with a mitigation.

### Gondola network interface

The payload connected to the gondola's network through a custom-built interface cable. I verified it against the agency interface requirements — continuity testing on each conductor and end-to-end link verification with the payload — working with the electronics shop engineer in Western's Department of Physics and Astronomy.

---

## Results

- The payload flew and the communications system operated for the duration of the flight. *(Add flight date and duration.)*
- Camera settings were adjusted from the ground during flight, and temperature was correlated against imaging performance in real time using the metadata written into each frame's headers.
- Full FITS frames, compressed preview images, and photometry were all returned to the ground station.
- The **PASTIS link proved far more reliable than its documentation suggested** — comparable to a home Wi-Fi connection. See below.

*(Add: number of frames captured, total data volume downlinked, achieved throughput, and a sample image if cleared for release.)*

---

## What I would do differently

Writing this section honestly is more useful than the rest of the document, so:

**1. Use TCP, not a hand-rolled reliability layer.**
I built a UDP protocol with acknowledgements and selective retransmission because the agency documentation described the network as unreliable. It was not. It behaved like ordinary Wi-Fi throughout. TCP would have given me the same guarantees for free and saved weeks. My protocol works and is genuinely robust, but it was overkill for the conditions that actually occurred, and I paid for that in schedule.

**2. Negotiate for more bandwidth at the start.**
Most of the complexity in this project — the compression pipeline, the time-budgeted image downscaling, the packet segmentation — exists only because the allocated bandwidth was very small. A request for a modest increase early on (on the order of 10 kB/s) would have removed the need for most of it. **Ask for the resource before you build the workaround.**

**3. Ask the agency engineers earlier.**
The CSA and CNES engineers had solved most of these problems before. I spent significant time building things that existing solutions already covered. Asking how they would approach a problem, before starting on it, would have been the single highest-return habit.

**4. Build only what is strictly required.**
Some of what I built was not on the critical path to science return. Under a fixed launch date, that time has to come from somewhere, and it came out of the things that mattered.

None of this was wasted — I learned a great deal building it — but a second flight would be delivered in a fraction of the time.

---

## Skills demonstrated

**Languages and tools:** C++, Python, Windows and Linux command line, Git
**Hardware:** Camera Link frame grabbers, RS-232 serial protocols, PCIe topology and link negotiation, thermoelectric cooler control, sensor temperature calibration
**Imaging:** FITS, OpenCV, astropy photometry, JPEG compression pipelines, 12-bit sensor data handling
**Networking:** UDP, packet segmentation, selective retransmission, bandwidth-constrained transfer, link degradation simulation
**Engineering practice:** design reviews, agency interface compliance, vendor escalation, field integration and test, flight operations documentation

---

## Credits

This was one student's contribution to a multidisciplinary team. The POET-STRATOS payload was led by **Dr. Stanimir Metchev** (Department of Physics and Astronomy / Institute for Earth and Space Exploration, Western University), with flight operations support from the **Canadian Space Agency** and **CNES** at the Timmins Stratospheric Balloon Base.

*(Add teammates and their contributions.)*

**Press:** [CBC News — Western University prof, students in northern Ontario to launch tennis ball-sized camera into the stratosphere](https://www.cbc.ca/news/canada/sudbury/western-university-weather-balloon-experiment-timmins-1.7619343)

---

**Ali Mohammed-Ali** — Electrical Engineering, Western University
[LinkedIn](https://linkedin.com/in/am-uwo)
