# Samsung SCX-4623FW on macOS

Printing and scanning from a networked Samsung SCX-4623FW on macOS, through CUPS and SANE,
and the scanning app planned on top of them.

## Language

### Scanning

**Source**:
Where the paper is read from: the **Flatbed** or the **ADF**. A property of one scan, not of a document.
_Avoid_: input, tray, mode

**Flatbed**:
The glass. Reads one sheet per scan.
_Avoid_: glass bed, platen

**ADF**:
The automatic document feeder. Reads a stack of sheets in one scan, front side only.
_Avoid_: feeder, loader, tray

**Scan**:
One run of the scanner over one source, producing one or more pages.
_Avoid_: pass (except inside a two-pass duplex), job, acquisition

**Preview**:
A quick low-resolution scan from the flatbed, shown to check the sheet and never added to a document.
_Avoid_: prescan, thumbnail

**Page**:
One scanned image of one side of one sheet.
_Avoid_: image, sheet, frame

**Document**:
The ordered set of pages accumulated across scans until it is saved. May mix sources.
_Avoid_: batch, session, file

**Two-pass duplex**:
Scanning double-sided originals with the ADF twice — all fronts, then the flipped stack for all backs — and interleaving the two passes into one document.
_Avoid_: duplex (alone), double-sided scan

**Interleave**:
Merging the two passes of a two-pass duplex into front/back order, taking the backs in reverse.
_Avoid_: collate, merge
