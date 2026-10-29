# Fiche 1 — État architectural, registres et mémoire (IA-64)

**Sources principales :** Manuel Intel® Itanium® Architecture Software Developer's Manual, Révision 2.3, Volume 1, Partie 1, Chapitre 3 (§ 3.1 à 3.2, pages `1:23` à `1:37`).

---

## 1. Vue d'ensemble des registres applicatifs (§ 3.1, Figure 3-1, p. `1:23` et `1:25`)

L'état architectural visible par un programme applicatif IA-64 comprend :

| Banque / Registre | Abrév. | Nombre | Largeur | Découpage et particularités | Référence |
|---|---|---|---|---|---|
| **Registres généraux** | `GR0` – `GR127` (`r0`–`r127`) | 128 | **64 bits + 1 bit NaT** (65 bits) | `GR0`–`GR31` : statiques (`GR0` = `0`, NaT = `0`). `GR32`–`GR127` : empilés (*stacked*, renommés par la pile de registres) et tournants (*rotating*). | § 3.1.2, p. `1:25` |
| **Registres flottants** | `FR0` – `FR127` (`f0`–`f127`) | 128 | **82 bits** (signe 1, exposant 17, mantisse 64) | `FR0`–`FR31` : statiques (`FR0` = `+0.0`, `FR1` = `+1.0`). `FR32`–`FR127` : tournants (96 registres). Exception différée codée par la valeur spéciale `NaTVal`. | § 3.1.3, p. `1:26` |
| **Registres de prédicat** | `PR0` – `PR63` (`p0`–`p63`) | 64 | **1 bit** | `PR0`–`PR15` : statiques (`PR0` = `1`). `PR16`–`PR63` : tournants (48 registres). | § 3.1.4, p. `1:26` |
| **Registres de branchement** | `BR0` – `BR7` (`b0`–`b7`) | 8 | **64 bits** | Statiques. Contiennent les adresses cibles des sauts indirects et les adresses de retour (`b0` par convention). | § 3.1.5, p. `1:26` |
| **Pointeur d'instruction** | `IP` | 1 | **64 bits** | Adresse du **bundle** (16 octets) en cours d'exécution. Les 4 bits de poids faible (`3:0`) valent toujours `0` en mode Itanium. | § 3.1.6, p. `1:27` |
| **Marqueur de cadre courant** | `CFM` | 1 | **38 bits** | Décrit le cadre de pile de registres courant (`sof`, `sol`, `sor`) et les bases de rotation (`rrb.gr`, `rrb.fr`, `rrb.pr`). Non lisible/modifiable directement par `mov`. | § 3.1.7, p. `1:27` |
| **Registres d'application** | `AR0` – `AR127` (`ar0`–`ar127`) | 128 | **64 bits** (ou sous-ensemble) | Registres spécialisés (RSE, boucles, sauvegarde d'état, compteurs, etc.). | § 3.1.8, p. `1:28` |
| **Masque utilisateur** | `UM` | 1 | **6 bits** (`5:0` de `PSR`) | Sous-ensemble du registre d'état `PSR` accessible en mode utilisateur (boutisme `be`, alignement `ac`, moniteurs `up`, indicateurs flottants `mfl`/`mfh`). | § 3.1.10, p. `1:33` |
| **Identifiants processeur** | `CPUID[0..n]` | $\ge 5$ | **64 bits** | Lecture seule via `mov r1 = cpuid[r3]`. `CPUID[0..1]` = vendeur ASCII, `CPUID[3]` = version, `CPUID[4]` = bits de fonctionnalités (`lb`, `sd`, `ao`, `ru`, `cz`, `x2`). | § 3.1.11, p. `1:34`–`1:35` |
| **Données de performance** | `PMD[0..m]` | Variable | **64 bits** | Compteurs matériels de performance. | § 3.1.9, p. `1:33` |

---

## 2. Règles sur les registres constants, réservés et ignorés (§ 3.1.1 à 3.1.4, p. `1:23`–`1:26`)

Le comportement du processeur sur les registres constants ou non définis est asymétrique et doit être reproduit exactement dans un simulateur :

| Cible | Lecture | Écriture | Référence |
|---|---|---|---|
| **`GR0` (`r0`)** | Vaut toujours `0` (et bit NaT = `0`). | **Faute `Illegal Operation`** (interdit d'utiliser `r0` comme destination). | § 3.1.2, p. `1:25` |
| **`FR0` (`f0`) et `FR1` (`f1`)** | `f0` = `+0.0`, `f1` = `+1.0`. | **Faute `Illegal Operation`** (ou faute d'enregistrement). | § 3.1.3, p. `1:26` |
| **`PR0` (`p0`)** | Vaut toujours `1` (vrai). | **Écriture ignorée sans faute** (*value written is discarded*). Permet d'annuler une destination d'instruction `cmp`. | § 3.1.4, p. `1:26` |
| **Registre réservé** | Faute `Illegal Operation` (ou `Reserved Register/Field` si non implémenté dans une banque à taille variable). | Faute `Illegal Operation`. | Table 3-1, p. `1:24` |
| **Registre ignoré** (ex. `AR48`–`63`, `AR112`–`127`, `CPUID[2]`) | Retourne `0`. | Valeur jetée sans faute. | Table 3-1, p. `1:24` ; Table 3-3, p. `1:28` |
| **Champ réservé** (`rv`) dans un registre | Retourne `0`. | Écrire une valeur non nulle déclenche une faute **`Reserved Register/Field`**. | Table 3-1, p. `1:24` |
| **Champ ignoré** (`ig`) dans un registre | Retourne `0` (sauf mention contraire). | Valeur jetée sans faute. | Table 3-1, p. `1:24` |

---

## 3. Format détaillé du `CFM` (Current Frame Marker, § 3.1.7, Figure 3-2, Table 3-2, p. `1:27`)

Le `CFM` fait **38 bits** (bits `37:0`) :

| Champ | Bits | Largeur | Rôle |
|---|---|---|---|
| `sof` | `6:0` | 7 bits | **Size of Frame** : taille totale du cadre de pile de registres courant (0 à 96 registres, à partir de `r32`). |
| `sol` | `13:7` | 7 bits | **Size of Locals** : taille de la zone locale (entrées + locaux). La zone de sortie (*outputs*) vaut `sof - sol`. |
| `sor` | `17:14` | 4 bits | **Size of Rotating** : nombre de registres généraux tournants divisé par 8. Le nombre réel est `8 * sor` (commence toujours à `r32`). |
| `rrb.gr` | `24:18` | 7 bits | **Register Rename Base (GR)** : base de renommage des registres généraux tournants. |
| `rrb.fr` | `31:25` | 7 bits | **Register Rename Base (FR)** : base de renommage des registres flottants tournants (`f32`–`f127`, modulo 96). |
| `rrb.pr` | `37:32` | 6 bits | **Register Rename Base (PR)** : base de renommage des prédicats tournants (`p16`–`p63`, modulo 48). |

---

## 4. Les registres d'application (`AR`) principaux (§ 3.1.8, Table 3-3, p. `1:28`–`1:33`)

Chaque `AR` dépend d'un type d'unité d'exécution (**M-unit** ou **I-unit**) pour les instructions `mov ar = ...` et `mov ... = ar` :

| Numéro | Nom | Unité | Rôle et structure | Référence |
|---|---|---|---|---|
| `AR 0`–`7` | `KR0`–`KR7` | **M** | *Kernel Registers* : lisibles à tout niveau de privilège, modifiables uniquement au niveau 0 (sinon faute `Privileged Register`). | § 3.1.8.1, p. `1:28`–`1:29` |
| `AR 16` | `RSC` | **M** | *Register Stack Configuration* : bits `1:0` = `mode` (lazy/eager), `3:2` = `pl`, `4` = `be`, `29:16` = `loadrs` (14 bits). | § 3.1.8.2, Fig. 3-3, p. `1:29` |
| `AR 17` | `BSP` | **M** | *Backing Store Pointer* (**lecture seule** ; écriture = faute `Illegal Operation`). Adresse mémoire où serait sauvegardé `GR32` du cadre courant (bits `2:0` = `0`). | § 3.1.8.3, Fig. 3-4, p. `1:29`–`1:30` |
| `AR 18` | `BSPSTORE` | **M** | *Backing Store Pointer for Memory Stores* : adresse mémoire du prochain *spill* du RSE (bits `2:0` ignorés). | § 3.1.8.4, Fig. 3-5, p. `1:30` |
| `AR 19` | `RNAT` | **M** | *RSE NaT Collection* : collecte les bits NaT des registres sauvegardés par le RSE. Bit 63 ignoré (vaut `0`). | § 3.1.8.5, Fig. 3-6, p. `1:30` |
| `AR 25` | `CSD` | **M** | *Compare and Store Data* : second registre de 8 octets pour `ld16`, `st16` et `cmp8xchg16`. | § 3.1.8.6, p. `1:30` |
| `AR 32` | `CCV` | **M** | *Compare and Exchange Compare Value* : valeur comparée par `cmpxchg`. | § 3.1.8.7, p. `1:30` |
| `AR 36` | `UNAT` | **M** | *User NaT Collection* : 64 bits stockant les bits NaT lors des instructions `st8.spill` et `ld8.fill` (indexé par les bits `8:3` de l'adresse mémoire). | § 3.1.8.8, p. `1:31` |
| `AR 40` | `FPSR` | **M** | *Floating-point Status Register* : contrôle et drapeaux d'état flottants (`sf0` à `sf3`). | § 3.1.8.9, p. `1:31` |
| `AR 44` | `ITC` | **M** | *Interval Time Counter* : compteur de temps 64 bits à fréquence fixe. | § 3.1.8.10, p. `1:31` |
| `AR 64` | `PFS` | **I** | *Previous Function State* : bits `37:0` = `pfm` (copie de `CFM`), `57:52` = `pec` (copie de `EC`, 6 bits), `63:62` = `ppl` (copie de `PSR.cpl`, 2 bits). Bits `51:38` et `61:58` réservés. | § 3.1.8.12, Fig. 3-7, p. `1:32` |
| `AR 65` | `LC` | **I** | *Loop Count* : compteur de boucle 64 bits (`br.cloop`, `br.ctop`, `br.cexit`). | § 3.1.8.13, p. `1:33` |
| `AR 66` | `EC` | **I** | *Epilog Count* : compteur d'épilogue sur **6 bits** (bits `5:0` ; bits `63:6` ignorés = `0`). | § 3.1.8.14, Fig. 3-8, p. `1:33` |

---

## 5. Modèle mémoire applicatif (§ 3.2, p. `1:36`–`1:37`)

- **Adressage :** mémoire adressable à l'octet via des pointeurs 64 bits (§ 3.2.1, p. `1:36`).
- **Unités adressables :** 1, 2, 4, 8, 10 (flottant précision étendue) et 16 octets (§ 3.2.2, p. `1:36`).
- **Justification dans les registres généraux :** une valeur de 1, 2 ou 4 octets chargée en registre général est toujours placée dans les bits de poids faible et **étendue par des zéros** (*zero-extended*) sur 64 bits (§ 3.2.2, p. `1:36` et § 4.4.1, p. `1:58`).
- **Alignement :**
  - Les bundles d'instructions (16 octets) sont **toujours alignés sur 16 octets** (§ 3.2.2, p. `1:36`).
  - Pour les données, si `UM.ac = 1`, tout accès non aligné sur sa frontière naturelle déclenche une faute `Unaligned Data Reference` (Table 3-6, p. `1:34`). Tout accès qui franchit une frontière de page de 4 Ko déclenche cette faute quel que soit `UM.ac` (§ 4.4, p. `1:57`).
- **Ordre des octets (Endianness, § 3.2.3, Figures 3-13 et 3-14, p. `1:36`–`1:37`) :**
  - Contrôlé par le bit `UM.be` (`0` = little-endian, `1` = big-endian) pour les chargements et écritures de données en mode Itanium.
  - **Exception importante :** la lecture des instructions (*instruction fetch*) est **toujours en little-endian** (le champ `template` est toujours dans l'octet 0 du bundle), quel que soit `UM.be` (§ 3.2.3, p. `1:37`). Le RSE a son propre bit `RSC.be` (Table 3-4, p. `1:29`).


---

## 6. Regard critique & limites architecturales (pour IA-86)

| Élément | Verdict | Analyse critique |
|---|---|---|
| **65e bit `NaT` sur chaque `GR` et 82 bits sur chaque `FR`** | ❌ **Coût disproportionné** | Ces largeurs non puissances de 2 (65 et 82 bits) cassent la symétrie des bus internes, interdisent de stocker directement un registre sur sa taille naturelle en mémoire et obligent à inventer des mécanismes dédiés (`UNAT`, `RNAT` toutes les 63 sauvegardes, `stf.spill` sur 16 octets) uniquement pour ne pas perdre 1 bit d'état spéculatif lors d'un *spill*. |
| **Comportement asymétrique à l'écriture (`r0` = faute vs `p0` = ignoré)** | ✅ **Bien pensé** | Ignorer l'écriture sur `p0` permet d'utiliser une seule instruction `cmp p1, p0 = ...` quand on n'a besoin que d'un seul prédicat, tandis que piéger l'écriture sur `r0` attrape les bogues de génération de code (puisqu'une instruction qui ne veut pas écrire de registre général n'a pas de raison d'exister, sauf `nop`). |
| **Fragmentation des registres spéciaux (`BR`, `AR(I)` vs `AR(M)`, `CFM`)** | ⚠️ **À simplifier** | Séparer 8 registres de branchement `BR` des `GR` et diviser les `AR` entre unité `M` et unité `I` multiplie les transferts `mov` inter-banques (et les stops `;;` associés). Dans IA-86, pouvoir sauter directement sur un `GR` (comme RISC-V) éviterait cette latence de préparation de saut indirect. |
