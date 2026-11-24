Sweep

One RTL-SDR dongle, one app that picks up everything — and never makes anything up.

Sweep is a wideband scanner for a single RTL-SDR Blog V4. Voice bands tune, scan and record. Data modes decode the payload and show it — aircraft and ships on a map, APRS stations with what they said, ACARS messages as text. Each subject gets its own workspace: the thing itself, a switchable instrument beside it, and the actions that belong to it.

The one rule the whole program is built around: nothing is claimed that the signal didn't prove. Every decoder checks a frame's own checksum or parity before it reports anything, every frequency that drives the tuner comes from a real database record rather than a model's memory, and the few places where a format couldn't be verified without an off-air capture say so on screen instead of printing a confident guess. Data pulled from the internet (the OpenSky overlay, the online aircraft lookup) is always marked as not something the antenna heard.

<!-- Add a screenshot: drag an image into any GitHub issue comment to get a URL, or drop a PNG in a docs/ folder and point the line below at it, then delete this comment. ![Sweep](docs/screenshot.png) -->
What you get
A wideband scanner that sweeps a whole band and stops the moment something keys up — listens, hangs on the conversation, then resumes where it left off.
Voice in every common mode — AM, narrow FM, wide FM, SSB (USB/LSB) and CW — with squelch, an adjustable noise floor, and per-band presets (airband, marine, weather, 2 m, 70 cm, FRS, FM broadcast).
Six working decoders — ADS-B, AIS, APRS, ACARS, SSTV and POCSAG — each one checksum-verified before it shows you anything.
An aircraft map with ADS-B traffic on a range-ringed radar display, a sonar sweep, a settable home location, and per-plane detail (identity worked out from the signal alone, plus an optional internet lookup for type, route and photo — clearly flagged as internet data).
A live spectrum and waterfall, including a 3-D view.
An antenna-tune SNR meter for aiming or trimming an antenna by watching the number climb.
Two honest band surveys — drone bands and 433/915 MHz ISM — that report the shape of what they find without inventing a device name or a sensor value.

The desktop app is sweep-gui (eframe/egui). The signal processing lives in sweep-core, which has no USB, no libusb and no platform code — desktop and a future Android build link the same crate and each supply their own capture layer.

Honest by design

This is the part worth reading even if you never run it.

Provenance beats plausibility. validate() rejects any scan plan whose frequency source isn't a database record, "user", or "rules". A language model that produced a frequency from memory does not get to drive the tuner.
Checksums gate everything. No frame is reported until its own CRC, parity or error-control passes. Where a format couldn't be verified without a real capture — MIC-E in APRS, the checksum span in ACARS, the nibble order of numeric POCSAG — the UI labels it a reading rather than a fact.
Surveys don't pretend to be decoders. The drone and ISM views measure energy, timing and structure and phrase every verdict as "consistent with". They never name a device or print a sensor value they can't prove, because the hundreds of undocumented ISM formats would have to be invented to do so, and that's the same fabrication the decoders refuse.
Internet data is marked as internet data. The OpenSky overlay and the online aircraft lookup are clearly not something the antenna received.
Decoders
mode	where	modulation	payload
ADS-B	1090 MHz	pulse position, 1 Mb/s	aircraft positions
AIS	161.975 / 162.025	GMSK 9600, HDLC	ship positions and names
APRS	144.390 MHz	AFSK 1200, AX.25	station positions, text
ACARS	131.550 MHz	AM, MSK 2400	airline/aircraft messages
SSTV	14.230 MHz	FM subcarrier, VIS	pictures
POCSAG	931.0125–931.9875	2FSK, BCH(31,21)	pager text messages

A seventh decoder, RS41 (radiosonde, 400–406 MHz), has a complete and Reed-Solomon-verified decode engine in sweep-core (RS(255,231) over GF(256), with a full test suite); wiring its GFSK demodulator into the live receiver is in progress.

AIS and APRS share hdlc.rs; APRS and ACARS share tone.rs. Writing those once means a bug found in one decoder is fixed in both — and every bug those two layers had was subtle enough to pass a casual read and was caught only by a test.

POCSAG, as an example of the rule

Every 32-bit codeword carries a BCH(31,21) check and a parity bit, so the tests encode valid batches from CCIR Recommendation 584, modulate them as NRZ audio with a DC offset and a clock error, slice them back and compare — and the decoder is told neither the bit rate nor the polarity in advance. It runs all three standard rates in both polarities at once and lets the error control pick the winner, which is safe precisely because a wrong lane produces nothing rather than rubbish. Two things it declines to claim: published function-bit mappings contradict each other, so a payload is read both as text and as digits and judged by content (and when both are plausible, both are shown); and the nibble order of numeric messages can't be verified without a capture, so numeric pages are labelled as a reading. Alphanumeric pages carry no such doubt.

Surveys

drone.rs measures where energy sits in the drone-relevant bands an R828D can reach — 433 MHz ISM, 868 MHz SRD, 902–928 MHz ISM, and 1.2/1.3 GHz analogue FPV video — and classifies what it finds by bandwidth and duty cycle. It identifies nothing, and says so: Remote ID, the only broadcast carrying a drone's identity and operator position, is at 2.4/5.8 GHz and this tuner stops at 1766 MHz. Two non-obvious decisions make it correct: the noise floor is tracked across the whole band rather than per window (a window's own median sits on any carrier wide enough to fill it, which would hide a wide FPV downlink), and the DC spur at each window centre is recorded as "no measurement" rather than as quiet (recorded as quiet it punches a hole through a wide carrier every 1.8 MHz and splits one signal into five).

ism.rs analyses the structure of 433/915 MHz bursts — their timing fingerprint, line coding, raw bits, repeats, and whether a standard CRC-8 closes over the bytes — without ever naming a device or reporting a sensor value. Those hundreds of undocumented device formats are what rtl_433 built from years of captures; inventing their bit layouts would be fabrication, so the survey stops at what the signal proves and hands you the raw hex and a .cu8 recording to take further.

The desktop app

sweep-gui is an instrument-panel front end — a dark terminal look with a big amber VFO, a phosphor-green spectrum, bracket-framed panels and amber signal alerts. It runs the exact same plan → scheduler → spectrum/demod pipeline as the headless scan example, so the two never drift apart.

Across the top is a VFO display, a tuning knob, a direct-entry keypad, the mode buttons (AM / NFM / WFM / USB / LSB / CW), and squelch, volume, RF-gain and antenna-preamp controls. The workspaces are:

Spectrum / Whole band — live trace, waterfall, and a 3-D waterfall.
Aircraft map — ADS-B traffic on an azimuthal-equidistant radar display with range rings and a sonar sweep. Set your location by clicking the map; select a plane for its track, altitude and speed, its identity derived from the signal alone (US tail number, country of registration and operator, all computed with no network), and an optional internet lookup that adds type, route and a photo — shown as internet data, never as something the antenna heard.
Antenna tune — a live SNR meter. Point at a steady signal and watch the number as you trim or aim; it's relative (the dongle isn't calibrated), so only the change matters.
Pictures / ACARS / APRS / Ships / Pagers — the decoder outputs, each in its own view.
ISM sensors / Drone bands — the two surveys.

Settings persist between runs.

cargo run --release -p sweep-gui

Type a band name or a bare frequency (e.g. 146.520) and tune. The dongle opens with a retry loop, so you can launch the app before plugging the RTL-SDR in. Needs the two DLLs from the Windows setup below next to the built .exe (or RTLSDR_LIB_DIR set).

Quick start
powershell
# Clone and run the desktop app (Windows, with Rust and the RTL-SDR set up — see below)
git clone https://github.com/OWNER/Sweep.git
cd Sweep
cargo run --release -p sweep-gui
# Or just run the test suite — no hardware, no dongle, nothing to install but Rust
cargo test -p sweep-core

189 tests, all hardware-independent. sweep-rtl loads librtlsdr at runtime, so cargo check --workspace type-checks everything without a dongle attached.

Windows setup
Zadig. Windows binds its DVB-T TV-tuner driver to the dongle by default and librtlsdr can't open it through that. Options → List All Devices, pick Bulk-In, Interface (Interface 0), install WinUSB. If two interfaces show up it's 0, not 1. Nearly every "my dongle doesn't work" thread ends here.
Run rtl_test.exe from the prebuilt rtl-sdr package before anything else. If that doesn't run clean, nothing here will help.
setx RTLSDR_LIB_DIR C:\rtl-sdr\x64, and copy rtlsdr.dll and libusb-1.0.dll next to your .exe. It builds without them and fails to start.
Disable USB selective suspend in Power Options. Laptop power profiles suspend the dongle mid-capture and the stream just stops.
Build natively. WSL needs usbipd-win for USB passthrough and it isn't worth the days it costs.
How it's built
utterance ─► rules::parse ─► ScanPlan ─► validate ─► resolve ─► RunPlan
                                                                  │
IQ ─► Spectrum ─► Integrator ─► NoiseFloor ─► Detector ─► Tracker ─┤
                                    │                              │
                                 Squelch ─► audio                 UI
crate / module	does
sweep-core/plan	ScanPlan types, JSON round-trip, validate()
sweep-core/resolve	packs a plan into the fewest 2 MHz tuner windows
sweep-core/rules	utterance → plan, without a model
sweep-core/spectrum	windowed FFT, calibrated dB, DC blanking, integration
sweep-core/detect	noise floor, detection clustering, tracking, squelch
sweep-core/demod	IF→audio: anti-aliased decimation, FM discriminator, de-emphasis, AM/SSB/CW
sweep-core/scheduler	window walking, stop-and-listen, priority, mode arbitration
sweep-core/source	IqSource trait plus a deterministic FakeSource
sweep-rtl	IqSource over librtlsdr (loaded at runtime via libloading)
sweep-gui	the eframe/egui desktop app

sweep-rtl also ships three examples that are worth running in order: first_light (open the dongle, confirm a real signal), listen (NFM to audio through cpal), and scan (the scheduler driving real hardware — the actual scanner, not a fixed-frequency demo).

cargo run --release -p sweep-rtl --example listen -- 162.550
cargo run --release -p sweep-rtl --example scan -- "listen to marine"
Five things that are load-bearing

Window count is blind time. resolve() packs channels greedily over sorted frequencies, which is optimal for fixed-width interval covering, so the window count is minimal. One window means the scanner never retunes and misses nothing; eleven windows for the airband means each one is unobserved 91% of the time.

Integration is correctness, not optimisation. Noise in a single FFT frame is Rayleigh distributed — bins land 8–11 dB above the mean constantly, so a +8 dB threshold against one frame produces a storm of false detections. Integrator averages 16 frames in the power domain, taking the spread from ~5.6 dB to ~1.4 dB. This was a real bug, caught by running an example, not by reading the code.

The sweep resumes where it left off. After a stop-and-listen ends, the scheduler picks up at the window it was on, not at window zero — otherwise the first window gets checked far more often than the last.

The noise floor must estimate noise, not signal. Priming by copying the first frame bakes in any carrier already transmitting at startup, so priming clamps every bin to the frame median + 3 dB; and bins more than 6 dB over the estimate are frozen out (with a slow creep) so an idling repeater isn't absorbed into the floor. Each tuner window keeps its own floor — a busy 2 m sub-band and a quiet aircraft sub-band are not equally loud.

Decimation without a low-pass filter is aliasing, not decimation. LowpassDecimator filters before it drops samples; the test that catches this plants a 20 kHz tone that would otherwise fold straight into the audible band and checks it's gone.

Roadmap
Wire the radiosonde decoder to the live receiver. The RS41 engine and its tests are done; it needs a GFSK demodulator feeding it on 400–406 MHz and a Radiosonde view.
Android capture. UsbManager permission → getFileDescriptor() into libusb_wrap_sys_device, with an intent-filter matching 0bda:2838 / 0bda:2832 and larger buffers (16 × 16384) than on Windows.
CTCSS via a Goertzel bank on the sub-300 Hz audio.
wgpu render layer (DX12 on Windows) for the waterfall.
License

MIT — see LICENSE. Use it, learn from it, build on it.

Sweep is a receive-only hobby project for a single RTL-SDR dongle. What you may listen to, decode or record is governed by the radio laws where you live; that's on you, not the code.# sweep
