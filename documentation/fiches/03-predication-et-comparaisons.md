# Fiche 3 — Prédication et instructions de comparaison (IA-64)

**Sources principales :** Manuel Intel® Itanium® Architecture Software Developer's Manual, Révision 2.3 :
- Volume 1, Partie 1, Chapitre 4 (§ 4.3, pages `1:54` à `1:57`).
- Volume 3, Chapitre 2 (`cmp`, `tbit`, `tnat`, `mov pr`) et Chapitre 4 (§ 4.2.2, p. `3:304`–`3:308` et § 4.3.6, p. `3:321`).

---

## 1. Principe de la prédication (§ 4.3.1, p. `1:54`)

La **prédication** consiste à conditionner l'exécution de presque chaque instruction par un **prédicat qualificatif** (*qualifying predicate*, champ `qp` de 6 bits, bits `5:0` de l'instruction 41 bits) désignant l'un des 64 registres de prédicat `PR0` à `PR63` :

- Si `PR[qp] == 1` (*vrai*), l'instruction s'exécute normalement et met à jour l'état architectural.
- Si `PR[qp] == 0` (*faux*), l'instruction **ne modifie pas l'état architectural** (et ses lectures/écritures sont ignorées pour les règles de dépendances entre registres), **à trois exceptions près** :
  1. Les comparaisons de type **inconditionnel** (`.unc`), qui écrivent `0` dans leurs deux prédicats cibles même quand `PR[qp] == 0` (§ 4.3.3, Table 4-9, p. `1:55`).
  2. Les instructions d'approximation flottante (`frcpa`, `fprcpa`, `frsqrta`, `fprsqrta`), qui sont de type `.unc` (Table 4-11, p. `1:56`).
  3. Les branchements de boucle `while` pipelinée (`br.wtop`, `br.wexit`), où `PR[qp]` sert de condition de poursuite de la boucle tandis que l'épilogue (`EC`) continue de tourner quand `PR[qp] == 0` (§ 4.5.1, p. `1:77`).

### Instructions qui ne peuvent PAS être prédiquées (§ 4.3.1, p. `1:54`)
Certaines instructions ont leur champ `qp` figé à `0` (`PR0`, toujours vrai) ou réservé (faute `Illegal Operation` si `qp != 0`, ex. `alloc` p. `1:49`) :
- Gestion de la pile de registres et rotation : `alloc`, `flushrs`, `loadrs`, `cover`, `clrrrb`.
- Branchements comptés et prédiction : `br.cloop`, `br.ctop`, `br.cexit`, `brp`.
- Système : `bsw` (*bank switch*), `epc` (*enter privileged code*), `rfi` (*return from interruption*).

---

## 2. Les instructions qui écrivent des prédicats (§ 4.3.2, Table 4-8, p. `1:54`–`1:55`)

Presque toutes les instructions de comparaison écrivent **deux** registres de prédicat (`p1` et `p2`) en une seule instruction :

| Mnémonique | Opération | Cibles | Particularités |
|---|---|---|---|
| `cmp` | Comparaison entière 64 bits (`r2` vs `r3`, ou `imm8` vs `r3`, ou `r3` vs `0`) | `p1, p2` | 10 relations (`eq`, `ne`, `lt`, `le`, `gt`, `ge`, `ltu`, `leu`, `gtu`, `geu`). En binaire, seules 4 relations de base (`eq`, `lt`, `ltu` et comparaisons à `0`) sont encodées, les autres sont des pseudo-instructions assembleur inversant `r2`/`r3` ou le bit `c` (*complement*). |
| `cmp4` | Comparaison entière 32 bits | `p1, p2` | Compare uniquement les bits `31:0` des deux sources (les bits `63:32` sont ignorés). |
| `tbit` | Test d'un bit dans un registre général (`r3{pos6}`) | `p1, p2` | Teste si le bit à la position immédiate `pos6` (`0` à `63`) vaut `0` (`.z`) ou `1` (`.nz`). |
| `tnat` | Test du bit NaT d'un registre général (`r3.nat`) | `p1, p2` | Teste si `r3` contient une exception différée (`.z` ou `.nz`). |
| `tf` | Test d'une fonctionnalité processeur (`CPUID[4]`) | `p1, p2` | Teste rapidement un bit de la moitié haute de `CPUID[4]`. |
| `fcmp` | Comparaison flottante (`f2` vs `f3`) | `p1, p2` | Relations `eq`, `lt`, `le`, `unord` et leurs négations. |
| `fclass` | Classification d'un flottant (`f2`) | `p1, p2` | Teste la classe de `f2` (zéro, infini, NaN, normal, dénormalisé, `NaTVal`, etc.). |
| `frcpa`, `frsqrta` | Approximation d'inverse ($1/x$) ou $1/\sqrt{x}$ | `f1, p2` | Écrit **un seul** prédicat `p2` (`1` si l'approximation est valide, `0` si appel logiciel requis). |

---

## 3. Les 5 types de comparaisons (§ 4.3.3, Table 4-9, p. `1:55`–`1:56`)

Soit `qp` la valeur du prédicat qualificatif (`PR[qp]`) et `res` le résultat booléen (`0` ou `1`) de la relation testée. La mise à jour des deux cibles `p1` et `p2` dépend du *completer* de type :

| Type | Completer | Action sur `p1` (1re cible) | Action sur `p2` (2e cible) | Usage typique |
|---|---|---|---|---|
| **Normal** | *(aucun)* | `if (qp) p1 = res` | `if (qp) p2 = !res` | `if / else` classique (`p1` et `p2` complémentaires si `qp == 1`, inchangés si `qp == 0`). |
| **Unconditional** | `.unc` | `if (qp) p1 = res`<br>`else p1 = 0` | `if (qp) p2 = !res`<br>`else p2 = 0` | `if` imbriqué : si la branche englobante est fausse (`qp == 0`), force `p1` et `p2` à `0` en un seul cycle. |
| **AND** (parallèle) | `.and` | `if (qp && !res) p1 = 0` | `if (qp && !res) p2 = 0` | Conjonction : ne fait rien si `res == 1`, met à `0` si `res == 0` (nécessite d'avoir initialisé la cible à `1` avant). |
| | `.andcm` | `if (qp && res) p1 = 0` | `if (qp && res) p2 = 0` | Conjonction avec le complément (`!res`). |
| **OR** (parallèle) | `.or` | `if (qp && res) p1 = 1` | `if (qp && res) p2 = 1` | Disjonction : met à `1` si `res == 1`, ne fait rien si `res == 0` (nécessite d'avoir initialisé la cible à `0` avant). |
| | `.orcm` | `if (qp && !res) p1 = 1` | `if (qp && !res) p2 = 1` | Disjonction avec le complément (`!res`). |
| **DeMorgan** (parallèle) | `.or.andcm` | `if (qp && res) p1 = 1` *(type OR)* | `if (qp && res) p2 = 0` *(type AND)* | Calcule en parallèle une disjonction dans `p1` ($p_1 = A \lor B$) et sa négation dans `p2` ($p_2 = \lnot A \land \lnot B$). |
| | `.and.orcm` | `if (qp && !res) p1 = 0` *(type AND)* | `if (qp && !res) p2 = 1` *(type OR)* | Inverse du précédent : `p1` est de type AND, `p2` de type OR. |

> **Règle d'écriture parallèle dans un même groupe (§ 3.4.2, p. `1:43` et § 4.3.3, p. `1:56`) :**
> Plusieurs comparaisons du **même groupe d'instructions** peuvent écrire dans le **même registre de prédicat** `pX` (avec `X > 0`) à condition que **toutes** les écritures visant `pX` soient de **type AND** (y compris la moitié AND d'un DeMorgan), ou que **toutes** soient de **type OR** (y compris la moitié OR d'un DeMorgan).

---

## 4. Comportement des comparaisons face à un bit `NaT` ou `NaTVal` (§ 4.3.3, Table 4-10, p. `1:56`)

Si l'un des registres sources d'une comparaison (sauf `tnat` et `fclass` testant `NaTVal`) a son bit `NaT == 1` (ou contient `NaTVal`), **aucune exception n'est levée** et les **deux** prédicats cibles `p1` et `p2` reçoivent le même traitement (ce qui désactive à la fois la branche `if` et la branche `else`) :

| Type de comparaison | Action sur `p1` **et** sur `p2` si une source est `NaT` / `NaTVal` |
|---|---|
| **Normal** | `if (qp) { p1 = 0; p2 = 0; }` |
| **Unconditional (`.unc`)** | `p1 = 0; p2 = 0;` (quel que soit `qp`) |
| **AND (`.and`, `.andcm`)** | `if (qp) { p1 = 0; p2 = 0; }` |
| **OR (`.or`, `.orcm`)** | **Non écrits** (inchangés) |
| **DeMorgan (`.or.andcm`, `.and.orcm`)** | **Non écrits** (inchangés) |

### Disponibilité des types selon l'instruction (Table 4-11, p. `1:56`)
- `cmp` et `cmp4` supportent les 5 types (**Normal, Unconditional, AND, OR, DeMorgan**) uniquement pour `==`, `!=` et les comparaisons signées par rapport à **`0`** (`r3 > 0`, `r3 >= 0`, `r3 < 0`, `r3 <= 0`). Toutes les autres relations (`r2 < r3`, non signées, etc.) ne supportent que **Normal** et **Unconditional**.
- `tbit`, `tnat` et `tf` supportent les **5 types**.
- `fcmp` et `fclass` ne supportent que **Normal** et **Unconditional**.

---

## 5. Transferts globaux de prédicats (`mov pr`, § 4.3.4, p. `1:57` ; Vol. 3, p. `3:321`)

Trois instructions (format `I23`, `I24`, `I25` sur l'unité **I**) transfèrent les prédicats en bloc (*broadside*) :

1. **`mov r1 = pr`** (format `I25`) : copie les 64 registres `PR0`–`PR63` dans les bits `63:0` du registre général `r1`. Attention aux dépendances : cette instruction **lit toujours les 64 prédicats** (§ 3.4.2, Note, p. `1:43`).
2. **`mov pr = r2, mask17`** (format `I23`) : copie les bits de `r2` vers les prédicats sélectionnés par un masque immédiat :
   - 1 bit de masque individuel pour chacun des prédicats statiques `PR1` à `PR15` (`PR0` reste à `1`).
   - 1 seul bit de masque global pour **tous** les prédicats tournants `PR16` à `PR63` en bloc.
   - Ne compte comme écrivain (pour les dépendances WAW/RAW) que sur les prédicats dont le bit de masque vaut `1`.
3. **`mov pr.rot = imm44`** (format `I24`) : initialise les 48 prédicats tournants (`PR16` à `PR63`) à partir d'un immédiat encodé sur 28 bits (`imm27a` + bit de signe `s`, décalé de 16 bits et étendu par le signe jusqu'au bit 63) :
   - `PR16` à `PR42` peuvent recevoir des valeurs indépendantes (`0` ou `1`).
   - `PR43` à `PR63` reçoivent tous le bit de signe `s` (`0` ou `1`).

> **Règle importante sur la rotation (§ 4.3.4, p. `1:57`) :**
> Les trois instructions `mov pr` accèdent aux prédicats tournants (`PR16`–`PR63`) **comme si la base de renommage `CFM.rrb.pr` valait `0`** (elles ignorent le renommage de rotation). Il faut donc exécuter `clrrrb` ou `clrrrb.pr` avant d'initialiser les prédicats d'une boucle pipelinée si `rrb.pr` n'est pas déjà nul.


---

## 6. Regard critique & limites architecturales (pour IA-86)

| Élément | Verdict | Analyse critique |
|---|---|---|
| **Coût de 6 bits (`qp`) sur 100 % des instructions** | ⚠️ **Taxe d'encodage lourde** | Réserver 6 bits (`qp`) dans **chaque** instruction de 41 bits consomme $14{,}6\,\%$ de tout le binaire, alors que la majorité des instructions d'un programme sont inconditionnelles (`qp = p0 = 0`). |
| **Double cible `p1, p2` et type `.unc` sur les comparaisons** | ✅ **Très efficace** | Générer en un cycle une condition et sa négation (`p1, p2`) ou annuler automatiquement les deux si la branche parente est fausse (`.unc`) élimine proprement les arbres de sauts dans les petits blocs `if / else if / else`. |
| **Restriction cachée des comparaisons parallèles (`.and`, `.or`, DeMorgan)** | ❌ **Goulot d'encodage** | À cause du manque de bits dans le format de 41 bits, `cmp.lt.and` ou `cmp.gt.or` **n'existent pas entre deux registres (`r2` et `r3`)** : ils ne sont encodables que pour `==`, `!=` ou une comparaison signée avec **`0`** (Table 4-11, p. `1:56`) ! Pour évaluer `(a < b && c < d)` en parallèle, le compilateur ne peut donc pas utiliser directement les comparaisons parallèles vantées par le manuel. |
| **`mov pr` qui ignore `CFM.rrb.pr`** | ⚠️ **Piège matériel** | Le fait que `mov pr` accède aux prédicats physiques sans passer par le renommage `rrb.pr` oblige à exécuter `clrrrb.pr` avant d'initialiser une boucle pipelinée ou complique la sauvegarde de contexte au milieu d'une boucle. |
