# Snapshot packing in binary

All numbers are encoded in little endian, and have no padding unless otherwise stated.
Floats follow the IEEE 754 standard.

## CFrame Schema
All fields in this schema are offset from the start of the `CFrame`.

| Offset | Size | Field |
|--------|------|-------|
| 0x00   | 4    | X (`f32`) |
| 0x04   | 4    | Y (`f32`) |
| 0x08   | 4    | Z (`f32`), two least-significant bits indicate index of dropped quaternion component |
| 0x0C   | 2    | q0 (`i16`) |
| 0x0E   | 2    | q1 (`i16`) |
| 0x10   | 2    | q2 (`i16`) |

Total size = `18` bytes

### Z-component packing
- The two least-significant bits (LSBs) of the Z component should be overwritten with the index of the dropped quaternion component.
- Treat Z as a `u32` when performing bit manipulation; bitwise operations on the float directly will not behave correctly.
- When reading the Z coordinate, masking the LSBs is optional — the introduced error is negligible (~1e-5).

### Example CFrame:

Random CFrame I got from `Workspace.CurrentCamera` lol

Position: -86.98088, 447.52756, -216.12019
Rotation: -13.207°, 58.252°, 0°

Compressed quaternion:
`i = 3, q0 = -3292, q1 = 15843, q2 = 1834`

| X           | Y           | Z           | q0    | q1    | q2    |
|-------------|-------------|-------------|-------|-------|-------|
|`36 F6 AD C2`|`87 C3 DF 43`|`C0 1E 58 C3`|`24 F3`|`E3 3D`|`2A 07`|

Errors that occur during (de)compression should be fatal. For example, if square root operations receive a negative number or return NaN, the process should be aborted.

### Compression
1. Get axis-angle representation of the CFrame rotation.
2. If the magnitude of the axis is significant enough,
multiply each component by `sin(angle / 2) / magnitude`. Otherwise, set the X to `sin(angle / 2)`, and set Y and Z to 0.
3. Get the `cos(angle / 2)` (`w`).
4. If the squared length - the sum of each component and `w` squared (i.e., `x*x + y*y + z*z + w*w`) - is insignificant enough, the quaternion components are `qx = 0`, `qy = 0`, `qz = 0`, and `qw = 1`. Otherwise, normalize the quaternion by making `qx, qy, qz, qw` equal to each respective component divided by the quaternion's magnitude.
5. Find the largest of `qx, qy, qz, qw`. The index should be the position of the largest component (0-3) *in that order*, where index 0 is `qx` and index 3 is `qw`. The largest value is the one that will be dropped.
6. Let `q0`, `q1`, and `q2` be the three remaining components that were not dropped (still in order - so if `qy` was dropped, `q0, q1, q2 = qx, qz, qw`). Multiply each of these by the sign of the dropped component (1 or -1, no 0) and then multiply again by 32767 to convert to an `i16`. Round each value to the nearest integer.
7. Store the index in the two least-significant bits of the Z component in the CFrame's position. Store the `q0`, `q1`, and `q2` values as `i16`s following the position values.

#### Decompression
1. Extract the index bits from the Z component in the CFrame's position.
2. Retrieve the `q0`, `q1`, and `q2` values from the binary data following the position values. Divide each by 32767 to convert back to floats.
3. Let `d = sqrt(1 - (q0*q0 + q1*q1 + q2*q2))`.
4. The component that was dropped is equal to `d`. For example, if the index was 2, then `qx, qy, qz, qw = q0, q1, d, q2`.

Converting the quaternion back to the CFrame in Roblox is as simple as passing it to `CFrame.new`:
`CFrame.new(x, y, z, qx, qy, qz, qw)` - see [docs](https://create.roblox.com/docs/reference/engine/datatypes/CFrame#new)

## Recording Schema
All fields in this schema are offset from the start of the `Recording`.

| Offset | Size | Field |
|--------|------|-------|
| 0x00   |  1   | Map name length (`u8`) - `S` |
| 0x01   | `S`  | Map name (utf8 string) - **no null terminator** |
| 0x01+S |  2   | CFrame count (`u16`) - `C` |
| 0x03+S |18*`C`| Exactly `C` `CFrame`s |

Recording size = `3 + S + 18*C` bytes

## Data Schema

The buffer will be made up of `N` `Recording`s. 

There is no header that precedes the list of recordings. Each `Recording` is stored sequentially in the buffer. This also means the only way to count the number of recordings is to read until the end of the buffer.

It is relatively simple and cheap, since the amount of CFrames is encoded in each `Recording`.

Data mismatches caused by incorrect length values should be considered fatal, since recovering from this is very difficult.
