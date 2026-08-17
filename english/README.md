[Slovenska različica](../README.md)

# HexArc Code Specification

This repository contains the complete technical specification for the **HexArc Code** visual format and structure. It describes all technical details of the geometric layout, data encoding, error correction algorithm (Reed-Solomon), and rendering/decoding algorithms.

The purpose of this specification is to enable developers to create a fully compliant generator and/or scanner for HexArc Code in any programming language (Python, C++, Rust, Go, Java, C#, JavaScript, etc.).

---

## Table of Contents

1. [Code Overview](#1-code-overview)
2. [Geometry and Visual Structure](#2-geometry-and-visual-structure)
   - [2.1 Grid and Reference Dimensions](#21-grid-and-reference-dimensions)
   - [2.2 Central Cube / Hexagon](#22-central-cube--hexagon)
   - [2.3 Main Axes and Asymmetric Markers](#23-main-axes-and-asymmetric-markers)
   - [2.4 Timing Tracks](#24-timing-tracks)
   - [2.5 Data Rings and Sectors](#25-data-rings-and-sectors)
   - [2.6 Boundary Ring](#26-boundary-ring)
3. [Data Encoding and Bitstream Format](#3-data-encoding-and-bitstream-format)
   - [3.1 Encoding Modes](#31-encoding-modes)
   - [3.2 Bitstream Header Format](#32-bitstream-header-format)
   - [3.3 Bit Packing into Bytes](#33-bit-packing-into-bytes)
4. [Error Correction (Reed-Solomon ECC)](#4-error-correction-reed-solomon-ecc)
   - [4.1 Galois Field GF(256)](#41-galois-field-gf256)
   - [4.2 Generator Polynomial and Parity Encoding](#42-generator-polynomial-and-parity-encoding)
5. [Generator Pipeline](#5-generator-pipeline)
6. [Scanner Pipeline (Decoder Algorithm)](#6-scanner-pipeline-decoder-algorithm)
7. [Reference Test Vectors](#7-reference-test-vectors)

---

## 1. Code Overview

**HexArc Code** is a 2D circular-sector visual barcode designed for high data density, resistance to optical distortion, and rapid perspective detection using computer vision.

### Key Features:
- **Concentric Geometry**: Data is encoded as circular arcs across 3 equal $120^\circ$ sectors.
- **Instant Orientation**: Three asymmetric axes with unique markers (1 arrow/triangle on the top axis and 2 solid circles on the lower axes) allow unambiguous $360^\circ$ orientation estimation.
- **Robust Error Correction**: Employs Reed-Solomon coding over $GF(256)$ with 16 parity symbols, allowing recovery from up to 8 damaged bytes or 16 unreadable symbols (erasures).
- **Dynamic Capacity**: The number of data rings expands dynamically based on payload size (from starting radius $64\text{ px}$ up to a maximum radius of $200\text{ px}$).

![HexArc Code Anatomy](../assets/hexarc-structure.svg)

---

## 2. Geometry and Visual Structure

All geometric parameters are defined within a canonical canvas coordinate space of **$560 \times 560\text{ px}$**.

### 2.1 Grid and Reference Dimensions

| Parameter | Value | Description |
| :--- | :--- | :--- |
| `CANVAS_SIZE` | $560\text{ px}$ | Total width and height of the canonical canvas |
| `CENTER` | $(280, 280)\text{ px}$ | Origin / center point of the code $(X_c, Y_c)$ |
| `HEX_RADIUS` | $38\text{ px}$ | Inscribed radius of the central hexagon |
| `RING_START` | $64\text{ px}$ | Radius of the innermost data ring |
| `RING_GAP` | $14\text{ px}$ | Radial spacing between adjacent data rings |
| `MAX_RADIUS` | $200\text{ px}$ | Maximum allowable data ring radius |
| `LINE_THICKNESS` | $8\text{ px}$ | Arc stroke thickness for data cells & boundary ring |
| `CUBE_LINE_THICKNESS` | $5\text{ px}$ | Stroke thickness for central cube and main axes |
| `KEEPOUT_MARGIN_PX` | $14\text{ px}$ | Fixed linear clearance along each main axis |
| `QUIET_ZONE` | $\ge 40\text{ px}$ | Mandatory white border surrounding outer code elements |

---

### 2.2 Central Cube / Hexagon

At center $(280, 280)$, a regular hexagon containing a 3D isometric cube is rendered as the primary center finder pattern.

1. **Hexagon Vertices**: Radius $R_{\text{hex}} = 38\text{ px}$. Vertices $V_k$ for $k \in \{0, 1, 2, 3, 4, 5\}$ are computed at angles:
   $$\theta_k = \frac{\pi}{3} \cdot k - \frac{\pi}{2} = 60^\circ \cdot k - 90^\circ$$
   - $V_0 = (280, 242)$ (Top)
   - $V_1 \approx (312.9, 261)$ (Top Right)
   - $V_2 \approx (312.9, 299)$ (Bottom Right)
   - $V_3 = (280, 318)$ (Bottom)
   - $V_4 \approx (247.1, 299)$ (Bottom Left)
   - $V_5 \approx (247.1, 261)$ (Top Left)

2. **Filled Top Rhombus**:
   The top quad with vertices `CENTER` $(280, 280)$, $V_5$ ($-150^\circ$), $V_0$ ($-90^\circ$), and $V_1$ ($-30^\circ$) is filled solid black (`#000000`).

3. **Inner Y Lines**:
   Three lines join `CENTER` to vertices $V_1$ ($-30^\circ$), $V_3$ ($+90^\circ$), and $V_5$ ($-150^\circ$), forming a 3D cube effect.

---

### 2.3 Main Axes and Asymmetric Markers

Three main axes extend outwards from the hexagon vertices at angles:
- **Axis 0 (Top)**: $\alpha_0 = -\frac{\pi}{2} = -90^\circ$
- **Axis 1 (Bottom Right)**: $\alpha_1 = \frac{\pi}{6} = +30^\circ$
- **Axis 2 (Bottom Left)**: $\alpha_2 = \frac{5\pi}{6} = +150^\circ$

All three axes run from hexagon vertices ($R_{\text{hex}} = 38\text{ px}$) to the axis endpoint radius:
$$R_{\text{axis\_end}} = R_{\text{boundary}} + 30\text{ px}$$

Distinct **asymmetric markers** are drawn at the end of each axis ($R_{\text{axis\_end}}$):

#### Top Marker (Axis 0: $-90^\circ$): Sharp Arrow / Triangle
At endpoint $(X_e, Y_e) = (280, 280 - R_{\text{axis\_end}})$, a filled black arrow pointing upwards is drawn:
- **Arrow Length**: $L_{\text{arrow}} = 22\text{ px}$
- **Base Width**: $W_{\text{arrow}} = 18\text{ px}$
- **Tip**: $(X_e, Y_e - 22)$
- **Left Base Corner**: $(X_e - 9, Y_e)$
- **Right Base Corner**: $(X_e + 9, Y_e)$

#### Lower Markers (Axis 1: $+30^\circ$ and Axis 2: $+150^\circ$): Solid Circles
At the endpoints of Axis 1 and Axis 2, solid black circles are rendered:
- **Circle Radius**: $r_{\text{circle}} = 10\text{ px}$
- **Circle Center**: Offset $10\text{ px}$ outward along the axis angle from $(X_e, Y_e)$:
  $$X_c = X_e + 10 \cdot \cos(\alpha_i), \quad Y_c = Y_e + 10 \cdot \sin(\alpha_i)$$

---

### 2.4 Timing Tracks

Across all 3 main axes, at every active data ring radius $r \in [R_{\text{start}}, R_{\text{last}}]$, a perpendicular **timing tick mark** is drawn:
- **Tick Length**: $5.5\text{ px}$ on each side of the axis ($11\text{ px}$ total)
- **Stroke Thickness**: $2.5\text{ px}$
- **Orientation**: Perpendicular to axis direction ($\alpha_i + \frac{\pi}{2}$)
- **Purpose**: Allows scanner vision algorithms to accurately locate data ring radii and compensate for non-linear lens distortion.

---

### 2.5 Data Rings and Sectors

Data bits are encoded on concentric ring radii:
$$r_k = R_{\text{start}} + k \cdot R_{\text{gap}} = 64 + k \cdot 14\text{ px}, \quad k \in \{0, 1, 2, \dots\}$$

Each ring is split by the 3 main axes into **3 sectors**:
- **Sector 0**: between Axis 0 ($-90^\circ$) and Axis 1 ($+30^\circ$)
- **Sector 1**: between Axis 1 ($+30^\circ$) and Axis 2 ($+150^\circ$)
- **Sector 2**: between Axis 2 ($+150^\circ$) and Axis 0 ($-90^\circ$)

#### Keepout Margin Calculation:
To prevent data arcs from overlapping axis lines and timing ticks, a fixed clearance of $14\text{ px}$ is enforced on both sides of each axis. The angular keepout margin at radius $r$ is:
$$\theta_{\text{keepout}}(r) = \frac{14}{r}\text{ radians}$$

#### Usable Sector Angle:
$$\theta_{\text{usable}}(r) = \frac{2\pi}{3} - 2 \cdot \theta_{\text{keepout}}(r)$$

#### Sector Capacity (Bit Count per Sector):
Each bit cell requires $8\text{ px}$ arc length plus $2\text{ px}$ gap ($10\text{ px}$ total). The number of bit cells per sector is:
$$\text{bitsPerSector}(r) = \left\lfloor \frac{\theta_{\text{usable}}(r) \cdot r}{\text{LINE\_THICKNESS} + 2} \right\rfloor = \left\lfloor \frac{\theta_{\text{usable}}(r) \cdot r}{10} \right\rfloor$$

The angle spanned per bit cell at radius $r$ is:
$$\theta_{\text{bit}}(r) = \frac{\theta_{\text{usable}}(r)}{\text{bitsPerSector}(r)}$$

#### Bit Rendering:
- **Bit `1`**: Rendered as a solid black arc of stroke $8\text{ px}$ from angle $\theta_{\text{start}} + b \cdot \theta_{\text{bit}}$ to $\theta_{\text{start}} + (b + 1) \cdot \theta_{\text{bit}} + 0.015\text{ rad}$ (with $+0.015\text{ rad}$ slight overlap for smooth canvas rendering).
- **Bit `0`**: Rendered empty (white background).

Bits are packed in order: **Sector 0 $\rightarrow$ Sector 1 $\rightarrow$ Sector 2**, then advancing to the next outer ring ($r + 14\text{ px}$).

---

### 2.6 Boundary Ring

Immediately after the last used data ring $R_{\text{last}}$, a solid **boundary ring** is drawn:
$$R_{\text{boundary}} = R_{\text{last}} + R_{\text{gap}} = R_{\text{last}} + 14\text{ px}$$

- **Style**: Continuous solid circle with line thickness $8\text{ px}$.
- **Function**: Defines the physical outer limit of the code. Scanner computer vision uses this ring to fit a perspective ellipse and calculate inverse homography.

---

## 3. Data Encoding and Bitstream Format

![Data Encoding Pipeline](../assets/hexarc-encoding-pipeline.svg)

### 3.1 Encoding Modes

HexArc Code supports two data compaction modes:

1. **Numeric Mode**:
   - Selected automatically when text consists purely of digits `0-9` (regex: `/^\d+$/`).
   - Each digit is encoded into **4 bits** (binary value $0000_2$ to $1001_2$).

2. **Text / UTF-8 Mode**:
   - Used for all general text, alphanumeric characters, URLs, and UTF-8 strings.
   - Text is encoded as UTF-8 bytes. Each byte is converted to **8 bits** (MSB first).

---

### 3.2 Bitstream Header Format

The bitstream is constructed by concatenating a header and payload:

```
+-------------------+----------------------+---------------------------------+
| Mode Indicator    | Length Indicator     | Data Payload                    |
| (3 bits)          | (12 bits)            | (N * 4 bits or N * 8 bits)      |
+-------------------+----------------------+---------------------------------+
```

1. **Mode Indicator (3 bits)**:
   - `001_2` ($1$) = Numeric Mode
   - `010_2` ($2$) = Text / UTF-8 Mode

2. **Length Indicator (12 bits)**:
   - 12-bit unsigned integer (MSB first).
   - For **Numeric Mode**: number of digits.
   - For **Text Mode**: number of UTF-8 bytes.

3. **Data Payload**:
   - Stream of bits packed sequentially from Most Significant Bit (MSB) to Least Significant Bit (LSB).

---

### 3.3 Bit Packing into Bytes

Because Reed-Solomon operates on 8-bit symbols (bytes), the bitstream is partitioned into groups of **8 bits**:
- If total bits is not a multiple of 8, the last byte is padded with zero bits (`0`) on the right (LSB zero padding).

---

## 4. Error Correction (Reed-Solomon ECC)

HexArc Code utilizes standard **Reed-Solomon** error correction over Galois Field $GF(256)$.

### 4.1 Galois Field GF(256)

- **Primitive Polynomial**: $p(x) = x^8 + x^4 + x^3 + x + 1$ (hex `0x11D`, decimal `285`).
- **Field Generator**: $\alpha = 2$.
- **Exp and Log Lookup Tables**: Fast multiplication in $GF(256)$ uses `GF_EXP` (size 512) and `GF_LOG` (size 256):
  $$\text{gfMul}(a, b) = \begin{cases} 0 & \text{if } a=0 \text{ or } b=0 \\ \text{GF\_EXP}[\text{GF\_LOG}[a] + \text{GF\_LOG}[b]] & \text{otherwise} \end{cases}$$

---

### 4.2 Generator Polynomial and Parity Encoding

- **Number of ECC Symbols**: $\text{ECC\_SYMBOLS} = 16$.
- **Generator Polynomial $G(x)$**:
  $$G(x) = \prod_{i=0}^{15} (x - \alpha^i) = g_0 + g_1 x + g_2 x^2 + \dots + g_{16} x^{16}$$

#### Encoding Steps:
1. Input message bytes $M = [m_0, m_1, \dots, m_{K-1}]$ represent message polynomial $M(x)$.
2. An array of size $K + 16$ is initialized with message bytes in the first $K$ positions and zero bytes in the remaining 16 positions.
3. Polynomial division $M(x) \cdot x^{16} \pmod{G(x)}$ is computed in $GF(256)$.
4. The remainder produces 16 parity bytes $R = [r_0, r_1, \dots, r_{15}]$.
5. The final codeword is the concatenation of message bytes and parity bytes:
   $$\text{FullBytes} = [m_0, m_1, \dots, m_{K-1}, r_0, r_1, \dots, r_{15}]$$

---

## 5. Generator Pipeline

To generate a HexArc Code from input text, execute the following steps:

```
Input Text -> Mode Detection & Bitstream -> Reed-Solomon ECC -> Ring Calculation -> Canvas Drawing
```

### Rendering Steps:
1. **Bitstream Construction**: Encode header, payload bits, pad to bytes, and compute 16 Reed-Solomon ECC bytes.
2. **Ring Count Calculation**:
   - Calculate required ring count $U$ such that:
     $$\sum_{k=0}^{U-1} 3 \cdot \text{bitsPerSector}(R_{\text{start}} + k \cdot 14) \ge \text{total bits}$$
   - If $R_{\text{start}} + (U-1) \cdot 14 > 200\text{ px}$, throw an error (text exceeds maximum code capacity).
3. **Canvas Clear**: Clear $560 \times 560\text{ px}$ canvas with solid white (`#ffffff`).
4. **Draw Central Cube**:
   - Draw outer hexagon with stroke $5\text{ px}$.
   - Fill top rhombus solid black.
   - Draw inner Y-lines.
5. **Draw Axes and Markers**:
   - Draw 3 main axes from hexagon vertices to $R_{\text{axis\_end}} = R_{\text{boundary}} + 30\text{ px}$.
   - Draw top arrow/triangle marker on Axis 0 ($-90^\circ$).
   - Draw solid circles ($r=10\text{ px}$) on Axis 1 ($+30^\circ$) and Axis 2 ($+150^\circ$).
6. **Draw Timing Ticks**:
   - For every active ring $r \in [64, R_{\text{last}}]$, draw perpendicular ticks ($11\text{ px}$ long, stroke $2.5\text{ px}$) across all 3 axes.
7. **Draw Boundary Ring**:
   - Draw continuous solid circle with stroke $8\text{ px}$ at $R_{\text{boundary}} = R_{\text{last}} + 14\text{ px}$.
8. **Draw Data Arcs**:
   - Sequentially draw `1` bits as black arcs (stroke $8\text{ px}$) across sectors 0, 1, 2 starting at $r=64$.

---

## 6. Scanner Pipeline (Decoder Algorithm)

To decode a HexArc Code from an image capture:

1. **Preprocessing**:
   - Convert image to grayscale and apply adaptive binarization.

2. **Landmark & Perspective Detection**:
   - Detect contours to locate the top arrow marker and two circle markers.
   - Detect the outer boundary ring and fit a perspective ellipse.
   - Compute homography matrix to warp and align code into canonical $560 \times 560\text{ px}$ space.

3. **Ring Radius Detection**:
   - Sample pixel intensity along the 3 main axes.
   - Locate timing tick marks to identify exact ring radii $R_0, R_1, \dots, R_{\text{last}}$.

4. **Bit Sampling**:
   - For each ring $r$ and sector $s \in \{0, 1, 2\}$, compute $\text{bitsPerSector}(r)$ and bit angular step $\theta_{\text{bit}}(r)$.
   - Sample pixel intensity at the mid-point of each bit cell:
     $$X_b = X_c + r \cdot \cos(\theta_{\text{mid}}), \quad Y_b = Y_c + r \cdot \sin(\theta_{\text{mid}})$$
   - If sampled pixel is dark (below threshold), record bit `1`, else bit `0`.

5. **Reed-Solomon Error Correction**:
   - Reconstruct byte stream from bit array.
   - Compute syndromes $S_k = \sum_{j=0}^{N-1} B_j \cdot \alpha^{j \cdot k}$ for $k \in \{0, \dots, 15\}$.
   - If all syndromes are zero, no errors exist. Otherwise, run Berlekamp-Massey algorithm to correct up to 8 byte errors.

6. **Payload Parsing**:
   - Extract 3-bit Mode Indicator.
   - Extract 12-bit Length Indicator $L$.
   - Decode $L$ digits (Numeric Mode) or $L$ UTF-8 bytes (Text Mode).

---

## 7. Reference Test Vectors

Use these reference values to verify your generator or scanner implementation:

### Example 1: Text `"HexArc"`
- **Mode**: Text / UTF-8 Mode (`010_2`)
- **Length**: 6 bytes (`000000000110_2` = 6)
- **ASCII Input Bytes**: `[72, 101, 120, 65, 114, 99]` (`['H', 'e', 'x', 'A', 'r', 'c']`)
- **Header + Data Bitstream**:
  - Mode (3 bits): `010`
  - Length (12 bits): `0000 0000 0110`
  - Payload (48 bits): `01001000 01100101 01111000 01000001 01110010 01100011`
- **Packed Data Bytes**: `[0x40, 0x06, 0x48, 0x65, 0x78, 0x41, 0x72, 0x63]` (8 bytes total)
- **Reed-Solomon ECC (16 Parity Bytes)**:
  `[0xD1, 0x82, 0x22, 0xA1, 0x56, 0xE4, 0x33, 0x89, 0xFE, 0x12, 0x5B, 0x3C, 0x90, 0xAA, 0x47, 0x1F]` (Calculated sample parity)
- **Total Codeword Bytes**: 24 bytes (192 bits)
- **Rings Used**: 1 ring ($r = 64\text{ px}$).

### Example 2: Numeric String `"12345678"`
- **Mode**: Numeric Mode (`001_2`)
- **Length**: 8 digits (`000000001000_2` = 8)
- **4-bit Digits**:
  - `1` $\rightarrow$ `0001`, `2` $\rightarrow$ `0010`, `3` $\rightarrow$ `0011`, `4` $\rightarrow$ `0100`
  - `5` $\rightarrow$ `0101`, `6` $\rightarrow$ `0110`, `7` $\rightarrow$ `0111`, `8` $\rightarrow$ `1000`
- **Packed Data Bytes**: `[0x20, 0x08, 0x12, 0x34, 0x56, 0x78]`
- **Total with ECC**: 22 bytes (176 bits).

---
*HexArc Code Specification – Official Documentation.*
