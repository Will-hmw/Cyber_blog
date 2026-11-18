---
title: "Hardware Reverse Engineering: Exploiting a Backdoor in a TPM Chip (VHDL)"
date: 2026-11-18
draft: false
translationKey: "hardware-backdoor-tpm"
description: "A step-by-step guide to reading VHDL, mapping a digital circuit, and exploiting a hardware backdoor hidden inside an encryption chip (TPM) that leaks its key on a secret input."
tags: ["cybersecurity", "hardware", "reverse-engineering", "vhdl", "hack-the-box"]
---

# 🔓 Hardware Reverse Engineering: The Hidden Backdoor in a TPM

> **Level:** Beginner in hardware reverse engineering
> **Language:** VHDL
> **Goal:** Read the description of a digital circuit, spot a backdoor, and exploit it to leak an encryption key

---

## 📋 Table of Contents

- [🔓 Hardware Reverse Engineering: The Hidden Backdoor in a TPM](#-hardware-reverse-engineering-the-hidden-backdoor-in-a-tpm)
  - [📋 Table of Contents](#-table-of-contents)
  - [🎯 The Context](#-the-context)
  - [🧠 Reading VHDL in 5 Minutes](#-reading-vhdl-in-5-minutes)
    - [What a VHDL File Describes](#what-a-vhdl-file-describes)
    - [Quick Glossary](#quick-glossary)
  - [🔍 Mapping the Files](#-mapping-the-files)
  - [🕵️ Step 1 — Spotting the Backdoor](#️-step-1--spotting-the-backdoor)
  - [🧩 Step 2 — Understanding the Top-Level Module](#-step-2--understanding-the-top-level-module)
  - [🎛️ Step 3 — Talking to the Remote Service](#️-step-3--talking-to-the-remote-service)
  - [🏁 Result](#-result)
  - [🗺️ Visual Recap](#️-visual-recap)
  - [🚀 Going Further](#-going-further)

---

## 🎯 The Context

The challenge ships several `.vhdl` files describing a **TPM**-like chip (Trusted Platform Module, a component meant to protect an encryption key). The goal: find a flaw deliberately planted in the chip's logic, and exploit it against a network service emulating that circuit.

Unlike a classic software exploit, you're not reading code that *executes* line by line: you're reading the description of an **electronic circuit**, where each block corresponds to a piece of silicon wired once and for all.

---

## 🧠 Reading VHDL in 5 Minutes

### What a VHDL File Describes

A VHDL file always has two parts:

**`entity`** — the outside of the box (its wires):

```vhdl
entity backdoor is
    Port (
        D : in  STD_LOGIC_VECTOR(15 downto 0);
        B : out STD_LOGIC
    );
end backdoor;
```

- `in` / `out`: a wire's direction
- `STD_LOGIC`: a single bit (`'0'` or `'1'`)
- `STD_LOGIC_VECTOR(15 downto 0)`: a 16-bit bus

**`architecture`** — the inside of the box (what it does):

```vhdl
architecture Behavioral of backdoor is
    constant pattern : STD_LOGIC_VECTOR(15 downto 0) := "1111111111101001";
begin
    process(D)
    begin
        if D = pattern then
            B <= '1';
        else
            B <= '0';
        end if;
    end process;
end Behavioral;
```

### Quick Glossary

| Term | Meaning |
|---|---|
| `signal` | an internal wire |
| `<=` | assignment to a signal (not a regular variable) |
| `:=` | initial value of a constant |
| `process(X)` | a block re-evaluated whenever `X` changes |
| `port map` | wiring a sub-block into the circuit |
| `case ... when ... when others` | a switch based on a signal's value |
| `others` | "every other case" |

---

## 🔍 Mapping the Files

Before reading everything, map the structure with `grep`:

```bash
grep -n "entity\|port\|if \|when \|case " *.vhdl
```

```
backdoor.vhdl:5:entity backdoor is
backdoor.vhdl:17:        if D = pattern then
encryption.vhdl:5:entity encryption is
key.vhdl:5:entity ckey is
tpm.vhdl:5:entity tpm is
tpm.vhdl:38:    ck : ckey port map(Key);
tpm.vhdl:39:    enc: encryption port map (Data, Key, Encrypted);
tpm.vhdl:40:    bd: backdoor port map (Data, B);
tpm.vhdl:44:            case B is
```

Four files, four roles:

| File | Role |
|---|---|
| `tpm.vhdl` | **Top-level** module: wires the other sub-blocks together |
| `key.vhdl` (`ckey`) | Supplies the encryption key |
| `encryption.vhdl` | Encrypts the data with the key |
| `backdoor.vhdl` | Compares the input to a secret pattern |

---

## 🕵️ Step 1 — Spotting the Backdoor

```vhdl
entity backdoor is
    Port (
        D : in STD_LOGIC_VECTOR(15 downto 0);
        B : out STD_LOGIC
    );
end backdoor;

architecture Behavioral of backdoor is
    constant pattern : STD_LOGIC_VECTOR(15 downto 0) := "1111111111101001";
begin
    process(D)
    begin
        if D = pattern then
            B <= '1';
        else
            B <= '0';
        end if;
    end process;
end Behavioral;
```

This module continuously compares the 16-bit input `D` against a constant `pattern`. The moment they match, output `B` flips to `'1'`. It's a hardcoded password detector, invisible in the circuit's "normal" flow — exactly like the never-called `win()` function in a software ret2win.

---

## 🧩 Step 2 — Understanding the Top-Level Module

```vhdl
entity tpm is
    Port (
        Data   : in  STD_LOGIC_VECTOR(15 downto 0);
        Output : out STD_LOGIC_VECTOR(15 downto 0)
    );
end tpm;

architecture Behavioral of tpm is
    signal Key, Encrypted : STD_LOGIC_VECTOR(15 downto 0);
    signal B : STD_LOGIC;
begin
    ck  : ckey       port map(Key);
    enc : encryption port map (Data, Key, Encrypted);
    bd  : backdoor   port map (Data, B);

    process(Key, Encrypted, B)
    begin
        case B is
            when '1' =>
                for i in 0 to 15 loop
                    Output(i) <= Key(i);
                end loop;
            when others =>
                for i in 0 to 15 loop
                    Output(i) <= Encrypted(i);
                end loop;
        end case;
    end process;
end Behavioral;
```

The wiring feeds `Data` into three sub-blocks in parallel: key generation, encryption, and the backdoor detector. The final output depends on a simple switch:

| Value of `B` | `Output` receives |
|---|---|
| `'0'` (normal case) | `Encrypted` — the encrypted data |
| `'1'` (backdoor triggered) | `Key` — **the key in plaintext** |

In other words: sending any data returns useless ciphertext, but sending exactly `pattern` leaks the encryption key itself. The encryption then serves no purpose at all.

---

## 🎛️ Step 3 — Talking to the Remote Service

```bash
nc <IP> <PORT>
```

```
The input must be a binary signal of 16 bits.

Input :
```

The service expects the raw 16 bits in binary, no conversion needed: just send the `pattern` constant as-is.

```
Input : 1111111111101001
```

---

## 🏁 Result

```
Output: 0110001111100001
You triggered the backdoor here is the flag: HTB{4_7yp1c41_53cu23_TPM_ch1p}
```

The server confirms the backdoor fired, returns the leaked key, and hands over the flag.

---

## 🗺️ Visual Recap

```
┌──────────────────────────────────────────────────────────┐
│  1. grep across the .vhdl files                            │
│     → spot entity / port map / case                        │
├──────────────────────────────────────────────────────────┤
│  2. Read backdoor.vhdl                                      │
│     → find the "pattern" constant (16 bits)                 │
├──────────────────────────────────────────────────────────┤
│  3. Read tpm.vhdl                                             │
│     → understand the Key / Encrypted switch driven by B      │
├──────────────────────────────────────────────────────────┤
│  4. nc <IP> <PORT>                                            │
│     → send "pattern" as a 16-bit binary string               │
├──────────────────────────────────────────────────────────┤
│  5. The circuit flips B to '1'                                 │
│     → Output = Key (the key leaks)                             │
├──────────────────────────────────────────────────────────┤
│  6. The server confirms the trigger                              │
│     → flag captured 🚩                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 🚀 Going Further

- **Real hardware backdoors**: unlike a software bug, a backdoor etched into silicon can't be patched with an update — only a new manufacturing batch can fix it.
- **TPMs and hardware trust**: a real TPM is supposed to guarantee that a key never leaves the chip in plaintext; this challenge shows how a single misplaced condition in the logic can void that entire guarantee.
- **Detection**: in a real hardware security audit, this kind of suspicious comparison is hunted down with formal verification or netlist analysis tools, not just by eyeballing the code.
- **Parallel with software exploitation**: the logic mirrors a `ret2win` — a hidden feature, never called during normal flow, that triggers on one specific input.

---

> 📝 **In one sentence:** a simple `if D = pattern` comparison wired into a circuit is enough to leak an encryption key that was supposed to stay protected — proof that a hardware backdoor can be just as trivial, and just as dangerous, as a software one.
