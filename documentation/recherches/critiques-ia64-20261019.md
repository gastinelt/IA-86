# IA-64 (Itanium) : ce que les critiques nous apprennent

Recherche du 19/10/2026 (Claude, Opus 5.5). Sources : articles, forums, listes de diffusion.
⚠️ Ce sont surtout des **avis** (forums, presse), pas des études vérifiées. Chaque point sera ensuite confronté au manuel Intel.

## 1. Les critiques qui reviennent le plus

| # | Point critique | Ce qu'on en dit | Piste pour notre langage |
|---|---|---|---|
| 1 | **Tout repose sur le compilateur** | Le parallélisme est décidé à la compilation (EPIC). Dès 1997, compilateur et architecture se sont révélés bien plus durs à réaliser que prévu. | Ne pas faire dépendre les performances d'un compilateur « magique ». |
| 2 | **Le comportement dynamique est imprévisible** | Le compilateur ne connaît ni les défauts de cache, ni le chemin réel des branchements. Phrase souvent citée : on ne peut pas planifier statiquement un comportement dynamique. | Laisser au matériel ce qui ne se connaît qu'à l'exécution (latence mémoire). |
| 3 | **Code trop gros** | Les cases vides des paquets d'instructions (NOP) gonflent le code. Il faut alors d'énormes caches : puces chères et gourmandes. | Encodage compact, sans case vide imposée. |
| 4 | **La microarchitecture est exposée** | Le jeu d'instructions fige des détails internes (nombre d'unités, latences), ce qui rend les puces suivantes difficiles à faire évoluer. | Séparer le contrat du langage et la puce. |
| 5 | **Prédication coûteuse** | Exécuter les deux branches puis jeter l'une consomme de l'énergie (avis contesté sur Hacker News). | Prédication oui, mais limitée. |
| 6 | **Rupture totale avec l'existant** | Incompatible avec x86, émulation x86 lente. AMD64 a gagné en gardant x86 et en ajoutant simplement des registres plus larges et plus nombreux. | Une transition douce compte autant que la technique. |
| 7 | **Peu de parallélisme dans les vrais programmes** | La plupart des logiciels sont séquentiels. Les gains espérés sont restés faibles (2 à 3 fois au mieux, selon un commentaire), sauf sur le calcul vectoriel. | Viser des usages précis (calcul, signal) plutôt que tout. |
| 8 | **« Trop propre »** | Linus Torvalds : Intel a jeté les bons côtés de x86 parce qu'ils étaient laids. | La beauté du design ne suffit pas, il faut mesurer. |

## 2. Les avis contraires (à garder en tête)
- Certains disent que le concept était bon et que c'est la technique de compilation de l'époque qui manquait.
- D'autres répondent qu'aucun compilateur ne l'aurait sauvé (le problème est le code existant, pas le compilateur).
- Le VLIW (la famille d'IA-64) **fonctionne bien** dans des domaines précis : processeurs de signal (DSP), anciens GPU.
- Certains pensent que le VLIW pourrait revenir, car les compilateurs ont progressé et la finesse de gravure stagne.

## 3. Ce qui manque encore (à chercher)
- [ ] La vraie citation de Donald Knuth sur les compilateurs (souvent citée, sortie de son contexte selon une source : il parlait du multicœur).
- [ ] Les articles de Raymond Chen sur l'Itanium (blog *The Old New Thing*).
- [ ] Real World Technologies (forum de David Kanter), plus technique.
- [ ] Le processeur **Mill** (projet qui reprend des idées proches et dit corriger les défauts d'Itanium).
- [ ] L'histoire complète : « The Itanic Saga » (abortretry.fail).

## Sources
- Hacker News : [14518758](https://news.ycombinator.com/item?id=14518758), [14331342](https://news.ycombinator.com/item?id=14331342), [37797645](https://news.ycombinator.com/item?id=37797645), [38114943](https://news.ycombinator.com/item?id=38114943), [48971566](https://news.ycombinator.com/item?id=48971566), [39097504](https://news.ycombinator.com/item?id=39097504)
- [The Register, « Who sank Itanic? » (2004)](https://www.theregister.com/2004/02/17/who_sank_itanic/)
- [The Inquirer, Torvalds](https://www.theinquirer.net/inquirer/news/1008015/linus-torvalds-itanium-threw-x86)
- [The Register, Linux retire Itanium (2021)](https://www.theregister.com/2021/02/01/linux_pulls_itanium_support)
- [EE Journal, « Sinking the Itanic »](https://www.eejournal.com/article/20130306-itanium/)
- [Real World Tech, Poulson](https://www.realworldtech.com/poulson/)
- [The Itanic Saga (pappp.net)](https://pappp.net/?p=59768)
- [Wikipedia, IA-64](https://en.wikipedia.org/wiki/IA-64)
