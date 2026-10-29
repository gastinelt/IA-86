# Fiche 10 — Exemples assembleur IA-64, encodage binaire manuel et coût réel des templates

**Source unique de vérité :** *Intel Itanium Architecture Software Developer's Manual*, Révision 2.3 :
- **Volume 1** (*Application Architecture*) : Chapitre 3 (modèle d'exécution, bundles, prédication, RSE, boucles pipelinées, p. `1:31`–`1:80`).
- **Volume 3** (*Instruction Set Reference*) : Chapitre 3 (dépendances d'instructions et stops obligatoires, Table 3-10 p. `3:25`), Chapitre 4 (*Instruction Formats*, formats 41 bits et formation des immédiats, p. `3:293`–`3:369`).

> **Statut de vérification :**
> - ✅ **Calculé et vérifié bit à bit** d'après les diagrammes officiels du Volume 3 Chapitre 4 (p. `3:293`–`3:369`, inspectés visuellement sur le PDF original).
> - ⚠️ **Non testé sur un assembleur/désassembleur IA-64 externe (`gas`/`objdump`)** : aucun binaire externe n'étant installé sur le poste, les mots de 41 bits et de 128 bits ci-dessous sont produits par application directe des formules de champs du manuel Intel.

---

## 1. Règles d'encodage binaire d'un bundle 128 bits et des formats 41 bits utilisés

### 1.1 Empaquetage d'un bundle de 128 bits en mémoire (petit-boutiste, p. `1:38`, `3:293`)

Un bundle IA-64 occupe **16 octets (128 bits) alignés sur 16 octets** :

$$\text{Bundle}_{128} = (\text{Slot}_2 \ll 87) \;\mid\; (\text{Slot}_1 \ll 46) \;\mid\; (\text{Slot}_0 \ll 5) \;\mid\; \text{Template}_{5}$$

| Champ | Bits du bundle | Largeur | Chevauchement d'octets en mémoire (little-endian) |
|---|---|---|---|
| `template` | `4:0` | 5 bits | Octet `0` (bits `4:0`) |
| `Slot 0` | `45:5` | 41 bits | Octet `0` (bits `7:5`) jusqu'à l'octet `5` (bits `5:0`) |
| `Slot 1` | `86:46` | 41 bits | Octet `5` (bits `7:6`) jusqu'à l'octet `10` (bits `6:0`) |
| `Slot 2` | `127:87` | 41 bits | Octet `10` (bit `7`) jusqu'à l'octet `15` (bits `7:0`) |

### 1.2 Formats d'instructions de 41 bits utilisés dans nos 3 programmes (Vol. 3 Chap. 4)

| Format | Unité / Slots | Instruction | Disposition exacte des 41 bits (`40:0`) | Réf. Vol. 3 |
|---|---|---|---|---|
| **`A1`** | A (`M` ou `I`) | `add` / `sub r1 = r2, r3` | `op=8 (40:37) \| 0 (36:33) \| x4 (32:29) \| x2b (28:27) \| r3 (26:20) \| r2 (19:13) \| r1 (12:6) \| qp (5:0)` (`add`: `0,0` ; `sub`: `1,1`) | p. `3:301` |
| **`A4`** | A (`M` ou `I`) | `adds r1 = imm14, r3` | `op=8 (40:37) \| s (36) \| x2a=2 (35:34) \| 0 (33) \| imm6d (32:27) \| r3 (26:20) \| imm7b (19:13) \| r1 (12:6) \| qp (5:0)` (`mov r1 = r3` est `adds r1 = 0, r3`) | p. `3:302`, `3:368` |
| **`A6`** | A (`M` ou `I`) | `cmp.lt` / `cmp.ltu p1, p2 = r2, r3` | `op (40:37 : C=lt, D=ltu) \| 0 (36:33) \| p2 (32:27) \| r3 (26:20) \| r2 (19:13) \| c (12) \| p1 (11:6) \| qp (5:0)` | p. `3:304` |
| **`I18`** | I (`I`) | `nop.i imm21` | `op=0 (40:37) \| i (36) \| 0 (35:33) \| x6=0x01 (32:27) \| 0 (26) \| imm20a (25:6) \| qp (5:0)` (`nop.i 0` = `0x00008000000`) | p. `3:319` |
| **`I21`** | I (`I`) | `mov b1 = r2` | `op=0 (40:37) \| 0 (36) \| x3=7 (35:33) \| 0 (32:22) \| wh=1 (21:20) \| r2 (19:13) \| 0 (12:9) \| b1 (8:6) \| qp (5:0)` | p. `3:320` |
| **`I22`** | I (`I`) | `mov r1 = b2` | `op=0 (40:37) \| 0 (36:33) \| x6=0x31 (32:27) \| 0 (26:16) \| b2 (15:13) \| r1 (12:6) \| qp (5:0)` | p. `3:321` |
| **`I24`** | I (`I`) | `mov pr.rot = imm44` | `op=0 (40:37) \| s (36) \| x3=2 (35:33) \| imm27a (32:6) \| qp (5:0)` où `imm44 = sign_ext(s<<43 \| imm27a<<16, 44)` | p. `3:321`, `3:369` |
| **`I26`** | I (`I`) | `mov.i ar3 = r2` | `op=0 (40:37) \| 0 (36:33) \| x6=0x2A (32:27) \| ar3 (26:20) \| r2 (19:13) \| 0 (12:6) \| qp (5:0)` (`PFS=64, LC=65, EC=66`) | p. `3:322` |
| **`I27`** | I (`I`) | `mov.i ar3 = imm8` | `op=0 (40:37) \| s (36) \| 0 (35:33) \| x6=0x0A (32:27) \| ar3 (26:20) \| imm7b (19:13) \| 0 (12:6) \| qp (5:0)` | p. `3:322` |
| **`I28`** | I (`I`) | `mov.i r1 = ar3` | `op=0 (40:37) \| 0 (36:33) \| x6=0x32 (32:27) \| ar3 (26:20) \| 0 (19:13) \| r1 (12:6) \| qp (5:0)` | p. `3:322` |
| **`M3`** | M (`M`) | `ld8 r1 = [r3], imm9` | `op=5 (40:37) \| s (36) \| x6=0x03 (35:30) \| hint=0 (29:28) \| i (27) \| r3 (26:20) \| imm7b (19:13) \| r1 (12:6) \| qp (5:0)` | p. `3:331`, `3:369` |
| **`M5`** | M (`M`) | `st8 [r3] = r2, imm9` | `op=5 (40:37) \| s (36) \| x6=0x33 (35:30) \| hint=0 (29:28) \| i (27) \| r3 (26:20) \| r2 (19:13) \| imm7a (12:6) \| qp (5:0)` | p. `3:332`, `3:369` |
| **`M34`** | M (`M` Slot 0) | `alloc r1 = ar.pfs, i, l, o, r` | `op=1 (40:37) \| 0 (36) \| x3=6 (35:33) \| 0 (32:31) \| sor (30:27, r>>3) \| sol (26:20, i+l) \| sof (19:13, i+l+o) \| r1 (12:6) \| qp=0 (5:0)` | p. `3:344`, `3:369` |
| **`M48`** | M (`M`) | `nop.m imm21` | `op=0 (40:37) \| i (36) \| 0 (35:31) \| x4=1 (30:27) \| 0 (26) \| imm20a (25:6) \| qp (5:0)` (`nop.m 0` = `0x00008000000`) | p. `3:349` |
| **`B2`** | B (`B`) | `br.cloop` / `br.ctop target25` | `op=4 (40:37) \| s (36) \| d (35) \| wh (34:33) \| imm20b (32:13) \| p (12) \| 0 (11:9) \| btype (8:6 : 5=cloop, 7=ctop) \| 0 (5:0)` | p. `3:353`, `3:369` |
| **`B3`** | B (`B`) | `br.call b1 = target25` | `op=5 (40:37) \| s (36) \| d (35) \| wh (34:33) \| imm20b (32:13) \| p (12) \| 0 (11:9) \| b1 (8:6) \| qp (5:0)` | p. `3:353` |
| **`B4`** | B (`B`) | `br.ret b2` | `op=0 (40:37) \| 0 (36) \| d (35) \| wh (34:33) \| x6=0x21 (32:27) \| 0 (26:16) \| b2 (15:13) \| p (12) \| 0 (11:9) \| btype=4 (8:6) \| qp (5:0)` | p. `3:353` |
| **`B8`** | B (`B`) | `clrrrb` / `clrrrb.pr` | `op=0 (40:37) \| 0 (36:33) \| x6 (32:27 : 0x04=clrrrb, 0x05=clrrrb.pr) \| 0 (26:0)` | p. `3:355` |
| **`B9`** | B (`B`) | `nop.b imm21` | `op=2 (40:37) \| i (36) \| x6=0x00 (35:30) \| 0 (29:26) \| imm20a (25:6) \| qp (5:0)` (`nop.b 0` = `0x04000000000`) | p. `3:356` |

> **Curiosité d'encodage révélée par le binaire :** `nop.i 0` et `nop.m 0` produisent exactement le même mot de 41 bits (`0x00008000000`, bit 27 à `1` sous l'opcode majeur `0`), tandis que `nop.b 0` vaut `0x04000000000` (opcode majeur `2`, car l'opcode `0` avec `x6=00` sur l'unité B est occupé par `break.b` !).

---

## 2. Programme 1 — Condition prédiquée sans branchement (`clamp_abs_diff`)

### 2.1 Algorithme et code assembleur

Fonction feuille calculant `r8 = min(|a - b|, max_val)` en entier 64 bits non signé (`a = r32`, `b = r33`, `max_val = r34`), **sans aucun saut conditionnel**, en utilisant :
1. Une comparaison `cmp.lt p1, p2 = r32, r33` qui positionne deux prédicats complémentaires (`p1 = 1` si `a < b`, `p2 = 1` si `a >= b`).
2. Deux soustractions `(p1) sub r8 = r33, r32` et `(p2) sub r8 = r32, r33` placées **dans le même groupe d'instructions** (le manuel autorise deux écritures vers le même registre `r8` dans le même groupe dès lors que leurs prédicats de garde sont mutuellement exclusifs, p. `1:40`).
3. Un plafonnement prédiqué via `cmp.ltu p3, p0 = r34, r8` suivi de `(p3) mov r8 = r34` (`adds r8 = 0, r34`) et `br.ret.sptk.many b0`.

```assembly
// uint64_t clamp_abs_diff(uint64_t a /*r32*/, uint64_t b /*r33*/, uint64_t max_val /*r34*/)
clamp_abs_diff:
    // Bundle 0 : Template 0x03 (M_MI_) — 2 groupes dans 1 seul bundle, 0 nop !
    { .mmi
        cmp.lt p1, p2 = r32, r33 ;;       // Slot 0 (M) : Groupe 1 (stop après Slot 0 !)
        (p1) sub r8 = r33, r32            // Slot 1 (M) : Groupe 2 (si a < b, r8 = b - a)
        (p2) sub r8 = r32, r33 ;;         // Slot 2 (I) : Groupe 2 (si a >= b, r8 = a - b)
    }
    // Bundle 1 : Template 0x01 (MII_) — Groupe 3 (forcé dans un bundle dédié !)
    { .mii
        cmp.ltu p3, p0 = r34, r8          // Slot 0 (M) : Groupe 3 (p3 = 1 si max_val <u r8)
        nop.i 0                           // Slot 1 (I) : nop forcé
        nop.i 0 ;;                        // Slot 2 (I) : nop forcé + stop
    }
    // Bundle 2 : Template 0x11 (MIB_) — Groupe 4
    { .mib
        (p3) adds r8 = 0, r34             // Slot 0 (M) : Groupe 4 (mov r8 = r34 si p3)
        nop.i 0                           // Slot 1 (I) : nop forcé
        br.ret.sptk.many b0 ;;            // Slot 2 (B) : Groupe 4 (retour au même cycle)
    }
```

### 2.2 Encodage binaire exact (calculé d'après Vol. 3 Chap. 4)

| Bundle (Offset) | Template | Slot 0 (41 bits) | Slot 1 (41 bits) | Slot 2 (41 bits) | Mot 128 bits (`MSB..LSB`) et octets mémoire (little-endian) |
|---|---|---|---|---|---|
| **Bundle 0** (`+0x00`) | `0x03` (`M_MI_`) | `0x18012140040` (`cmp.lt`) | `0x1002a042201` (`(p1) sub`) | `0x1002a140202` (`(p2) sub`) | `0x80150a0101400a810880700242800803`<br>`03 08 80 42 02 70 80 08 81 0a 40 01 01 0a 15 80` |
| **Bundle 1** (`+0x10`) | `0x01` (`MII_`) | `0x1a0008440c0` (`cmp.ltu`) | `0x00008000000` (`nop.i 0`) | `0x00008000000` (`nop.i 0`) | `0x00040000000002000000340010881801`<br>`01 18 88 10 00 34 00 00 00 02 00 00 00 00 04 00` |
| **Bundle 2** (`+0x20`) | `0x11` (`MIB_`) | `0x10802200203` (`(p3) adds`) | `0x00008000000` (`nop.i 0`) | `0x00108001100` (`br.ret`) | `0x00840008800002000000210044004071`<br>`71 40 00 44 00 21 00 00 00 02 00 80 08 00 84 00` |

### 2.3 Diagnostic architectural sur le Programme 1

- **Pourquoi 6 instructions utiles occupent-elles 3 bundles (9 slots, 33,3 % de `nop`) au lieu de 2 bundles ?**
  - Le Bundle 0 est parfait (0 `nop`) grâce au template `0x03` (`M_MI_`) qui place un stop juste après le Slot 0 (`cmp.lt`), permettant aux deux `sub` prédiqués d'occuper les Slots 1 et 2.
  - En revanche, pour les Groupes 3 (`cmp.ltu`) et 4 (`(p3) mov` + `br.ret`), il faudrait un bundle `MIB` possédant un stop **après le Slot 0** (`M_IB_`) afin de placer `cmp.ltu` en Slot 0 (M), un stop `;;`, `(p3) adds` en Slot 1 (I) et `br.ret` en Slot 2 (B).
  - Or **le template `M_IB_` n'existe pas dans la Table 4-2 d'Itanium** (seul `MI_B_` `0x13`, avec stop après le Slot 1, existe !). Après le stop de `MI_B_`, il ne reste qu'un slot `B`, incapable d'exécuter `(p3) adds r8 = 0, r34` !
  - **Conséquence directe :** L'absence du seul template `M_IB_` force l'émetteur de code à ajouter **1 bundle entier (16 octets) et 3 `nop`**, gonflant la taille de la fonction de **+50 %** (48 octets au lieu de 32 octets).

---

## 3. Programme 2 — Appel de fonction et cadre de pile RSE (`sum_plus_one` & `double_val`)

### 3.1 Algorithme et code assembleur

Fonction non-feuille `sum_plus_one(x, y)` qui :
1. Alloue un cadre RSE via `alloc r34 = ar.pfs, 2, 2, 1, 0` (`in = 2` : `r32..r33` ; `local = 2` : `r34..r35` pour sauvegarder `ar.pfs` et `b0` ; `out = 1` : `r36` qui deviendra automatiquement le `r32` de la sous-fonction lors de `br.call`).
2. Sauvegarde `b0` dans `r35` et calcule l'argument sortant `r36 = r32 + r33` **dans le même groupe que `alloc`** (autorisé par le manuel Vol. 3 p. `3:20`).
3. Appelle `double_val(r36)` via `br.call.sptk.many b0 = double_val` (située à `+3` bundles = `+0x30` octets du bundle d'appel).
4. Ajoute `1` au résultat `r8` tout en restaurant `ar.pfs = r34` et `b0 = r35`, puis marque un **stop obligatoire** (Table 3-10 p. `3:25` : dépendance RAW sur `AR[PFS]` et `BR[b0]`) avant `br.ret.sptk.many b0`.

```assembly
// uint64_t sum_plus_one(uint64_t x /*r32*/, uint64_t y /*r33*/)
sum_plus_one:
    // Bundle 0 (+0x00) : Template 0x01 (MII_) — Prologue complet en 1 groupe, 0 nop
    { .mii
        alloc r34 = ar.pfs, 2, 2, 1, 0    // Slot 0 (M) : sof=5, sol=4, sor=0
        mov r35 = b0                      // Slot 1 (I) : sauvegarde b0 dans local r35
        add r36 = r32, r33 ;;             // Slot 2 (I) : prépare out0 (r36 = x + y)
    }
    // Bundle 1 (+0x10) : Template 0x11 (MIB_) — Appel relatif vers Bundle 4 (+3 bundles)
    { .mib
        nop.m 0                           // Slot 0 (M) : nop forcé
        nop.i 0                           // Slot 1 (I) : nop forcé
        br.call.sptk.many b0 = double_val ;; // Slot 2 (B) : renomme r36 -> r32 de l'appelé
    }
    // Bundle 2 (+0x20) : Template 0x01 (MII_) — Calcul post-appel + restauration PFS et b0
    { .mii
        adds r8 = 1, r8                   // Slot 0 (M) : r8 = 2*(x+y) + 1
        mov.i ar.pfs = r34                // Slot 1 (I) : restaure AR[PFS] (unité I uniquement)
        mov b0 = r35 ;;                   // Slot 2 (I) : restaure b0 (unité I uniquement) + stop RAW
    }
    // Bundle 3 (+0x30) : Template 0x11 (MIB_) — Retour (forcé dans un bundle séparé par le stop RAW)
    { .mib
        nop.m 0                           // Slot 0 (M) : nop forcé
        nop.i 0                           // Slot 1 (I) : nop forcé
        br.ret.sptk.many b0 ;;            // Slot 2 (B) : restaure CFM depuis AR[PFS] et retourne
    }

// Sous-fonction feuille appelée (située à +0x40, soit +3 bundles depuis le Bundle 1)
double_val:
    // Bundle 4 (+0x40) : Template 0x11 (MIB_)
    { .mib
        add r8 = r32, r32                 // Slot 0 (M) : r8 = 2 * in0 (pas besoin d'alloc !)
        nop.i 0                           // Slot 1 (I) : nop forcé
        br.ret.sptk.many b0 ;;            // Slot 2 (B) : retour immédiat dans le même groupe
    }
```

### 3.2 Encodage binaire exact (calculé d'après Vol. 3 Chap. 4)

| Bundle (Offset) | Template | Slot 0 (41 bits) | Slot 1 (41 bits) | Slot 2 (41 bits) | Mot 128 bits (`MSB..LSB`) et octets mémoire (little-endian) |
|---|---|---|---|---|---|
| **Bundle 0** (`+0x00`) | `0x01` (`MII_`) | `0x02c0040a880` (`alloc`) | `0x001880008c0` (`mov r35=b0`) | `0x10002140900` (`add r36`) | `0x80010a04800062000230058008151001`<br>`01 10 15 08 80 05 30 02 00 62 00 80 04 0a 01 80` |
| **Bundle 1** (`+0x10`) | `0x11` (`MIB_`) | `0x00008000000` (`nop.m 0`) | `0x00008000000` (`nop.i 0`) | `0x0a000007000` (`br.call +3`) | `0x50000038000002000000000100000011`<br>`11 00 00 00 01 00 00 00 00 02 00 00 38 00 00 50` |
| **Bundle 2** (`+0x20`) | `0x01` (`MII_`) | `0x10800802200` (`adds r8`) | `0x00154044000` (`mov.i pfs`) | `0x00e00146000` (`mov b0=r35`) | `0x07000a30000055011000210010044001`<br>`01 40 04 10 00 21 00 10 01 55 00 00 30 0a 00 07` |
| **Bundle 3** (`+0x30`) | `0x11` (`MIB_`) | `0x00008000000` (`nop.m 0`) | `0x00008000000` (`nop.i 0`) | `0x00108001100` (`br.ret`) | `0x00840008800002000000000100000011`<br>`11 00 00 00 01 00 00 00 00 02 00 80 08 00 84 00` |
| **Bundle 4** (`+0x40`) | `0x11` (`MIB_`) | `0x10002040200` (`add r8`) | `0x00008000000` (`nop.i 0`) | `0x00108001100` (`br.ret`) | `0x00840008800002000000200040804011`<br>`11 40 80 40 00 20 00 00 00 02 00 80 08 00 84 00` |

### 3.3 Diagnostic architectural sur le Programme 2

- **Deux goulots d'étranglement structurels de l'épilogue IA-64 :**
  1. `mov.i ar.pfs = r34` (`I26`) et `mov b0 = r35` (`I21`) sont **toutes deux réservées à l'unité `I`** (impossible de les placer dans le slot `M` d'un bundle `MI_B_`).
  2. La Table 3-10 (p. `3:25`) impose un **stop `;;` obligatoire** entre l'écriture de `b0`/`ar.pfs` et le `br.ret b0`.
  - Résultat : tout épilogue de fonction non-feuille doit séparer les restaurations (`MII_`) du retour (`MIB_`), laissant **2 `nop` obligatoires** devant `br.ret` (et de même 2 `nop` dans le bundle du `br.call` dès que le calcul des arguments a nécessité un stop préalable). Sur 15 slots au total, **5 slots sont des `nop` (33,3 %)**.

---

## 4. Programme 3 — Boucle comptée (`br.cloop`) face au pipeline logiciel (`br.ctop`)

Nous comparons la même fonction `void vec_add(uint64_t *a /*r32*/, uint64_t *b /*r33*/, uint64_t c /*r34*/, uint64_t n /*r35*/)` calculant `b[i] = a[i] + c` pour $n \ge 1$ :
- **Version 3A (`vec_add_cloop`) :** boucle comptée simple sans rotation de registres.
- **Version 3B (`vec_add_ctop`) :** boucle pipelinée modulo à **3 étages** (Stage 0 : `ld8`, Stage 1 : `add`, Stage 2 : `st8`, $II = 1$ groupe/itération).

### 4.1 Version 3A : `vec_add_cloop` (4 bundles, 64 octets)

```assembly
vec_add_cloop:
    // Bundle 0 (+0x00) : Template 0x0B (M_I_I_) — 3 groupes dans 1 seul bundle, 0 nop !
    { .mii
        adds r3 = -1, r35 ;;              // Slot 0 (M) : Groupe 1 (r3 = n - 1) + stop
        mov.i r2 = ar.lc ;;               // Slot 1 (I) : Groupe 2 (sauve ar.lc, préservé ABI) + stop
        mov.i ar.lc = r3 ;;               // Slot 2 (I) : Groupe 3 (initialise LC = n - 1) + stop
    }
.loop:
    // Bundle 1 (+0x10) : Template 0x03 (M_MI_) — Groupes 1 et 2 du corps de boucle
    { .mmi
        ld8 r8 = [r32], 8 ;;              // Slot 0 (M) : charge a[i], r32 += 8 + stop
        add r9 = r8, r34                  // Slot 1 (M) : r9 = a[i] + c
        nop.i 0 ;;                        // Slot 2 (I) : nop forcé + stop avant st8
    }
    // Bundle 2 (+0x20) : Template 0x11 (MIB_) — Groupe 3 du corps de boucle (saut à -1 bundle)
    { .mib
        st8 [r33] = r9, 8                 // Slot 0 (M) : stocke b[i], r33 += 8
        nop.i 0                           // Slot 1 (I) : nop forcé
        br.cloop.sptk.few .loop ;;        // Slot 2 (B) : si LC > 0, LC-- et saute à +0x10 (-1 bundle)
    }
    // Bundle 3 (+0x30) : Template 0x13 (MI_B_) — Épilogue en 1 seul bundle
    { .mib
        nop.m 0                           // Slot 0 (M) : nop forcé
        mov.i ar.lc = r2 ;;               // Slot 1 (I) : restaure ar.lc + stop interne
        br.ret.sptk.many b0 ;;            // Slot 2 (B) : retour
    }
```

| Bundle (Offset) | Template | Slot 0 (41 bits) | Slot 1 (41 bits) | Slot 2 (41 bits) | Mot 128 bits (`MSB..LSB`) et octets mémoire (little-endian) |
|---|---|---|---|---|---|
| **Bundle 0** (`+0x00`) | `0x0B` (`M_I_I_`) | `0x119fa3fe0c0` (`adds r3=-1,r35`) | `0x00194100080` (`mov.i r2=lc`) | `0x00154106000` (`mov.i lc=r3`) | `0x00aa0830000065040020233f47fc180b`<br>`0b 18 fc 47 3f 23 20 00 04 65 00 00 30 08 aa 00` |
| **Bundle 1** (`+0x10`) | `0x03` (`M_MI_`) | `0x0a0c2010200` (`ld8 r8=[r32],8`) | `0x10002210240` (`add r9=r8,r34`) | `0x00008000000` (`nop.i 0`) | `0x00040000004000884090141840204003`<br>`03 40 20 40 18 14 90 40 88 00 40 00 00 00 04 00` |
| **Bundle 2** (`+0x20`) | `0x11` (`MIB_`) | `0x0acc2112200` (`st8 [r33]=r9,8`) | `0x00008000000` (`nop.i 0`) | `0x091ffffe140` (`br.cloop -1`) | `0x48fffff0a00002000000159842244011`<br>`11 40 24 42 98 15 00 00 00 02 00 a0 f0 ff ff 48` |
| **Bundle 3** (`+0x30`) | `0x13` (`MI_B_`) | `0x00008000000` (`nop.m 0`) | `0x00154104000` (`mov.i lc=r2`) | `0x00108001100` (`br.ret`) | `0x00840008800055041000000100000013`<br>`13 00 00 00 01 00 00 10 04 55 00 80 08 00 84 00` |

### 4.2 Version 3B : `vec_add_ctop` (8 bundles, 128 octets)

> **Piège architectural majeur découvert lors de l'écriture de `vec_add_ctop` :**
> En IA-64, les arguments d'entrée d'une fonction arrivent en `r32..r35` (`a=r32, b=r33, c=r34, n=r35`). Or la fenêtre de registres généraux rotatifs allouée par `alloc` commence **toujours à `r32`** par pas de **8 registres** (`r32..r39` pour `r = 8`, Vol. 1 §3.3.1).
> Par conséquent, dès le premier `br.ctop`, **`r32..r35` tournent et écrasent les pointeurs `a`, `b` et la constante `c` !** Toute fonction utilisant des `GR` rotatifs doit donc impérativement **évacuer ses arguments invariants de boucle hors de `r32..r39`** vers des registres statiques (`r14 = a`, `r15 = b`, `r16 = c`) dans le prologue !
> En revanche, `AR[EC]` n'a **pas** besoin d'être sauvegardé manuellement : lors du `br.call`, le matériel a déjà copié l'`EC` de l'appelant dans `AR[PFS].pec`, que `alloc` sauvegarde dans `r40` et que `br.ret` restaure automatiquement !

```assembly
vec_add_ctop:
    // Bundle 0 (+0x00) : Template 0x01 (MII_) — Allocation r=8 (r32..r39 rotatifs, r40..r41 statiques)
    { .mii
        alloc r40 = ar.pfs, 4, 6, 0, 8    // Slot 0 (M) : sof=10, sol=10, sor=1 (8 rotatifs r32..r39)
        mov.i r41 = ar.lc                 // Slot 1 (I) : sauve ar.lc dans le registre statique r41
        adds r2 = -1, r35 ;;              // Slot 2 (I) : r2 = n - 1
    }
    // Bundle 1 (+0x10) : Template 0x00 (MII) — Évacuation de a, b, c hors de la fenêtre rotative r32..r39 !
    { .mii
        adds r14 = 0, r32                 // Slot 0 (M) : r14 = a (statique)
        adds r15 = 0, r33                 // Slot 1 (I) : r15 = b (statique)
        adds r16 = 0, r34                 // Slot 2 (I) : r16 = c (statique, pas de stop requis)
    }
    // Bundle 2 (+0x20) : Template 0x11 (MIB_) — Initialisation de LC et remise à zéro des RRB
    { .mib
        nop.m 0                           // Slot 0 (M) : nop forcé (mov.i et clrrrb exigent I et B)
        mov.i ar.lc = r2                  // Slot 1 (I) : LC = n - 1
        clrrrb ;;                         // Slot 2 (B) : RRB.gr = 0, RRB.fr = 0, RRB.pr = 0 + stop
    }
    // Bundle 3 (+0x30) : Template 0x01 (MII_) — Initialisation de EC = 3 et PR.rot (p16=1, p17..p63=0)
    { .mii
        nop.m 0                           // Slot 0 (M) : nop forcé (mov.i et mov pr.rot exigent I)
        mov.i ar.ec = 3                   // Slot 1 (I) : EC = 3 étages
        mov pr.rot = 1 << 16 ;;           // Slot 2 (I) : p16 = 1, p17..p63 = 0 + stop
    }
.loop:
    // Bundle 4 (+0x40) : Template 0x00 (MII, SANS STOP !) — 1re moitié du groupe noyau (II = 1 groupe)
    { .mii
        (p16) ld8 r32 = [r14], 8          // Slot 0 (M) : Stage 0 (charge a[i] -> r32, qui deviendra r33)
        (p17) add r34 = r33, r16          // Slot 1 (I) : Stage 1 (r34 = r33 + c, qui deviendra r35)
        nop.i 0                           // Slot 2 (I) : nop forcé (st8 exige un slot M au bundle suivant)
    }
    // Bundle 5 (+0x50) : Template 0x11 (MIB_, AVEC STOP !) — 2nde moitié du même groupe noyau
    { .mib
        (p18) st8 [r15] = r35, 8          // Slot 0 (M) : Stage 2 (stocke r35 dans b[i-2])
        nop.i 0                           // Slot 1 (I) : nop forcé
        br.ctop.sptk.few .loop ;;         // Slot 2 (B) : fait tourner PR/GR, boucle vers +0x40 (-1 bundle)
    }
    // Bundle 6 (+0x60) : Template 0x01 (MII_) — Restauration de LC et PFS (qui contient l'EC appelant)
    { .mii
        nop.m 0                           // Slot 0 (M) : nop forcé (mov.i exige I)
        mov.i ar.lc = r41                 // Slot 1 (I) : restaure AR[LC]
        mov.i ar.pfs = r40 ;;             // Slot 2 (I) : restaure AR[PFS] + stop RAW obligatoire
    }
    // Bundle 7 (+0x70) : Template 0x11 (MIB_) — Retour
    { .mib
        nop.m 0                           // Slot 0 (M) : nop forcé
        nop.i 0                           // Slot 1 (I) : nop forcé
        br.ret.sptk.many b0 ;;            // Slot 2 (B) : restaure CFM et AR[EC] depuis AR[PFS]
    }
```

| Bundle (Offset) | Template | Slot 0 (41 bits) | Slot 1 (41 bits) | Slot 2 (41 bits) | Mot 128 bits (`MSB..LSB`) et octets mémoire (little-endian) |
|---|---|---|---|---|---|
| **Bundle 0** (`+0x00`) | `0x01` (`MII_`) | `0x02c08a14a00` (`alloc`) | `0x00194100a40` (`mov.i r41=lc`) | `0x119fa3fe080` (`adds r2=-1,r35`) | `0x8cfd1ff0400065040290058114294001`<br>`01 40 29 14 81 05 90 02 04 65 00 40 f0 1f fd 8c` |
| **Bundle 1** (`+0x10`) | `0x00` (`MII`) | `0x10802000380` (`adds r14=0,r32`) | `0x108021003c0` (`adds r15=0,r33`) | `0x10802200400` (`adds r16=0,r34`) | `0x840110020042008400f0210040007000`<br>`00 70 00 40 00 21 f0 00 84 00 42 00 02 10 01 84` |
| **Bundle 2** (`+0x20`) | `0x11` (`MIB_`) | `0x00008000000` (`nop.m 0`) | `0x00154104000` (`mov.i lc=r2`) | `0x00020000000` (`clrrrb`) | `0x00100000000055041000000100000011`<br>`11 00 00 00 01 00 00 10 04 55 00 00 00 00 10 00` |
| **Bundle 3** (`+0x30`) | `0x01` (`MII_`) | `0x00008000000` (`nop.m 0`) | `0x00054206000` (`mov.i ec=3`) | `0x00400000040` (`mov pr.rot`) | `0x02000000200015081800000100000001`<br>`01 00 00 00 01 00 00 18 08 15 00 20 00 00 00 02` |
| **Bundle 4** (`+0x40`) | `0x00` (`MII`) | `0x0a0c0e10810` (`(p16) ld8`) | `0x10001042891` (`(p17) add`) | `0x00008000000` (`nop.i 0`) | `0x00040000004000410a2454181c210200`<br>`00 02 21 1c 18 54 24 0a 41 00 40 00 00 00 04 00` |
| **Bundle 5** (`+0x50`) | `0x11` (`MIB_`) | `0x0acc0f46212` (`(p18) st8`) | `0x00008000000` (`nop.i 0`) | `0x091ffffe1c0` (`br.ctop -1`) | `0x48fffff0e0000200000015981e8c4251`<br>`51 42 8c 1e 98 15 00 00 00 02 00 e0 f0 ff ff 48` |
| **Bundle 6** (`+0x60`) | `0x01` (`MII_`) | `0x00008000000` (`nop.m 0`) | `0x00154152000` (`mov.i lc=r41`) | `0x00154050000` (`mov.i pfs=r40`) | `0x00aa0280000055054800000100000001`<br>`01 00 00 00 01 00 00 48 05 55 00 00 80 02 aa 00` |
| **Bundle 7** (`+0x70`) | `0x11` (`MIB_`) | `0x00008000000` (`nop.m 0`) | `0x00008000000` (`nop.i 0`) | `0x00108001100` (`br.ret`) | `0x00840008800002000000000100000011`<br>`11 00 00 00 01 00 00 00 00 02 00 80 08 00 84 00` |

### 4.3 Et quand le noyau `br.ctop` tient-il en 1 seul bundle sans aucun `nop` ? (Noyau 3C : réduction `sum += a[i]`)

Si la boucle ne fait pas d'écriture mémoire par itération (par exemple une réduction `sum += a[i]`), le noyau pipeliné n'a plus que **3 instructions** (`ld8` en M, `add` en I, `br.ctop` en B) et tient dans **1 seul bundle `0x11` (`MIB_`) avec 0 `nop` (100 % de densité)** et un déplacement de branchement nul (`0`) :

| Bundle | Template | Slot 0 (`(p16) ld8 r32=[r14],8`) | Slot 1 (`(p17) add r8=r8,r33`) | Slot 2 (`br.ctop .loop` `disp=0`) | Mot 128 bits et octets mémoire (LE) |
|---|---|---|---|---|---|
| **Noyau 3C** | `0x11` (`MIB_`) | `0x0a0c0e10810` | `0x10002110211` | `0x080000001c0` | `0x40000000e0400084408454181c210211`<br>`11 02 21 1c 18 54 84 40 84 00 40 e0 00 00 00 40` |

---

## 5. Bilan quantitatif global et enseignements directs pour IA-86

### 5.1 Tableau comparatif des 4 programmes encodés

| Programme | Bundles statiques (Octets) | Slots totaux | Instructions utiles | `nop` forcés (% gaspillé) | Groupes dynamiques (Cycles min.) pour $N$ itérations | Bundles dynamiques exécutés |
|---|---|---|---|---|---|---|
| **1. `clamp_abs_diff`** | 3 (48 B) | 9 | 6 | **3 (33,3 %)** | 4 groupes | 3 bundles (48 B) |
| **2. `sum_plus_one` + `double_val`** | 5 (80 B) | 15 | 10 | **5 (33,3 %)** | 5 groupes | 5 bundles (80 B) |
| **3A. `vec_add_cloop`** | 4 (64 B) | 12 | 9 | **3 (25,0 %)** | **$3N + 5$ groupes** | **$2N + 2$ bundles** |
| **3B. `vec_add_ctop`** | 8 (128 B) | 24 | 17 | **7 (29,2 %)** | **$N + 7$ groupes** (gain ×3 dès $N \gg 1$, égal dès $N=1$) | **$2N + 10$ bundles** (+128 B dynamiques fixes) |
| **3C. Noyau `vec_sum_ctop`** | 1 (16 B, noyau seul) | 3 | 3 | **0 (0,0 %)** | $N + 1$ groupes (noyau seul) | $N + 1$ bundles |

### 5.2 Les 4 leçons concrètes prouvées par l'encodage manuel (pour IA-86)

| Problème observé en écrivant l'assembleur IA-64 | Cause matérielle exacte dans la spécification Intel | Règle de conception correspondante pour IA-86 |
|---|---|---|
| **1. Le « mur des 4 instructions » dans les boucles (`3B` vs `3C`)** | Un bundle fixe de 3 slots loge parfaitement `ld + add + br.ctop` (3 instructions = 1 bundle, 0 `nop`), mais dès qu'on ajoute un `st` (`ld + add + st + br.ctop` = 4 instructions), il faut **2 bundles (6 slots)**, donc **2 `nop` obligatoires (33 %)** exécutés à chaque itération ! | ❌ Ne jamais imposer de paquet de taille fixe à 3 slots. Utiliser un **encodage de longueur variable ou à préfixe de groupe sans `nop` de remplissage**. |
| **2. L'absence du template `M_IB_` (`Programme 1`)** | Sur 5 bits de template, seules 24 combinaisons existent : `MI_B_` (`0x13`) existe, mais **`M_IB_` n'existe pas**, ce qui interdit de placer 1 instruction suivie d'un stop puis d'une instruction ALU + branchement dans le même bundle (+50 % de taille sur `clamp_abs_diff`). | ✅ Séparer totalement le **marquage de fin de groupe d'instructions** (1 bit de stop par instruction ou champ de longueur de groupe) du **routage vers les unités d'exécution**. |
| **3. Le télescopage entre arguments `r32+` et registres rotatifs (`Programme 3B`)** | La fenêtre rotative `GR` commence obligatoirement en `r32` par blocs de 8 (`r32..r39`), exactement là où l'ABI place les arguments d'entrée (`in0..in7`), obligeant le prologue à copier tous les arguments invariants (`a, b, c`) dans des registres statiques avant `br.ctop`. | ✅ Si un mécanisme de file/renommage de boucle est retenu dans IA-86, il ne doit **jamais** partager les mêmes numéros logiques que les registres d'arguments d'entrée `in0..in7`. |
| **4. L'embouteillage de l'unité `I` dans les prologues/épilogues (`Programmes 2 & 3B`)** | `mov b0`, `mov.i ar.pfs`, `mov.i ar.lc`, `mov.i ar.ec` et `mov pr.rot` sont **toutes réservées aux seuls slots `I`**, laissant les slots `M` vides (`nop.m 0`) dans les prologues et épilogues, et exigeant un stop obligatoire avant `br.ret`. | ✅ Fusionner la restauration du cadre et le retour en une seule instruction atomique (`ret.frame`) et autoriser les transferts de registres système sur toutes les unités entières. |
