# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this directory is

The published project (git remote `tsentavta/metka`, public) is the **web terminal** `web/terminal.html` for an STM32 LoRa beacon, plus `README.md`. The README documents the device's ASCII protocol: commands `GET`, `STATON`, `STATOFF` and `FREQ=`, `OK!`/`ERR0R` replies, telemetry format, and the differences between firmware versions. Keep the README in sync when the protocol knowledge changes. The terminal is a single self-contained HTML file using Web Serial (Chrome/Edge only). It has no build step. `window.__beaconFeed(text)` feeds raw device text into the parser, so you can test it without hardware.

Before sending anything to a connected device, remember that `STATOFF` and `FREQ=` change the beacon's live state. Never put the device into DFU/bootloader mode or flash firmware.

`app/` is git-ignored and must not be published. The rest of this section describes it.

`app/` is **not a source tree**. It holds only a built Gradle `application` distribution (`installDist` layout) of a Java Swing desktop app called **SleepTerminal** (window title "SleepTerminal v4.3"). There are no `.java` files, no build script, no tests and no git repository. Any "code change" means first getting the original source, or decompiling `app/lib/app.jar`. Ask the user which one before you start.

- `app/bin/app.bat` / `app/bin/app`: Gradle-generated launch scripts. The main class is `javatest.Com_Terminal_MainFrame`, and the classpath lists every jar in `app/lib/`.
- `app/lib/app.jar`: the application itself. All classes are in package `javatest`, plus large map images in `images/`.
- `app/lib/*.jar`: dependencies: jSerialComm 2.10.3 (serial ports), jsoup 1.21.1, and Guava 31.1 with its transitive jars.
- `app/bin/SleepTerminal.uset`: the persisted connection settings (see below).

## Running

The classes are compiled for Java 17 (class file major version 61), so the app needs a Java 17+ runtime. Java is not on PATH on this machine. Set `JAVA_HOME`, or put `java` on PATH, before you launch:

```bash
app/bin/app.bat
```

JVM options go through `JAVA_OPTS` or `APP_OPTS`. The app is a GUI, so running it needs a desktop session.

## Architecture (from the compiled classes)

- **`Com_Terminal_MainFrame`**: the main Swing window (NetBeans GUI-builder style, with dozens of anonymous `ActionListener` inner classes). It shows the terminal log and the RSSI/distance readouts ("UWB" output in `Add_VisualOut_UWB`). It draws positions on site map images (`images/*_map*.png`). It can also log received data to a text file. One path is hard-coded in the jar: `C:\Users\User\Documents\SerialData_0014.txt`.
- **`ComSettings_Frame`**: the connection layer and its settings dialog. It talks to the device in one of two ways:
  - over a serial port via jSerialComm (port, bitrate, data bits, parity, stop bits);
  - over a TCP `Socket` to an IP and port (default `192.168.2.211:4001`), read by `ReadIPThread` and written by `SendThread`.
  It formats traffic as Hex, Ascii or the device's own "Sleep"/"USOSleep" frame formats (`>>>` marks sent data, `<<<` marks received data). It also reads and writes the `.uset` settings file.
- **`Mlib`**: shared helpers. It holds circular byte and string buffers (`CircBuffer`, `CircStringBuffer`), the frame encode/decode routines (`ByteArrayToUSOSleep`, `USOSleepToByteArray`), the saved settings-window geometry, and drawing utilities.
- **Positioning**: `Triangulation`, `Triangle`, `Square`, `BasePoint`, `TriangPointes`, `Point_Triangle` and `Algorithms` work out a tag's position from distances to fixed base points (circle intersection). They raise errors such as "not enough points … to find a position".
- **Panels**: `SendPanel` sends manual commands, `UsoPanel` handles the USO device commands and prompts, and `SpeedTestPanel` measures link Tx/Rx throughput.

### `SleepTerminal.uset` format

The file is one comma-separated line, for example:

```
ttyACM0,0,2,0,0,0,192.168.2.211,4001,
```

The fields are:
1. Serial port name.
2–6. Indices for the bitrate, data bits, parity and stop-bit combo boxes, plus the connection type. These are indices into the UI combo boxes, not literal values.
7. IP address.
8. TCP port.

The file sits next to the launcher in `app/bin/`.
