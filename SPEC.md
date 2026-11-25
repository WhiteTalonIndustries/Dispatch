# Dispatch: a 64-FSK HF Text Mode with ALE Link Layer

**Status:** Working Draft v0.28 (not frozen)
**Name:** Dispatch. Short tag for waterfalls, logs and mode lists: **DS64** (proposed). The name
"MFSK64" is already used by fldigi for a different mode and MUST NOT be used for this one.
**Licences:** Section 1.6.
**Date:** 2026-11-23

> Draft conventions. Sections marked **[SIM]** are backed by results from the reference
> simulator (`mfsk64_sim.py`). Sections marked **[PROPOSED]** are design proposals not yet
> validated by simulation. Sections marked **[TBD]** are open. Nothing in this draft is
> normative until v1.0, when a reference modulator/demodulator and test vectors agree.

---

## 1. Scope and conventions

**Intended use.** Contact between known stations: emergency communications (EmComm) and
business-band style traffic, not CQ calling, contests, or activator-style operating such as POTA and SOTA (no spots, no
portable-suffix identifiers). Consequences for the design: nearly
every frame is addressed (ADDR = 1 is the normal case; ADDR = 0 is for bulletins and net
announcements), links are set up between stations that already know each other, and the click
palette (Section 5.4.1) is weighted toward status, resource, location and weather words.


1.1 This document specifies Dispatch: (a) a physical layer for sending short text messages over
HF single-sideband audio channels using 64-ary orthogonal FSK, and (b) an Automatic Link
Establishment (ALE) layer built on it (Part 2, Section 8). Nothing in Sections 3-7 depends on ALE.

1.2 Design goals, in priority order:
1. Deliver one 140-character-class message in under 30 seconds.
2. Decode at negative SNR (target: reliable decode near -16 dB in 2500 Hz AWGN).
3. Occupy well under 2.8 kHz (about 500 Hz).
4. Be fully and publicly documented, so it qualifies as a "specified digital code" for amateur use.
5. Be implementable from this document alone, with test vectors.

1.3 The key words MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be read as in
RFC 2119 / RFC 8174 when, and only when, they appear in capitals.

1.4 Versioning: semantic versioning of the specification. Any change that alters the
on-air waveform or bit-level frame format is a MAJOR change. Protocol version is carried in
every frame (Section 5.2). Receivers MUST ignore frames with an unknown major version.

1.5 Normative annexes are frozen as explicit tables, never as "seed + algorithm", so that
no implementation depends on a particular random-number library.

1.6 **Licences.** Dispatch is published in three layers with different licences. No patents are
asserted on the mode.

| Layer | Contents | Licence |
|---|---|---|
| Specification | this document and its annexes (tables, vectors, model, dictionary) | MIT |
| Mode implementation | the reference code in `tools/` and the Rust core library `dispatch-core` (modem, codes, codec, frames, ALE state machine) | MIT |
| Application | the cross-platform Rust application (user interface, audio, rig control, and optional FT8, JS8 and RTTY decoders) | GPL-3.0-or-later |

Reasons:
- MIT for the mode lets anyone implement Dispatch, including in closed or differently licensed
  software, which helps adoption and keeps the mode a public, specified digital code (Section 2.2).
- The application is GPL because it may include or link GPL code: the WSJT-X FT8 decoder and
  JS8Call are GPL-3.0. GPL-3.0-or-later (not GPL-2.0-only) is needed to combine with them, and with
  Apache-2.0 Rust crates. MIT code can be used inside a GPL program, so the core library goes into
  the application unchanged, and it stays MIT for everyone else.
- The core library MUST NOT contain or depend on GPL code. FT8, JS8 and RTTY support belongs in
  the application layer only. (ft8_lib, an MIT FT8 implementation, could also be used in the core
  layer if that is ever wanted, but it is not needed there.)
- The specification and the mode are both MIT, so anyone may copy, adapt and redistribute them,
  including inside GPL or closed software. The application's GPL licence does not affect them.
  This is not legal advice; confirm the licence choices before the first public release.

1.7 **Reference application (informative).** A self-contained, cross-platform application
written in Rust (macOS, Windows, Linux), structured as a Cargo workspace:
- `dispatch-core` (MIT): pure Rust; modulator, demodulator, sync, LDPC encoders and decoders,
  text codec and dictionary, identifiers, frame builder and parser, ALE state machine. No audio,
  file or network access, so it can be tested on its own against Annexes C, D and F and the
  conformance test (Section 10.4).
- `dispatch-app` (GPL-3.0-or-later): the station: the ALE procedure run on 30 s UTC slots over
  an abstract radio, a simulator in which several stations share one fading channel in virtual
  time, and the command line. No dependencies.
- `dispatch-radio` (GPL-3.0-or-later): the radio for `dispatch-app`: audio capture and playback
  at 48 kHz with exact 4:1 conversion (Section 9.7), sample-count-to-UTC mapping that follows the
  sound card's clock, and PTT on a serial control line (Digirig: RTS) or VOX. Binary
  `dispatch-station`.
- Interface profiles (`dispatch-station --rig`): `digirig-mobile` (CM108 audio, PTT on RTS of
  its CP2102 serial port), `digirig-lite` (CM108 audio, PTT on the CM108's GPIO3 through a USB HID
  output report, as in Direwolf and Hamlib; macOS and Linux) and `vox`. PTT and the Ctrl-C release
  use operating-system calls directly; the only third-party crate is `cpal` for audio.
- `dispatch-modes` (GPL-3.0-or-later): RTTY, FT8 and JS8 encoders and decoders, used by
  `dispatch-station monitor` and the `modes` commands. FT8 decodes 78% of what WSJT-X decodes
  from ft8_lib's 27 on-air test recordings, with identical text. Both decode live traffic
  through a Digirig (24 Nov 2026: FT8 on 40 m at 7.074 MHz, and JS8Call frames), which also
  confirms the station's sample-count-to-UTC mapping (FT8 DT about +0.1 s).
- `dispatch-gui` (GPL-3.0-or-later; egui/eframe, the only other third-party crates): window
  with waterfall and tabs for FT8, JS8, RTTY and Dispatch; transmit in all four; JS8 directed
  commands, groups and saved messages; message store; automatic operation (8.13, 8.14). Verified
  on the air on 25 Nov 2026 (JS8: directed queries and replies, a custom group answered by a JS8Call
  2.2 station; reports through PSKReporter).
- `dispatch-tui` (in `dispatch-radio`; macOS and Linux): the same station in a terminal, drawn
  with ANSI escape codes. The GUI and terminal versions share `dispatch-app::session` (settings,
  log, message store), so either can be used with the same configuration.
- Planned (GPL-3.0-or-later): click palette, UTC time check, Morse identification for the
  business profile, ADIF logging.
- On macOS the application MUST be a signed app bundle that declares microphone use
  (NSMicrophoneUsageDescription), or capture returns silence (Section 9.7).
- Status (v0.25): `dispatch/` holds the Cargo workspace with `dispatch-core` and `dispatch-cli`
  (both MIT, no dependencies), `dispatch-app` and `dispatch-radio` (GPL). The core is bit-exact with Annexes C, D and F, its modulator output
  is within 1 LSB of the Annex D and Annex F WAVs, and its receiver passes the Section 10.4 test
  with the same results as the Python reference (Section 10.4). The core also holds the short ALE
  frame (`short`), the receiver's soft-value output for combining (`Receiver::observe`,
  `Receiver::decode_llr`) and the sender and responder state machines of Section 8.5 (`ale`),
  tested end to end at sample level (Section 8.9). `dispatch-app` runs whole exchanges between
  simulated stations, including the channel check of Section 8.5 and monitoring without automatic
  replies; `dispatch-station` runs the same station on a Digirig and is not yet tested on the air.

---

## 2. Regulatory profiles (US; informative)

This section is informative and is not legal advice. It records how this mode relates to the
rules as read on 2026-11-24 from the eCFR and the Cornell LII copy of 47 CFR. Operators outside
the US MUST check local rules, and everyone SHOULD verify the current text before relying on it.
Where this section gives an interpretation, no FCC ruling on this mode is known.

### 2.A Amateur profile (Part 97; PROFILE = 0, the default)

2.1 **Bandwidth.** HF data emissions are limited to a 2.8 kHz bandwidth (97.307(f)(3)) rather
than a symbol rate. Part 97 defines bandwidth as the width outside which the mean power is at
least 26 dB below the mean power within the band (97.3(a)(8)). This mode measures about 527 Hz
at -26 dB (Section 3.2), well inside 2.8 kHz.

2.2 **Documentation.** 97.309(a)(4) allows, as a specified digital code, any technique whose
technical characteristics are documented publicly, such as CLOVER, G-TOR or PacTOR, used to
facilitate communications. Publishing this specification with its annexes and reference source
is intended to meet that. Implementations MUST NOT apply any undocumented transform to
over-the-air data. Unspecified codes are also allowed (97.309(b)), but must not be used to
obscure meaning, and a Regional Director may order a station to stop using one.

2.3 **No obscured meaning.** 97.113(a)(4) prohibits messages encoded to obscure their meaning.
Compression and the dictionary (Section 5.4) are acceptable only because both are public and
fixed. There MUST be one public dictionary and no private or per-group code tables, and the
ALE layer MUST NOT include encryption. Integrity checks (CRC) are fine.

2.4 **No business traffic.** 97.113(a)(3) prohibits communications in which the licensee or
control operator has a pecuniary interest, including communications on behalf of an employer.
There is an exception for emergency preparedness and disaster drills on an employer's behalf,
limited to the drill and its immediate testing; drills not sponsored by government are limited
to one hour per week, except up to 72 hours at a time no more than twice a calendar year.
Business traffic belongs in the business profile (Section 2.B), under a licence for that service.

2.5 **Station identification.** 97.119(a) requires the call sign at the end of each
communication and at least every 10 minutes during one. 97.119(b)(3) allows identification by a
specified digital code when the communication is by data. Every frame carries the sender's
identifier in a documented, public format (Section 5.3), so each transmission identifies the
station in the same emission and code as the traffic. This is the same approach as FT8 and JS8,
which also pack call signs into their frames. Limits:
- The identifier holds up to 6 characters (or a 7-character form, Section 5.3) without a slash.
  Indicators the rules require MUST be sent in the message text, at least every 10 minutes and
  at the end of the communication. These are: a self-assigned indicator if used (97.119(c)); the
  control operator's own call sign after the station's when the control operator holds a higher
  class (97.119(e)); /KT, /AG or /AE when operating on upgrade privileges (97.119(f)); and for
  reciprocal operation the location prefix, such as W4/VE3ABC, plus at least once per
  intercommunication the city and state (97.119(g)). The application SHOULD add these to the
  text automatically from its settings.
- US amateur call signs have at most 6 characters and always fit. The header carries the
  station's own call sign; prefixes and suffixes go in the text as above.

2.6 **Automatic control.** 97.221 allows automatically controlled data stations on 6 m and up,
and on these HF segments: 28.120-28.189, 24.925-24.930, 21.090-21.100, 18.105-18.110,
14.0950-14.0995 and 14.1005-14.112, 10.140-10.150, 7.100-7.105 and 3.585-3.600 MHz. Elsewhere, an
automatically controlled station may transmit only when responding to interrogation by a station
under local or remote control, and then no transmission may occupy more than 500 Hz.
Consequences:
- Scanning and listening are receive-only and always permitted.
- Initiating a CALL MUST be done under local or remote control by a control operator.
- A full frame of this mode measures about 527 Hz at -26 dB (Section 3.2), so under that reading
  it does not meet the 500 Hz condition. Automatic responses (RESP, ACK, automatic repeats) MUST
  therefore be either confined to the 97.221(b) segments, sent with a control operator present,
  or sent in a frame format narrower than 500 Hz. The short ALE frames (Section 8.3) are 277 Hz
  wide at -26 dB, and automatic replies use only short frames, so they are possible anywhere
  data is allowed (Section 8.6).
- Automatic responses MUST ship OFF by default, and the application MUST show clearly when the
  station is transmitting under automatic control.

2.7 **Interference.** Stations MUST listen before transmitting on a channel and SHOULD back off
when the channel is busy. Control operators MUST be able to stop automatic operation at once.

### 2.B Business profile (Part 90; PROFILE = 1) **[PROVISIONAL]**

The business profile is for stations licensed under Part 90 (Industrial/Business Pool). It is not
a way to send business traffic on amateur frequencies. The operator MUST hold a licence that
covers the frequency, the emission and the use. Findings so far:

2.8 **HF (below 25 MHz) is narrow.** Part 90 HF use under 90.266 requires 90.35(c)(1)
eligibility and a communications plan, and is limited to circuits of high national importance
where other means are unavailable, or standby and backup circuits for domestic communications
disrupted by disaster. Emissions are limited to 2K80J3E, 100HA1A, 100HA1B and those in
90.237(g). 90.237(g) lists A2B and F2B radioteleprinter emissions on J3E-certified transmitters,
and A1B, A2B, F1B, F2B, J2B and A3C for standby backup circuits. This mode is text for automatic
reception on an SSB transmitter, which would most plausibly be classed J2B (about 500HJ2B). That
fits only the backup-circuit case. HF business use therefore needs the specific eligibility and
licensing, and possibly a frequency coordinator or the FCC to confirm the emission class.

2.9 **VHF/UHF.** Part 90 stations above 25 MHz normally use FM; this mode through an FM radio's
audio input is an F2B or F2D emission. 90.207 allows data emissions in several cases, including
under digital-voice authority subject to 90.233, any modulation meeting 90.209 on 3.6 kHz
channels, and other emissions case by case. The licence MUST list a suitable emission designator.
FM operation of this mode has not yet been simulated (deviation, de-emphasis, squelch tail).

2.10 **Identification is different.** 90.425 requires the call sign during each transmission or
exchange of transmissions, or once each 15 minutes (30 in the Public Safety Pool) during
continuous operation, by voice in English or International Morse. Stations using non-voice
emissions must identify in unscrambled form, using A3E, F3E or G3E, or Morse. The identifier
inside the frame therefore does NOT satisfy Part 90. In the business profile the application
MUST send a Morse identification: tone 1200 +/- 800 Hz, 20-25 words per minute, at 40% +/- 10% of
maximum modulation if fed into the microphone input (90.425(b)). It MUST send it at the end of
each exchange and at least every 15 minutes during continuous operation, using a slot of its own
when it does not fit the 2.4 s gap after a frame. A 7-character call sign at 20 words per minute
takes about 5-6 s, so it needs a slot.

2.11 **Call signs.** Part 90 call signs are often 7 characters (for example WQAB123 or KAE1234).
The identifier field holds these as 7-character forms (Section 5.3).

2.12 **Still to settle.** Which Part 90 frequencies and pools this profile targets; the emission
designator; whether a frequency coordinator must approve data on the channel; FM simulation.

---

## 3. Physical layer **[SIM]**

### 3.1 Parameters

| Parameter | Value |
|---|---|
| Modulation | 64-ary orthogonal FSK, one tone per symbol, Gaussian-smoothed transitions (GFSK, BT = 4) |
| Bits per symbol | 6 |
| Sample rate (reference) | 12000 Hz |
| Symbol duration T | 1536 samples = 128 ms |
| Symbol rate | 7.8125 baud |
| Tone spacing | 1/T = 7.8125 Hz (equal to symbol rate; orthogonal under noncoherent detection) |
| Tones | 64, index k = 0..63 |
| Tone frequency | f(k) = f0 + k x 7.8125 Hz |
| Occupied bandwidth | 493 Hz (99% power); 544 Hz at -30 dB; -55 dB of power outside 950-1550 Hz |
| Default f0 | 1000 Hz audio (SHOULD be configurable; band edge at 1000 Hz plus 500 Hz) |

3.2 **Waveform (GFSK).** Continuous-phase FSK with Gaussian frequency smoothing. The
instantaneous frequency is f(t) = f0 + 7.8125 * sum_k a_k g(t - kT), where a_k is the tone index
of symbol k and g is a rectangle of length T convolved with a Gaussian of bandwidth-time product
BT = 4:

    g(t) = 1/2 [ erf(pi*sqrt(2/ln2)*BT*(t/T + 1/2)) - erf(pi*sqrt(2/ln2)*BT*(t/T - 1/2)) ]

truncated to |t| <= 1.5T. Before the first and after the last symbol the tone index is held at
the first and last values. The phase is the running integral of f(t). The transmission starts
and ends with a 16 ms (T/8) raised-cosine amplitude ramp. Shaping MUST NOT lengthen the symbol
period or move the tone centres. Reference code: `modulate` in `mfsk64_modem.py`.

Measured (`tools/shaping2.py`, 12 kHz, random data):

| Shaping | Energy in correct bin | -26 dB width | -40 dB width | Power outside 950-1550 Hz |
|---|---|---|---|---|
| None (rectangular) | 0 dB | 579 Hz | 929 Hz | -33 dB |
| GFSK BT = 6 | -0.10 dB | 535 Hz | 617 Hz | -48 dB |
| **GFSK BT = 4 (this spec)** | **-0.28 dB** | **527 Hz** | **587 Hz** | **-55 dB** |
| GFSK BT = 2 (FT8 value) | -1.05 dB | 524 Hz | 552 Hz | -70 dB |
| Raised-cosine transition, 15% of T | -0.28 dB | 535 Hz | 604 Hz | -49 dB |

BT = 2 (the FT8 value) costs too much here: with 64 tones a typical transition jumps about 21
tones, and the sweep spreads energy across many bins. BT = 4 gives about 22 dB less splatter for
a measured sensitivity loss of about 0.35 dB at the 50% point (AWGN, 80 trials per point,
`tools/shaping_perf.py`).

3.3 **Bit-to-tone mapping.** Natural binary: 6 bits (MSB first) give the tone index. Gray
mapping gives no benefit for orthogonal FSK, because every wrong tone is equally likely.

3.4 **Frequency and timing tolerance.** Transmitters MUST hold audio-frequency error within
+/- 1 Hz of nominal and sample-clock error within +/- 50 ppm over a frame. Receivers MUST search
at least +/- 5 Hz in frequency and +/- 1.0 s in time around the nominal values (Section 4.3). Receivers SHOULD tolerate linear audio-frequency drift of at
least 0.3 Hz/s (8 Hz over a frame) and SHOULD track drift as in Section 4.3. Receivers SHOULD
tolerate sample-clock error of +/-300 ppm (measured: no loss at 300 ppm, about 20% frame loss at
1000 ppm and -16 dB; Section 9.3).

3.5 **Receiver front end (normative for Demodulators).** A Demodulator MUST remove steady
carriers and suppress impulsive noise before the symbol FFTs.

*Carrier removal.* The reference receiver takes one FFT over the whole 30 s buffer (about
0.03 Hz resolution). Any bin more than 13 dB above the local median level (1001-bin running
median), and 3 bins on each side, is set to zero before the inverse FFT. The wanted signal spends
only 1/64 of the time on any tone, so its long-FFT bins stay far below the threshold.

*Noise blanker.* A Demodulator MUST suppress impulsive noise, for example with a noise blanker. The reference blanker sets to
zero every sample whose magnitude exceeds 4 robust standard deviations, plus 2 ms on each side.
The robust standard deviation is measured locally (median |x| of each 6 ms block, then the median
over 7 blocks, about 40 ms), so the blanker keeps working when rig AGC moves the level. Without it, static crashes of 2 ms at +30 dB above the noise and 5 per
second caused 100% frame loss at -16 dB; with it, 0% (Section 9.3). It does not trigger on the
signal itself at any SNR tested (up to +20 dB).

3.6 **Interference-aware decoding (reference receiver).** Each candidate is decoded up to three
times, each pass only if the previous CRC failed. Pass 1 uses the soft metric of Section 3.2.1.
Pass 2 is the same after per-symbol gain normalisation (Section 3.7). Pass 3, also
gain-normalised, erases cells (symbol,
tone) that interference has hit: a cell's likelihood ratio is set to 1, so it says nothing either
way. A cell is busy when its power exceeds 4 N0. A bin that is busy in at least 20% of the
surrounding 17 symbols carries interference, since the wanted signal visits a bin only 1/64 of the
time and noise exceeds 4 N0 about 2% of the time. Busy regions are grown by one bin and one symbol.
Any single cell above 4 (Es + N0) is also erased. N0 and Es are re-estimated from cells that are not
busy. Passes 2 and 3 alone cost about 0.1-0.25 dB on clean signals; running them only when pass 1
fails avoids that cost. On 300 noise-only slots (900 candidates, three passes each) no pass
produced a CRC-valid frame.
Reference code: `_decode_plain`, `_decode_erase` in `mfsk64_modem.py`.

3.7 **Per-symbol gain normalisation (reference receiver).** Rig AGC raises and lowers signal and
noise together, so the frame no longer has one noise level. Each symbol is divided by its own
level: the median bin power over that symbol and its neighbours (3 symbols x 64 bins). The
coarse sync search also divides each spectrogram row by its median. Reference code:
`gain_normalise` in `mfsk64_modem.py`.

### 3.2.1 Demodulation (informative)

For each symbol, correlate the received signal with each of the 64 tones over one symbol
period (or take a 1536-point FFT; bin spacing equals the tone spacing). Let r_m be the complex
bin value and Es/N0 the per-symbol signal-to-noise ratio. The soft metric is
`ln I0(2*sqrt(Es/N0) * |r_m|)`, and each bit's LLR is the log-sum-exp of the metrics of the
tones whose label has that bit 0, minus the same for bit 1. Positive LLR means bit 0.

---

## 4. Frame structure

### 4.1 DATA frame **[SIM]**

| Item | Value |
|---|---|
| Information bits (including CRC-16) | 576 |
| Coded bits | 1152 (rate 1/2) |
| Data symbols | 192 |
| Sync symbols | 24 (Section 4.3) |
| Total | 216 symbols = 27.648 s |
| Slot | 30 s, starting on UTC :00 and :30 (Section 8.3) |

Transmission starts 0.5 s after the slot boundary (nominal), leaving 1.85 s of guard
at the end for propagation delay and clock error.

### 4.2 Bit pipeline

```
payload (560 bits) -> CRC-16 (+16) -> 576 info bits
 -> LDPC (1152,576) -> bit interleave -> 6-bit groups -> tone index -> waveform
```

### 4.3 Synchronization **[SIM]**

**Pattern.** 24 sync symbols in three 8-symbol blocks at frame symbol positions 0-7,
104-111 and 208-215 (data fills 8-103 and 112-207). Each block is an order-8 Costas array
c(0..7); the tone index is 8*c + 4, so the sync tones are spread across tones 4..60 and both time
and frequency offsets give unambiguous correlation peaks.

| Block | Costas array | Tone indices |
|---|---|---|
| 1 | 0 1 4 6 5 3 7 2 | 4 12 36 52 44 28 60 20 |
| 2 | 0 7 4 6 2 1 5 3 | 4 60 36 52 20 12 44 28 |
| 3 | 1 6 5 3 7 4 0 2 | 12 52 44 28 60 36 4 20 |

These are arrays #0, #40 and #80 of the 444 order-8 Costas arrays in lexicographic order of
the permutation. Using three different arrays suppresses false peaks when only some blocks align.

**Reference receiver search (informative).**
1. Coarse: spectrogram of the audio mixed by each of -5, -2.5, 0, +2.5, +5 Hz, with a hop of T/8
   (192 samples). Each frequency bin is divided by its own median over time, and every cell's
   value is softly limited (linear up to 30, logarithmic above), so carriers and strong signals
   in some bins do not swamp the metric. The refine and drift-tracking metrics use the same soft
   limit. The metric for a candidate start time is the sum of bin energies at the 24
   expected (symbol, tone) pairs. Start times from 0 to 2.0 s are tried. The three best, well
   separated peaks are kept.
2. Refine: around each candidate, +/-192 samples in 32-sample steps and +/-1.25 Hz in
   0.625 Hz steps.
3. Demodulate and decode each candidate in order, and stop at the first frame whose CRC passes.
   Noise power per bin is estimated as the median bin power divided by ln 2. Es is estimated
   from the energy at the sync tones.

**Measured behaviour (sample-level reference modem, `mfsk64_modem.py`).**
- The sync search finds the frame in 97-100% of trials from -19 dB (AWGN) upward and in all
  ITU channel profiles tested.
- Compared with decoding the same noisy audio using the true timing and frequency, the sync
  loss is about 0.1 dB in AWGN and at most about 0.3 dB in fading, which is within the noise of
  the 40-60 trials per point.
- **Drift tracking.** After the refine step, the receiver finds the best frequency offset
  separately for each sync block (+/-8 Hz around the refined value in 0.25 Hz steps), fits a
  straight line through the three block centres, and demodulates each symbol at its own offset.
  If the fitted change over the frame is under 0.5 Hz, the mean of the three values is used.
  Measured at -15 dB (30 trials, `tools/drift_test.py`): without tracking, 0.2 Hz/s is lossless
  and 0.3 Hz/s loses 33% of frames; with tracking, 0.5 Hz/s (14 Hz over the frame) is lossless,
  0.7 Hz/s loses 17%, and 1.0 Hz/s fails because the coarse search has no drift hypothesis.
  Tracking caused no loss at -17 dB with zero drift.
- **Sync threshold and false alarms.** The normalised sync metric is the mean sync-tone bin
  power divided by the noise estimate. For noise alone the best
  candidate of a slot scores about 1.7 on average, because the search keeps the highest of many tries. On 300 noise-only slots (900 decode
  attempts, `tools/fa_test.py`) its maximum was 2.44, and LDPC never converged, so no false
  frame was produced. Weak signals that decoded had a metric of 5.6 or more. Receivers SHOULD
  skip candidates with a metric below 3.0. With LDPC non-convergence on noise (fewer than 3 in
  900 at 95% confidence) and the 16-bit CRC, the false-frame rate is below about 5e-8 per
  attempt, or roughly one false frame in 7 years of continuous monitoring with 3 candidates per
  30 s slot.

---

## 5. Payload and source coding

### 5.1 CRC **[SIM]**

CRC-16/CCITT-FALSE: polynomial 0x1021, init 0xFFFF, MSB-first, no reflection, no final XOR,
computed over the 560 payload bits and appended MSB-first. A frame is accepted only if the
CRC matches. Test vector: **[TBD]**.

### 5.2 Payload layout **[PROPOSED]**

| Bits | Field |
|---|---|
| 4 | Frame type (Section 8.2) |
| 4 | Protocol version (high 2 bits: major; low 2 bits: minor) |
| 1 | ADDR: 1 = a destination identifier follows the body (footer present); the normal case |
| 1 | PROFILE: 0 = amateur (default), 1 = business (Section 5.6) |
| 1 | MORE: 1 = the message continues in the next part (Section 5.7) |
| 3 | PART: part number of a chained message, 0 to 7 (0 for a single-frame message) |
| 6 | Link ID (0 for broadcast; 1-63 assigned during link setup) |
| 32 | HEADER identifier: sender (Section 5.3) |
| 476 or 508 | Message body (compressed text, Section 5.4): 476 bits when ADDR = 1, 508 when ADDR = 0 |
| 32 | FOOTER identifier: destination (present only when ADDR = 1) |

Total 560 bits. The CRC makes 576. The body is the same coded text in both cases; ADDR = 1 trades
32 bits of text capacity (about 9 characters) for an addressed frame. A bulletin or net
announcement sets ADDR = 0.

### 5.3 Station identifier packing **[SIM]**

Every frame carries the sender's identifier in the header, so each frame identifies its station
(Section 2) without relying on the text. An addressed frame also carries the destination
identifier in the footer, so an identifier appears at both ends of the payload. The CRC covers the
whole frame, so the second copy adds no error protection; it is there for addressing and logging.

One format is used in both profiles. An identifier is 1 to 6 characters from 0-9 and A-Z (so it
holds any amateur callsign of six characters or fewer, such as W4ABC, KD9XYZ, VE3ABC, 3DA0AB, and
any six-character business identifier). There is no modifier field: /P, /R and similar go in the
message text. Packing: identifiers are numbered in order of length and then base-36 value; there
are 2,238,976,116 of them (36 + 36^2 + ... + 36^6), which is less than 2^32, so the field is
32 bits. Values from 2,238,976,116 up are reserved. An identifier longer than six characters, or
with other punctuation, is not supported. Reference code and tests: `tools/callsign.py`
(`pack_biz`, `unpack_biz`).

- **Worked example.** KQ4TGD packs to 1315412113 (0x4E679891). `tools/callsign.py` checks it.
- **7-character forms.** Values above the 1-6 character range hold three 7-character forms,
  which cover common Part 90 and GMRS call signs: 4 letters + 3 digits (WQAB123), 3 letters + 4
  digits (KAE1234) and 2 letters + 5 digits (KA12345). Form k (in that order) starts at a base
  value; the value is base + (letters as a base-26 number) x 10^digits + (digits as a number).
  The bases are 2,238,976,116, 2,695,952,116 and 2,871,712,116, and all identifiers are below
  2,939,312,116; the values from there up to 2^32 are reserved. Reference: `pack_id`, `unpack_id`
  in `tools/callsign.py`; WQAB123 packs to 2,636,465,239.
- **Why not plain text.** In the text model a callsign costs about 40 to 60 bits, because
  uppercase letters and digits are rare in the training text. The packed form is a fixed 32 bits
  and does not depend on the model.
- **Why not a tighter amateur-only form.** A prefix-digit-suffix packing needs only 28 bits but
  cannot hold calls such as 3DA0AB or any business identifier. One format for both profiles
  costs 4 bits per field and removes the special cases. `tools/callsign.py` keeps the 30-bit
  form (`pack`, `unpack`, with a modifier) as a reference only.
- **Tactical names.** EmComm traffic also uses tactical names (for example a hospital or an
  EOC). The header and footer carry the station's licensed or assigned identifier. Tactical
  names go in the message text, and the dictionary already has some of them as words.
- **Capacity.** With the v1 model (Section 5.4), 508 bits holds about 151 characters on average and
  476 bits about 139. An unaddressed frame has about 10 characters of margin over the 140-character
  goal. An addressed frame has none, so ordinary text is on the edge, and the chain flag (Section
  5.7) carries longer messages. The averages move when the text model is retrained.

### 5.4 Text compression (codec v1) **[SIM]**

Message text is converted to symbols, coded with a static order-2 model and a 32-bit arithmetic
coder, and padded with zero bits to the message body (476 or 508 bits, Section 5.2). Reference
code: `tools/textcodec.py`; frozen model and vectors: Annex C.

- **Symbols (99).** 0-94 = ASCII 0x20-0x7E; 95 = LF; 96 = EOT; 97 = ESC, followed by one raw byte
  (8 bits, flat probability) for every byte of non-ASCII (UTF-8) text; 98 = WORD, followed by a
  dictionary index and a case code (Section 5.4.1). TAB is coded as space and CR is dropped. The
  message is the symbols followed by EOT.
- **Model (Annex C).** An order-1 table (99 x 99) and the listed order-2 rows (452 in the v1
  model). The context is the last two symbols (a, b); at the start of a message it is (LF, LF),
  after a raw byte the context symbol is ESC, and after a WORD it is the last two characters of
  the word as written. If (a, b) is listed in the order-2 table, that row is used; otherwise the
  order-1 row for b. Every row holds 99 integers, each at least 1, summing to exactly 32768.
- **Coder.** 32-bit integer arithmetic coder (Witten-Neal-Cleary with bit-follow underflow
  handling), MSB first, with `rng = high - low + 1`, `high = low + rng*hi/total - 1`,
  `low = low + rng*lo/total` (integer division). Symbols and dictionary indices use
  total = 32768, raw bytes total = 256, case codes total = 4 (lower 2, Capitalised 1, UPPER 1).
  Termination: increment the pending count, then emit 0 if low < 2^30, else 1, followed by the
  pending opposite bits. The decoder treats bits past the end as zeros and stops at EOT.
- **Fitting a frame.** A sender MUST include only as much text as fits in the body (476 or 508
  bits). Longer text is sent as a chained message (Section 5.7), split at the last space that
  fits, or at the longest fitting prefix if there is no space.
- **Measured (v1 model, 28 handwritten held-out messages).** 3.41 bits/char including EOT and
  coder flush (v0 without dictionary: 3.66; order-0 Huffman: 4.98; ASCII: 7.0). Capacity over
  17 windows of that text: 476-bit body, 104-167 characters, mean 139; 508-bit body, 109-180,
  mean 151. Digit- and callsign-dense text is at the low end.
- **Caveat.** The model was trained on two public texts (Newton's *Opticks* and the Python
  language reference) plus 28 handwritten messages (weighted x16). The test set is a separate 28
  handwritten messages from the same author, so these numbers are optimistic. The model MUST be
  retrained on real amateur-radio and EmComm traffic before v1.0, and changing it is a MAJOR
  change to the over-the-air format (Section 1.4).

#### 5.4.1 Word dictionary and click palette **[SIM]**

The purpose of the dictionary is shorter transmissions for long words. It does not hide meaning.
It is one public table of up to 220 words or phrases, each with a public 4-digit code. Draft
contents: `corpus/dictionary_v0_draft.txt` (`code|category|text`), carried into the model file.

- **Operator model (click palette).** The application shows the table as a palette grouped by
  category. Clicking an entry inserts it into the message. The compose buffer stores the entry
  as a private-use marker character followed by the 4-digit code: U+E000 for lower case, U+E001
  for Capitalised, U+E002 for UPPER. A clicked code is therefore never confused with digits the
  operator typed: a typed 9999 is four digit characters, a clicked entry is a marker plus a code.
  The application shows the word; it MAY also show the code. An encoder MUST reject a marker
  followed by anything other than a valid code.
- **On-air coding.** An entry is sent as a WORD symbol, its index (0-219, in table order) and a
  case code. The 4-digit code itself is not transmitted: the index costs fewer bits, and CRC and
  LDPC already protect the frame.
- **Receiver output.** The receiver shows each WORD as the word in the coded case. On the air a
  clicked entry and a typed word that the encoder matched to an entry are identical, so a
  receiver that reproduces the compose-buffer form (with markers) marks both. Reference:
  `decode_bits(..., markers=True)` and `canonical()` in `tools/textcodec.py`.
- **Encoder policy (not normative for decoders).** Clicked entries always use WORD symbols. Typed
  text MAY be matched to single-word entries of 4 or more letters, and to multi-word phrases
  always, provided the case is one of the three forms. Shorter words stay as characters because
  WORD costs more than the letters.
- **Probabilities.** Index probabilities are half trained frequency, half uniform, so an
  unseen entry costs no more than about 9 bits.
- **Code properties** (checked by script): codes are distinct, differ in at least 2 digits, and
  swapping two neighbouring digits never gives another valid code, so single-digit typos and
  neighbour swaps are caught at entry.
- **Contents.** 94 entries are the supplied "Code Table No. 1". The other 126 are a draft in
  nine categories (replies, time, radio, places, weather, emergency, people, linking words).
  Contents, order and code assignment are **[TBD]**; changing them changes the model file and is a
  MAJOR change.
- **Regulatory.** There MUST be exactly one public dictionary, published with this specification;
  private or per-user code tables are not allowed (Section 2.3).
- **Measured (v1 model).** Clicked words cost about 16 bits each including the space (uniform
  random clicks), so about 29 fit a 476-bit body. The Annex D frame "Emergency power outage at the
  shelter. need fuel and water by 1400. ALL OK here." (81 characters, 6 entries) is
  well inside one addressed frame.

### 5.6 Operating profile **[PROPOSED]**

The application has one checkbox, "Business band use", **off by default**. Off is the amateur
profile (PROFILE = 0): the US amateur rules in Section 2. On is the business profile (PROFILE = 1).

- **On the air.** PROFILE is one header bit (Section 5.2). A receiver applies the rules of the
  frame's profile, and ignores frames of the other profile unless the operator
  chooses to monitor both. The PHY, frame, FEC and text codec are the same in both profiles.
- **Identifiers.** Both profiles use the same 32-bit identifier (Section 5.3).
- **Identification.** The business profile MUST add a Morse identification as Section 2.10
  requires; the in-frame identifier does not satisfy Part 90.
- **Application behaviour.** Turning the box on clears the amateur-only settings, asks the
  operator to enter the licence or authorisation identifier to be used, and shows a notice that
  the operator is responsible for the rules of the service and frequency in use. The profile
  selects local rules; it does not grant any authority to transmit.
- **Dictionary.** The same public dictionary is used in both profiles. Business-specific entries
  are **[TBD]**, if the 220 slots allow.
- **Regulatory text.** Section 2.B.

### 5.7 Chained messages **[SIM]**

A message that does not fit one body is sent as up to 8 parts, one per frame, and joined
after decoding. About 1,100 characters of ordinary text is the limit.

- **Header fields.** MORE = 1 on every part except the last. PART counts 0, 1, 2, ... A
  single-frame message has MORE = 0 and PART = 0. The fields cost 4 bits of body, which is
  about 1 character per frame; the 4 bits come out of the body, not the link ID.
- **Splitting (sender).** Each part is coded on its own, starting from the start context, and
  holds as much text as fits the body. A part ends before the last space that fits, and that space
  is the first character of the next part, so the receiver joins by plain concatenation. Text
  with no space is cut at the longest fitting prefix. Reference code: `split_message` and
  `join_parts` in `tools/textcodec.py`; test: `tools/chain_test.py`.
- **Joining (receiver).** Parts with the same sender, destination and link ID, and consecutive
  PART numbers, are joined once the part with MORE = 0 has arrived. If a part is missing, the
  application shows the parts it has with a visible gap marker. It does not guess. PART 0 after a
  higher part starts a new message.
- **Tested.** 200 random messages of 150 to 1000 characters, at body sizes 476 and 508, split and
  rejoin exactly; they need 3.2 to 3.4 parts on average and at most 6.
- **Cost.** Each part starts from the start context, which costs a few bits per part, and a
  chained message takes one 30 s slot per part, so a 4-part message takes 2 minutes. Repeat and
  acknowledgement of single parts belong to the ALE layer (Section 8).

---

## 6. Forward error correction **[SIM]**

6.1 **Code.** Systematic rate-1/2 irregular repeat-accumulate style LDPC code,
(N, K) = (1152, 576). Codeword = [u (576 bits) | p (576 bits)].

6.2 **Parity-check matrix.** H = [Hs | Hp]. Hp is the dual-diagonal accumulator: row i
contains parity bits p_i and p_(i-1) (row 0 contains only p_0). Hs is given explicitly in
Annex A. Properties of the frozen matrix: info-column weights 8 (40%) and 3 (60%); row
weights 4-5 in Hs; no 4-cycles.

6.3 **Encoding.** s = Hs u (mod 2); p_i = s_0 xor s_1 xor ... xor s_i.

6.4 **Interleaver.** Fixed permutation in Annex B. Transmitted bit j is coded bit perm[j].

6.5 **Decoder (informative).** Sum-product (tanh) belief propagation with soft LLRs from
Section 3.2.1, up to 80 iterations, stopping early when all parity checks are satisfied. The
frame is accepted only if the CRC also passes. Normalised min-sum with scale
factor 0.8 is an acceptable alternative: on identical frames (150 per point, AWGN,
`tools/minsum_test.py`) it lost about 0.05-0.1 dB against sum-product. Plain min-sum lost about
0.35 dB, normalised 0.75 about 0.1-0.15 dB, and offset min-sum (0.3) about 0.05-0.1 dB.

6.6 **Alternatives considered.** A K=7 rate-1/2 convolutional code (171/133 octal) with
Viterbi decoding, as used in MFSK16, was simulated and is about 0.5-1 dB less sensitive.

### Annex A (normative): LDPC matrix, info part

File: `annex/ldpc_1152_576_info_columns.txt`
SHA-256: `baa84c68bc1a4e8fad49190defabde40946d6cdd90ad9bf0a0623912337eed4e`

### Annex B (normative): interleaver

File: `annex/interleaver_1152.txt`
SHA-256: `ebb7ff61d388f93745d54061365085adf9a8041c25963b846547d204656e6b54`

Both tables are provisional in v0.1 and will be re-frozen, with new digests, before v1.0 if
the simulation work changes them. They were generated by `tools/gen_annex.py`.

### Annex C (normative): text model and test vectors

File: `annex/textmodel_v1.txt` (dictionary with codes and index frequencies, order-1 and
order-2 tables)
SHA-256: `cc80528c9f1d20bbc6b375adcd958e80ab796e4275b1726ef15d0a78404187fe`

File: `annex/textmodel_v1_vectors.txt` (compose-buffer text, displayed text, bit length and
476-bit body for seven cases, including clicked entries)
SHA-256: `994b87e091dc7f3db5b62f42e94f1e102eea3c7d3b0593d789d031f3db81223e`

Dictionary source: `corpus/dictionary_v0_draft.txt`
SHA-256: `092d335f474ec7c489f437581bc8eefb82fce5de21d0987a7d3eedc81e4b8602`

Rebuild: `tools/textcodec.py build annex/textmodel_v1.txt corpus/dictionary_v0_draft.txt
corpus/books/opticks.txt corpus/books/pydoc_topics.txt` (from the project root). Provisional until
the model is retrained (Section 5.4). The v0 model (no dictionary) and its codec are kept as
`annex/textmodel_v0.txt` and `tools/textcodec_v0.py` for reference only.

---

## 7. Why 64 tones (informative)

At fixed frame time and power, energy per bit is fixed, so tone count affects only the
efficiency of noncoherent orthogonal signaling. Computed Eb/N0 required per coded bit at
raw BER 0.10: 1.09 dB for 32 tones, 0.84 dB for 64, 0.66 dB for 128, 0.51 dB for 256. Going
from 64 to 128 tones gains about 0.2 dB while doubling the bandwidth, so 64 was chosen.
For more sensitivity, lengthening the frame is more effective: doubling the time gives
about 3 dB.

---

## 8. ALE link layer (Part 2) **[SIM]**

Part 2 covers addressed messages between two stations on one agreed channel, which is how EmComm
and business nets normally work: link set-up, delivery with acknowledgement and repeats, and
channel access. Channel scanning is deferred (Section 8.10). It follows the functional model of
the HF ALE standards (FED-STD-1045A, MIL-STD-188-141) without claiming compatibility with them.
All transmit behaviour is subject to Section 2.

### 8.1 Timing

Stations MUST be synchronised to UTC within 0.5 s (GPS, NTP or equivalent). Time is divided
into 30 s slots starting on the minute and half-minute.
- A DATA frame (27.648 s) uses a whole slot and nominally starts 0.5 s into it.
- Short frames (12.288 s, Section 8.3) use half slots: sub-slot A starts 0.5 s into the slot,
  sub-slot B 15.5 s into it. A CALL and its RESP, or a POLL and its ACK, fit in one slot.
- Receivers search start times within +/-1 s of nominal; the reference receiver searches
  2.5 s from the start of each (sub-)slot.
- The reference station closes its short-frame receive windows at 14.0 s (sub-slot A) and 29.0 s
  (sub-slot B): a frame up to 1 s late has ended by then, and the remaining 1.5 s is the time to
  decode before the RESP or ACK at 15.5 s, or the next slot's frame at 30.5 s.

### 8.2 Frame types

| Type | Name | Frame | Sent by | Purpose |
|---|---|---|---|---|
| 0 | DATA | full (Section 4) | sender | message part |
| 1 | CALL | short | sender | open a link; carries the number of parts |
| 2 | RESP | short | receiver | accept a CALL; carries the SNR it measured |
| 3 | ACK | short | receiver | which parts it holds; reply to POLL |
| 4 | SOUND | short | any | LQA beacon |
| 5 | TERM | short | sender | link finished |
| 6 | POLL | short | sender | ask for an ACK |
| 7-15 | reserved | | | |

### 8.3 Short frame physical layer

- **Modulation.** 32-ary GFSK (BT = 4, Section 3.2) on tones 16-47 of the DATA tone grid, that is
  f0 + 125 Hz to f0 + 367 Hz, so short frames sit in the middle of the channel. 5 bits per symbol,
  natural binary, symbol period and spacing as in Section 3.1.
- **Frame.** 96 symbols (12.288 s): Costas sync blocks at symbols 0-7, 44-51 and 88-95, using the
  three arrays of Section 4.3 with short-frame tone 4c + 2; 72 data symbols carry 360 coded bits.
- **Code.** LDPC (360, 120), systematic IRA with the same structure as Annex A: info columns of
  degree 6 (30%) and 3 (70%), dual-diagonal parity, no 4-cycles; fixed interleaver. Annex F.
- **Bandwidth.** 247 Hz (99% power), 277 Hz at -26 dB, 328 Hz at -40 dB, so under the 500 Hz
  limit of 97.221(c) for automatic replies (Section 2.6).
- **Sensitivity** (sample-level, sync search, `tools/ale_phy.py sweep`): AWGN frame loss 12% at
  -18 dB and 0% at -17 dB (DATA: 78% at -17 dB); ITU Moderate 30% at -16 dB and 0% at -14 dB (DATA:
  52% at -14 dB). Short frames are 1.5-2 dB more robust than DATA frames, so the link control
  frames are not the weak point. A rate-1/2 version (9.2 s) was rejected: it was only 0.5 dB
  better than DATA in AWGN and had an error floor on fading (3.5% at -12 dB).

### 8.4 Short frame payload (104 bits + CRC-16 = 120 info bits)

| Bits | Field |
|---|---|
| 4 | type (Section 8.2) |
| 2 | version (0) |
| 1 | PROFILE (Section 5.6) |
| 6 | link ID (1-63; chosen at random by the caller in CALL, then used by both stations) |
| 32 | from: sender of this frame (Section 5.3) |
| 32 | to: addressee of this frame |
| 16 | info, by type (below) |
| 11 | reserved, 0 |

Info field: CALL = parts - 1 (3 bits), priority (2 bits: 0 routine, 1 priority, 2 emergency,
3 test), 11 reserved. RESP = SNR (6 bits, signed, dB in 2500 Hz, -32 to +31), accept (1 bit),
9 reserved. ACK = part bitmap (8 bits, bit k set when part k is held), SNR (6 bits), 2 reserved.
POLL, TERM, SOUND = 16 reserved. The CRC is CRC-16/CCITT-FALSE as in Section 5.1.

All fields are sent most significant bit first, in the order of the tables. Within the info
field, numbered from bit 15 (sent first) to bit 0: CALL parts - 1 in bits 15-13 and priority in
12-11; RESP SNR in 15-10 (two's complement) and accept in 9; ACK bitmap in 15-8, where part k has
value 2^k (so part 7 is sent first), and SNR in 7-2. A receiver MUST discard a short frame with
type 0 or 7-15, version other than 0, or an identifier outside the range of Section 5.3.

### 8.5 Procedure

1. **Channel check.** Before a CALL the sender MUST have listened to the channel for the previous
   two slots and seen no frame of this mode that is not its own and no link in progress (a CALL or
   RESP heard without a TERM or 3 idle slots after it).
2. **Link set-up.** The sender's control operator starts the call. The sender sends CALL in
   sub-slot A; the addressee answers RESP in sub-slot B of the same slot. Without a RESP the sender
   repeats the CALL in the next slot, up to 5 slots in all.
3. **Data round.** From the next slot, the sender sends every part not yet acknowledged, in
   ascending order, one per slot. DATA frames carry the link ID, PART and MORE (Section 5.2).
4. **Poll.** In the slot after a round, the sender sends POLL in sub-slot A and the receiver
   answers ACK in sub-slot B with its part bitmap. Without an ACK the sender polls again in the
   next slot, up to 8 slots. The sender starts a new round only after it has received an ACK.
5. **Repeats and combining.** Because a new round only follows an ACK the sender received, the
   receiver knows which parts the round carries: the parts missing from its last ACK, in ascending
   order, one per slot. It MUST add the soft values (LLRs) of every copy of a part, including
   copies that did not decode, and decode the sum. Section 9.4 and 8.8 give the gain.
6. **End.** When an ACK shows every part, the sender sends TERM in sub-slot A of the next slot.
   No reply is sent. A sender gives up after 40 slots (20 minutes).
7. **Duplicates.** A receiver that decodes a payload it already holds on the same link keeps one
   copy. A new message uses a new CALL and link ID.
8. **Bulletins** (ADDR = 0, no addressee) use no ALE. They SHOULD be sent twice in consecutive
   slots so receivers can combine the copies.
9. **Reference responder (informative).** `dispatch-core` (`ale::Responder`) maps the slots after
   its last RESP or ACK to the parts missing from it, one per slot. In a slot with no short frame
   for the link it runs the DATA receiver; a copy that decodes on its own is stored by its PART
   field (and realigns the count if a slot was miscounted); otherwise, if the strongest DATA
   candidate has a sync metric of at least 3 (noise alone gives about 1.6), its soft values are
   added to the sum for the expected part and the sum is decoded. It answers a repeated CALL with
   the same link ID again, and drops a link after 12 slots in which nothing of it was heard.

### 8.6 Automatic operation

The receiving station only ever sends short frames (RESP, ACK), and only in reply to a CALL or POLL
addressed to it. Short frames are under 500 Hz (Section 8.3), so under 97.221(c) an unattended
receiving station may answer on any amateur frequency where data is allowed, not only in the
97.221(b) segments. The sending station transmits DATA frames, which are wider than 500 Hz, so it
MUST be under local or remote control (its operator starts the CALL), unless it is in a 97.221(b)
segment. Automatic replies MUST still ship off by default (Section 2.6). In the business profile
the Morse identification of Section 2.10 also applies.

### 8.7 LQA

RESP and ACK report the SNR the receiving station measured on the frame it answered. Stations
keep a table of (peer, channel, time, SNR) from these reports and from every frame they decode.
SOUND beacons MAY be sent only where Section 2.6 allows and no more than once per hour per channel.

### 8.8 Measured delivery (`tools/ale_sim.py`)

Slot-level simulation of the procedure above with the symbol-level physical layer (sync assumed
found, which the sample-level results support down to these SNRs), 3-part message, 100 sessions
per point, channel gain continuous over the session (Gaussian Doppler spectrum):

| Channel | SNR | Delivered, no combining | Delivered, combining | Mean time (combining) |
|---|---|---|---|---|
| AWGN | -18 dB | 0% | 100% | 4.7 min |
| AWGN | -17 dB | 100% | 100% | 3.1 min |
| AWGN | -16 dB | 100% | 100% | 2.5 min |
| Rayleigh, 1 Hz Doppler | -16 dB | 9% | 100% | 5.1 min |
| Rayleigh, 1 Hz Doppler | -14 dB | 100% | 100% | 3.5 min |
| Rayleigh, 0.1 Hz Doppler | -16 dB | 78% | 92% | 6.9 min |
| Rayleigh, 0.1 Hz Doppler | -14 dB | 99% | 99% | 4.4 min |
| Rayleigh, 0.1 Hz Doppler | -12 dB | 100% | 100% | 3.3 min |

The shortest exchange for a 3-part message is 5 slots, 2.5 minutes: CALL/RESP, three DATA,
POLL/ACK. Retry limits were tuned on slow fading: with 3 CALL and 3 POLL attempts, delivery at
-16 dB with 0.1 Hz Doppler was 54%; with 5 and 8 it is 92-94%.

### 8.9 Sample-level runs of the procedure

The Rust core runs whole exchanges at sample level: both stations use their real receivers
(short-frame search in each sub-slot, DATA search with drift tracking, soft combining) on 30 s
slots of audio with white noise and up to 0.17 s of timing error that changes from slot to slot.
3-part message, AWGN, 24 sessions per point (`ale_sample_level_sweep` in
`dispatch/crates/dispatch-core/tests/ale.rs`; slots counted up to and including TERM, so the
shortest exchange is 6 slots):

| SNR | Delivered | Mean slots | Mean time |
|---|---|---|---|
| -16 dB | 24/24 | 6.0 | 3.0 min |
| -17 dB | 24/24 | 9.5 | 4.8 min |
| -18 dB | 23/24 | 14.2 | 7.1 min |

The one failure at -18 dB was a CALL not answered in 5 attempts. These agree with Section 8.8
(sync assumed there): at -18 dB no single DATA frame decodes reliably and delivery rests on
combining.

The same runs on fading channels (`dispatch_core::channel`, Watterson model as in Section 9.1;
each direction fades independently and continuously over the session, with its own frequency
offset of up to +/-2 Hz), 20 sessions per point (`DISPATCH_ALE_CHAN=...`):

| Channel | SNR | Delivered | Mean slots | Mean time |
|---|---|---|---|---|
| ITU Moderate (1 ms, 0.5 Hz) | -12 dB | 20/20 | 6.1 | 3.0 min |
| ITU Moderate | -14 dB | 20/20 | 8.4 | 4.2 min |
| ITU Moderate | -16 dB | 20/20 | 13.4 | 6.7 min |
| Rayleigh, 1 Hz | -14 dB | 20/20 | 8.5 | 4.2 min |
| Rayleigh, 1 Hz | -16 dB | 19/20 | 13.5 | 6.8 min |
| Rayleigh, 0.1 Hz | -12 dB | 20/20 | 7.5 | 3.7 min |
| Rayleigh, 0.1 Hz | -14 dB | 20/20 | 10.2 | 5.1 min |
| Rayleigh, 0.1 Hz | -16 dB | 16/20 | 17.5 | 8.8 min |

The failures were CALLs not answered in 5 attempts (four) and one POLL not answered in 8. With 20
sessions per point these agree with Section 8.8 (Rayleigh 0.1 Hz at -16 dB: 92% there, 80% +/- 9%
here). Not yet run at sample level: collisions with a third station and drift between slots.

### 8.10 Scanning (future)

A later revision may add time-synchronous scanning: a published schedule maps each slot to a
channel, stations listen on the scheduled channel, and a caller calls on the channel the callee
is listening to, chosen from LQA. Asynchronous scanning is not practical with 12-28 s frames.

### 8.11 Rate adaptation (future)

A later revision may add faster or slower PHY tiers chosen from LQA SNR.

### 8.12 Calling frequencies (informative)

Stations SHOULD call on these dial frequencies (upper sideband, signal 1000-1500 Hz above the
dial), which follow FT8 (.074) and JS8 (.078) and keep clear of their passbands:

| Band | Dial | Signal |
|---|---|---|
| 40 m | 7.084 MHz | 7.0850-7.0855 MHz |
| 20 m | 14.084 MHz | 14.0850-14.0855 MHz |

Both are in the US data segments. They are outside the 97.221(b) automatic-control segments,
which is allowed because automatic replies are short frames under 500 Hz (Section 2.6). Busy
RTTY contest weekends can occupy 14.080-14.090; the channel check (Section 8.5) then waits.
Other regions and bands are for a later revision.

### 8.13 Store and forward (application convention)

A message sent over an ALE link whose text begins `TO:` followed by an identifier and a space
(`TO:KQ4TGD MEET AT 1900`) is a message for that station, left with the receiving station. The
receiving station stores it with the original sender's identifier and does not show it as its
own. A station that holds messages MAY deliver them when it next decodes any frame from the
addressee (Section 8.5, as a new message with the text followed by ` DE ` and the original
sender's identifier). Delivery starts a CALL, so it MUST be done under local or remote control
unless the station is in a 97.221(b) segment (Section 8.6); it is off by default. A delivered
message is marked delivered and not sent again; an attempt that fails is not repeated for 30
minutes. The convention matches JS8Call's `MSG TO:` so that a station can bridge the two.

### 8.14 Automatic operation in the reference application (informative)

All of the following are off by default and shown while active (Section 2.6):
- Dispatch: answering CALLs addressed to the station (RESP and ACK only, Section 8.6);
  forwarding held messages (8.13); SOUND beacons, at most once an hour (8.7), held off for 10
  minutes after anything addressed to the station is heard. A SOUND frame has link ID 0 and is
  addressed to its sender.
- JS8 (for interworking): JS8Call's automatic replies to directed queries (SNR?, GRID?, INFO?,
  STATUS?, HEARING?, AGN?, QUERY MSGS, QUERY MSG n), storing MSG and MSG TO: with an ACK, and
  heartbeat acknowledgements; none to @ALLCALL queries, and the same station and command at most
  once in two minutes. Buffered commands carry JS8Call's CRC-16/KERMIT checksum. These are replies
  to interrogation in a 50 Hz signal, so they may run unattended under 97.221(c).
- FT8 auto-sequencing and the JS8, FT8 and RTTY beacons start transmissions: they assume a control
  operator is present.

---

## 9. Performance (reference simulators; informative) **[SIM]**

SNR is quoted in a 2500 Hz reference bandwidth (Es/N0 = SNR2500 + 25.05 dB for T = 128 ms).
Frame loss counts a frame as lost if the CRC fails or the payload is wrong.

### 9.1 Symbol-level simulator (ideal sync, `mfsk64_sim.py`)

| Channel | Level | Conv (K=7) | LDPC (this spec) |
|---|---|---|---|
| AWGN | 50% frame loss | -16.9 dB | -17.3 dB |
| AWGN | 0% frame loss (200 trials) | -15.5 dB | -16.25 dB |
| Rayleigh flat fading, 1 Hz Doppler spread, no CSI | 50% | -14.0 dB | -14.5 dB |
| same | 1% | -11.0 dB | -12.0 dB |

Uncoded symbol error rates match the theoretical noncoherent M-FSK expression to within sampling
error at Es/N0 = 6, 8, 10 and 12 dB.

### 9.2 Sample-level reference modem (real sync search, `mfsk64_modem.py`)

12 kHz audio, GFSK BT = 4, noise blanker and drift tracking on, random start time within 0-1 s of
the slot start, random audio-frequency offset within +/-3 Hz, two-path Watterson channels with
Gaussian Doppler spectrum (Doppler spread = 2 sigma). 40 trials per point (60 for AWGN).

| Channel | 50% frame loss | First SNR with 0 failures |
|---|---|---|
| AWGN | -16.8 dB | -16.0 dB |
| Flat Rayleigh, 1 Hz | -14.0 dB | about -12 dB (2.5% at -12 dB) |
| ITU Good (0.5 ms, 0.1 Hz) | -14.0 dB | -8 dB (5% still lost at -10 dB) |
| ITU Moderate (1 ms, 0.5 Hz) | -14.2 dB | about -12 dB (5-7% at -13 to -12 dB) |
| ITU Poor (2 ms, 1 Hz) | -13.9 dB | about -12 dB (2.5% at -12 dB) |

These are about 0.3 dB worse than the unshaped v0.12 modem, which matches the measured cost of
GFSK shaping. Trial counts are small: treat individual numbers as +/-0.5 dB, and the
zero-failure points as upper-bound estimates of the true 1% point.

### 9.3 Impairments (`tools/impair_test.py`, `tools/drift_test.py`)

At -16 dB AWGN, 40 trials per case:

| Impairment | Frame loss |
|---|---|
| None | 0% |
| Sample clock +100 ppm / +300 ppm | 0% / 0% |
| Sample clock +1000 ppm / -1000 ppm | 15% / 20% |
| Static crashes 5/s, +30 dB, no blanker / blanker | 100% / 0% |
| Static crashes 20/s, +30 dB, no blanker / blanker | 100% / 20% |
| Static crashes 20/s, +40 dB, no blanker / blanker | 100% / 20% |
| Blanker on, no impulses | 0% |

Drift: see Section 4.3. Clock error above about 300 ppm needs timing-rate tracking, which is
**[TBD]** (the same per-block method as for frequency would work). Not yet simulated: AGC,
receiver IF filtering, and interference from other signals.

### 9.4 Repeat with soft combining (`tools/combine_test.py`)

The same frame is sent in two consecutive 30 s slots. The receiver keeps the LLRs of a frame
that fails (best candidate with sync metric >= 3) and, when the repeat arrives, decodes the sum
of the two LLR sets. Frame loss (60-100 trials per point):

| Channel, SNR | One frame | Two tries, either decodes | Two tries, soft combined |
|---|---|---|---|
| AWGN -18 dB | 100% | 100% | 25% |
| AWGN -17 dB | 80% | 57% | 0% |
| ITU Good -17 dB | 92% | 84% | 50% |
| ITU Good -15 dB | 61% | 44% | 16% |
| ITU Good -13 dB | 33% | 13% | 0% |
| ITU Good -11 dB | 10% | 1% | 0% |
| ITU Moderate -15 dB | 87% | 65% | 0% |
| ITU Moderate -13 dB | 20% | 2% | 0% |

Soft combining gains about 2-3 dB over one frame on AWGN and closes most of the slow-fading gap:
on ITU Good the 0%-loss point moves from about -8 dB to about -13 dB. Recommendation for Part 2:
a sender that gets no ACK repeats the identical frame in its next transmit slot, and the
receiver MUST combine. A repeat costs one slot, so it should be triggered by ARQ, not sent
blindly (except for bulletins, where a fixed second copy is reasonable).

### 9.5 Interference (`tools/interf_test.py`)

Wanted signal at -14 dB SNR (about 3 dB above the AWGN threshold), 30 trials per case, full
receiver with sync search. SIR is wanted-signal power divided by interferer power. Interferers:
steady carriers anywhere in the passband; FT8-like signals (8-GFSK, BT 2, 6.25 Hz spacing,
12.64 s) placed inside the passband ("x4": two signals in each of two 15 s periods, so the band is
occupied for the whole frame); a second frame of this mode, either offset by 250 Hz (half
overlapping) or on the same frequency with different timing.

| Interferer | SIR | v0.13 receiver | v0.15 receiver |
|---|---|---|---|
| None | | 0% | 0% |
| Carrier | 0 dB | 93% | 0% |
| Carrier | -10 dB | 100% | 0% |
| Carrier | -20 dB | 100% | 0% |
| Three carriers | -10 dB | 100% | 0% |
| One FT8 signal | 0 dB | 3% | 0% |
| One FT8 signal | -10 dB | 100% | 0% |
| One FT8 signal | -20 dB | 100% | 3% |
| Busy FT8 (whole frame) | -10 dB | 100% | 27% |
| Same mode, +250 Hz | 0 dB | 20% | 20% |
| Same mode, +250 Hz | -6 dB | 100% | 100% |
| Same mode, same frequency | +6 dB | 0% | 0% |
| Same mode, same frequency | 0 dB | 47% | 43% |

The front end and decoder of Sections 3.5-3.7 and 4.3 cost no measurable sensitivity
without interference (150 trials at -16.5 dB, all variants within one or two frames).
**Application feature (informative, not part of this standard).** An application MAY
also include FT8, JS8 and RTTY decoders; the reference application now does (Section 1.7), and
its FT8 decoder already subtracts the signals it decodes before searching again. Besides showing that traffic to the operator, a
decoded signal can be rebuilt and subtracted from the audio before this mode is decoded
(successive interference cancellation, as WSJT-X does for FT8). This would mainly help the
hard cases above: an interferer 20 dB stronger, or a band busy for the whole frame. Decoder
licences differ (ft8_lib: MIT; WSJT-X and JS8Call: GPL). Operating away from the FT8 and JS8
sub-bands remains the simplest protection.

Collisions with another frame of this mode are not helped, because that signal looks exactly like
the wanted one. Avoiding them is a channel-access problem for Part 2 (listen before transmit,
slot assignment).

### 9.6 Rig filters and AGC (`tools/rxchain_test.py`)

The received audio passes through a model of the rig before the reference receiver: an IF
filter (8th-order elliptic, 1 dB ripple, 60 dB stop band, causal, so with its real group delay),
optional passband tilt, and a peak AGC (2 ms attack; fast = 100 ms decay; slow = 1 s decay with
200 ms hang). Wanted signal at -15 dB SNR, audio 1000-1500 Hz, 30 trials per case.

| Rig setting | Frame loss |
|---|---|
| No rig model | 0% |
| SSB filter 300-2700 Hz | 0% |
| SSB filter with 10 dB passband tilt | 0% |
| Data filter 1000-1500 Hz (exact fit) | 0% |
| Data filter 500 Hz, 50 Hz off centre | 0% |
| Data filter 500 Hz, 100 Hz off centre | 47% |
| Data filter 500 Hz, 250 Hz off centre | 100% |
| Data filter 600 Hz (950-1550, or 50 Hz off centre) | 0% |
| SSB filter, fast or slow AGC | 0% |
| Fast or slow AGC, keyed CW 30-40 dB above the wanted signal at 2200 Hz | 0% |
| Fast or slow AGC, static crashes 5/s at +30 dB | 0% |
| Slow AGC, static crashes 5/s at +60 dB | 0% |
| Fast AGC, static crashes 5/s at +60 dB | 43% |

Without the local blanker level and gain normalisation (Sections 3.5, 3.7), the last two rows
were 15% and 100%.

**Operating guidance (informative).**
- Use the SSB filter (2.4 kHz or wider) or a data filter of 600 Hz or more, centred on the
  signal. A 500 Hz filter leaves only about 50 Hz of tuning margin. The application SHOULD show
  the audio range in use (f0 to f0 + 500 Hz) so the operator can match the filter, or move f0 to
  the centre of the rig's data filter.
- Use slow AGC or AGC off. The rig's own noise blanker may be on.
- Strong signals elsewhere in the passband do not matter as long as the rig and sound card are
  not overloaded (clipping was not modelled).

### 9.7 Reference sound card: Digirig (`tools/digirig_test.py`, `tools/txdrive_test.py`)

The reference station interface is a Digirig Mobile, which uses a C-Media CM108B USB codec.
Datasheet values used in the model: 16-bit ADC, 48 or 44.1 kHz only, 88.5 dB dynamic range,
internal oscillator (no crystal), and no stated clock accuracy.

**Implementation requirements (normative for applications using CM108-class interfaces).**
- Capture and play at 48 kHz and convert to and from the 12 kHz reference rate by an exact
  factor of 4, with a proper anti-alias low-pass filter. 44.1 kHz MUST NOT be used unless the
  application resamples it exactly.
- The application SHOULD show the input level (noise RMS and peak, in dBFS) and warn when samples
  reach full scale.
- On macOS, audio capture needs microphone permission, which is granted per application. A program
  started over SSH or as a background service receives silence with no error. The application
  SHOULD run as a normal desktop application, request permission on first use, and warn if the
  input is exactly zero.

**Receive level (model).** Rig SSB filter and slow AGC, then the Digirig input scaled so the audio
RMS is the given level, ADC noise, clipping at full scale, 16-bit quantisation at 48 kHz, then
decimation to 12 kHz. Signal at -16.5 dB (near threshold), 60-100 trials per case, so differences
under about 10 points are noise. Reference loss without the Digirig at this SNR: 24-32%.

| Input level (noise RMS) | Clipped samples | Frame loss |
|---|---|---|
| -30 dBFS (recommended) | 0% | 24-40% (same as without the Digirig) |
| -80 dBFS (near the ADC floor) | 0% | 43% |
| -90 dBFS (below the ADC floor) | 0% | 82% |
| -6 dBFS | 4.6% | 37% |
| 0 dBFS | 32% | 65% |
| 0 dBFS, keyed CW 30 dB above the wanted signal | 34% | 100% |
| -12 dBFS, keyed CW 40 dB above the wanted signal | 0% | 43% |
| -6 dBFS, keyed CW +40 dB | 1.8% | 47% |
| 0 dBFS, keyed CW +40 dB | 35% | 100% |

The mode is tolerant of input level: anything from about -80 to -6 dBFS works. Clipping on noise
alone costs a few tenths of a dB (hard limiting a weak signal in noise loses about 1 dB at most).
Clipping caused by a strong signal in the passband is the real hazard: the strong signal takes
over the clipped waveform and suppresses the weak one. Recommended setting: noise RMS about
-30 dBFS, with the strongest signals in the passband still below full scale. Use the Digirig's
20 dB receive attenuator if the OS input control cannot bring the level down.

**Sample clock.** The CM108B runs from an internal oscillator, and its clock accuracy is not
specified. One Digirig Mobile measured -4.3 ppm (standard error 2.4 ppm) against a Mac mini's clock
over 300 s (`tools/digirig_measure.py`); a second 60 s run gave -0.6 ppm (standard error 29 ppm). The receiver tolerates +/-300 ppm with no loss (Section 9.3). Applications SHOULD
measure the actual capture rate against the system clock over a few minutes, warn if it is more
than 200 ppm off, and MAY resample to correct it.

**Measured receive floor (one Digirig Mobile, radio connected but off, macOS input volume 70).**
- Noise in the 1000-1500 Hz signal band: -92.5 dBFS, so a full-scale sine is 89.5 dB above it
  (datasheet: 88.5 dB dynamic range).
- Hum: 60 Hz at -66 dBFS with harmonics (120, 180, 240, 300 ... 1020 Hz), a typical ground loop.
- USB frame noise: lines at 1000, 2000 and 3000 Hz (USB full speed sends a frame every 1 ms). The
  1000 Hz line is on tone 0 when f0 = 1000 Hz, 32 dB above the floor.
- 28 lines in 20 Hz - 6 kHz were more than 10 dB above the local noise. In 30 s of audio the
  carrier remover notches 88 long-FFT bins in 950-1550 Hz, about 3.5 Hz in all.
- Mixing this recording into simulated frames at -16 dB SNR (`tools/digirig_floor_test.py`, 60
  trials per level) caused no loss at receive levels of -30, -50, -60 or -70 dBFS, even at
  -70 dBFS, where the hum is louder than the received noise.
- Measuring on macOS: the measurement must run in a terminal that has microphone permission.
  Over SSH, start it with `open -a Terminal measure.command`; output goes to
  `measurements/last_run.log`.

**Transmit drive.** The Digirig output is known to be hot and easily overdrives the radio. The
waveform has a constant envelope, so a correctly driven transmitter reproduces it cleanly, but
clipping in the Digirig output or the radio's audio stages creates spurious signals inside the
SSB passband, next to the wanted 500 Hz:

| Transmit audio (f0 = 1000 Hz) | Spurious power outside the signal, 200-3000 Hz | Worst spur vs signal |
|---|---|---|
| Correct drive | -55 dB | -46 dB |
| 3 dB over, symmetric clipping | -37 dB | -35 dB |
| 10 dB over, symmetric clipping | -29 dB | -26 dB |
| 3 dB over, asymmetric clipping | -25 dB | -25 dB |
| 10 dB over, asymmetric clipping | -21 dB | -21 dB |
| Soft compression, 6 dB over | -43 dB | -34 dB |

With f0 = 1500 Hz the results are similar, so moving f0 does not help. Operators MUST set
transmit drive so the radio's ALC shows no action (or minimal action), set the transmit audio
level in the operating system rather than relying on the radio, and SHOULD check the transmitted
signal on a second receiver or a web SDR. The application SHOULD generate audio no higher than
-6 dBFS, so the digital path itself never clips.

---

## 10. Conformance

10.1 Classes: **Modulator**, **Demodulator**. (An **ALE node** class will be added in Part 2.)

10.2 A conforming Modulator MUST, given the 560-bit payloads in Annex D, produce info bits,
codewords and tone sequences identical to the vectors, and audio whose waveform follows Section
3.2 within the tolerance of Section 3.4. The WAV hashes are informative (floating-point rounding
may differ between implementations).

10.3 A conforming Demodulator MUST decode the noise-free reference audio files in Annex D and
the noisy slot file at -10 dB, and SHOULD decode the noisy slot file at -14 dB. It MUST also pass
the statistical test of Section 10.4.

10.4 **Statistical test (Annex E).** `tools/conformance.py generate <dir>` writes 1,300 test slots
(30 s, 12 kHz, 16-bit mono WAV, noise at -20 dBFS RMS, about 0.9 GB) from fixed seeds, with a
manifest of the payload in each. Each slot holds at most one frame, starting 0-1.5 s into the slot,
with an audio offset within +/-4.5 Hz. The Demodulator under test processes every file as a
separate slot, with no knowledge of the manifest, and writes one line per decoded frame:
`<file name> <560-bit payload as 140 hex digits>`. `tools/conformance.py check <dir> <results>`
then scores it. A frame counts only if its payload matches; any other decoded payload is a false
frame.

| Case | Condition | SNR (2500 Hz) | Slots | Max lost | Reference receiver lost |
|---|---|---|---|---|---|
| A1 | AWGN | -15.5 dB | 200 | 16 | 0 |
| G1 | ITU Good (0.5 ms, 0.1 Hz) | -10 dB | 200 | 25 | 7 |
| M1 | ITU Moderate (1 ms, 0.5 Hz) | -12 dB | 100 | 20 | 5 |
| P1 | ITU Poor (2 ms, 1 Hz) | -12 dB | 100 | 15 | 1 |
| D1 | AWGN, linear drift 0.3 Hz/s | -15 dB | 100 | 10 | 0 |
| I1 | AWGN, steady carrier 10 dB above the signal | -14 dB | 100 | 10 | 0 |
| I2 | AWGN, FT8-type signal 10 dB above the signal | -14 dB | 100 | 10 | 0 |
| I3 | AWGN, static crashes 5/s at +30 dB | -15 dB | 100 | 10 | 0 |
| N1 | Noise only | - | 300 | - | - |

At most 1 false frame is allowed over the whole set (the reference receiver produced none).

The Rust receiver (`dispatch results`, Section 1.7) lost exactly the same numbers of frames in
every case (0, 7, 5, 1, 0, 0, 0, 0) with no false frames, on the same 1,300 slots.

How the limits were set (binomial, using the reference receiver's measured loss):
- The reference receiver passes every case with probability above 0.9999.
- A receiver 1 dB less sensitive in AWGN (A1 run 1 dB lower: 30 of 200 lost) passes A1 with
  probability about 0.002. A receiver 2 dB worse on fading channels (G1, M1, P1 run 2 dB lower:
  20, 53 and 51 lost) passes with probability under 0.004 on each.
- D1, I1, I2 and I3 test that the receiver has the defences of Sections 3.5-3.7 and 4.3. Without
  drift tracking, carrier removal, interference erasure or a noise blanker, measured loss in
  those conditions is 33-100%, so the case fails.

The run takes about 6 minutes with the reference receiver on a laptop. `run_all.py` runs a
4-slot-per-case version in quick mode (limits scaled down) and the full set with `--full`.

### Annex D (normative): test vectors

Directory `annex/annexD/`: `vectors.txt` and eight WAV files (12 kHz, 16-bit mono). Generator:
`tools/frame.py vectors annex/annexD` (needs `annex/textmodel_v1.txt`). Contents:
- CRC-16/CCITT-FALSE check: ASCII "123456789" gives 0x29B1.
- Six frames: all-zero payload, all-one payload, a random payload (NumPy seed 1), an addressed
  text frame (KQ4TGD to W4ABC), a bulletin text frame (KQ4TGD, ADDR = 0) and a clicked-entry frame
  (KQ4TGD to W4ABC), each with payload,
  info (payload + CRC), codeword, the 216 tone indices and a clean WAV starting at sample 0.
- Two 30 s noisy slot files with the random payload at -14 dB and -10 dB, starting at 0.5 s with
  an audio offset of +1.7 Hz. The reference receiver decodes both, from the 16-bit files.
- The text frames use the v1 text model (Annex C). Case `text_clicked` is built from palette
  entries plus typed text; `text` is the compose-buffer form and `shown` is what the receiver displays.

10.4 The reference implementations are `mfsk64_sim.py` (symbol level) and `mfsk64_modem.py`
(sample level, 12 kHz audio). Annex D was generated from it; the vectors
must be regenerated and frozen when the text model and dictionary are final.


### Annex F (normative): short ALE frame code

File: `annex/ldpc_360_120_info_columns.txt` (info part of H; parity part is the accumulator)
SHA-256: `ed34b2b55fb4c5d042b31600f46a2a1e9747cbf4b6fc884d76bcdbb9b2df7d48`

File: `annex/interleaver_360.txt` (transmitted bit j is coded bit perm[j])
SHA-256: `c0343efcfc7894f28f524121491593a88a587da221b9c887fb29940fad897a7e`

Generator and check: `tools/gen_annex_short.py [annex] [--check]`.

Test vectors: `annex/short_vectors.txt` (payloads all zeros, all ones and a random one, each with
info bits, codeword before interleaving, and the 96 frame tones) and `annex/short_random_seed5.wav`
(the random case as 16-bit audio at 12 kHz, peak 16384, frame at sample 0). Generator:
`tools/gen_short_vectors.py [annex]`. The Rust core reproduces the tones exactly and the audio
within 1 LSB (`dispatch/crates/dispatch-core/tests/ale.rs`).
---

## 11. Security considerations

- No confidentiality: transmissions are plain text by regulation (Section 2.3).
- No authentication: a CRC detects errors, not forgery. Anyone can transmit a CALL or
  TERM with another station's callsign. Operators SHOULD treat link-layer addressing as
  convenience, not identity.
- Denial of service: an adversary can send false TERMs or occupy slots. Nodes SHOULD
  ignore a TERM that does not match an active link ID and sequence.
- Automatic operation: see Section 2.5. Implementations that transmit unattended
  without regard to Section 97.221 put the operator in violation.

---

## 12. Open issues

1. Choose the final 220 dictionary entries and their 4-digit codes, and retrain the text model on
   real amateur-radio and EmComm traffic (v1 uses mostly book text); rebuild Annex C, C and D
   vectors. (Section 5.4)
2. Timing-rate tracking for sample-clock error above 300 ppm, and a drift hypothesis in the
   coarse search for drift above 0.5 Hz/s. (Sections 3.4, 4.3)
3. Measure the transmitted spectrum on real Digirig hardware, and the receive floor with the
   radio on. (Section 9.7)
4. Validation on recorded real HF audio, and on-air tests.
6. ALE: sample-level runs of the procedure on fading channels, collisions with a third station,
   and scanning (Sections 8.9, 8.10).
7. Business profile: target frequencies and pools, emission designator, coordinator approval,
   Morse identification timing, FM simulation. (Section 2.B)

Closed in v0.19: identification rules for the in-frame identifier (2.5, 2.10).
Closed in v0.21: ALE short frame, link procedure with POLL/ACK and soft combining (Section 8).
Closed in v0.20: statistical conformance test (10.4).
Closed in v0.16: sound-card overload and transmit drive, Digirig reference interface (9.7).
Closed in v0.15: rig IF filters and AGC (3.5, 3.7, 9.6).
Closed in v0.14: interference from carriers and FT8-type signals (3.5, 3.6, 4.3, 9.5).
Closed in v0.13: pulse shaping (3.2), drift tracking and false-alarm rate (4.3), slow-fading
outage (9.4), min-sum (6.5), test vectors (Annex D), clock offset and impulsive noise (3.5, 9.3).

---

## 13. References

- ARRL, "MFSK16 specification" (16 tones, 15.625 baud, K=7 rate-1/2 FEC).
- FCC 23-93, Report and Order, WT Docket 16-239 (HF bandwidth limit replaces baud-rate limit).
- 47 CFR Part 97, in particular 97.113, 97.119, 97.221, 97.307, 97.309.
- FED-STD-1045A, Automatic Link Establishment; MIL-STD-188-141.
- J. Taylor, K1JT, "The FT4 and FT8 Communication Protocols", QEX (for weak-signal mode design practice).
- S. Sklar, J. Proakis: noncoherent M-ary orthogonal signaling and detection theory.

## Revision history

| Version | Date | Notes |
|---|---|---|
| 0.1 | 2026-11-23 | First working draft: PHY, LDPC, simulator results, proposed ALE layer |
| 0.2 | 2026-11-23 | Sample-level reference modem, Costas sync design and search, ITU channel results, measured bandwidth; ALE deferred to Part 2 |
| 0.3 | 2026-11-23 | Text compression defined: order-2 static model with arithmetic coder, Annex C, test vectors |
| 0.4 | 2026-11-23 | Proposed word dictionary (220 entries, public 4-digit codes) and prototype codec |
| 0.5 | 2026-11-23 | Click-palette model: hidden marker, WORD symbol, encoder policy, draft 220-entry contents, measurements |
| 0.6 | 2026-11-23 | Callsign packing (30 bits); header sender call in every frame, optional footer destination call; body 484/514 bits |
| 0.7 | 2026-11-23 | Intended use (EmComm, business band; addressed contacts) recorded; tactical-name and non-amateur identifier notes; open issue on non-amateur regulatory profile |
| 0.8 | 2026-11-23 | PROFILE bit (amateur default / business), Section 5.6; link ID reduced to 6 bits |
| 0.9 | 2026-11-23 | Business-profile identifier: 1-6 alphanumerics plus modifier, 34 bits; business body 510/476 bits |
| 0.10 | 2026-11-23 | One 34-bit identifier (1-6 alphanumerics + modifier) for both profiles; body 476/510 bits |
| 0.11 | 2026-11-23 | Chained messages (MORE and PART fields, Section 5.7); link ID stays 6 bits, body 472/506 |
| 0.12 | 2026-11-23 | Identifier modifier (/P, /R, /M) removed; identifier is 32 bits; body 476/508 bits |
| 0.13 | 2026-11-24 | GFSK BT 4 shaping; noise blanker; drift tracking; sync threshold and false-alarm rate; repeat with soft combining; min-sum results; Annex D test vectors; performance tables re-run |
| 0.13.1 | 2026-11-24 | Full test suite (`run_all.py --full`) reproduced on macOS; sync-metric noise level and clock-offset figures corrected |
| 0.14 | 2026-11-24 | Carrier removal, per-bin normalised and soft-limited sync metrics, two-pass decoding with interference erasure; interference results (9.5) |
| 0.14.1 | 2026-11-24 | Note: optional FT8/JS8/RTTY decoders and interference subtraction as a future application feature |
| 0.15 | 2026-11-24 | Rig filter and AGC results (9.6); local-level noise blanker; per-symbol gain normalisation; three-pass decoding; operating guidance |
| 0.16 | 2026-11-24 | Digirig (CM108B) reference interface: 48 kHz capture, receive level and clipping, clock, transmit drive (9.7) |
| 0.16.1 | 2026-11-24 | Full suite (21 tests) reproduced on macOS; Digirig table: clipping with CW +30 dB |
| 0.16.2 | 2026-11-24 | Measured Digirig clock (-4.3 ppm); macOS microphone-permission note |
| 0.17 | 2026-11-24 | Measured Digirig receive floor, hum and USB lines; recorded floor causes no loss (9.7) |

| 0.18 | 2026-11-24 | Text codec v1: dictionary built in, click markers U+E000-E002, 476/508-bit bodies; frozen model and vectors (Annex C); Annex D regenerated with a clicked-entry frame |

| 0.19 | 2026-11-24 | Regulatory research: Section 2 split into amateur (Part 97) and business (Part 90) profiles; 500 Hz automatic-control finding; Part 90 Morse ID requirement; 7-character identifier forms |
| 0.20 | 2026-11-24 | Statistical demodulator conformance test (10.4, tools/conformance.py); blind reference receiver `receive()` |
| 0.21 | 2026-11-24 | ALE Part 2: short frame (32-GFSK, LDPC (360,120), 277 Hz), payload, procedure with POLL/ACK and soft combining, automatic-operation rules, delivery simulation; Annex F |
| 0.22 | 2026-11-24 | Named Dispatch (tag DS64); licences: spec and mode MIT, application GPL-3.0-or-later, no patents asserted; Rust application structure (1.6, 1.7) |
| 0.23 | 2026-11-24 | Rust `dispatch-core` and `dispatch-cli` (MIT): bit-exact with Annexes C and D, passes the conformance test identically to the reference |
| 0.24 | 2026-11-24 | Rust port of the short ALE frame and the ALE procedure (sender and responder state machines, soft combining); bit layout of the info field (8.4); reference responder behaviour (8.5); sample-level delivery results (8.9); Annex F test vectors |
| 0.25 | 2026-11-24 | Station: `dispatch-app` (ALE station on UTC slots, multi-station simulator, CLI) and `dispatch-radio` (`dispatch-station` with Digirig Mobile, Digirig Lite and VOX profiles); short-frame receive windows (8.1); sample-level delivery on fading channels (8.9) |
| 0.26 | 2026-11-24 | `dispatch-modes`: RTTY, FT8 and JS8 encode/decode; live monitoring in `dispatch-station` (1.7) |
| 0.27 | 2026-11-24 | Calling frequencies 7.084 and 14.084 MHz (8.12); bulletins and recording scan in the station; band presets and FT-857/897 CAT |
| 0.28 | 2026-11-25 | Store and forward (8.13); automatic operation in the application and SOUND frame addressing (8.14); `dispatch-gui` and `dispatch-tui` (1.7) |
