# Fiche 5 — Spéculation de contrôle (NaT) et spéculation de données (ALAT) (IA-64)

**Sources principales :** Manuel Intel® Itanium® Architecture Software Developer's Manual, Révision 2.3 :
- Volume 1, Partie 1, Chapitre 4 (§ 4.4.4 et § 4.4.5, pages `1:60` à `1:69`).
- Volume 3, Chapitre 2 (`ld`, `chk.s`, `chk.a`, `invala`, `st8.spill`, `ld8.fill`, pages `3:33`, `3:130`, `3:151`–`3:159`, `3:247`).

---

## 1. Spéculation de contrôle (*Control Speculation*, § 4.4.4, p. `1:60`–`1:63`)

### 1.1 Objectif
Exécuter un chargement mémoire à longue latence (`ld.s`, `ldf.s`, `ldfp.s`) et la chaîne de calculs qui en dépend **avant** le branchement conditionnel qui décide si ce calcul est réellement nécessaire, sans déclencher prématurément une exception (ex. pointeur nul ou page absente).

### 1.2 Jeton d'exception différée : bit `NaT` et `NaTVal` (§ 4.4.4.1, p. `1:60`–`1:61`)
Lorsqu'une exception est différée (*deferred*) par un chargement spéculatif :
- **Dans un registre général (`GR0`–`GR127`) :** le **65e bit** du registre, appelé **bit `NaT`** (*Not a Thing*), est mis à **`1`** (et les 64 bits de données reçoivent une valeur dépendante de l'implémentation sur laquelle le logiciel ne doit jamais compter ; dans beaucoup d'implémentations, `0`).
- **Dans un registre flottant (`FR0`–`FR127`) :** le registre reçoit un encodage pseudo-zéro spécial sur 82 bits appelé **`NaTVal`** (*Not a Thing Value* : signe = `0`, exposant = `0x1FFFE`, mantisse = `0`, Vol. 1 § 5.1.2 p. `1:86`).

### 1.3 Propagation et génération (§ 4.4.4.2, p. `1:61`)
1. **Qui génère un jeton `NaT` / `NaTVal` ?** Uniquement les **chargements spéculatifs** (`ld.s`, `ld.sa`, `ldf.s`, `ldf.sa`, `ldfp.s`, `ldfp.sa`), ou une recharge `ld8.fill` / `ldf.fill` depuis un état sauvegardé.
2. **Qui propage un jeton `NaT` / `NaTVal` ?** Presque toutes les **instructions de calcul** qui écrivent dans un `GR` ou un `FR` (`add`, `sub`, `shl`, `extr`, `fma`, etc.) : si l'une de leurs sources a `NaT == 1` (ou `NaTVal`), elles ne lèvent aucune faute et mettent `NaT = 1` (ou `NaTVal`) sur leur registre destination.
   - *Cas particulier du post-incrément d'adresse (`ld r1 = [r3], r2`, Vol. 3 p. `3:151`) :* si `r2.nat == 1`, le bit `r3.nat` est mis à `1` sans lever de faute. En revanche, si `r3.nat == 1` sur un chargement **non spéculatif**, une faute `Register NaT Consumption` est levée !
3. **Qui consomme un jeton `NaT` en levant une faute ?** Toute **instruction non spéculative** (ex. `st`, chargement non spéculatif utilisant `r3` comme adresse, division/sémaphore, `mov to br/ar`) qui lit un registre avec `NaT == 1` ou `NaTVal` déclenche immédiatement une faute **`Register NaT Consumption`** (§ 4.4.4.4, p. `1:62`).
4. **Action sur les comparaisons (§ 4.4.4.3, p. `1:61`–`1:62` et Table 4-10, p. `1:56`) :** les comparaisons (`cmp`, `cmp4`, `tbit`, `fcmp`) ne lèvent pas de faute mais mettent leurs deux prédicats cibles à `0` (pour les types Normal, `.unc` et `.and`) ou les laissent inchangés (types `.or` et DeMorgan). Pour tester explicitement un `NaT`, on utilise **`tnat`** (pour un `GR`) ou **`fclass`** (pour un `FR`).

### 1.4 Vérification et récupération : `chk.s` (§ 4.4.4.2, p. `1:61`)
À l'endroit du programme où le résultat spéculatif devient réellement nécessaire :
- On exécute **`chk.s r2, target25`** (formats `I20` sur I-unit ou `M20` sur M-unit) ou **`chk.s.m f2, target25`** (`M21`).
- Si `r2.nat == 0` (ou `f2 != NaTVal`) : la spéculation a réussi, l'exécution continue en séquence.
- Si `r2.nat == 1` (ou `f2 == NaTVal`) : `chk.s` saute au code de récupération à `target25`, qui ré-exécute le chargement avec une instruction **non spéculative** (`ld`) puis revient.

### 1.5 Sauvegarde et restauration d'un registre avec son bit `NaT` (`st8.spill` / `ld8.fill`, § 4.4.4.6, p. `1:62`–`1:63`)
Pour sauvegarder sur la pile mémoire un registre général sans perdre son bit `NaT` ni déclencher de faute `Register NaT Consumption` :
- **`st8.spill [r3] = r2`** :
  1. Calcule l'index de bit `pos = r3{8:3}` (les bits 8 à 3 de l'adresse mémoire, soit le numéro du mot de 8 octets modulo 64).
  2. Copie `r2.nat` dans le bit `AR[UNAT]{pos}`.
  3. Écrit les 64 bits de `r2` en mémoire à `[r3]` (si `r2.nat == 0`).
- **`ld8.fill r1 = [r3]`** :
  1. Lit 8 octets en mémoire à `[r3]` dans `r1`.
  2. Restaure `r1.nat = AR[UNAT]{r3{8:3}}`.
- Pour les flottants, `stf.spill` et `ldf.fill` transfèrent directement **16 octets** en mémoire au format registre (82 bits) sans toucher à `UNAT` et sans lever de faute si le registre contient `NaTVal`.

---

## 2. Spéculation de données (*Data Speculation* et `ALAT`, § 4.4.5, p. `1:63`–`1:69`)

### 2.1 Objectif : franchir un `st` potentiellement aliasé (§ 4.4.5.1, p. `1:64`)
Lorsqu'un chargement `ld r6 = [r8]` suit une écriture `st [r4] = r12` et que le compilateur ne peut pas prouver statiquement que `r4 != r8` (*dépendance mémoire ambiguë*), la spéculation de données permet de **remonter le chargement avant le `st`** sous la forme d'un **chargement avancé** (*advanced load* : `ld.a`, `ldf.a`, `ldfp.a`).

### 2.2 La table matérielle `ALAT` (*Advanced Load Address Table*, § 4.4.5.3, p. `1:65`–`1:67`)
Chaque entrée de l'`ALAT` mémorise qu'un chargement avancé a eu lieu :
- **Étiquette de registre (*ALAT register tag*)** : identifie le registre physique cible (`GR` ou `FR`, pour `ldfp.a` seul le 1er registre cible est inscrit).
- **Adresse physique mémoire** et **taille de l'accès** (`1`, `2`, `4`, `8`, `10` ou `16` octets).

#### Cycle de vie d'une entrée `ALAT` :
1. **Allocation par `ld.a` (`ld1.a`, `ld2.a`, `ld4.a`, `ld8.a`, `ldf.a`, `ldfp.a`, p. `1:65`–`1:66`) :**
   - Si une entrée avec la même étiquette de registre existait déjà dans l'`ALAT`, elle est **supprimée**.
   - Une nouvelle entrée `(reg_tag, phys_addr, size)` est insérée dans l'`ALAT`.
   - La donnée est chargée en mémoire dans le registre cible (et son bit `NaT` est mis à `0`).
2. **Invalidation automatique par collision mémoire (§ 4.4.5.3.2, p. `1:67`) :**
   - Dès qu'une écriture mémoire (`st`, `st.rel`, `st.spill`, `stf`), une instruction atomique (`cmpxchg`, `xchg`, `fetchadd`), une invalidation de TLB (`ptc.ga`) ou une éviction de ligne de cache touche une plage d'adresses physiques qui **chevauche** (*overlaps*) l'intervalle `[phys_addr, phys_addr + size - 1]` d'une entrée de l'`ALAT`, cette entrée est **supprimée de l'`ALAT`**.
   - *Note :* Les écritures automatiques du RSE (*RSE spills*) ne sont pas vérifiées contre l'`ALAT` et n'invalident pas ses entrées (p. `1:67`).
3. **Invalidation explicite par instruction :**
   - `invala` : invalide **toutes** les entrées de l'`ALAT`.
   - `invala.e r1` / `invala.e f1` : invalide l'entrée correspondant au registre `r1` ou `f1`.
   - *Obligation logicielle (p. `1:67`) :* le logiciel doit invalider l'`ALAT` lorsqu'il exécute `clrrrb` si des registres tournants ont été cibles de `ld.a`.

> **Flexibilité architecturale importante (p. `1:66`–`1:67`) :**
> La taille et l'associativité de l'`ALAT` dépendent du processeur. Une implémentation a le droit d'invalider n'importe quelle entrée à tout moment (par ex. par manque de place ou à la granularité d'une ligne de cache entière), et **même d'implémenter une `ALAT` de zéro entrée** (auquel cas tous les `ld.c` et `chk.a` se comportent toujours comme un échec de spéculation / *ALAT miss*).

### 2.3 Les deux méthodes de vérification et de récupération (§ 4.4.5.2, Figures 4-2 et 4-3, p. `1:64`–`1:66`)

| Cas d'usage | Instruction de vérification | Si l'entrée est présente dans l'`ALAT` (*Hit*) | Si l'entrée est absente de l'`ALAT` (*Miss* / collision) |
|---|---|---|---|
| **1. Seul le `ld.a` a été remonté** (aucun calcul dépendant n'a encore été exécuté avant le `st`) | **`ld.c` (*Check Load*)**<br>`ld8.c.clr r6 = [r8]` ou `ld8.c.nc r6 = [r8]` | Spéculation réussie : aucun rechargement n'est nécessaire (`r6` est inchangé).<br>*(Rappel § 3.4.1 : une instruction du **même groupe** après `ld.c` a le droit de lire `r6` !)* | Collision détectée : `ld.c` **recharge immédiatement** la bonne valeur depuis la mémoire `[r8]` dans `r6` (sans sauter). |
| **2. Le `ld.a` ET des calculs dépendants** (ex. `add r5 = r6, r7`) ont été remontés avant le `st` | **`chk.a` (*Advanced Load Check*)**<br>`chk.a.clr r6, recover` ou `chk.a.nc r6, recover` | Spéculation réussie : continue en séquence sans sauter. | Collision détectée : `chk.a` **saute à `recover`**, qui recharge `r6`, refait `add r5 = r6, r7`, puis revient après `chk.a`. |

### 2.4 Les completers `.clr` vs `.nc` (§ 4.4.5.5, p. `1:68`–`1:69`)
- **`.clr` (*clear*)** : indique que l'entrée `ALAT` ne servira plus et la **supprime** de l'`ALAT` après la vérification (pour `chk.a.clr`, la suppression n'est garantie que si le test réussit ; en cas de saut vers `recover`, le code de récupération doit faire `invala.e` s'il a besoin que l'entrée soit absente).
- **`.nc` (*no clear*)** : conserve l'entrée dans l'`ALAT` en cas de *hit*, et **ré-alloue** une nouvelle entrée dans l'`ALAT` en cas de *miss* sur un `ld.c.nc` (très utile pour un chargement invariant sorti d'une boucle mais vérifié à chaque itération).

---

## 3. Combinaison des deux spéculations : `ld.sa` (§ 4.4.5.4, p. `1:68`)

Un chargement peut être **à la fois** spéculatif en contrôle et en données grâce au completer **`.sa`** (*speculative advanced load* : `ld1.sa`, `ld2.sa`, `ld4.sa`, `ld8.sa`, `ldf.sa`, `ldfp.sa`) :
- **Si aucune exception ne survient lors du chargement :** se comporte exactement comme un `ld.a` (charge la donnée, met `NaT = 0` et alloue une entrée dans l'`ALAT`).
- **Si une exception est différée lors du chargement :**
  1. Supprime toute entrée existante dans l'`ALAT` pour ce registre cible.
  2. **N'alloue aucune nouvelle entrée** dans l'`ALAT`.
  3. Met le bit **`NaT = 1`** (ou écrit `NaTVal`) dans le registre cible.
- **Conséquence :** comme aucune entrée n'est présente dans l'`ALAT` lorsqu'une exception a été différée, **un seul `ld.c` ou `chk.a` suffit pour détecter à la fois une collision mémoire et une exception différée !**
