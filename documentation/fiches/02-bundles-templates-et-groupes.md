# Fiche 2 — Bundles, templates, groupes d'instructions et dépendances (IA-64)

**Sources principales :** Manuel Intel® Itanium® Architecture Software Developer's Manual, Révision 2.3 :
- Volume 1, Partie 1, Chapitre 3 (§ 3.3 à 3.5, pages `1:38` à `1:45`).
- Volume 3, Chapitre 4 (Introduction et § 4.1, pages `3:293` à `3:295`).

---

## 1. Types d'instructions et unités d'exécution (§ 3.3, Table 3-9, p. `1:38` ; Vol. 3, Table 4-1, p. `3:293`)

Chaque instruction IA-64 appartient à l'un des **6 types d'instructions** et s'exécute sur un type d'unité d'exécution :

| Type d'instruction | Description | Unité d'exécution compatible | Exemples d'instructions |
|---|---|---|---|
| **A** (*Integer ALU*) | Arithmétique, logique et comparaisons entières | **I-unit** ou **M-unit** | `add`, `sub`, `and`, `or`, `xor`, `shladd`, `cmp`, `cmp4` |
| **I** (*Non-ALU integer*) | Décalages, champs de bits, transferts BR/PR/AR(I) | **I-unit** uniquement | `shl`, `shr`, `extr`, `dep`, `tbit`, `tnat`, `sxt`, `zxt`, `mov br`, `mov pr`, `mov.i ar` |
| **M** (*Memory*) | Accès mémoire, pile de registres, spéculation, AR(M) | **M-unit** uniquement | `ld`, `st`, `alloc`, `chk.s`, `chk.a`, `invala`, `mov.m ar` |
| **F** (*Floating-point*) | Calculs et comparaisons flottants, `xma` | **F-unit** uniquement | `fma`, `fms`, `fnma`, `fcmp`, `fclass`, `xmpy` |
| **B** (*Branch*) | Sauts, appels, retours, boucles, `clrrrb`, `rfi` | **B-unit** uniquement | `br.cond`, `br.call`, `br.ret`, `br.cloop`, `br.ctop`, `clrrrb` |
| **L+X** (*Extended*) | Constante 64 bits ou saut long (occupe **2 slots** : slot 1 = `L`, slot 2 = `X`) | **I-unit** (opcodes majeurs `0`–`7`, ex. `movl`) ou **B-unit** (opcodes majeurs `8`–`F`, ex. `brl`) | `movl`, `brl.cond`, `brl.call`, `nop.x`, `break.x` |

---

## 2. Structure du bundle 128 bits (§ 3.3, Figure 3-15, p. `1:38`)

Trois instructions de **41 bits** et un champ **template de 5 bits** sont empaquetés dans un conteneur de **128 bits** (16 octets, toujours aligné sur 16 octets) appelé **bundle** :

```text
 127                  87 86                   46 45                    5 4      0
+-----------------------+-----------------------+-----------------------+--------+
|   instruction slot 2  |   instruction slot 1  |   instruction slot 0  |template|
|        41 bits        |        41 bits        |        41 bits        | 5 bits |
+-----------------------+-----------------------+-----------------------+--------+
```

- En mémoire, chaque bundle est stocké en **little-endian** (§ 3.2.3, p. `1:37` et § 3.4, p. `1:39`) : l'octet `0` du bundle contient les bits `4:0` (`template`) et les 3 bits de poids faible du `slot 0`.
- L'ordre d'exécution séquentiel au sein d'un bundle va toujours du **slot 0** au **slot 2** (§ 3.4, p. `1:39`).
- Dans un template `MLX` (`0x04` ou `0x05`), l'instruction étendue de 82 bits utilise les 41 bits du **slot 1 (`L-unit`)** pour l'immédiat supplémentaire (`imm41` ou `imm39`) et les 41 bits du **slot 2 (`X-unit`)** pour l'opcode majeur, le prédicat `qp` et le reste des champs (Vol. 3, Table 4-4, p. `3:297`).

---

## 3. Table complète des 32 templates et position des stops (Table 3-10, p. `1:38`–`1:39` et Vol. 3, Table 4-2, p. `3:294`)

> **Contrôlé visuellement sur les PDF (29/10/2026).**
> Un `;;` indique un **stop architectural** (double trait dans la table du manuel).
> **Loi binaire :** sur les 24 templates valides, le **bit 0 (`template & 1`) vaut `1` si et seulement s'il y a un stop à la fin du bundle (après le slot 2)**.

| Code (hex) | Binaire | Mnémonique | Slot 0 | Slot 1 | Slot 2 | Stops dans le bundle |
|---|---|---|---|---|---|---|
| `0x00` | `00000` | `MII` | M-unit | I-unit | I-unit | Aucun |
| `0x01` | `00001` | `MII_` | M-unit | I-unit | I-unit `;;` | Après slot 2 |
| `0x02` | `00010` | `MI_I` | M-unit | I-unit `;;` | I-unit | Après slot 1 |
| `0x03` | `00011` | `MI_I_` | M-unit | I-unit `;;` | I-unit `;;` | Après slot 1 **et** après slot 2 |
| `0x04` | `00100` | `MLX` | M-unit | L-unit | X-unit | Aucun |
| `0x05` | `00101` | `MLX_` | M-unit | L-unit | X-unit `;;` | Après slot 2 |
| `0x06` | `00110` | — | *Réservé* | *Réservé* | *Réservé* | Faute `Illegal Operation` |
| `0x07` | `00111` | — | *Réservé* | *Réservé* | *Réservé* | Faute `Illegal Operation` |
| `0x08` | `01000` | `MMI` | M-unit | M-unit | I-unit | Aucun |
| `0x09` | `01001` | `MMI_` | M-unit | M-unit | I-unit `;;` | Après slot 2 |
| `0x0A` | `01010` | `M_MI` | M-unit `;;` | M-unit | I-unit | Après slot 0 |
| `0x0B` | `01011` | `M_MI_` | M-unit `;;` | M-unit | I-unit `;;` | Après slot 0 **et** après slot 2 |
| `0x0C` | `01100` | `MFI` | M-unit | F-unit | I-unit | Aucun |
| `0x0D` | `01101` | `MFI_` | M-unit | F-unit | I-unit `;;` | Après slot 2 |
| `0x0E` | `01110` | `MMF` | M-unit | M-unit | F-unit | Aucun |
| `0x0F` | `01111` | `MMF_` | M-unit | M-unit | F-unit `;;` | Après slot 2 |
| `0x10` | `10000` | `MIB` | M-unit | I-unit | B-unit | Aucun |
| `0x11` | `10001` | `MIB_` | M-unit | I-unit | B-unit `;;` | Après slot 2 |
| `0x12` | `10010` | `MBB` | M-unit | B-unit | B-unit | Aucun |
| `0x13` | `10011` | `MBB_` | M-unit | B-unit | B-unit `;;` | Après slot 2 |
| `0x14` | `10100` | — | *Réservé* | *Réservé* | *Réservé* | Faute `Illegal Operation` |
| `0x15` | `10101` | — | *Réservé* | *Réservé* | *Réservé* | Faute `Illegal Operation` |
| `0x16` | `10110` | `BBB` | B-unit | B-unit | B-unit | Aucun |
| `0x17` | `10111` | `BBB_` | B-unit | B-unit | B-unit `;;` | Après slot 2 |
| `0x18` | `11000` | `MMB` | M-unit | M-unit | B-unit | Aucun |
| `0x19` | `11001` | `MMB_` | M-unit | M-unit | B-unit `;;` | Après slot 2 |
| `0x1A` | `11010` | — | *Réservé* | *Réservé* | *Réservé* | Faute `Illegal Operation` |
| `0x1B` | `11011` | — | *Réservé* | *Réservé* | *Réservé* | Faute `Illegal Operation` |
| `0x1C` | `11100` | `MFB` | M-unit | F-unit | B-unit | Aucun |
| `0x1D` | `11101` | `MFB_` | M-unit | F-unit | B-unit `;;` | Après slot 2 |
| `0x1E` | `11110` | — | *Réservé* | *Réservé* | *Réservé* | Faute `Illegal Operation` |
| `0x1F` | `11111` | — | *Réservé* | *Réservé* | *Réservé* | Faute `Illegal Operation` |

*Note a de la Table 3-10 (p. `1:39`) :* Le template `MLX` s'appelait initialement `MLI` ; par compatibilité, le slot `X` accepte `break.i` et `nop.i` en plus des instructions `X`.

---

## 4. Groupes d'instructions (*Instruction Groups*, § 3.4, p. `1:39`–`1:41`)

### 4.1 Définition et frontières (§ 3.4, p. `1:40`)
Un **groupe d'instructions** est une séquence d'instructions consécutives (en avançant de slot en slot et de bundle en bundle) qui se termine dès que l'un des événements suivants survient :
1. Un **stop architectural** (`;;` encodé dans le template).
2. Un **branchement pris** (*taken branch*).
3. Une faute `Break Instruction` causée par `break.b`, ou une faute `Illegal Operation` dans l'espace d'opcodes de type B.

**Attention — Ce qui NE termine PAS un groupe d'instructions (p. `1:40`) :**
- La fin d'un bundle sans stop (ex. template pair `0x00`, `0x08`, `0x0C`...) : le groupe continue dans le bundle suivant.
- Un branchement **non pris**.
- Les instructions `break` autres que `break.b` (`break.i`, `break.m`, `break.f`, `break.x`).
- Les vérifications de spéculation (`chk.s`, `chk.a`, `fchkf`), même lorsqu'elles sautent au code de récupération.
- `rfi` et `brl` s'ils ne sont pas suivis d'un stop.

### 4.2 Modèle d'exécution en 4 phases (§ 3.4, p. `1:39`–`1:40`)
Chaque instruction traverse logiquement 4 phases : 1. *fetch* → 2. *read* → 3. *execute* → 4. *update*.
- **Entre deux groupes séparés par un stop :** toutes les lectures (*read*) du second groupe ont lieu après toutes les mises à jour (*update*) du premier groupe (latence architecturale unitaire).
- **À l'intérieur d'un même groupe :**
  - **Registres :** chaque instruction se comporte comme si elle lisait l'état des registres **avant** toute mise à jour par n'importe quelle instruction du même groupe (sauf exceptions § 3.4.1).
  - **Mémoire et ALAT :** chaque instruction se comporte comme si elle lisait la mémoire et l'ALAT **après** les mises à jour de toutes les instructions **précédentes** du même groupe.

---

## 5. Règles de dépendances dans un même groupe (§ 3.4 à 3.4.4, p. `1:41`–`1:44`)

> **Règle d'or sur la prédication (p. `1:41`) :** Les interdictions de dépendances ci-dessous ne s'appliquent qu'entre les instructions dont les lectures et écritures **ne sont pas dynamiquement désactivées par leur prédicat qualificatif (`qp = 0`)**. Si deux instructions écrivent le même registre dans le même groupe mais ont des prédicats mutuellement exclusifs (ex. `(p1)` et `(p2)` où un seul vaut `1`), c'est **légal**.
> **Exception :** Les branchements de boucle (`br.cloop`, `br.ctop`, `br.cexit`, `br.wtop`, `br.wexit`) sont **toujours** considérés comme lecteurs/écrivains de leurs ressources, même si leur prédicat vaut `0` (p. `1:43`).

### 5.1 Tableau récapitulatif des dépendances au sein d'un groupe

| Type de dépendance | Registres (GR, FR, PR, BR, AR, CFM) | Mémoire et ALAT |
|---|---|---|
| **RAW** (*Read-After-Write*) | ❌ **Interdit** (sauf 4 exceptions § 5.2) | ✅ **Autorisé** : un `ld` voit la valeur du `st` précédent à la même adresse. |
| **WAW** (*Write-After-Write*) | ❌ **Interdit** (sauf 3 exceptions § 5.3) | ✅ **Autorisé** : le dernier `st` du groupe l'emporte en mémoire. |
| **WAR** (*Write-After-Read*) | ✅ **Autorisé** (sauf 1 exception sur `PR63` § 5.4) | ✅ **Autorisé** : un `st` après un `ld` ne modifie pas la valeur lue par ce `ld`. |

*Note :* `PR0` est exclu de toute restriction de dépendance (écritures ignorées, lecture = `1`). En revanche, `IP` se comporte en série : les dépendances RAW et WAW sur `IP` sont permises (chaque lecture de `IP` donne l'adresse du bundle qui la contient, p. `1:42`).

### 5.2 Les 4 exceptions où un RAW registre est AUTORISÉ dans un même groupe (§ 3.4.1, p. `1:42`–`1:43`)
1. **`alloc` et `CFM` :** `alloc` doit être la **première instruction du groupe** et modifie implicitement `CFM`. Toutes les instructions suivantes du même groupe qui accèdent aux registres empilés (`r32`–`r127`) voient immédiatement le nouveau cadre alloué par `alloc`.
   - *Attention :* restent **interdits** dans le même groupe après `alloc` : `flushrs`, `loadrs`, `mov = ar.bspstore`, `mov = ar.rnat`, tous les branchements (`br.call`, `br.ret`, `br.ctop`, `br.cexit`, `br.wtop`, `br.wexit`, `br.ia`), `clrrrb`, `cover`, `rfi`.
2. **`ld.c` (*check load*) :** une instruction qui lit le registre cible d'un `ld.c` précédent dans le même groupe est **autorisée** et reçoit la nouvelle valeur éventuellement rechargée par `ld.c`.
3. **Écritures avant un branchement du même groupe :**
   - Écrire dans un **registre de branchement (`BR`)** ou dans **`AR[PFS]`** (par une instruction non-branchement), puis exécuter un branchement qui les lit dans le même groupe est **autorisé** (le branchement voit la nouvelle valeur).
   - Écrire dans un **registre de prédicat (`PR`)** par une instruction **non flottante** (ex. `cmp`, `tbit`, `tnat`, `mov pr`), puis exécuter un branchement conditionnel qui lit ce prédicat dans le même groupe est **autorisé** !
   - ❌ **Interdit avant un branchement dans le même groupe :** écrire dans `AR[LC]` ou `AR[EC]` (il faut obligatoirement un stop `;;` entre `mov ar.lc = ...` et `br.cloop`/`br.ctop`), ou écrire un prédicat avec une comparaison **flottante** (`fcmp`, `fclass`, `frcpa`) lu par un branchement du même groupe.
4. **`ld8.fill` et `st8.spill` sur `AR[UNAT]` :** les dépendances RAW sur `UNAT` entre instructions `spill`/`fill` sont vérifiées **au niveau du bit** (bits `8:3` de l'adresse mémoire). Plusieurs `st8.spill`/`ld8.fill` peuvent coexister dans le même groupe s'ils touchent des bits différents de `UNAT`.

### 5.3 Les 3 exceptions où un WAW registre est AUTORISÉ dans un même groupe (§ 3.4.2, p. `1:43`)
1. **Comparaisons parallèles :** plusieurs instructions de type comparaison (`cmp`, `cmp4`, `tbit`, `tnat`, `tf`, `fcmp`, `fclass`, `frcpa`, `frsqrta`) peuvent cibler le **même prédicat** dans un même groupe si :
   - elles sont **toutes de type AND** (`.and`, `.andcm`, ou la cible AND d'un DeMorgan), **ou**
   - elles sont **toutes de type OR** (`.or`, `.orcm`, ou la cible OR d'un DeMorgan), **ou**
   - la cible est **`PR0`**.
2. **Instructions flottantes sur `FPSR` et `PSR` :** plusieurs instructions flottantes dans un groupe combinent leurs écritures d'état par un OU logique (`OR`).
3. **`st8.spill` multiples sur `UNAT` :** autorisés dans le même groupe tant qu'ils n'écrivent pas le **même bit** de `UNAT`.

### 5.4 L'unique cas où un WAR registre est INTERDIT dans un même groupe (§ 3.4.3, p. `1:44`)
- **Lecture de `PR63` par une instruction de type B** suivie (sans stop intermédiaire) d'un branchement de boucle pipelinée (`br.ctop`, `br.cexit`, `br.wtop`, `br.wexit`) qui écrit `PR63` avant rotation : **interdit**.

### 5.5 Sanction d'une violation de dépendance (§ 3.4.4 et § 3.5, p. `1:44`–`1:45`)
- Le comportement du programme est **architecturalement indéfini** (*undefined behavior*).
- Les processeurs qui implémentent une détection matérielle déclenchent une faute **`Illegal Dependency Fault`** sur la **seconde** instruction en cause (le lecteur pour un RAW, le second écrivain pour un WAW, l'écrivain de `PR63` pour le WAR).


---

## 6. Regard critique & limites architecturales (pour IA-86)

| Élément | Verdict | Analyse critique |
|---|---|---|
| **Templates 5 bits combinant types d'unités ET position des stops** | ❌ **Rigidité excessive** | Sur 5 bits (24 templates valides), il est impossible de représenter toutes les combinaisons d'unités (`F`, `B`, `M`, `I`) et de stops internes (seuls `MI_I` et `M_MI` ont un stop au milieu !). Le compilateur est forcé d'insérer massivement des `nop.m`/`nop.i`/`nop.b` dès qu'un stop tombe après le slot 0 ou 1 d'un autre triplet, ce qui gonfle le binaire et gaspille le cache L1I. |
| **Le paradoxe de la vérification des dépendances (§ 3.4.1–3.4.4)** | ❌ **Promesse EPIC non tenue** | EPIC promettait de supprimer la logique matérielle de détection des dépendances inter-instructions. En réalité, la spécification accumule tellement d'exceptions (`alloc`, `ld.c`, écriture `BR`/`PFS`/`PR` avant saut, `UNAT` au bit près, `PR63` en WAR) et demande au processeur de lever `Illegal Dependency Fault` que le matériel doit quand même embarquer un réseau complexe de comparateurs de registres. |
| **Instruction 64 bits `movl` / `brl` limitée aux slots 1+2 du seul template `MLX`** | ⚠️ **Contrainte lourde** | Charger une constante 64 bits impose le template `MLX` (`0x04`/`0x05`), dont le slot 0 est obligatoirement `M`. Impossible de combiner `movl` avec un branchement ou un flottant dans le même bundle. |
| **Séparation entre « groupement parallèle explicite (stops) » et « routage vers les unités »** | 💡 **Piste clé pour IA-86** | L'idée d'indiquer explicitement les frontières de dépendance (stops) est intéressante, mais **la coupler rigidement au routage d'unités (`M/I/F/B`) par paquets de 3 slots de 41 bits est l'erreur centrale d'encodage d'IA-64**. |
