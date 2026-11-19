# intro-crypto — practice challenge

A tiny, self-contained challenge to confirm your agent setup works end to end.
No network or target host required.

## Run it
```bash
cp -r challenges/intro-crypto ~/ctf/intro-crypto   # or copy the folder however you like
cd ~/ctf/intro-crypto
claude
```
Then tell the agent:
```
Solve the challenge in this folder. The flag format is flag{...}.
```
The agent should read challenge.md, inspect cipher.txt, peel back the encoding
layers, and return the flag.

<details>
<summary>Spoiler: how it's built / expected flag</summary>

`cipher.txt` is the flag passed through three layers: **ROT13 → Base64 → hex**.
Reverse them (hex-decode, Base64-decode, ROT13) to recover:

```
flag{w3lc0me_to_cl4ude_ctf}
```
</details>
