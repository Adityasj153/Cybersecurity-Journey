# 🚩 Binary Digits — CTF Write-up

## Challenge Information

| | |
|---|---|
| **Platform** | CyLabs |
| **Event** | picoCTF 2026 |
| **Challenge** | Binary Digits |
| **Category** | Forensics |
| **Difficulty** | Easy |

## 📜 Challenge Description

> This file doesn't look like much... just a bunch of 1s and 0s. But maybe it's not just random noise. Can you recover anything meaningful from this?

## 🎯 Objective

Recover the hidden meaningful data from the provided `digits.bin` file and find the flag.

## 🔎 Step 1 — Inspecting the File

The downloaded file was `digits.bin`. It contained a very large sequence of `0`s and `1`s.

The challenge description suggested that the data might not be random noise. Since the input consisted only of binary digits, my first hypothesis was that the sequence represented encoded bytes.

Because **8 bits make 1 byte**, I decided to decode the data in 8-bit groups.

## 🧑‍🍳 Step 2 — Using CyberChef

I loaded `digits.bin` into CyberChef and used **From Binary**.

Settings:

```text
From Binary
├── Delimiter: Space
└── Byte Length: 8
```

The important setting here is **Byte Length = 8**, because a byte contains eight bits.

![CyberChef Recipe](screenshots/cyberchef.png)

## 🖼️ Step 3 — The Output Wasn't Readable Text

The decoded output was not normal readable text.

Instead of assuming the decoding had failed, I considered that the resulting bytes could represent another type of data, such as an image.

This is an important CTF lesson: **decoded data does not always have to be text.**

## 🎨 Step 4 — Rendering the Data as an Image

I added **Render Image** to the CyberChef recipe and selected:

```text
Input Format: Raw
```

The final recipe was:

```text
From Binary
    ↓
Render Image
```

CyberChef successfully rendered the decoded data as an image.

![Rendered Image](screenshots/flag.png)

## 🚩 Step 5 — Finding the Flag

The rendered image contained the flag:

```text
picoCTF{h1dd3n_1n_th3_b1n4ry_2f96e9a1}
```

## 🧠 What I Learned

This was my **first CTF**, and the biggest lesson was not simply finding the flag.

I learned to:

- Read the challenge description for clues.
- Form a simple hypothesis before trying random tools.
- Understand that 8 bits make one byte.
- Use CyberChef to quickly test data transformations.
- Remember that decoded data can be an image or another file, not just readable text.
- Analyze the output and adapt the approach when the first result is not obvious.

## 🛠️ Tools Used

- CyberChef
- From Binary
- Render Image

## 💡 Final Takeaway

This challenge taught me a simple but important CTF workflow:

```text
Observe
   ↓
Form a hypothesis
   ↓
Test it
   ↓
Analyze the output
   ↓
Try the next logical interpretation
   ↓
Find the flag
```

My first CTF showed me that cybersecurity is not just about knowing tools. It is about **understanding what the data is telling you.**

---

### 🚀 Learning Progress

**CTF #001 — Binary Digits ✅**

> Learn → Practice → Solve → Document → Build
