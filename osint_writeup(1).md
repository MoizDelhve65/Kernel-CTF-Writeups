# Problem Title: Race you to the finish line!
## DeliverHellVeil
## OSINT
## Medium

### Extract the 11, and smash that like button!
### (Instagram Handle: [First word of the group] + [Last word of the track] + [When it arrives]. Format: @word_word_number)
### Eleven lines unlock the gate; six words inside each verse.



# Solution:

## 1. Decoding the Base64

The challenge provides eleven lines, each containing six words. The hint states:

> Eleven lines unlock the gate; six words inside each verse.

Each word is classified as either positive (`1`) or negative (`0`). Using this rule, every line is converted into a six-bit binary number.

- **Positive words = `1`**
- **Negative words = `0`**

### Binary extraction

| Line | Binary |
|---|---|
| Life breaks; light heals, hope shines. | `101111` |
| Death kills; rot destroys dark joy. | `000001` |
| Hate ruins; fear crushes dead ash. | `000000` |
| Love weeps; pain mends, peace blooms. | `100111` |
| Joy fades; builders build ruined graves. | `101100` |
| Pain heals; light saves sweet decay. | `011110` |
| Truth wins; love suffers grim death. | `111000` |
| Light grows; warmth saves freezing life. | `111101` |
| Hate creates foul rot, heroes die. | `010010` |
| Hope fails; builders save bright ruins. | `101110` |
| Love wins; truth kills vile liars. | `111000` |

Combining the eleven six-bit groups gives:

```text
101111 000001 000000 100111 101100 011110
111000 111101 010010 101110 111000
```

Each six-bit group corresponds to a Base64 character. Converting the groups using the standard Base64 alphabet produces:

```text
vBAnse49Su4
```

## 2. Finding the YouTube Video

The clue **“smash that like button”** points to YouTube.

Using the decoded string as the video ID gives the following URL:

https://www.youtube.com/watch?v=vBAnse49Su4

The next clue is **“Race you to the finish line!”**, which suggests looking at the end of the video or its tracklist.

## 3. Identifying the Finnish Band

The video’s tracklist contains a Finnish band:

- **Artist:** Scarlet Youth
- **Track:** Gleaming Endless Ocean
- **Country:** Finland
- **Timestamp:** 36:38

The challenge's Instagram hint specifies the following format:

```text
@word_word_number
```

It instructs us to combine the first word of the group, the last word of the track, and the time when it arrives.

Applying these clues:

- First word of the group: `scarlet`
- Last word of the track: `ocean`
- Timestamp: `3638`

This produces the Instagram handle:

```text
@scarlet_ocean_3638
```

## 4. Retrieving the Flag

The Instagram handle is the next lead in the challenge. Following it leads to a Pastebin link containing the flag.

The recovered flag is:

```text
kernel{g3t_l0st_in_th3_w4ll_0f_gu1t4rs}
```

## Final Answer

**Flag:**

`kernel{g3t_l0st_in_th3_w4ll_0f_gu1t4rs}`
