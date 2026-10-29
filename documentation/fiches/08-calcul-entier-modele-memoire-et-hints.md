# Fiche 8 — Calcul entier, sémaphores, modèle mémoire, hints et multimédia (IA-64)

**Sources principales :** Manuel Intel® Itanium® Architecture Software Developer's Manual, Révision 2.3, Volume 1, Partie 1, Chapitre 4 (§ 4.2, § 4.4.1–4.4.3, § 4.4.6–4.4.7 et § 4.5.2–4.9, pages `1:50` à `1:60` et `1:69` à `1:84`).

---

## 1. Instructions de calcul entier (§ 4.2, p. `1:50`–`1:53`)

Les instructions arithmétiques, logiques et de pointeurs 32 bits sont de **type A** (exécutables sur **I-unit** ou **M-unit**), tandis que les décalages et manipulations de champs de bits sont de **type I** (**I-unit** uniquement).

### 1.1 Arithmétique, logique et constantes (§ 4.2.1, 4.2.2 et 4.2.5, Tables 4-3, 4-4 et 4-7, p. `1:51`–`1:53`)
- **Addition / soustraction à 2 ou 3 entrées (`add`, `sub`) :**
  - Forme normale : `add r1 = r2, r3` et `sub r1 = r2, r3`.
  - Forme à 3 entrées (« `+1` » / « `-1` ») : `add r1 = r2, r3, 1` ($r_2 + r_3 + 1$) et `sub r1 = r2, r3, 1` ($r_2 - r_3 - 1$).
  - Formes immédiates signées : `adds r1 = imm14, r3` (immédiat 14 bits), `addl r1 = imm22, r3` (immédiat **22 bits**, mais `r3` est restreint à **`r0`, `r1`, `r2` ou `r3`** !), et `sub r1 = imm8, r3` (8 bits, attention : calcule $\text{imm8} - r_3$).
- **Décalage et addition (`shladd`, p. `1:51`) :** `shladd r1 = r2, count2, r3` calcule $(r_2 \ll \text{count2}) + r_3$ avec $\text{count2} \in \{1, 2, 3, 4\}$ (indexation de tableaux d'éléments de 2, 4, 8 ou 16 octets en 1 cycle).
- **Constantes larges (§ 4.2.5, p. `1:53`) :**
  - Jusqu'à 22 bits signés : pseudo-instruction `mov r1 = imm22` (encodée comme `addl r1 = imm22, r0`).
  - Sur 64 bits : **`movl r1 = imm64`** (type `L+X`, seule instruction non-branchement occupant **2 slots** dans un bundle `MLX`).
- **Multiplication et division entières (§ 4.2.1, p. `1:51`) :**
  - Sur l'unité I (`pmpy` / `mpy4` / `mpyshl4` — *Note : `mpy4` et `mpyshl4` ont été retirés ou remplacés par `xmpy` selon les révisions, voir Vol. 3*), la multiplication entière 64 bits complète (`xma.l`, `xma.h`, `xma.hu`) s'exécute **dans l'unité flottante (F-unit)** sur les registres `FR` (voir [Fiche 9](file:///C:/Users/theog/Documents/_CODE/10-ia64/IA86/documentation/fiches/09-modele-flottant-xmpy-et-ia32.md)).
  - **Aucune instruction de division entière** : la division entière est réalisée en logiciel via les instructions flottantes (`frcpa` + `fma`).
- **Opérations logiques (Table 4-4, p. `1:52`) :** `and`, `or`, `xor`, et **`andcm`** ($r_2 \land \lnot r_3$, avec un registre ou un immédiat `imm8` en 1er opérande).

### 1.2 Pointeurs et entiers 32 bits (§ 4.2.3, Table 4-5, p. `1:52`)
- **`addp4` et `shladdp4`** (*appelés `addp`/`shladdp` dans le texte du Vol. 1 p. `1:52`*) : additionnent deux valeurs 32 bits, mettent à zéro les bits `63:32`, **sauf les bits `62:61` qui reçoivent une copie des bits `31:30` du second opérande** (pour sélectionner le registre de région virtuelle IA-64 lors de la traduction de pointeurs 32 bits).
- **Extensions de signe et de zéro :** `sxt1`, `sxt2`, `sxt4` et `zxt1`, `zxt2`, `zxt4` (sur I-unit).

### 1.3 Champs de bits et décalages (§ 4.2.4, Table 4-6, p. `1:52`–`1:53`, sur I-unit)
- **Décalages variables :** `shl`, `shr` (signé), `shr.u` (non signé) d'un nombre de bits donné par un registre.
- **Extraction (`extr`, `extr.u`) :** extrait un champ de longueur `len` à partir du bit `pos`, le cale à droite (bit 0) et l'étend par le signe (`extr`) ou par des zéros (`extr.u`). Un décalage à droite par une constante `shr r1 = r3, count` est un simple alias assembleur de `extr`.
- **Dépôt (`dep`, `dep.z`) :** insère un champ de longueur `len` à la position `pos` soit dans un second registre (`dep` — *shift-left-mask-merge*), soit dans un registre de zéros (`dep.z`). Un décalage à gauche par une constante `shl r1 = r2, count` est un alias de `dep.z`.
- **Décalage d'entonnoir sur 128 bits (`shrp`, *Shift Right Pair*) :** concatène deux registres sources `r2:r3` (128 bits) et en extrait 64 bits à partir d'une position immédiate `count6` (permet aussi de faire une rotation de bits 64 bits si `r2 == r3`).

---

## 2. Accès mémoire spéciaux (`16 octets`), sémaphores et hiérarchie de cache (§ 4.4.1–4.4.3 et § 4.4.6, p. `1:58`–`1:60` et `1:69`–`1:72`)

### 2.1 Chargements/écritures 16 octets et sémaphores atomiques (§ 4.4.1–4.4.3, p. `1:58`–`1:60`)
- **`ld16` et `st16` :** transfèrent atomiquement **16 octets** alignés entre la mémoire et la paire `(r1, AR[CSD])` (sans post-incrément d'adresse).
- **Les 3 familles de sémaphores atomiques (§ 4.4.3, p. `1:59`–`1:60`) :** toujours non spéculatifs, sans post-incrément, et lèvent obligatoirement `Unaligned Data Reference` si l'adresse n'est pas alignée sur sa taille naturelle (même si `UM.ac == 0`, p. `1:57`) :
  1. **`xchg1` / `xchg2` / `xchg4` / `xchg8 r1 = [r3], r2`** : échange atomiquement `[r3]` et `r2` (toujours avec sémantique *acquire*).
  2. **`cmpxchg1` / `2` / `4` / `8.{acq,rel} r1 = [r3], r2, ar.ccv`** : lit `[r3]` (étendu par des zéros) dans `r1` ; s'il est égal à **`AR[CCV]`**, écrit `r2` en `[r3]`.
  3. **`cmp8xchg16.{acq,rel} r1 = [r3], r2, ar.csd, ar.ccv`** : compare **8 octets** en `[r3]` avec `AR[CCV]` ; s'ils sont égaux, écrit **16 octets** (`r2` et `AR[CSD]`) à l'adresse alignée sur 16 octets contenant `[r3]`.
  4. **`fetchadd4` / `fetchadd8.{acq,rel} r1 = [r3], inc3`** : lit `[r3]` dans `r1`, ajoute l'immédiat signé **`inc3` $\in \{-16, -8, -4, -1, +1, +4, +8, +16\}$**, et réécrit la somme en `[r3]`.

### 2.2 Hints de localité cache et préchargement (§ 4.4.6.1, Table 4-18 et 4-19, p. `1:69`–`1:72`)
Chaque niveau de cache est modélisé avec une structure **temporelle** et une structure **non temporelle** :
- Completers de localité sur `ld`, `st`, `lfetch` :
  - *(aucun)* : temporel à tous les niveaux (L1+).
  - **`.nt1`** : non temporel au niveau 1 (sur `ld` et `lfetch`).
  - **`.nt2`** : non temporel aux niveaux 1 et 2 (sur `lfetch` uniquement).
  - **`.nta`** : non temporel à **tous** les niveaux (sur `ld`, `st`, `lfetch` ; sur `st.nta`, indique aussi un *streaming store*).
- **`ld.bias`** : hint demandant l'acquisition immédiate de la ligne de cache en état exclusif (*Modified/Exclusive*) en prévision d'un `st` imminent sur la même ligne.
- **Préchargement explicite (`lfetch`, `lfetch.fault`) et implicite :** tout `ld`, `st` ou `lfetch` avec **post-incrément d'adresse** précharge implicitement la ligne contenant la **nouvelle adresse post-incrémentée** !
- **Vidage de cache et cohérence d'instructions (§ 4.4.6.2, p. `1:72`–`1:73`) :**
  - Les caches d'instructions **ne sont pas cohérents en matériel** avec les caches de données.
  - Après écriture de code en mémoire (JIT, chargement dynamique), le logiciel doit exécuter la séquence **`fc.i`** (*Flush Cache Instruction*) $\to$ **`sync.i`** $\to$ **`srlz.i`**.

---

## 3. Modèle d'ordonnancement mémoire (*Weak Ordering*, § 4.4.7, Table 4-20, p. `1:73`–`1:74`)

En dehors des dépendances de données vers la même adresse, **les lectures et écritures mémoire ordinaires (*unordered*) peuvent être réordonnées dans n'importe quel ordre par le processeur**. L'ordre inter-cœurs est contrôlé par **4 sémantiques** :

| Sémantique | Instructions | Garantie d'ordonnancement |
|---|---|---|
| **Unordered** | `ld`, `st`, `ld.s`, `ld.a`, `ld.c` | Aucune garantie d'ordre par rapport aux accès à d'autres adresses. |
| **Acquire (`.acq`)** | `ld.acq`, `ld.c.clr.acq`, `xchg`, `cmpxchg.acq`, `fetchadd.acq` | Rendu visible **avant** tous les accès mémoire *suivants* dans l'ordre du programme. |
| **Release (`.rel`)** | `st.rel`, `cmpxchg.rel`, `fetchadd.rel` | Rendu visible **après** tous les accès mémoire *précédents* dans l'ordre du programme. |
| **Fence (`mf`)** | `mf` (*Memory Fence*) | Barrière bidirectionnelle complète (*release + acquire*) : tous les accès précédents sont visibles avant tout accès suivant. |

**Matrice d'ordonnancement (Table 4-20, p. `1:73` — `O` = ordonné, `-` = réordonnancement permis) :**

| 1er accès $\downarrow$ \ 2nd accès $\rightarrow$ | Fence (`mf`) | Acquire (`.acq`) | Release (`.rel`) | Unordered |
|---|---|---|---|---|
| **Fence (`mf`)** | **O** | **O** | **O** | **O** |
| **Acquire (`.acq`)** | **O** | **O** | **O** | **O** |
| **Release (`.rel`)** | **O** | `-` *(peut passer avant !)* | **O** | `-` |
| **Unordered** | **O** | `-` | **O** | `-` |

> **Point clé (Table 4-20) :** Un `st.rel` suivi d'un `ld.acq` à une adresse différente **peut être réordonné** (`-` à l'intersection `release` $\to$ `acquire`) ! Pour interdire le réordonnancement *Store-Load* (par ex. algorithme de Dekker/Peterson), il faut obligatoirement une barrière **`mf`**.

---

## 4. Hints de branchement (`brp`), multimédia (SIMD) et chaînes (§ 4.5.2–4.9, p. `1:78`–`1:84`)

### 4.1 Hints sur les branchements et instruction `brp` (§ 4.5.2–4.5.3, Tables 4-26 à 4-28, p. `1:78`–`1:79`)
Chaque branchement porte 3 completers de hint :
1. **Stratégie de prédiction (`wh`)** : `.sptk` (*Static Taken*), `.spnt` (*Static Not-Taken* — n'alloue pas d'entrée dans le prédicteur dynamique), `.dptk` (*Dynamic Taken*), `.dpnt` (*Dynamic Not-Taken*).
2. **Préchargement séquentiel (`ph`)** : `.few` (quelques lignes à la cible) ou `.many` (beaucoup de lignes).
3. **Désallocation du prédicteur (`dh`)** : `.clr` (libère l'entrée du cache de branchement après usage) ou *(aucun)*.
- **Instruction `brp` (*Branch Predict*, p. `1:79`) :** instruction dédiée qui avertit le matériel plusieurs cycles à l'avance de l'adresse d'un futur branchement, de sa cible et de son importance (`.imp` pour activer un petit prédicteur ultra-rapide de boucle). `mov br = ...` peut aussi embarquer ces hints pour préparer un saut indirect.

### 4.2 Multimédia SIMD dans les registres généraux (§ 4.6, Tables 4-29 à 4-31, p. `1:79`–`1:82`)
Les registres généraux 64 bits `GR` sont découpés en vecteurs de **8 octets (`1-byte`), 4 mots de 16 bits (`2-byte`) ou 2 mots de 32 bits (`4-byte`)** :
- **Arithmétique parallèle :** `padd1/2/4`, `psub1/2/4` (modulo `.sss` saturé signé, `.uuu`/`.uus` saturé non signé), moyenne `pavg1/2` (`.raz`), `pshladd2`/`pshradd2`, comparaison `pcmp1/2/4` (écrit un masque de `0xFF..` ou `0x00..` dans le `GR` cible, **et non** dans un prédicat !), `pmpy2.r`/`pmpy2.l`, `pmpyshr2`, somme des différences absolues `psad1` (pour l'estimation de mouvement vidéo), `pmin1/2`, `pmax1/2`.
- **Décalages et permutations :** `pshl2/4`, `pshr2/4`, entrelacement `mix1/2/4.{l,r}`, `unpack1/2/4.{l,h}`, `pack2/4.{sss,uss}`, et permutation arbitraire `mux1` / `mux2`.

### 4.3 Chaînes de caractères, comptage de bits et privilège (§ 4.8–4.9, p. `1:83`–`1:84`)
- **Recherche d'octet/mot nul (`czx1.{l,r}`, `czx2.{l,r}`, Table 4-33, p. `1:83`–`1:84`) :** renvoie l'indice (`0` à `7` en `1-byte`, ou `8` si aucun zéro) du premier élément nul en balayant de gauche à droite (`.l`, big-endian) ou de droite à gauche (`.r`, little-endian) — conçu pour accélérer `strlen`/`strcpy`.
- **Bits (`popcnt`, `clz`, Table 4-34, p. `1:84`) :** `popcnt` compte le nombre de bits à `1` ; `clz` compte le nombre de zéros de tête sur 64 bits.
- **Élévation de privilège sans interruption (`epc`, *Enter Privileged Code*, § 4.9, p. `1:84`) :** augmente le niveau de privilège (vers celui indiqué par l'entrée TLB de la page d'instructions, marquée *promote*) sans sauter ni déclencher d'interruption, à condition que `PFS.ppl >= PSR.cpl` (au retour, `br.ret` redescend le privilège à `PFS.ppl`).

---

## 5. Regard critique & limites architecturales (pour IA-86)

| Mécanisme | Verdict | Analyse critique |
|---|---|---|
| **Modèle mémoire `.acq` / `.rel` / `mf`** | ✅ **À garder** | Très propre et visionnaire (adopté depuis par ARMv8 et C++11). Évite les barrières globales coûteuses sur chaque verrou. |
| **Asymétrie de `addl` (22 bits limité à `r0`–`r3`)** | ❌ **À corriger** | Dans un slot de 41 bits, `addl` ne dispose que de 2 bits pour le registre source (`r0` à `r3`). Cela force par convention ABI à figer `r1 = gp` (*Global Pointer*) et oblige à utiliser un bundle `MLX` (`movl` sur 2 slots) dès qu'on veut additionner une constante $> 14$ bits à un autre registre. |
| **`AR[CCV]` et `AR[CSD]` implicites sur `cmpxchg`** | ⚠️ **Discutable** | Comme une instruction M-unit de 41 bits ne peut pas encoder 3 registres généraux sources + 1 destination, `cmpxchg` impose de charger d'abord la valeur attendue dans `mov ar.ccv = rX ;;` (avec un stop !), ce qui allonge toute opération CAS (*Compare-And-Swap*) et complique la réentrance si une fonction imbriquée écrase `CCV`. |
| **Instructions `brp` et prolifération des hints** | ❌ **À rejeter** | Consommer des slots d'instructions entiers dans le binaire pour guider un prédicteur de branchement (`brp`) ou figer la topologie des caches (`.nt1`/`.nt2`) vieillit très mal dès que la microarchitecture change. Un prédicteur matériel moderne (TAGE) fait mieux sans polluer le code. |
| **Dualité `pcmp` (masque dans `GR`) vs `cmp` (dans `PR`)** | ⚠️ **Incohérent** | IA-64 possède 64 registres de prédicat `PR`, mais ses instructions SIMD (`pcmp`) n'utilisent pas les prédicats et écrivent des masques de bits dans les `GR` (comme MMX), sans instruction directe pour convertir ce masque en prédicat ou faire un `blend` octet par octet ! |
