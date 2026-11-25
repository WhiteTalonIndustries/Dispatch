# Dispatch — test build

Thanks for testing. Dispatch is a new HF text mode (64-tone MFSK with an automatic link layer)
by KQ4TGD. The station program also works FT8, JS8 and RTTY, so you can check your setup on
busy modes before trying Dispatch.

This is a **test build**: expect rough edges, and please report what you find (end of this
page). Transmitting needs an amateur licence; you are the control operator.

## What's in the package

| | |
|---|---|
| `Dispatch.app` (macOS) / `bin/dispatch-gui` (Linux) | the window: waterfall, FT8, JS8, RTTY and Dispatch |
| `dispatch-tui` | the same station in a terminal (macOS, Linux) |
| `dispatch-station` | command-line tools: `devices`, `levels`, `radios`, `tune`, `tone`, `record`, ... |
| `README.md`, licences, `SPEC.md`, source | |

The window and the terminal version share settings (`~/.dispatch/gui.conf`), the log of
everything heard and sent (`~/.dispatch/ALL.TXT`) and stored messages (`~/.dispatch/inbox.txt`).

## Install

**macOS (11 or later).** Unzip, move `Dispatch.app` to Applications. The app is not notarized
by Apple, so the first time:
- macOS 15 and later: open it, dismiss the warning, then System Settings → Privacy & Security →
  "Dispatch was blocked" → **Open Anyway**.
- macOS 14 and earlier: right-click (Control-click) the app → **Open** → Open.
- Or in Terminal: `xattr -dr com.apple.quarantine /Applications/Dispatch.app`

Allow **microphone** access when asked: that is how the program hears the radio. For the
terminal tools, run them from Terminal (or iTerm/Warp) and allow that app the microphone in
System Settings → Privacy & Security → Microphone. Copy `bin/*` somewhere on your PATH if you
like, e.g. `sudo cp bin/* /usr/local/bin/` (same `xattr` command for them if macOS blocks them).

**Linux (x86_64; glibc 2.35 or later: Ubuntu 22.04, Debian 12, Fedora 36 and newer).**
Unpack and run `./install.sh`. It puts the programs in `~/.local/bin` and adds a menu entry, and
tells you about two permissions:
- serial ports (CAT, Digirig Mobile PTT): `sudo usermod -aG dialout $USER`, then log out and in;
- Digirig Lite PTT: `sudo cp 99-dispatch-digirig.rules /etc/udev/rules.d/ && sudo udevadm control --reload`.

You need ALSA (`libasound2`), which every desktop install has. Raspberry Pi and other
processors: build it yourself from the source (`packaging/build-linux.sh`, notes at its top).

## First run

1. Start Dispatch. In the **⚙ Radio** panel enter your **callsign** and **grid**.
2. **Radio model**: choose your radio. That sets how the transmitter is keyed (Digirig PTT line,
   CAT command or VOX) and the CAT protocol and rate, and **"Setting up the …"** shows which
   Digirig cable and port to use and the radio settings to make (data mode, CAT rate, data
   input). Radios with their own USB sound card (IC-7300, IC-705, FT-991A, FT-710, TS-590, ...)
   don't need a Digirig: set **Audio in/out** to the radio's "USB Audio CODEC".
   `dispatch-station radios` lists the radios; `dispatch-station radios ft897` shows the setup.
3. With CAT, press **Test CAT** (it reads your frequency). **CAT tune** then sets the radio to the
   dial frequency shown at the top for the mode and band.
4. Press **▶ Start**. Band noise should show about **-30 dBFS** at RX (top right); set it with
   the sound card's input level (macOS: System Settings → Sound → Input; Linux: alsamixer).
5. Transmit level: the **TX level** slider (default -10 dBFS). Adjust it, or the radio's data
   input level, for full power with **no ALC**. `dispatch-station tone --rig ...` sends a steady
   tone for this.

Radio in **data mode**: USB-D / DATA-U / DIG (USER-U on the FT-817/857/897). Many radios ignore
the data jack audio in plain USB.

## Frequencies (dial, USB/data)

| | 40 m | 20 m |
|---|---|---|
| FT8 | 7.074 | 14.074 |
| JS8 | 7.078 | 14.078 |
| RTTY | around 7.080 | around 14.080 |
| **Dispatch** | **7.084** | **14.084** |

Dispatch's signal is 1000-1500 Hz above the dial (7.0850-7.0855 MHz).

## What to test

1. **Receive FT8 and JS8** for a few minutes: decodes per slot compared with WSJT-X / JS8Call,
   and the DT column (should be within about ±0.5 s; if not, check your computer's clock).
2. **Transmit JS8**: send `SNR?` to a station you hear well (click its line, Cmd → SNR?), and see
   whether it answers. PSKReporter shows who heard you.
3. **JS8 messages**: `CALL MSG text` to a JS8Call station — does it land in their inbox?
4. **FT8 QSO**: double-click a CQ; with "FT8 auto-sequence" on, it runs the QSO.
5. **Dispatch between two testers** (schedule a time; 7.084 or 14.084): one sends
   `THEIRCALL your message` (an addressed message with automatic link set-up), the other has
   **Answer Dispatch calls** on. Also try a bulletin (text without a callsign first).
6. **The terminal version**: `dispatch-tui` (`/help` lists commands).

## Automatic operation

All off by default (Radio panel, or `/auto`, `/seq`, `/beacon` in the terminal version).
Answering queries (JS8) and Dispatch calls may run unattended under FCC 97.221(c). Beacons, FT8
auto-sequencing and forwarding held Dispatch messages start transmissions: only use them while
you are at the controls (locally or remotely). The window shows **AUTO TX** when it transmits
on its own.

## Reporting

Please send:
- what you did, what happened, and what you expected;
- your OS and version, radio, interface (Digirig Mobile/Lite, radio USB, other);
- `~/.dispatch/ALL.TXT` (or the part around the problem), `~/.dispatch/tui.log` if you used the
  terminal version, and screenshots;
- for audio problems, the output of `dispatch-station devices` and `dispatch-station levels`.

Licences: Dispatch's application is GPL-3.0-or-later (LICENSE), its core MIT (LICENSE-MIT); the
source is included. Third-party components: THIRD-PARTY-LICENSES.txt.

73, KQ4TGD
