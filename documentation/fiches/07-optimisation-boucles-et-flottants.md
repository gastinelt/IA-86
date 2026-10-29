# Fiche 7 — Guide d'optimisation : pipeline logiciel de boucles et calcul flottant (Volume 1, Partie 2, Chap. 5 et 6)

**Sources principales :** Manuel Intel® Itanium® Architecture Software Developer's Manual, Révision 2.3, Volume 1, Partie 2 (*Optimization Guide*), Chapitres 5 et 6 (pages `1:181` à `1:218`).

---

## 1. Pipeline logiciel de boucles (*Software Pipelining*, Chap. 5, p. `1:181`–`1:204`)

### 1.1 Pourquoi dérouler une boucle (*Loop Unrolling*) ne suffit pas (§ 5.3.1, p. `1:182`–`1:183`)
Soit la boucle élémentaire `for (i=0; i<N; i++) *p6++ = *p5++ + r9;` (`ld4` latence 2 cycles, `add` 1 cycle, `st4`).
- Sans optimisation, une itération prend **4 cycles** (cycles 0, 2, 3).
- En déroulant la boucle 2 ou 4 fois (et en dédoublant les pointeurs d'induction `r5`, `r15`, `r25`, `r35`), le code grossit de $2\times$ à $4\times$ et il reste toujours un cycle de bulle pendant la latence des chargements.

### 1.2 Principe du pipeline logiciel et *Initiation Interval* (`II`, § 5.3.2 à 5.4, p. `1:183`–`1:188`)
Le pipeline logiciel découpe une itération source en $S$ **étages** de durée constante **`II`** (*Initiation Interval*, nombre de cycles entre le lancement de deux itérations successives).

Exemple complet du manuel (§ 5.4.3.2, Table 5-1, p. `1:187`–`1:188`) avec **`II = 1` cycle** et **$S = 4$ étages** (l'étage 2 est vide pour absorber les 2 cycles de latence du `ld4`) :
- Étage 1 (`p16`) : `(p16) ld4 r32 = [r5], 4` *(écrit `r32` ; après 2 rotations, à l'étage 3, cette valeur est dans **`r34`** !)*
- Étage 2 (`p17`) : *(vide — attente de la latence mémoire)*
- Étage 3 (`p18`) : `(p18) add r35 = r34, r9` *(lit `r34`, écrit `r35` ; après 1 rotation, à l'étage 4, cette valeur est dans **`r36`** !)*
- Étage 4 (`p19`) : `(p19) st4 [r6] = r36, 4` *(lit `r36` et l'écrit en mémoire)*

**Code assembleur complet (sans aucune expansion de code : 1 seul cycle par itération en régime permanent) :**
```assembly
    mov lc = 199            // 200 itérations : LC = N - 1
    mov ec = 4              // 4 étages : EC = S
    mov pr.rot = 1 << 16 ;; // p16 = 1, p17..p63 = 0 (suppose rrb.pr == 0)
L1:
    (p16) ld4 r32 = [r5], 4
    (p18) add r35 = r34, r9
    (p19) st4 [r6] = r36, 4
    br.ctop.sptk L1 ;;      // Tout le corps tient en 1 groupe (1 cycle) !
```

**Trace d'exécution (Table 5-1, p. `1:188`) :**
- **Cycles 0 à 2 (Prologue)** : remplissage progressif (`p16..p19` passent de `1000` à `1100` puis `1110`, `LC` descend de `199` à `197`, `EC = 4`).
- **Cycles 3 à 199 (Noyau / *Kernel*)** : pipeline plein (`p16..p19 = 1111`) : chaque cycle exécute simultanément le `ld4` de l'itération $k$, l'`add` de $k-2$ et le `st4` de $k-3$. Au cycle 199, `LC` atteint `0`.
- **Cycles 200 à 202 (Épilogue)** : `LC = 0`, `br.ctop` injecte des `0` dans `p16` (`0111`, `0011`, `0001`) et décrémente `EC` (`3`, `2`, `1`). À la fin du cycle 202 (`EC == 1`), `br.ctop` fait une dernière rotation (`p16..p19 = 0000`, `EC = 0`) et sort de la boucle.

> **Règle d'allocation des registres (Note p. `1:188`) :** Les variables modifiées à chaque itération (*loop-variant*) vont dans les **registres tournants** (`r32+`), mais les **variables d'induction post-incrémentées** (`r5`, `r6`) doivent impérativement rester dans les **registres statiques** (`r1`–`r31`) pour ne pas tourner !

---

## 2. Cas complexes de boucles pipelinées (§ 5.5, p. `1:190`–`1:204`)

### 2.1 Boucles `while` pipelinées (`br.wtop`) et étages spéculatifs (§ 5.5.1, Table 5-2, p. `1:190`–`1:192`)
Dans une boucle `while`, la condition de continuation n'est connue qu'après avoir chargé la donnée (ex. à l'étage 3). Pour démarrer une nouvelle itération à chaque cycle (`II = 1`), les étages qui précèdent le `cmp` (étages 1 et 2) doivent être **spéculatifs** (`ld4.s` sans prédicat d'étage) :
- Le `cmp.ne p17, p0 = r34, r0` à l'étage 3 écrit `p17`, qui sert **immédiatement** de prédicat qualificatif à `(p17) br.wtop.sptk L1 ;;` dans le même groupe, puis devient après rotation **`p18`**, le prédicat du premier étage non spéculatif (`chk.s`, `cmp`, `st4`) du cycle suivant !
- Comme `p17` vaut `0` pendant les $K$ premiers cycles spéculatifs du prologue (avant que le premier `cmp` ne s'exécute), `br.wtop` décrémente `EC` pendant ce prologue. Il faut donc initialiser **`EC = (# étages d'épilogue) + (# étages spéculatifs)`** (dans l'exemple p. `1:191`–`1:192` : $0 + 2 = 2$).

### 2.2 Prédicats internes au corps de boucle (§ 5.5.2, p. `1:192`–`1:193`)
Si la boucle d'origine contient déjà un `if/else` prédiqué (`fcmp.ge.unc p1, p2 = ...`), ces instructions **ne reçoivent pas un prédicat d'étage `p16+`**, mais utilisent des prédicats tournants dédiés (`p31, p32` écrits par `fcmp.ge.unc` sous le prédicat d'étage `(p20)`, qui deviennent `p32, p33` à l'étage suivant pour garder les `stfs`).

### 2.3 Boucles à sorties multiples et `br.cexit` (§ 5.5.3 et § 5.5.6, p. `1:193`–`1:200`)
- **`br.cexit`** est l'inverse de `br.ctop` : il continue la boucle quand il **n'est pas pris** (*fall-through*) et sort de la boucle quand il **est pris**.
- Il permet de dérouler une boucle avant de la pipeliner lorsque le nombre d'unités mémoire est limitant (ex. 3 accès mémoire pour 2 unités M : `II = 2` sans déroulage, mais `II = 3` pour 2 itérations après déroulage, soit **`1,5 cycle/itération`**, § 5.5.6 p. `1:198`–`1:199`).

### 2.4 Pièges d'interaction entre rotation des registres et `ALAT` (§ 5.5.5, p. `1:196`–`1:197`)
Un `ld.a r32 = [r8]` placé dans une boucle pipelinée cible un **nouveau registre physique à chaque rotation** et alloue donc **une nouvelle entrée dans l'`ALAT` à chaque itération** !
- Sur un processeur ayant une `ALAT` de **32 entrées**, si le `ld.c` correspondant est exécuté **32 rotations plus tard** (`(p48) ld4.c r64 = [r8]`), le 32e `ld.a` vient tout juste d'écraser la 1re entrée de l'`ALAT` : **100 % des `ld.c` échouent (*ALAT miss*)** ! Il faut vérifier dans les 31 itérations (`p47` / `r63`) ou utiliser des registres statiques en déroulant.

### 2.5 Réductions (`fma`) et élimination de chargements redondants (§ 5.5.7 et § 5.5.9, p. `1:200`–`1:204`)
- **Réduction (`sum += a[i] * b[i]`, § 5.5.7, p. `1:200`–`1:201`) :** si la latence de `fma` est de 5 cycles, au lieu de dérouler 5 fois la boucle, la rotation des registres flottants (`fma f32 = f4, f9, f37` avec `f33..f37` initialisés à `0`) entrelace automatiquement **5 sommes partielles indépendantes** en une seule instruction `fma` (`II = 1` cycle), qu'on additionne à la sortie de la boucle !
- **Chargement redondant (`a[i] + a[i+1]`, § 5.5.9, p. `1:204`) :** en chargeant `a[0]` dans `r33` avant la boucle et `a[i+1]` dans `r32` à chaque tour, la rotation fait automatiquement passer `a[i+1]` de `r32` à `r33` au tour suivant, éliminant un chargement sur deux et toute instruction `mov` sans dérouler la boucle.

---

## 3. Optimisations des calculs flottants (Chap. 6, p. `1:205`–`1:218`)

1. **Format uniforme 82 bits et `fma` (§ 6.3.1 et § 6.3.2, p. `1:207`–`1:210`) :**
   - Tous les chargements (`ldfs`, `ldfd`, `ldfe`) convertissent à la volée vers le format registre 82 bits (1 bit signe, 17 bits exposant, 64 bits mantisse).
   - L'instruction de base est le **Fused Multiply-Add (`fma`)** ($f_1 = f_3 \times f_4 + f_2$ avec un seul arrondi final).
2. **Absence d'instructions matérielles de division et de racine carrée (§ 6.3.3, p. `1:211`–`1:212`) :**
   - IA-64 **n'a pas d'instruction `fdiv` ni `fsqrt` directe** : la division et la racine carrée se font **en logiciel** par itérations de Newton-Raphson sur `fma`/`fnma`, amorcées par **`frcpa`** (*Floating-point Reciprocal Approximation*, précision ~8 bits, écrit `f8` et le prédicat `p6`) ou **`frsqrta`** ($1/\sqrt{x}$).
   - En double précision, une division prend **10 instructions (8 groupes)** en mode débit maximal ou **13 instructions (7 groupes)** en mode latence minimale ; une racine carrée prend **14 à 17 instructions** (p. `1:211`).
   - *Avantage revendiqué par Intel :* dans une boucle vectorisable, les `fma` de plusieurs divisions successives peuvent être **pipelinés logiciellement**, alors qu'un diviseur matériel classique bloque le pipeline. *(En contrepartie, cela gonfle fortement la taille du code — point important pour notre Phase 2).*
3. **Les 4 champs d'état flottants indépendants (`s0` à `s3` dans `FPSR`, § 6.3.5, p. `1:212`–`1:214`) :**
   - **`s0`** est le champ d'état architectural principal (IEEE 754).
   - **`s1`** est utilisé par les étapes intermédiaires des séquences de division/racine carrée (qui peuvent déborder ou être inexactes sans que le résultat final le soit) : ses drapeaux d'exception sont ignorés, seules la première instruction (`frcpa.s0`) et la dernière (`fma.d.s0`) écrivent dans `s0`.
   - **`s2` et `s3`** servent aux calculs flottants spéculatifs (ou à l'arithmétique d'intervalles avec des modes d'arrondi distincts) pour ne pas polluer `s0` tant que la spéculation n'est pas validée.
4. **Instructions `fmin`/`fmax` non commutatives et `ldfp` (§ 6.3.6.2 et § 6.3.7.1, p. `1:214`–`1:217`) :**
   - `fmin f1 = f2, f3` équivaut exactement à ` (f2 < f3) ? f2 : f3 ` : si `f2` ou `f3` est `NaN`, la comparaison `<` est fausse et l'instruction renvoie toujours le **second opérande `f3`**. Selon l'ordre des opérandes dans une boucle de recherche de minimum, on peut donc choisir d'ignorer les `NaN` ou de les propager (p. `1:215`).
   - **`ldfp8` / `ldfps` / `ldfpd` (*Load Pair*, p. `1:216`–`1:217`) :** charge deux flottants contigus en une seule instruction M, ce qui double la bande passante mémoire (ex. produit scalaire `DDOT` à 2 itérations par cycle). **Contrainte matérielle stricte :** les deux registres cibles doivent être une paire de registres **physiques** pair/impair (attention à la rotation !), sinon faute `Illegal Operation`.
