# RS01 tools

Unofficial browser tools for the **Contec RS01** sleep apnea screen meter (USB name “Sleep apnea screen meter”, USB ID `28E9:01F3`), in a single page, [index.html](index.html):

- **Set the clock**: sets the device’s clock from your computer, over USB.
- **Convert nights for OSCAR**: reads a night’s recording from the RS01’s card, shows SpO2 and pulse, and converts it into a file [OSCAR](https://www.sleepfiles.com/OSCAR/) can import.

Everything runs in your browser. Nothing is uploaded anywhere.

> These tools are not made or endorsed by Contec. The protocol and file formats were worked out from the device’s files and from the ResMon Plus 3.0.2.2 software. They are not medical software: use them for personal tracking, and rely on the official software and your doctor for anything clinical.

## Set the RS01 clock

1. Open [index.html](index.html) in **Chrome or Edge** on a computer. Safari and Firefox don’t support USB HID devices (WebHID).
2. Switch the RS01 on and plug it in with a USB cable that carries data (charge-only cables won’t work).
3. Press **Set the RS01 clock**. The first time, Chrome asks which device to allow: pick “Sleep apnea screen meter”.

The page shows success when the RS01 confirms. **Test: set it 2 minutes ahead** lets you check visually that it works before setting the real time.

WebHID only works when the page is served over HTTPS (or opened as a local file) and shown directly in the tab, not inside another site’s frame. GitHub Pages works.

## Convert nights for OSCAR

1. Open [index.html](index.html) in any recent browser (this part doesn’t need Chrome).
2. Plug in the RS01: its card shows up as a disk. Drop a night’s folder (the `.HD` file and its `.SEG` files) onto the page, or the whole card to see every night.
3. For each night, press **Download for OSCAR (.zip)** and unzip it.
4. In OSCAR, choose **File › Import › Viatom/Wellue** and select the file named after the date and time, with no extension (for example `20261102052830`).

The archive also holds a `.csv` file with second-by-second SpO2 and pulse, for a spreadsheet.

Limits:

- OSCAR has no RS01 importer, so nights are converted to the Viatom/Wellue (O2Ring) oximetry format. Only **SpO2 and pulse** are imported; the nasal airflow and the pleth wave are not.
- The default resolution is one point per second, every reading the RS01 took. If your OSCAR version rejects the file, choose 4 s, the O2Ring’s own rhythm (each point is then the mean of a 4-second window).

## Publishing on GitHub Pages

Put the files in a repository, then in **Settings › Pages** choose “Deploy from a branch”, branch `main`, folder `/ (root)`. After a minute the tools are at `https://<your-name>.github.io/<repository>/`.

## Python versions

The `python/` folder holds command-line equivalents, tested on macOS.

```bash
python3 -m venv ~/rs01env && source ~/rs01env/bin/activate
pip install hidapi

python3 python/rs01_hid_clock.py                  # dry run: shows the packet
python3 python/rs01_hid_clock.py --send           # sets the clock
python3 python/rs01_to_oscar.py /Volumes/<CARD> -o ~/Desktop/oscar
```

## Technical notes

### USB

The RS01 exposes two USB interfaces: a mass-storage disk (its TF card) and a HID interface. There is no serial port; older ResMon versions (2.x) talk to an earlier serial-based model and don’t work with this one.

HID reports are 64 bytes in both directions, with no report ID. Commands put the bytes at the start of the report and pad with zeros. Command bytes have the high bit set, data bytes are 7-bit, and the checksum is the sum of the preceding bytes `& 0x7F`.

| Command | Bytes sent | Expected reply |
|---|---|---|
| Set clock | `83 YY MM DD hh mm ss 00 00 CS` (YY = year − 2000) | `F3 00 73` |
| Query (sent by ResMon Plus on connect) | `81 01 …` | starts with `F1` |
| Switch mode (sent by ResMon Plus after the clock) | `84 01 00 00 00 00 00 CS` | `F4 00 74` |
| Unknown command | anything else | `F0 70` |

The tools only send the clock command.

### Recording files

Each night is a `.HD` file plus `001.SEG`, `002.SEG`, … (600 seconds per segment).

`.HD` is a 512-byte ASCII header. The start time is at offset `0x16` as `YYYYMMDDHHMMSS`; the signal table that follows declares Flow (50 Hz, 2 bytes, 0–4095 ↔ −511…511 Pa), PR (1 Hz, 1 byte, bpm), SpO2 (1 Hz, 1 byte, %) and Pleth (100 Hz, 1 byte).

`.SEG` starts with a 512-byte header, then one 206-byte record per second:

| Offset | Size | Content |
|---|---|---|
| 0 | 2 | `FF FF` sync |
| 2 | 2 | record number, uint16 little-endian, from 1 |
| 4 | 100 | Flow, 50 × uint16 little-endian |
| 104 | 1 | pulse rate (0 = no reading) |
| 105 | 1 | SpO2 (0 = no reading) |
| 106 | 100 | Pleth, 100 × uint8 |

### Output file

The OSCAR file uses the Viatom/Wellue layout: a 40-byte header (`uint16` signature `0x0003`, `uint16` year, then month, day, hour, minute, second as bytes, `uint32` file size, `uint16` duration in seconds, zero padding), then 5-byte records `SpO2, pulse, invalid flag, motion, vibration`. Records without a valid reading are written as `FF FF 01 00 00`.
