Dispatch on Windows (test build)
================================

This is the first Windows build and has not been tried on Windows yet. Please report
anything that doesn't work, with a screenshot or the exact error text.

Running it
  Unzip the folder anywhere and double-click Dispatch.exe. Windows SmartScreen will say
  "Windows protected your PC" because the program isn't signed: click "More info", then
  "Run anyway". dispatch-station.exe is the command-line tool (run it from a Command Prompt
  or PowerShell; "dispatch-station --help" lists its commands).

Digirig Mobile
  Install the Silicon Labs CP210x driver if Windows doesn't show a COM port when the
  Digirig is plugged in (Device Manager > Ports). In the Radio panel, choose the Digirig's
  "USB Audio Device" for input and output, and set the PTT port to its COM port (COM3, say).

CAT control (new on Windows, least tested)
  Choose your radio in the Radio panel and press "Test CAT": it should show the dial
  frequency. The truSDX appears as its own COM port (USB-SERIAL CH340): pick that one.
  If Test CAT fails, note the exact message; meanwhile key by the Digirig's PTT line or
  VOX and set the dial by hand.

Not on Windows yet
  - Digirig Lite PTT (its CM108 GPIO): use VOX meanwhile.
  - The terminal version (dispatch-tui).

Clock
  FT8 and JS8 need the PC's clock within a second: Settings > Time & language > Date & time
  > "Sync now".
