# Reflections at 17:30

## Author: DeliverHellVeil

**Category:** Forensics / Steganography 

## Difficulty: Medium-Hard

---

## Problem Description
 
> **Description:**  
> An item vanished from the back-wall display shelf during the afternoon shift. Security pulled a single frame (`cam_frame_0730.jpg`) from the entrance camera right as the suspect stood near the counter, but the camera's privacy filter scrambled the shelf display and bundled the incident backup directly into the capture file.  
>  
> Reconstruct the display, unlock the point-of-sale ledger, and identify the **receipt ID** of the active transaction for the missing item at the exact moment this frame was captured.  


---

## 1. Initial Reconnaissance & Metadata Analysis

Opening `cam_frame_0730.jpg` in a standard image viewer reveals a grainy, green-tinted CCTV frame from `CAM_01 [ENTRANCE]` timestamped `2026-03-14 17:30:00`. 

Two visual details stand out immediately:
1. **Redacted Shelves:** All three rows of display shelves on the back wall are obscured by heavy multi-colored static/noise bars, hiding the item tags and the empty slot.
2. **Mirrored Feed:** The glowing green `EXIT` sign above the door on the right is horizontally reversed (`TIXE`), while the `CAM_01` OSD timestamp in the top-left corner reads normally.

We start by inspecting the file's EXIF metadata and binary structure using `exiftool` and `binwalk`:

```bash
exiftool cam_frame_0730.jpg
binwalk cam_frame_0730.jpg
```

* **`exiftool` Output:** Reveals `Image Description : Planogram: v3 (Exclusive)`.
* **`binwalk` Output:** Identifies a multi-part polyglot structure inside `cam_frame_0730.jpg`:
  1. **JPEG image data** at offset `0x0` (the redacted base camera feed).
  2. **PNG image data** (`1920 x 1080`) immediately following the JPEG End-of-Image (`FF D9`) marker.
  3. **Zip archive data** appended at the end of the PNG stream, containing `planogram_v3.png`, `notes.txt`, and `ledger.zip`.

---

## 2. Carving & Extracting the Embedded Files

We extract the appended ZIP archive and carve out the embedded PNG layer using `foremost` or `binwalk`:

```bash
# Extract all embedded files (JPEG, PNG, and ZIP)
foremost -i cam_frame_0730.jpg -o extracted_files

# Alternatively, using binwalk to extract the archive:
rm -rf extractions
binwalk -e cam_frame_0730.jpg
```

If carving the PNG manually via Python, we can slice the file cleanly from the PNG magic bytes (`\x89PNG\r\n\x1a\n`) to the end of its `IEND` chunk:

```python
with open("cam_frame_0730.jpg", "rb") as f:
    data = f.read()

png_start = data.find(b"\x89PNG\r\n\x1a\n")
png_end = data.find(b"IEND") + 8

with open("shelf_mask.png", "wb") as f:
    f.write(data[png_start:png_end])
```

Inspecting the extracted files from the ZIP archive gives us:
* **`notes.txt`:**
  > *"Camera clock runs +11 minutes ahead. The mirror is convex and fitted opposite the back wall. The planogram diagram shows every slot in reading order: left to right, top to bottom. It's an exclusive display—two halves that cancel each other out. The ledger is keyed to the item that went missing."*
* **`planogram_v3.png`:** A visual schematic of the `3 x 4` display shelf grid mapping `SLOT #01` through `SLOT #12` to their 4-character alphanumeric SKUs:
  * **Row 1 (Slots 01–04):** `A1F3`, `B7K2`, `C9D4`, `E0M8`
  * **Row 2 (Slots 05–08):** `H5J1`, `K3P6`, `Q8R2`, `T4W9`
  * **Row 3 (Slots 09–12):** `X2Y7`, `Z6N0`, `L1U5`, `G3V8`
* **`ledger.zip`:** An encrypted ZIP archive containing `pos_ledger.csv`.

---

## 3. De-Redacting the CCTV Feed via Bitwise XOR

Opening the carved `shelf_mask.png` on its own shows a mostly black `1920x1080` frame with three horizontal strips of random colored noise at the exact coordinates of the redacted shelves. Neither `cam_frame_0730.jpg` nor `shelf_mask.png` displays readable text individually.

However, both the EXIF description (`Planogram: v3 (Exclusive)`) and `notes.txt` (*"It's an exclusive display—two halves that cancel each other out"*) hint at an **Exclusive OR (XOR)** operation (`Unredacted_Scene XOR Redacted_JPG`).

To cancel out the noise and restore the shelves, we combine `cam_frame_0730.jpg` and `shelf_mask.png` using a **Bitwise XOR** operation.

### Method A: GUI (Stegsolve / GIMP / Photopea)
1. Open `cam_frame_0730.jpg` in **Stegsolve**.
2. Navigate to `Analyse` -> `Image Combiner` and select the extracted `shelf_mask.png`.
3. Cycle through the operations until reaching **XOR**—the static noise cancels out completely, revealing the plaques on the shelves.

### Method B: Python Script
We can also XOR the two images and horizontally flip the result in one step to correct for the convex mirror reflection mentioned in `notes.txt`:

```python
import numpy as np
from PIL import Image, ImageOps

# 1. Load the base JPEG and the carved PNG mask
jpg_arr = np.array(Image.open("cam_frame_0730.jpg").convert("RGB"))
mask_arr = np.array(Image.open("shelf_mask.png").convert("RGB"))

# 2. Bitwise XOR the two layers to cancel out the noise boxes
unredacted_arr = np.bitwise_xor(jpg_arr, mask_arr)
unredacted_img = Image.fromarray(unredacted_arr)
unredacted_img.save("unredacted_mirror.png")

# 3. Flip horizontally to undo the convex mirror reflection
true_wall_img = ImageOps.mirror(unredacted_img)
true_wall_img.save("solved_wall_true_orientation.png")
```

---

## 4. Identifying the Missing Item (Mirror Orientation Trap)

Looking at the unredacted camera feed (`unredacted_mirror.png`), the empty dark slot appears on the **middle shelf, 2nd column from the left**. 

However, `notes.txt` explicitly warns us:
> *"The mirror is convex and fitted opposite the back wall."*

Because we are looking at a mirror reflection (proven by the backwards `TIXE` sign and backwards text on the plaques), left and right are inverted! When we view `solved_wall_true_orientation.png` (flipped horizontally so `EXIT` and the plaques read left-to-right):

| Row / Col | Col 1 (Left) | Col 2 | Col 3 | Col 4 (Right) |
| :--- | :---: | :---: | :---: | :---: |
| **Top Shelf** | `A1F3` (Slot #01) | `B7K2` (Slot #02) | `C9D4` (Slot #03) | `E0M8` (Slot #04) |
| **Middle Shelf** | `H5J1` (Slot #05) | `K3P6` (Slot #06) | **`[EMPTY]` (Slot #07)** | `T4W9` (Slot #08) |
| **Bottom Shelf** | `X2Y7` (Slot #09) | `Z6N0` (Slot #10) | `L1U5` (Slot #11) | `G3V8` (Slot #12) |

Cross-referencing **Slot #07** (Row 2, Column 3 in true reading order) with `planogram_v3.png` reveals the missing SKU:
* **Missing SKU:** `Q8R2`

---

## 5. Unlocking the POS Ledger & Timeline Correlation

Using `Q8R2` as the password, we extract `ledger.zip`:

```bash
unzip -P Q8R2 ledger.zip
cat pos_ledger.csv
```

This gives us the Point-of-Sale transaction log (`pos_ledger.csv`):

```csv
receipt_id,timestamp,event,sku,ref
R-0301,2026-03-14 09:12:40,SCAN,K3P6,
R-0588,2026-03-14 15:30:05,SCAN,Q8R2,
R-0592,2026-03-14 15:45:10,REFUND,Q8R2,R-0588
R-0701,2026-03-14 16:58:22,SCAN,Q8R2,
R-0703,2026-03-14 17:05:40,VOID,Q8R2,R-0701
R-0744,2026-03-14 17:15:02,SCAN,K3P6,
R-0766,2026-03-14 17:17:40,SCAN,Q8R2,
R-0770,2026-03-14 17:20:10,VOID,Q8R2,R-0766
R-0771,2026-03-14 17:20:30,SCAN,Q8R2,
```

### Analyzing the Clock Drift & Transactions:
1. **Camera Timestamp:** The CCTV overlay reads `2026-03-14 17:30:00`.
2. **Clock Offset:** `notes.txt` states: *"Camera clock runs +11 minutes ahead."*
3. **Actual Real-World Time of Frame:**
   $$\text{17:30:00} - \text{11 minutes} = \text{17:19:00}$$

Now we trace the history of `Q8R2` in `pos_ledger.csv` at the exact real-world capture time (`17:19:00`):
* `15:30:05` (`R-0588`): `Q8R2` is scanned, but refunded at `15:45:10` (`R-0592`).
* `16:58:22` (`R-0701`): `Q8R2` is scanned, but voided at `17:05:40` (`R-0703`).
* **`17:17:40` (`R-0766`):** `Q8R2` is scanned 1 minute and 20 seconds prior to the `17:19:00` frame capture (and remains the active scan until it is later voided at `17:20:10` under `R-0770` and re-scanned at `17:20:30` under `R-0771`).

---

## Final Solution Summary

* **Missing Slot Index:** `7` (`SLOT #07`)
* **Missing Item SKU (Zip Password):** `Q8R2`
* **True Camera Capture Time:** `2026-03-14 17:19:00
* **Flag / Active Receipt ID:** `kernel{R-0766}` 
