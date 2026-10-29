# Fiche 4 — Pile de registres (RSE), rotation et boucles pipelinées (IA-64)

**Sources principales :** Manuel Intel® Itanium® Architecture Software Developer's Manual, Révision 2.3 :
- Volume 1, Partie 1, Chapitre 4 (§ 4.1, pages `1:47` à `1:50` et § 4.5, pages `1:74` à `1:78`).
- Volume 1, Partie 1, Chapitre 3 (§ 3.1.7 et § 3.1.8, pages `1:27` à `1:33`).

---

## 1. La pile de registres généraux (*Register Stack*, § 4.1, p. `1:47`–`1:50`)

### 1.1 Partitionnement des 128 registres généraux
- **`GR0` à `GR31` (32 registres) : statiques**, visibles par toutes les procédures (sauvegardés par convention logicielle caller/callee-saved).
- **`GR32` à `GR127` (96 registres max.) : empilés (*stacked*)**, locaux à chaque procédure grâce à un renommage matériel automatique lors des appels et retours.

Chaque procédure voit un **cadre de pile de registres** (*register stack frame*) commençant toujours à `GR32`, de taille totale `sof` (*size of frame*, $0 \le \text{sof} \le 96$), divisé en deux zones contiguës (§ 4.1.1, p. `1:47`–`1:48`) :
1. **Zone locale (*local area*)** : de taille `sol` (*size of locals*), allant de `GR32` à `GR(32 + sol - 1)`. Contient les paramètres reçus de l'appelant (*inputs*) et les variables locales de la procédure.
2. **Zone de sortie (*output area*)** : de taille `sof - sol`, allant de `GR(32 + sol)` à `GR(32 + sof - 1)`. Sert à passer les paramètres à la prochaine procédure appelée.

> **Accès hors du cadre courant (p. `1:48`) :**
> - Lire un registre empilé au-delà du cadre courant (`>= 32 + sof`) retourne une **valeur indéfinie** (sans lever d'exception, § 3.5, p. `1:45`).
> - Écrire dans un registre empilé au-delà du cadre courant déclenche une **faute `Illegal Operation`**.

### 1.2 Fonctionnement de `br.call`, `alloc` et `br.ret` (§ 4.1.1, Figure 4-1, p. `1:48`–`1:49`)

Prenons l'exemple officiel du manuel (Figure 4-1, p. `1:49`) où `procA` appelle `procB` :

| Étape | Action matérielle sur `CFM`, `PFS` et le renommage | État dans l'exemple (Fig. 4-1) |
|---|---|---|
| **1. Avant l'appel** (`procA`) | `procA` a un cadre de taille $\text{sof}_a = 21$ et une zone locale $\text{sol}_a = 14$. Sa zone de sortie fait $\text{sof}_a - \text{sol}_a = 7$ registres (`r46` à `r52`). | `CFM` : `sol = 14`, `sof = 21` (`r32`–`r45` locaux, `r46`–`r52` sorties). |
| **2. Appel (`br.call` / `brl.call`)** | 1. Copie `CFM`, `EC` et `PSR.cpl` dans **`AR[PFS]`** (`pfm`, `pec`, `ppl`).<br>2. Écrit l'adresse de retour **`IP + 16`** dans le registre de branchement cible `b1`.<br>3. Renomme les registres empilés pour que le 1er registre de sortie de l'appelant (`GR(32 + sol_a)`) devienne **`GR32`** de l'appelé.<br>4. Fixe le nouveau `CFM` : **`sol = 0`**, **`sof = sof_a - sol_a`**, **`sor = 0`**, et remet les trois bases de rotation **`rrb.gr = rrb.fr = rrb.pr = 0`**. | `PFS.pfm` reçoit `(14, 21)`.<br>`CFM` de `procB` : `sol = 0`, `sof = 7`. Les anciens `r46`–`r52` de `procA` sont maintenant vus comme `r32`–`r38` par `procB`. |
| **3. Allocation (`alloc`)** | En assembleur : `alloc r1 = ar.pfs, i, l, o, r` (entrées $i$, locaux $l$, sorties $o$, tournants $r$).<br>1. Copie l'ancien `AR[PFS]` dans le registre général `r1`.<br>2. **Sans changer la base de renommage**, redimensionne le cadre courant : **`sol = i + l`**, **`sof = i + l + o`**, **`sor = r / 8`**. | `procB` fait `alloc` avec $\text{sol}_{b2} = 16$ (`r32`–`r47`) et $\text{sof}_{b2} = 19$ (`r48`–`r50` en sortie). Les registres fraîchement alloués (`r39`–`r50`) et leurs bits NaT ont des valeurs indéfinies. |
| **4. Retour (`br.ret`)** | `procB` a préalablement restauré `AR[PFS]` depuis `r1`.<br>1. Copie `PFS.pfm` dans **`CFM`** (restaure `sol`, `sof`, `sor`, `rrb.*` et la base de renommage de `procA`), `PFS.pec` dans **`EC`**, et `PFS.ppl` dans `PSR.cpl` (sans jamais augmenter le privilège).<br>2. Saute à l'adresse contenue dans `b2`. | `CFM` redevient `(14, 21)`, `procA` retrouve ses registres `r32`–`r52` intacts (avec les valeurs éventuellement modifiées par `procB` dans `r46`–`r52`). |

### 1.3 Contraintes strictes sur `alloc` (§ 4.1.2, p. `1:49` et § 4.5.1, p. `1:76`)
- `alloc` **doit être la première instruction d'un groupe d'instructions** (sinon comportement indéfini).
- Son prédicat qualificatif `qp` **doit être `PR0`** (sinon faute `Illegal Operation`).
- `alloc` ne peut modifier `sor` que si **toutes les bases de rotation (`rrb.gr`, `rrb.fr`, `rrb.pr`) valent `0`** (sinon faute `Reserved Register/Field`, Vol. 3).

### 1.4 Le moteur RSE (*Register Stack Engine*, § 3.1.8.2–3.1.8.5, p. `1:29`–`1:30` et § 4.1.2, p. `1:50`)
Lorsque les appels imbriqués dépassent le nombre de registres physiques disponibles, le **RSE** sauvegarde (*spill*) et restaure (*fill*) automatiquement les cadres anciens en mémoire (dans le *Backing Store*) sans intervention du programme :
- **`AR[BSP]`** (*lecture seule*) : pointe vers l'adresse mémoire correspondant à `GR32` du cadre courant.
- **`AR[BSPSTORE]`** : pointe vers l'adresse où le RSE écrira le prochain registre sauvegardé.
- **`AR[RNAT]`** : toutes les **63 sauvegardes** consécutives de registres généraux en mémoire, lorsque les bits `8:3` de `BSPSTORE` valent `111111` (`0x3F`, soit le 64e mot de 8 octets), le RSE écrit le registre `AR[RNAT]` (qui a collecté les 63 bits NaT des 63 registres précédents) à cette adresse mémoire, puis continue.
- Instructions explicites de synchronisation (§ 4.1.2, Table 4-2, p. `1:50`, aucune ne peut être prédiquée) :
  - `flushrs` (1re instruction d'un groupe) : force le RSE à vider en mémoire tous les cadres actifs précédant le cadre courant.
  - `loadrs` (1re instruction d'un groupe) : force le RSE à recharger depuis la mémoire `RSC.loadrs` octets.
  - `cover` (dernière instruction d'un groupe) : crée un nouveau cadre vide (`sof = sol = 0`) au-dessus du cadre courant (utilisé par les gestionnaires d'interruption).

---

## 2. Rotation des registres (*Rotating Registers*, § 4.5.1, p. `1:75`–`1:76`)

Pour accélérer les boucles par **pipeline logiciel** (*modulo-scheduled loops*) sans recopier les registres d'une itération à l'autre, trois zones de registres peuvent « tourner » d'un cran vers le haut (`X` devient `X+1`, et le dernier reboucle sur le premier) à chaque exécution d'un branchement de boucle pipelinée (`br.ctop`, `br.cexit`, `br.wtop`, `br.wexit`) :

| Banque | Zone tournante | Taille de la zone | Base de renommage dans `CFM` |Formule d'indexation physique / logique |
|---|---|---|---|---|
| **Registres généraux (`GR`)** | `GR32` à `GR(32 + 8*sor - 1)` | Programmable : **`8 * sor`** ($0, 8, 16, \dots, 96$). Ne tourne que si `sor > 0`. | `CFM.rrb.gr` (7 bits) | $\text{Index} = 32 + ((r - 32 + \text{rrb.gr}) \bmod (8 \times \text{sor}))$ |
| **Registres flottants (`FR`)** | `FR32` à `FR127` | Fixe : **96 registres** (tourne toujours). | `CFM.rrb.fr` (7 bits) | $\text{Index} = 32 + ((f - 32 + \text{rrb.fr}) \bmod 96)$ |
| **Registres de prédicat (`PR`)** | `PR16` à `PR63` | Fixe : **48 prédicats** (tourne toujours). | `CFM.rrb.pr` (6 bits) | $\text{Index} = 16 + ((p - 16 + \text{rrb.pr}) \bmod 48)$ |

- **Mécanisme de rotation (p. `1:76`, Table 4-25) :** lors d'une rotation, `rrb.gr` (si `sor > 0`), `rrb.fr` et `rrb.pr` sont **décrémentés de 1** modulo la taille de leur région respective. Ainsi, la valeur qui était lue dans le registre logique $X$ (correspondant à $X + \text{rrb}$) se retrouve au tour suivant dans le registre logique $X+1$ (car $(X+1) + (\text{rrb}-1) = X + \text{rrb}$).
- **Remise à zéro des bases (`clrrrb`, Table 4-25, p. `1:76`) :**
  - `clrrrb` remet `rrb.gr`, `rrb.fr` et `rrb.pr` à `0`.
  - `clrrrb.pr` remet uniquement `rrb.pr` à `0`.
  - `clrrrb` et `clrrrb.pr` **doivent être la dernière instruction d'un groupe d'instructions**.

---

## 3. Branchements et boucles (§ 4.5 et § 4.5.1, p. `1:74`–`1:78`)

### 3.1 Règles générales sur les branchements (§ 4.5, p. `1:74`–`1:75`)
- Les cibles de branchement sont toujours alignées sur un bundle (16 octets, slot 0).
- Un branchement **n'est pas obligé d'être la dernière instruction d'un groupe** : un même groupe peut contenir plusieurs branchements. Dès qu'un branchement est pris, les instructions suivantes du groupe ne sont pas exécutées.
- **Exception pour les branchements de boucle (`br.cloop`, `br.ctop`, `br.cexit`, `br.wtop`, `br.wexit`, p. `1:75`) :** ils sont **autorisés uniquement dans le slot 2 d'un bundle**. S'ils sont placés en slot 0 ou en slot 1, ils déclenchent une faute **`Illegal Operation`**.

### 3.2 Boucle comptée simple : `br.cloop` (p. `1:75`)
- N'utilise **pas** de prédicat qualificatif (`qp` est forcé à `0` dans le format `B2`, Vol. 3 p. `3:297`), ni `EC`, ni la rotation des registres.
- Si `AR[LC] > 0` : décrémente `LC` (`LC = LC - 1`) et **prend le branchement** (continue la boucle).
- Si `AR[LC] == 0` : ne modifie pas `LC` et **ne prend pas le branchement** (sort de la boucle).
- Pour exécuter $N$ itérations ($N \ge 1$), on initialise donc **`LC = N - 1`**.

### 3.3 Boucles pipelinées comptées : `br.ctop` et `br.cexit` (§ 4.5.1, p. `1:76`–`1:77`)
Le corps de la boucle est découpé en $S$ étages de pipeline logiciel, contrôlés par les **prédicats d'étage** (*stage predicates*) commençant architecturalement à **`PR16`** (1er étage = `p16`, 2e étage = `p17`, etc.) :
- **Initialisation avant la boucle (p. `1:77`) :**
  - **`AR[LC] = N - 1`** (un de moins que le nombre d'itérations $N$).
  - **`AR[EC] = S`** (égal au nombre d'étages du pipeline logiciel).
  - Prédicats tournants initialisés : **`PR16 = 1`** et `PR17`..`PR63 = 0` (par ex. via `mov pr.rot = 1 << 16`).
- **Comportement exact à chaque exécution de `br.ctop` / `br.cexit` (p. `1:77`, contrôlé visuellement) :**
  *(Note : le branchement écrit dans `PR63` **avant** de faire tourner les registres ; après la rotation d'un cran, cette valeur écrite dans `PR63` devient le nouveau **`PR16`** !)*

| Phase | Condition à l'entrée du branchement | Action sur `LC` et `EC` | Rotation (`rrb--`) ? | Nouveau `PR16` (via `PR63`) | Direction (`ctop` : pris = continuer ; `cexit` : inverse) |
|---|---|---|---|---|---|
| **Prologue & Noyau** (*Kernel*) | **`LC > 0`** | `LC = LC - 1`<br>(`EC` inchangé) | **Oui** | **`PR16 = 1`** (active une nouvelle itération) | **Continue** la boucle (`ctop` pris) |
| **Épilogue** (vidage du pipeline) | **`LC == 0` et `EC > 1`** | `EC = EC - 1`<br>(`LC` reste à `0`) | **Oui** | **`PR16 = 0`** (éteint l'étage 1, décale les `1` restants) | **Continue** la boucle (`ctop` pris) |
| **Dernier tour d'épilogue** (sortie) | **`LC == 0` et `EC == 1`** | `EC = 0`<br>(`LC` reste à `0`) | **Oui** | **`PR16 = 0`** | **Sort** de la boucle (`ctop` non pris) |
| **Boucle déjà terminée** | **`LC == 0` et `EC == 0`** | `LC` et `EC` inchangés | **Non** (`rrb` inchangés) | **`PR63 = 0`** (sans rotation) | **Sort** de la boucle (`ctop` non pris) |

### 3.4 Boucles pipelinées `while` : `br.wtop` et `br.wexit` (§ 4.5.1, p. `1:77`–`1:78`)
- N'utilisent **pas** `LC` : la condition de poursuite est donnée par le **prédicat qualificatif `PR[qp]`** (`1` pendant le noyau, `0` pendant l'épilogue).
- Tant que `PR[qp] == 1` : la boucle continue, `EC` n'est pas modifié, les registres tournent et **`PR16` est mis à `0` après rotation** (dans une boucle `while`, c'est le calcul de condition dans le corps de boucle qui allume le prédicat du premier étage utile).
- Quand `PR[qp] == 0` : même comportement d'épilogue avec `EC` (`EC > 1` continue et décrémente `EC` ; `EC == 1` sort, décrémente `EC` et tourne ; `EC == 0` sort sans tourner et met `PR63 = 0`).
