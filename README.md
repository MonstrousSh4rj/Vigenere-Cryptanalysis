# Vigenere Cipher and Cryptanalysis

This project implements the Vigenere cipher in C++ (encryption and decryption, including file-based operation) along with a Python toolkit that breaks Vigenere-encrypted ciphertext without knowing the key in advance. The cryptanalysis tool recovers both the key length and the key itself using statistical methods, not brute force.

## Contents

- `Vigenere_Cipher.cpp` — encrypts and decrypts text or files using a Vigenere cipher
- `find_key_length.py` — scans candidate key lengths and scores each using Index of Coincidence
- `find_key_letters.py` — recovers the actual key letters using chi-squared frequency analysis
- (optional) a combined script that runs the full pipeline end to end

## Background: how the Vigenere cipher works

The Vigenere cipher shifts each letter of the plaintext by an amount determined by a repeating key. Each letter of the alphabet is assigned a number, A=0 through Z=25. To encrypt a letter:

```
cipher_number = (plain_number + key_number) mod 26
```

To decrypt:

```
plain_number = (cipher_number - key_number + 26) mod 26
```

The key is shorter than the message, so it repeats. For example, with the key `CAT` and the message `ATTACKATDAWN`, the key is applied as `CATCATCATCAT`, one key letter per plaintext letter.

Because the key repeats, Vigenere is really a set of interleaved Caesar ciphers, one Caesar cipher per key letter. This is the property the cryptanalysis tool exploits.

## Background: how the cryptanalysis works

The attacker only has the ciphertext. Two questions need to be answered in order: how long is the key, and what are the actual key letters.

### Step 1: Finding the key length with Index of Coincidence

If the key length is guessed correctly, splitting the ciphertext into that many groups (group 1 = every Nth letter starting at position 1, group 2 = every Nth letter starting at position 2, and so on) produces groups where every letter in a given group was shifted by the same key letter. Each group is then just a simple Caesar cipher.

The Index of Coincidence (IC) measures how likely two randomly chosen letters from a piece of text are the same letter. Real English text has a skewed letter distribution (E, T, A are common; Q, Z, X are rare), which gives it an IC around 0.065 to 0.070. Purely random letters give an IC around 0.038, since with 26 equally likely letters the chance of any two matching is 1/26.

A Caesar shift does not change this skew, it only relabels which letter is common. So if a guessed key length is correct, each resulting group still has an IC close to English's value. If the guessed key length is wrong, letters that were actually shifted by different key letters get mixed into the same group, which flattens the distribution and pulls the IC down toward the random baseline.

The formula used per group:

```
IC = sum over each letter of [ n * (n - 1) ]  /  ( N * (N - 1) )
```

where `n` is how many times a specific letter appears in the group and `N` is the total number of letters in the group.

The tool tries every candidate key length (1 through 15 in this project), computes the average IC across that length's groups, and looks for the smallest length that produces a strong spike in IC. Multiples of the true key length also produce elevated IC values, which is why the smallest spiking length is chosen rather than the single highest score.

### Step 2: Recovering each key letter with chi-squared

Once the key length is known, each group is a simple Caesar cipher with an unknown single shift (0 through 25). For each possible shift, the tool undoes that shift on the whole group and compares the resulting letter frequency distribution against the known English letter frequency table using the chi-squared statistic:

```
chi-squared = sum over each letter of [ (observed - expected)^2 / expected ]
```

Here `observed` is how many times a letter actually appears in the decrypted attempt, and `expected` is how many times it should appear if the text were real English, calculated as the English frequency percentage for that letter multiplied by the group length.

A low chi-squared score means the decrypted attempt's letter frequencies closely match real English. The tool tries all 26 shifts for a group and keeps the shift with the lowest score. That shift, converted back to a letter (shift 0 = A, shift 1 = B, and so on), is the key letter for that group.

This is repeated for every group, and the recovered key letters are read off in order to reconstruct the full key.

## Usage

### Encrypting and decrypting (C++)

Compile and run `Vigenere_Cipher.cpp`. The program presents a menu to encrypt or decrypt either typed text or a file, given a key consisting only of letters.

### Breaking an unknown key (Python)

Run the key length finder first:

```
python3 find_key_length.py
```

Paste in the ciphertext when prompted. The script prints the average IC for every key length from 1 to 15 and reports its best guess.

Then run the key recovery script using the identified length to recover the actual key letters and decrypt the message.

## Notes on this implementation

The `sequence_file` function in the C++ program advances the key index for every character it processes, including spaces, rather than only advancing on letters. This is a valid design choice and encryption and decryption remain consistent with each other, but it means the cryptanalysis grouping must match this behavior. Two grouping schemes are possible when breaking a Vigenere ciphertext:

- Group by letter-only position, where the key only advances on actual letters and spaces are skipped entirely
- Group by raw character position, where the key advances on every character including spaces

Which scheme is correct depends on how the ciphertext was originally generated. Using the wrong scheme will produce a flat, noisy IC scan with no clear key length, even though the underlying cipher is not broken, only misidentified. This project's cryptanalysis scripts assume one specific scheme; if analyzing ciphertext from a different Vigenere implementation, confirm which scheme it uses before trusting the results.

## Requirements

- A C++ compiler (g++ or equivalent) for the cipher program
- Python 3 for the cryptanalysis scripts, no external libraries required beyond the standard library
