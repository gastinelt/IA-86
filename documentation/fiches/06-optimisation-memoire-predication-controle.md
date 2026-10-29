# Fiche 6 — Guide d'optimisation : mémoire, prédication et flot de contrôle (Volume 1, Partie 2, Chap. 1 à 4)

**Sources principales :** Manuel Intel® Itanium® Architecture Software Developer's Manual, Révision 2.3, Volume 1, Partie 2 (*Optimization Guide*), Chapitres 1 à 4 (pages `1:137` à `1:180`).

---

## 1. Objet de la Partie 2 et modèle machine de référence (Chap. 1 et 2, p. `1:137`–`1:146`)

- **Clarification sur l'avertissement de la p. `1:137` :** Le manuel écrit *« Application developers are not advised to use this as a guide to assembly language programming »* uniquement pour préciser que cette Partie 2 expose des **techniques d'optimisation pour compilateurs** sur une implémentation générique hypothétique, et non la référence normative de l'assembleur (qui est dans la Partie 1 et le Volume 3).
- **Modèle machine hypothétique utilisé dans tous les exemples de la Partie 2 (Note p. `1:137`) :**
  - **Unités par cycle :** 2 unités mémoire (**M-unit**) et 4 unités entières (**ALU**).
  - **Latences supposées :** opérations ALU = **1 cycle** ; chargement entier (`ld`) depuis le cache L1 = **2 cycles** ; chargement flottant (`ldf`) = **9 cycles** (p. `1:193`) ; `fma` = **5 cycles** (p. `1:200`) ; pénalité de mauvaise prédiction de branchement = **10 cycles** (p. `1:164`).

---

## 2. Optimisations des accès mémoire et de la spéculation (Chap. 3, p. `1:147`–`1:162`)

### 2.1 Pourquoi la spéculation de données (`ld.a` / `ALAT`) est plus délicate que la spéculation de contrôle (`ld.s` / `NaT`) (§ 3.5.1, p. `1:157`)
Le manuel souligne plusieurs coûts et pièges propres à la spéculation qu'il faudra garder en tête lors de la Phase 2 (analyse critique) :
1. **Interférence entre chargements avancés dans l'`ALAT` :** un chargement spéculatif en contrôle (`ld.s`) ne peut jamais faire échouer un autre `ld.s` (chaque registre a son propre bit `NaT`). À l'inverse, l'`ALAT` a une **capacité matérielle finie** et une associativité dépendante du processeur : émettre un nouveau `ld.a` peut **évincer une entrée `ALAT` existante** et provoquer un faux échec (*false miss*) sur un `ld.c` / `chk.a` ultérieur !
2. **Allongement de la durée de vie des registres (*register pressure*, p. `1:157`) :** même si le code de récupération (*recovery code* d'un `chk.s` ou `chk.a`) n'est jamais exécuté, son existence oblige le compilateur à **garder vivants tous les registres sources** de la chaîne spéculée jusqu'au point de `chk`, ce qui augmente la pression sur les registres et le coût des *spills/fills* du RSE.
3. **Probabilité cumulée de conflit mémoire (§ 3.5.2, p. `1:158`) :** si l'on remonte un `ld.a` au-dessus de deux écritures `*p1` (probabilité d'aliasing $0{,}30$) et `*p2` (probabilité $0{,}40$), la probabilité d'au moins un conflit grimpe à $1 - (0{,}70 \times 0{,}60) = 58\,\%$. Il est également risqué de remonter un `ld.a` au-dessus d'un **appel de fonction** (qui peut faire beaucoup de `st` ou saturer l'`ALAT`).
4. **Expansion de la taille du code (§ 3.5.3, Figure 3-3, p. `1:158`–`1:159`) :** remonter un chargement au-dessus d'un point de convergence de deux branches (`Block B` et `Block C`) oblige à dupliquer le chargement dans chaque branche, sauf si on le remonte encore plus haut dans le bloc dominateur (`Block A`).

### 2.2 Trois techniques pour réduire le coût de la spéculation (§ 3.5.4 à 3.5.6, p. `1:159`–`1:161`)
1. **Combiner plusieurs vérifications `chk.s` en une seule grâce à la propagation du bit `NaT` (§ 3.5.6, p. `1:161`) :**
   ```assembly
   ld4.s r1 = [r10]      // Chargement spéculatif 1
   ld4.s r2 = [r20]      // Chargement spéculatif 2
   add   r3 = r1, r2 ;;  // Propage le NaT si r1 OU r2 a échoué
   // ... autres instructions ...
   chk.s r3, recover     // UN SEUL chk.s sur r3 suffit avant d'utiliser r1, r2 et r3 !
   st4   [r30] = r1
   st4   [r40] = r2
   st4   [r50] = r3
   ```
2. **Sortir des invariants de boucle malgré un pointeur ambigu grâce à `chk.a.nc` (§ 3.5.5, p. `1:160`) :**
   Dans `while (cond) { c = a + b; *ptr++ = c; }`, on calcule `ld4.a r1=[&a]`, `ld4.a r2=[&b]` et `add r3=r1,r2` **une seule fois avant la boucle**, et dans la boucle on exécute `chk.a.nc r1, recover1` et `chk.a.nc r2, recover2`. Le completer **`.nc` (*no clear*) est indispensable** ici : avec `.clr`, l'entrée `ALAT` serait effacée dès la 1re itération et toutes les itérations suivantes échoueraient !
3. **Avantages et piège du post-incrément (`ld r2 = [r1], imm`, § 3.5.4, p. `1:159`–`1:160`) :**
   - *Gain :* fusionne un `ld` et un `add` en une seule instruction (économise un slot I/M et double le débit des paires load+add).
   - *Piège :* crée une dépendance directe sur le registre d'adresse `r1` modifié ; parfois le compilateur a intérêt à séparer `ld` et `add` pendant l'ordonnancement, puis à les refusionner après coup si un post-incrément est possible.

---

## 3. Quand la prédication (*if-conversion*) fait perdre en performance (Chap. 4, § 4.2, p. `1:163`–`1:171`)

La conversion d'un `if / else` en code prédiqué sans branchement (*if-conversion*, § 4.2.3.1, p. `1:165`–`1:166`) supprime les pénalités de mauvaise prédiction, permet de remonter ou descendre des `st` au-delà d'un saut ou d'un label (*upward/downward code motion*, § 4.2.3.3–4.2.3.4, p. `1:167`–`1:168`), et évite de polluer le cache avec des accès mémoire dont le prédicat est faux (§ 4.2.3.5, p. `1:168`).

**Mais le manuel démontre par le calcul deux cas où prédiquer RALENTIT le programme (§ 4.2.4, p. `1:168`–`1:170`) :**

### Cas A : Chemins d'exécution très déséquilibrés (*Unbalanced Execution Paths*, § 4.2.4.1–4.2.4.4, p. `1:169`–`1:170`)
Exemple : `if (r4) r3 = r2 + r1;` (**2 cycles**) `else r3 = r2 * r1;` (**18 cycles** via `setf` + `xma.l` + `getf`).
- En code prédiqué, même quand `p2` est faux (`if` pris), les instructions prédiquées par `(p2)` occupent le flux et leurs stops retardent la suite : le chemin court passe de **2 cycles à 5 cycles**.
- **Résultat (p. `1:169`) :**
  - Si le `if` est pris 50 % du temps sans erreur de prédiction : code avec branchement = $0{,}5 \times 2 + 0{,}5 \times 18 = \mathbf{10\text{ cycles}}$ vs code prédiqué = $0{,}5 \times 5 + 0{,}5 \times 18 = \mathbf{11{,}5\text{ cycles}}$ (**perdant**).
  - Même avec 70 % dans le `if` et 10 % d'erreurs de prédiction (à 10 cycles) : branchement = $\mathbf{7{,}8\text{ cycles}}$ vs prédiqué = $\mathbf{8{,}9\text{ cycles}}$ (**encore perdant**).
  - La prédication ne devient gagnante que si le chemin long domine et que le branchement est très mal prédit (ex. 30 % d'erreurs : $16{,}2$ vs $14{,}1$ cycles).

### Cas B : Saturation des unités d'exécution (*Overlapping Resource Usage*, § 4.2.4.5–4.2.4.6, p. `1:170`)
Exemple : le bloc `then` fait 4 accès mémoire (`2 ld + 2 st` = **2 cycles** sur 2 unités M) et le bloc `else` fait aussi 4 accès mémoire (**2 cycles**).
- Si l'on fusionne `then` et `else` par prédication, le processeur doit émettre **8 instructions mémoire**, ce qui prend obligatoirement **4 cycles** sur 2 unités M (même si la moitié a son prédicat à `0` !).
- Avec seulement 10 % d'erreurs de prédiction : code avec branchement = $2 + 0{,}10 \times 10 = \mathbf{3\text{ cycles}}$ vs code prédiqué = **4 cycles** (**perdant**).

---

## 4. Optimisations avancées du flot de contrôle (Chap. 4, § 4.3 à 4.5, p. `1:171`–`1:179`)

### 4.1 Réduction d'expressions booléennes en 1 cycle par comparaisons parallèles (§ 4.3.1, p. `1:172`–`1:173`)
- **Disjonction `if (rA || rB || rC || rD)` :** après avoir initialisé `p1 = 0` (`cmp.ne p1, p0 = r0, r0 ;;`), quatre `cmp.ne.or p1, p0 = rX, r0` dans le **même groupe** calculent le OU à 4 entrées en **un seul cycle** et peuvent même être suivis de `(p1) br.cond if_block` dans le même groupe !
- **Condition complexe avec `if` ET `else` (`if (rA == 0 || rB == 10) ... else ...`, p. `1:173`) :** l'utilisation du type **DeMorgan** (`.or.andcm`) permet de calculer simultanément `p1` (en mode OR, initialisé à `0`) et son complément `p2` (en mode AND, initialisé à `1`) :
  ```assembly
  cmp.ne          p1, p2 = r0, r0 ;;   // Initialise p1 = 0, p2 = 1
  cmp.eq.or.andcm p1, p2 = rA, r0      // Si rA == 0 : p1 = 1, p2 = 0
  cmp.eq.or.andcm p1, p2 = rB, 10 ;;   // Si rB == 10 : p1 = 1, p2 = 0
  (p1) add        r1 = r2, r3          // Branche if
  (p2) sub        r4 = r5, r6          // Branche else
  ```

### 4.2 Branchements multi-voies (*Multiway Branches*, § 4.3.2, p. `1:173`–`1:174`)
Un seul bundle de template **`.bbb`** (`0x16` ou `0x17`) permet de tester **3 cibles de saut différentes + 1 sortie séquentielle (4 voies)** en un seul cycle :
```assembly
{ .bbb
  (p1) br.cond label_B
  (p2) br.cond label_C
  (p3) br.cond label_D
}
// Fall-through vers label_A si p1, p2, p3 sont faux
```

### 4.3 Écriture multiple sous prédicats exclusifs et économie de registres (§ 4.3.3, p. `1:174`–`1:175`)
- Plusieurs instructions d'un **même groupe** peuvent écrire dans le **même registre `rA`** si leurs prédicats qualificatifs sont mutuellement exclusifs (`switch` exécuté en 1 cycle : `(p1) add rA=rB,rC`, `(p2) add rA=rE,rF`, `(p3) sub rA=rH,rI ;;`).
- Cela permet aussi aux deux branches d'un `if/else` prédiqué de **réutiliser les mêmes registres temporaires** (`r1`, `r7`) au lieu d'allouer des registres distincts par branche, réduisant les *spills* du RSE (§ 4.3.3.2, p. `1:175`).

### 4.4 Alignement du flux d'instructions sur les lignes de cache (§ 4.3.4.1, p. `1:176`)
Si une étiquette `L` est fréquemment cible d'un saut (ex. tête de boucle) **et** que le premier groupe d'instructions en `L` s'étend sur plus d'un bundle, il est recommandé de l'aligner sur une frontière de ligne de cache d'instructions (ou un bundle pair) pour éviter que le matériel ne coupe l'émission (*split issue*) entre les deux bundles.

### 4.5 Astuce matérielle pour les boucles d'attente (*Spinlocks* / *Multi-threading*, § 4.5.1, p. `1:177`–`1:178`)
Sur un processeur Itanium multi-threadé (SMT), un thread qui attend qu'un verrou mémoire se libère peut exécuter un **chargement avancé `ld.a` sur l'adresse du verrou**, suivi de l'instruction **`hint @pause`** : le processeur cède alors ses ressources aux autres threads matériels et **réveille automatiquement la priorité du thread dès que l'entrée `ALAT` du verrou est invalidée par l'écriture d'un autre cœur** !
