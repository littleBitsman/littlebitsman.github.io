# Snapshot packing in binary

All numbers are encoded in little endian, and have no padding unless otherwise stated.
Floats follow the IEEE 754 standard.

## Data Schema

| Offset | Size | Field |
|--------|------|-------|
| 0x00   | 8    | Version header (utf8 string, e.g. `SNAPV1.3`) |
| 0x08   | 1    | Recording count (`u8`) - `N` |
| 0x09   | ...  | Exactly `N` `Recording`s, stored sequentially |

Unlike earlier revisions, the recording count is stored explicitly rather than
inferred by reading to the end of the buffer. A reader must check the version
header against the version it was built for and treat any mismatch as fatal -
do not attempt to decode a buffer produced by a different version.

> [!NOTE]
> Latest snapshot version is V1.3; the header is `SNAPV1.3`.

## Recording Schema
All fields in this schema are offset from the start of the `Recording`.

| Offset  | Size | Field |
|---------|------|-------|
| 0x00    | 1    | Map name length (`u8`) - `S` |
| 0x01    | `S`  | Map name (utf8 string) - **no null terminator** |
| 0x01+S  | 2    | Frame count (`u16`) - `C` |
| 0x03+S  | 4    | Total duration in milliseconds (`u32`) - sum of every frame's `dt`, provided so a reader can know playback length without pre-scanning every frame |
| 0x07+S  | ...  | Exactly `C` `Frame`s, stored sequentially |

Recording size is variable and depends on how many frames are stored as
absolute versus delta (see Frame Schema below) - it can no longer be computed
as a fixed multiple of `C`.

The **first** frame in every recording must be an absolute frame. Readers
should treat any recording whose first frame is not absolute as corrupt and
fatal.

## Frame Schema

Each frame begins with a 1-byte bitfield that determines its shape:

| Bitfield value | Meaning |
|-----------------|---------|
| `0b100`         | Absolute frame - full position + quaternion follows |
| bit `0b001` set | Delta frame includes a position delta |
| bit `0b010` set | Delta frame includes a rotation delta |
| `0b000`         | Delta frame with neither - position and rotation unchanged since the previous frame (this still consumes a frame slot, e.g. for a timing-only tick) |

Note `0b100` is a distinct sentinel, not a flag combinable with `0b001`/`0b010`
- an absolute frame never carries delta fields.

Every `KEYFRAME_INTERVAL`-th frame (currently every 60th) is forced absolute
regardless of motion, to bound accumulated quantization drift from chained
deltas. A frame is also forced absolute if the position delta since the
previous frame exceeds `MAX_POS_DELTA` (8.0 studs) or the rotation delta
exceeds `MAX_ROT_DELTA` (0.2 rad) - i.e. deltas are only used when both the
translation and rotation since the last frame are small enough to fit the
delta encoding's precision and range.

### Absolute frame

| Offset | Size | Field |
|--------|------|-------|
| 0x00   | 1    | Bitfield (`u8`) - always `0b100` |
| 0x01   | 2    | Delta-time since previous frame in ms (`u16`) |
| 0x03   | 18   | `CFrame` (see CFrame Schema below) |

Total size = `21` bytes

### Delta frame

| Offset | Size | Field |
|--------|------|-------|
| 0x00   | 1    | Bitfield (`u8`) - `0b000`, `0b001`, `0b010`, or `0b011` |
| 0x01   | 2    | Delta-time since previous frame in ms (`u16`) |
| 0x03   | 6    | Position delta (3x `i16`) - **present only if bit `0b001` is set** |
| 0x03 or 0x09 | 6 | Rotation delta (3x `i16`) - **present only if bit `0b010` is set**; offset depends on whether the position delta field is present |

Total size = `3`, `9`, or `15` bytes depending on which optional fields are present.

#### Position delta
Each `i16` component is `round(delta * POS_DELTA_SCALE)`, where `POS_DELTA_SCALE = 4096`
and `delta` is the difference between the current frame's position and the
previous frame's position, per axis. To decode: `delta = i16_value / POS_DELTA_SCALE`,
then add to the previous frame's position.

The position-delta bit is only set if the squared delta magnitude is at least
`FP_EPSILON` (`1e-6`) - deltas smaller than this are treated as no change and
the bit is left unset, saving 6 bytes for effectively-stationary frames.

#### Rotation delta
Computed as the axis-angle rotation from the previous frame's orientation to
the current one (i.e. `previous:ToObjectSpace(current)`'s axis-angle). Each
`i16` component is `round(axis[i] * angle * ROT_DELTA_SCALE)`, where
`ROT_DELTA_SCALE = 32767 / MAX_ROT_DELTA`. To decode: reconstruct the vector
by dividing each component by `ROT_DELTA_SCALE`, take its magnitude as the
angle and its normalized direction as the axis, then apply
`previous * CFrame.fromAxisAngle(axis, angle)`.

The rotation-delta bit is only set if the resulting angle exceeds `ROT_EPSILON`
(`1e-4` rad) **and** the quantized components aren't all zero - this avoids
spending 6 bytes to encode a rotation too small to be meaningfully represented
at this quantization scale.

Errors that occur during (de)compression should be fatal. For example, if
square root operations receive a negative number or return NaN, the process
should be aborted. (In practice, decoders should clamp the value under the
square root to a minimum of 0 before taking the root, since quantization
rounding can otherwise push it very slightly negative - see CFrame
decompression step 3.)

## CFrame Schema (non-delta encoded)
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
- When reading the Z coordinate, masking the LSBs is optional - the introduced error is negligible (~1e-5).

### Compression
1. Get axis-angle representation of the CFrame rotation.
2. If the magnitude of the axis is significant enough,
multiply each component by `sin(angle / 2) / magnitude`. Otherwise, set the X to `sin(angle / 2)`, and set Y and Z to 0.
3. Get the `cos(angle / 2)` (`w`).
4. If the squared length - the sum of each component and `w` squared (i.e., `x*x + y*y + z*z + w*w`) - is insignificant enough, the quaternion components are `qx = 0`, `qy = 0`, `qz = 0`, and `qw = 1`. Otherwise, normalize the quaternion by making `qx, qy, qz, qw` equal to each respective component divided by the quaternion's magnitude.
5. Find the largest of `qx, qy, qz, qw`. The index should be the position of the largest component (0-3) *in that order*, where index 0 is `qx` and index 3 is `qw`. The largest value is the one that will be dropped.
6. Let `q0`, `q1`, and `q2` be the three remaining components that were not dropped (still in order - so if `qy` was dropped, `q0, q1, q2 = qx, qz, qw`). Multiply each of these by the sign of the dropped component (1 or -1, no 0) and then multiply again by 32767 to convert to an `i16`. Round each value to the nearest integer.
7. Store the index in the two least-significant bits of the Z component in the CFrame's position. Store the `q0`, `q1`, and `q2` values as `i16`s following the position values.

### Decompression
1. Extract the index bits from the Z component in the CFrame's position.
2. Retrieve the `q0`, `q1`, and `q2` values from the binary data following the position values. Divide each by 32767 to convert back to floats.
3. Let `d = sqrt(max(0, 1 - (q0*q0 + q1*q1 + q2*q2)))`. Clamping under the square root prevents NaN from quantization rounding pushing the argument slightly negative.
4. The component that was dropped is equal to `d`. For example, if the index was 2, then `qx, qy, qz, qw = q0, q1, d, q2`.

Converting the quaternion back to the CFrame in Roblox is as simple as passing it to `CFrame.new`:
`CFrame.new(x, y, z, qx, qy, qz, qw)` - see [docs](https://create.roblox.com/docs/reference/engine/datatypes/CFrame#new)

#### Example CFrame
Random CFrame I got from `Workspace.CurrentCamera` lol

Position: -86.98088, 447.52756, -216.12019
Rotation: -13.207°, 58.252°, 0°

Compressed quaternion:
`i = 3, q0 = -3292, q1 = 15843, q2 = 1834`

| X           | Y           | Z           | q0    | q1    | q2    |
|-------------|-------------|-------------|-------|-------|-------|
|`36 F6 AD C2`|`87 C3 DF 43`|`C0 1E 58 C3`|`24 F3`|`E3 3D`|`2A 07`|
