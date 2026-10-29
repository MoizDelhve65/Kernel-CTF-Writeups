# Challenge Title: Bio-Digital Transmission

## Problem Description

We intercepted a strange audio signal. Intelligence suggests this single file contains both a hidden message and the key needed to decrypt it, but the audio sounds like pure noise. Can you extract the data, find the key, and decipher the flag?

**Assets provided:**

* `transmission.wav` (A single audio file)

## Hint

*“A biological string can't pass the gate.”*

## Solution / Writeup

### Step 1: Metadata Extraction (Finding the Ciphertext)

Since we are only given a `.wav` file, the first instinct should be to check for hidden data within the file itself before analyzing the audio.

1. Use a tool like **ExifTool** via command line (`exiftool transmission.wav`), or simply right-click the file and view its properties/details.

2. Looking through the metadata tags (such as Artist, Title, or Comment), you will discover a strange, seemingly random string of text.

3. Note this string down—this is our encrypted flag (ciphertext), and it appears to be encrypted using a Vigenère cipher. Now we need the key.

### Step 2: Audio Analysis (Extracting the Biological String)

Next, we need to analyze the actual audio of `transmission.wav`. Listening to it directly doesn't reveal much, which is a common indicator that data is hidden visually in the audio spectrum.

1. Open the audio file using **Audacity** (or any similar audio analysis tool).

2. Click on the track name dropdown on the left and select **Spectrogram** view.

3. You will see distinct tones stepping between four specific frequency bands.

4. By analyzing the frequencies (or recognizing biological themes), map the four distinct frequency levels to the four DNA bases: **A, G, T, C**.

5. Reading the tones chronologically, we extract the following 16-character DNA sequence:
   `GAGGGGTAGACCGACT`

### Step 3: The Translation (Applying the Hint)

We now have a biological string: `GAGGGGTAGACCGACT`. However, this string doesn't work as a Vigenère key on its own.

This is where the hint comes into play: *“A biological string can't pass the gate.”*
Computers (logic gates) don't process biological sequences; they process binary. We need to convert this entire DNA sequence into digital data (0s and 1s). Since there are four bases, each base can be represented by a 2-bit state (from `00` to `11`).

Using the standard ascending mapping from the lowest frequency tone to the highest:

* **A** = `00`
* **G** = `01`
* **T** = `10`
* **C** = `11`

First, let's translate the whole biological string into a continuous binary sequence:

* DNA: `G A G G G G T A G A C C G A C T`
* Binary: `01 00 01 01 01 01 10 00 01 00 11 11 01 00 11 10`

Full binary string:
`01000101010110000100111101001110`

### Step 4: Grouping for ASCII (Finding the Key)

To read this binary as standard text, we need to convert it into ASCII format. Standard ASCII characters are 8 bits (1 byte) long. 

Because each DNA base equals 2 bits, grouping the sequence into sets of **4 bases** gives us exactly the 8-bit bytes we need. Let's group our continuous binary string into 8-bit chunks and translate them using **CyberChef** (or any binary-to-text converter):

* `01000101` (G A G G) = **E**
* `01011000` (G G T A) = **X**
* `01001111` (G A C C) = **O**
* `01001110` (G A C T) = **N**

The extracted secret key is **EXON** (a fittingly biological term for a coding sequence of DNA).

### Step 5: Decrypting the Flag

With both the ciphertext (from the metadata) and the key (from the spectrogram) in hand, the final step is decryption.

1. Open **CyberChef**.

2. Drag the **Vigenère Decode** operation into the recipe.

3. Input the ciphertext found in Step 1 into the main input box.

4. Set the key to `EXON`.

5. The output will reveal the decrypted flag!

**Flag:** `flag{...}`