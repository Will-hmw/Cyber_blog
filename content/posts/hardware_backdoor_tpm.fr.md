---
title: "Rétro-ingénierie matérielle : exploiter une backdoor dans une puce TPM (VHDL)"
date: 2026-11-18
draft: false
translationKey: "hardware-backdoor-tpm"
description: "Guide pas à pas pour lire du VHDL, cartographier un circuit numérique et exploiter une backdoor matérielle cachée dans une puce de chiffrement (TPM) qui fait fuiter sa clé sur une entrée secrète."
tags: ["cybersécurité", "hardware", "reverse-engineering", "vhdl", "hack-the-box"]
---

# 🔓 Rétro-ingénierie matérielle : la backdoor cachée dans un TPM

> **Niveau :** Débutant en rétro-ingénierie matérielle
> **Langage :** VHDL
> **Objectif :** Lire la description d'un circuit numérique, repérer une backdoor et l'exploiter pour faire fuiter une clé de chiffrement

---

## 📋 Table des matières

- [🔓 Rétro-ingénierie matérielle : la backdoor cachée dans un TPM](#-rétro-ingénierie-matérielle--la-backdoor-cachée-dans-un-tpm)
  - [📋 Table des matières](#-table-des-matières)
  - [🎯 Le contexte](#-le-contexte)
  - [🧠 Lire du VHDL en 5 minutes](#-lire-du-vhdl-en-5-minutes)
    - [Ce que décrit un fichier VHDL](#ce-que-décrit-un-fichier-vhdl)
    - [Lexique rapide](#lexique-rapide)
  - [🔍 Cartographier les fichiers](#-cartographier-les-fichiers)
  - [🕵️ Étape 1 — Repérer la backdoor](#️-étape-1--repérer-la-backdoor)
  - [🧩 Étape 2 — Comprendre le module principal](#-étape-2--comprendre-le-module-principal)
  - [🎛️ Étape 3 — Interagir avec le service distant](#️-étape-3--interagir-avec-le-service-distant)
  - [🏁 Résultat](#-résultat)
  - [🗺️ Récapitulatif visuel](#️-récapitulatif-visuel)
  - [🚀 Pour aller plus loin](#-pour-aller-plus-loin)

---

## 🎯 Le contexte

Le challenge fournit plusieurs fichiers `.vhdl` décrivant une puce de type **TPM** (Trusted Platform Module, un composant censé protéger une clé de chiffrement). Le but : trouver une faille volontairement implantée dans la logique de la puce, et l'exploiter contre un service réseau qui émule ce circuit.

Contrairement à un exploit logiciel classique, on ne lit pas du code qui *s'exécute* ligne par ligne : on lit la description d'un **circuit électronique**, où chaque bloc correspond à une portion de silicium câblée une fois pour toutes.

---

## 🧠 Lire du VHDL en 5 minutes

### Ce que décrit un fichier VHDL

Un fichier VHDL comporte toujours deux parties :

**`entity`** — l'extérieur de la boîte (ses fils) :

```vhdl
entity backdoor is
    Port (
        D : in  STD_LOGIC_VECTOR(15 downto 0);
        B : out STD_LOGIC
    );
end backdoor;
```

- `in` / `out` : direction d'un fil
- `STD_LOGIC` : un seul bit (`'0'` ou `'1'`)
- `STD_LOGIC_VECTOR(15 downto 0)` : un bus de 16 bits

**`architecture`** — l'intérieur de la boîte (ce qu'elle fait) :

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

### Lexique rapide

| Mot | Sens |
|---|---|
| `signal` | un fil interne |
| `<=` | affectation d'un signal (pas une variable classique) |
| `:=` | valeur initiale d'une constante |
| `process(X)` | bloc réévalué à chaque changement de `X` |
| `port map` | branchement d'une sous-boîte dans le circuit |
| `case ... when ... when others` | un aiguillage selon la valeur d'un signal |
| `others` | « tous les autres cas » |

---

## 🔍 Cartographier les fichiers

Avant de tout lire, on repère la structure avec `grep` :

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

Quatre fichiers, quatre rôles :

| Fichier | Rôle |
|---|---|
| `tpm.vhdl` | Module **principal** : assemble les autres sous-boîtes |
| `key.vhdl` (`ckey`) | Fournit la clé de chiffrement |
| `encryption.vhdl` | Chiffre la donnée avec la clé |
| `backdoor.vhdl` | Compare l'entrée à un motif secret |

---

## 🕵️ Étape 1 — Repérer la backdoor

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

Ce module compare en permanence l'entrée `D` (16 bits) à une constante `pattern`. Dès qu'elles sont identiques, la sortie `B` passe à `'1'`. C'est un détecteur de mot de passe câblé en dur, invisible dans le flot « normal » du circuit — exactement comme la fonction `win()` jamais appelée dans un ret2win logiciel.

---

## 🧩 Étape 2 — Comprendre le module principal

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

Le câblage branche `Data` dans trois sous-boîtes en parallèle : la génération de clé, le chiffrement, et le détecteur de backdoor. La sortie finale dépend d'un simple aiguillage :

| Valeur de `B` | `Output` reçoit |
|---|---|
| `'0'` (cas normal) | `Encrypted` — la donnée chiffrée |
| `'1'` (backdoor déclenchée) | `Key` — **la clé en clair** |

Autrement dit : envoyer n'importe quelle donnée renvoie du chiffré inutile, mais envoyer exactement `pattern` fait fuiter la clé de chiffrement elle-même. Le chiffrement n'a alors plus aucune utilité.

---

## 🎛️ Étape 3 — Interagir avec le service distant

```bash
nc <IP> <PORT>
```

```
The input must be a binary signal of 16 bits.

Input :
```

Le service attend directement les 16 bits en binaire, sans conversion : il suffit de recopier la constante `pattern` telle quelle.

```
Input : 1111111111101001
```

---

## 🏁 Résultat

```
Output: 0110001111100001
You triggered the backdoor here is the flag: HTB{4_7yp1c41_53cu23_TPM_ch1p}
```

Le serveur confirme le déclenchement de la backdoor, renvoie la clé qui a fuité, et livre le flag.

---

## 🗺️ Récapitulatif visuel

```
┌──────────────────────────────────────────────────────────┐
│  1. grep sur les .vhdl                                     │
│     → repérer entity / port map / case                     │
├──────────────────────────────────────────────────────────┤
│  2. Lire backdoor.vhdl                                      │
│     → trouver la constante "pattern" (16 bits)              │
├──────────────────────────────────────────────────────────┤
│  3. Lire tpm.vhdl                                            │
│     → comprendre l'aiguillage Key / Encrypted selon B        │
├──────────────────────────────────────────────────────────┤
│  4. nc <IP> <PORT>                                           │
│     → envoyer "pattern" en binaire sur 16 bits               │
├──────────────────────────────────────────────────────────┤
│  5. Le circuit bascule B à '1'                                │
│     → Output = Key (la clé fuite)                             │
├──────────────────────────────────────────────────────────┤
│  6. Le serveur valide le déclenchement                         │
│     → flag récupéré 🚩                                        │
└──────────────────────────────────────────────────────────┘
```

---

## 🚀 Pour aller plus loin

- **Backdoors matérielles réelles** : contrairement à un bug logiciel, une porte dérobée gravée dans le silicium ne peut pas être corrigée par une mise à jour — seul un nouveau lot de fabrication le peut.
- **TPM et confiance matérielle** : un vrai TPM est censé garantir qu'une clé ne quitte jamais la puce en clair ; ce challenge illustre à quel point une seule condition mal placée dans la logique peut annuler toute cette garantie.
- **Détection** : dans un vrai audit de sécurité matérielle, on cherche ce genre de comparaison suspecte avec des outils de vérification formelle ou d'analyse de netlist, pas seulement à l'œil nu.
- **Comparaison avec l'exploitation logicielle** : la logique est la même que pour un `ret2win` — une fonctionnalité cachée, jamais appelée dans le flot normal, qui se déclenche sur une entrée précise.

---

> 📝 **En une phrase :** une simple comparaison `if D = pattern` câblée dans un circuit suffit à faire fuiter une clé de chiffrement censée rester protégée — la preuve qu'une backdoor matérielle peut être aussi triviale, et aussi dangereuse, qu'une backdoor logicielle.
