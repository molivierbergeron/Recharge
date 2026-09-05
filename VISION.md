# Recharge — Vision, epics et séquence

*Audit du 8 août 2026, sur le commit `4e2dfac`. Ce fichier est une photographie horodatée : il n'est pas mis à jour après coup.*

---

## 1. Vision produit

**Ce que ça fait.** Répondre à une seule question, au moment où on branche l'auto : dans combien de temps je reviens la chercher. Le produit intègre la courbe d'acceptation réelle de la batterie par pas de 1 % (`index.html:594-604`) au lieu d'appliquer une puissance moyenne plate, ce qui lui permet de rester juste sur les charges longues et sur la fin de charge, là où la puissance s'effondre.

**Ce que ça remplace.** L'estimation du tableau de bord de l'auto, qui dérive. Sur la session mesurée du 29 juillet, l'auto a annoncé successivement 9 h 21, 9 h 26 puis 9 h 35 pour la même charge, pendant que le modèle recalé tenait 9 h 28 depuis trois points d'observation différents (commits `500d797`, `980e2fb`, `5e85719`). Ce que le CPO gagne n'est pas un chiffre plus précis dans l'absolu, c'est un chiffre **stable** : il peut planifier une course de 30 minutes au lieu de retourner voir l'écran de l'auto toutes les dix minutes.

**Où ça s'arrête.** Le produit ne lira jamais l'état de l'auto en direct : pas de compte connecté, pas d'API constructeur, pas de secret à stocker — c'est ce qui lui permet de rester une page statique installable et fonctionnelle hors ligne, sans backend à maintenir. Il ne cartographiera jamais les bornes ni ne guidera vers elles : c'est le métier des applications de réseau, déjà installées. Et il ne servira jamais un autre véhicule : la courbe de charge codée en dur (`index.html:432-445`) est celle de cette auto-là, et un sélecteur de modèle transformerait un instrument calibré en base de données approximative.

### État

| | |
|---|---|
| **Statut** | Vivant — dernier commit de code il y a 8 jours |
| **Dernière modification** | 2026-07-31, `4e2dfac` « Recentrer l'interface sur la question réellement posée » |
| **Stack réelle** | HTML + CSS + JavaScript vanille dans un fichier unique de 1 206 lignes. Service worker (`sw.js`, 48 lignes), manifeste PWA. Aucun build, aucun `package.json`, aucune bibliothèque tierce (vérifié : `find` sur le dépôt ne retourne ni `package.json` ni fichier de test) |
| **Dépendances externes** | GitHub Pages pour l'hébergement, GitHub Actions pour la publication (`.github/workflows/pages.yml`). Aucun appel réseau sortant à l'exécution : le seul `fetch` du code est `sw.js:37`, restreint à la même origine par le garde `sw.js:33` |
| **Points de rupture** | 1. Le déclencheur de déploiement code en dur le nom de branche `claude/polestar-charge-estimator-pp0lsk` (`pages.yml:8`). 2. La publication est un `push --force` sur `gh-pages` (`pages.yml:45`), sans point de retour. 3. Trois écritures `localStorage` avalent leur erreur en silence (`index.html:537`, `index.html:756`, et la lecture `index.html:496`). 4. L'échec d'enregistrement du service worker est ignoré (`index.html:1202`), donc le mode hors ligne peut ne pas fonctionner sans que rien ne le signale |
| **Coût récurrent** | 0 $. Dépôt public (`"private":false` retourné par l'API GitHub), donc Pages et Actions sont sur le palier gratuit. Aucune API tierce, aucun quota consommé |

---

## 2. Epics

### Epics socle

**S1 — Fiabilité.** Le calcul lui-même est sain : 41 fonctions déclarées, aucune fonction morte, aucun `TODO`/`FIXME`, aucun bloc de code commenté (vérifié par `grep` sur `index.html`, `sw.js`, `pages.yml`). Un passage Playwright sur la page complète ne produit aucune erreur console. Le problème n'est pas que ça casse, c'est que **ça casserait sans le dire** : il n'existe aucun gestionnaire d'erreur global, donc une exception dans `renderResult` (`index.html:825`) laisse à l'écran le dernier nombre calculé, qui reste parfaitement plausible. Le mode de panne d'une calculatrice, c'est le mauvais chiffre affiché avec assurance. S'y ajoute le déclencheur de déploiement codé en dur (`pages.yml:8`) : une branche renommée ne produit pas un échec, elle ne produit **aucune exécution**, donc aucune notification.

**S2 — Coût.** Rien à signaler. 0 $, aucun appel réseau sortant, aucun quota. La seule ressource consommée est du temps d'Actions sur un dépôt public, gratuit et non plafonné en pratique pour 9 commits.

**S3 — Vitesse et friction.** La position de départ est excellente : `restoreUi` (`index.html:730-745`) restaure borne, cible, température et niveau, donc rouvrir l'app à la même borne affiche déjà une réponse — zéro geste. La friction est concentrée sur le seul champ qui change à **chaque** session : le niveau actuel. Les cinq raccourcis (`index.html:302-306`) sont tous câblés sur la cible (`index.html:1001` et `index.html:1004`), jamais sur le niveau actuel, qui ne dispose que de boutons ±1 (`index.html:284-285`) et d'un curseur 0-99 (`index.html:288`). Passer de 20 % mémorisé à 58 % réel demande 38 appuis, ou un pointage au curseur à 3,4 pixels par point de pourcentage sur un écran de 390 px.

**S4 — Analytics et observabilité.** Rien n'est mesuré. Aucun compteur, aucun journal, aucune trace. Le produit ne sait pas s'il sert, ni s'il répond juste. C'est le manque le plus lourd de conséquence : le recalage du 29 juillet, qui a corrigé une erreur de 10 minutes sur 40, a été déclenché par une intuition du CPO — « je trouve ça un peu trop safe » — et non par une donnée. Le prochain écart devra à nouveau attendre d'être ressenti.

**S5 — Dette technique.** Deux dettes concrètes, aucune cosmétique. D'abord le `README.md`, qui décrit un modèle qui n'existe plus : il annonce « Les pertes n'allongent donc pas la durée » (`README.md:47`) alors que le commit `5e85719` a précisément inversé cette règle (`index.html:579-584`, `index.html:596-598`), et ses trois tableaux de référence donnent tous des valeurs périmées (détail en item 1). Ensuite le stockage de configuration : `saveConfig` sérialise l'objet entier (`index.html:537`) et `loadConfig` le superpose aux défauts (`index.html:496`), si bien qu'un seul passage dans le panneau de réglages fige **toutes** les valeurs, y compris celles jamais touchées. Cette dette a déjà coûté : le commit `5e85719` a dû faire passer la clé à `v2` (`index.html:482-485`) pour que les nouveaux défauts atteignent l'appareil, ce qui a effacé au passage la capacité personnalisée du CPO.

### Epics produit

**P1 — L'estimation se corrige à partir des recharges déjà faites**
- *Objectif :* que chaque session branchée laisse une trace exploitable, et que les constantes du modèle se déduisent de ces traces au lieu d'être ajustées à la main dans le code.
- *On saura que c'est atteint quand :* une correction du modèle entre en vigueur sans qu'une ligne de `DEFAULTS` ait été éditée.

**P2 — Le courant alternatif sort de l'à-peu-près**
- *Objectif :* donner à la charge maison et surtout à la charge au chalet la même assise mesurée que la charge rapide, aujourd'hui seule branche calibrée sur du réel.
- *On saura que c'est atteint quand :* une charge annoncée « 2 j 1 h » au chalet se termine à moins d'une demi-heure de l'heure affichée au branchement.

**P3 — Répondre à « quand brancher », pas seulement « quand ce sera prêt »**
- *Objectif :* résoudre le calcul dans l'autre sens — partir d'une heure de départ voulue et d'un niveau visé pour donner l'heure de branchement.
- *On saura que c'est atteint quand :* le CPO décide de brancher tout de suite ou d'attendre au lendemain matin sans faire de soustraction de tête.

---

## 3. Séquence par thème

### Maintenant

À la fin de cette vague, le produit dit la vérité sur lui-même et signale ses propres pannes : la documentation décrit le modèle réellement en vigueur, une exception cesse de se déguiser en résultat valide, et une modification du modèle se heurte à un filet de tests avant d'atteindre le téléphone. La deuxième plus grosse inconnue du modèle — les pertes en courant alternatif — est mesurée plutôt que supposée.

Rien ici n'ajoute de fonctionnalité. Tout corrige quelque chose de faux ou de muet aujourd'hui, et c'est ce qui rend le reste sûr à construire.

### Ensuite

À la fin de cette vague, le produit accumule ses propres sessions et s'en sert : il connaît sa capacité réelle plutôt que la valeur de fiche technique, il sait dire de combien il s'est trompé la dernière fois, et il répond à la question inverse — quand brancher pour être prêt à telle heure.

C'est la vague où le produit cesse d'être un calculateur figé pour devenir un instrument qui se règle. Elle n'a de sens qu'après la première : calibrer sur des données quand rien ne vérifie le modèle, c'est déplacer l'erreur sans la voir.

### Un jour

Le produit chiffrerait le coût de la session, et l'avertissement au-delà de 90 % passerait d'une règle abstraite à un montant en dollars. La configuration cesserait de figer les défauts, si bien qu'une correction du modèle atteindrait l'appareil sans effacer les mesures personnelles.

Ces deux chantiers sont réels mais aucun ne fait mal aujourd'hui. À garder en vue, à ne pas ouvrir tant que les deux premières vagues ne sont pas closes.

---

## 4. Items

| # | Item | Epic | Bénéfice concret | Effort | Sessions |
|---|---|---|---|---|---|
| 1 | Réaligner le README sur le modèle en vigueur | S5 | Dans six mois, la seule page d'entrée n'enseigne plus huit valeurs fausses | S | 1 |
| 2 | Rendre visible l'échec du calcul | S1 | Un chiffre périmé cesse de passer pour un chiffre juste | XS | 0,25 |
| 3 | Déclencher le déploiement sur la branche par défaut | S1 | Un renommage de branche ne fait plus disparaître les publications sans bruit | XS | 0,25 |
| 4 | Mesurer les pertes en courant alternatif | P2 | Retire 223 minutes d'incertitude cumulée pour une charge observée | XS | 0,25 |
| 5 | Donner des raccourcis au niveau actuel | S3 | 38 appuis ramenés à 3 pour la saisie faite à chaque session | XS | 0,25 |
| 6 | Figer le modèle sous tests | S1 | La quatrième correction du modèle ne réintroduit plus la faute de la première | M | 2 |
| 7 | Journaliser les sessions de recharge | P1 | Les mesures cessent de transiter par des photos et un assistant | L | 4 |
| 8 | Déduire la capacité utile du journal | P1 | Retire la première source d'incertitude : 235 min d'écart au chalet | M | 2 |
| 9 | Afficher l'écart entre estimation et réalité | S4 | Une dérive du modèle se voit avant d'être ressentie | M | 2 |
| 10 | Résoudre dans le sens inverse | P3 | La question du chalet — quand brancher — cesse d'être un calcul de tête | M | 2 |
| 11 | Chiffrer le coût de la session | S3 | L'avertissement au-delà de 90 % devient un montant, donc une décision | M | 2 |
| 12 | Cesser de figer les défauts dans la configuration | S5 | Une correction du modèle atteint l'appareil sans effacer les mesures | L | 4 |

**Total sessions estimées : 20**

---

### [1] Réaligner le README sur le modèle en vigueur

**Constat.** Le `README.md` documente le modèle d'avant le 29 juillet. Huit divergences vérifiées :

| README | Réalité |
|---|---|
| `README.md:47` « Les pertes n'allongent donc pas la durée » | `index.html:596-598` — le temps se calcule sur la puissance nette |
| `README.md:61` 75-85 % → 45 kW | `index.html:440` → 55 kW |
| `README.md:62` 85-90 % → 28 kW | `index.html:443` → 34 kW |
| `README.md:72` BRCC 50 → facteur 0,85 | `index.html:457` → 0,98 |
| `README.md:73-74` BRCC 100 et 180 → 0,85 | `index.html:460-461` → 0,90 |
| `README.md:84` Frais → ×0,9 | `index.html:469` → ×0,95 |
| `README.md:85` Froid → plafond seul | `index.html:472` → plafond 70 kW **et** facteur 0,85 |
| `README.md:37-38` « l'heure de retour vaut maintenant + durée » | `index.html:857-860` — l'heure de branchement est modifiable |

S'y ajoutent trois absences : les pertes DC passées de 5 à 7,2 % (`index.html:426-429`), l'affichage en jours au-delà de 24 h, et le nommage du jour d'arrivée — tous introduits par `5e85719` et `4e2dfac`.

**Bénéfice.** Le CPO rouvre le dépôt en février, lit le tableau de la courbe DC pour se remettre en tête son propre modèle, et repart avec 45 kW en mémoire là où le code dit 55. Toute décision prise ensuite — ajuster un facteur, interpréter un écart — part d'un chiffre faux.

**Proposition.** Reprendre les trois tableaux de référence (`README.md:55-63`, `README.md:67-74`, `README.md:81-87`) et le paragraphe sur les pertes (`README.md:40-49`) depuis les valeurs de `DEFAULTS`. Documenter les deux comportements ajoutés le 31 juillet dans la section *Utilisation*. Purger la section *Suite* (`README.md:139-144`), dont trois des quatre entrées vivent maintenant dans le carnet Notion.

**Dépend de.** Rien.

**Confiance que ça règle le constat.** 95 %

---

### [2] Rendre visible l'échec du calcul

**Constat.** Aucun gestionnaire d'erreur global : le `grep` sur `window.addEventListener('error'`, `onerror` et `unhandledrejection` ne retourne rien dans `index.html`. Les seuls `catch` existants avalent leur erreur sans trace (`index.html:537`, `index.html:756`, `index.html:1202`). Or `renderResult` réécrit le résultat par affectation d'`innerHTML` (`index.html:862-867`) : si une exception survient avant, le balisage précédent reste intact à l'écran.

**Bénéfice.** Le CPO consulte la barre ancrée, lit « 40 min », part faire une course — et le nombre datait de la borne précédente parce qu'une exception a interrompu la mise à jour. Il revient à une auto pas prête. Un bandeau qui dit « calcul interrompu » lui fait rouvrir l'app au lieu de partir.

**Proposition.** Ajouter un écouteur `window.addEventListener('error', ...)` et son équivalent `unhandledrejection` dans la section *Démarrage* (`index.html:1189-1203`), qui affiche un bandeau persistant et vide le résultat plutôt que de le laisser périmé. Réutiliser le style `.note` déjà défini (`index.html:192-196`).

**Dépend de.** Rien.

**Confiance que ça règle le constat.** 85 % — couvre les exceptions synchrones du rendu, pas un état incohérent qui ne lèverait pas d'erreur.

---

### [3] Déclencher le déploiement sur la branche par défaut

**Constat.** `pages.yml:6-10` déclenche la publication sur deux noms de branche codés en dur, dont `claude/polestar-charge-estimator-pp0lsk` — un nom généré, qui se trouve être la branche par défaut du dépôt (confirmé par `git remote show origin`). Un renommage ne provoque pas un échec de workflow : il ne provoque **aucune exécution**, donc aucune notification.

**Bénéfice.** Le CPO renomme la branche par défaut en `main` un soir de ménage, pousse trois correctifs les semaines suivantes, et consulte pendant tout ce temps une version du site figée à l'ancienne — sans qu'un seul signal ne l'avertisse.

**Proposition.** Remplacer la liste de branches par un filtre sur la branche par défaut, ou à défaut ajouter une étape finale qui compare le SHA publié sur `gh-pages` au SHA du commit déclencheur et échoue si l'écart persiste. Fichier unique, `.github/workflows/pages.yml`.

**Dépend de.** Rien.

**Confiance que ça règle le constat.** 90 %

---

### [4] Mesurer les pertes en courant alternatif

**Constat.** `index.html:429` fixe `acLoss: 0.10`. Cette valeur date du commit initial `1e0e5a5` et n'a jamais été confrontée à une mesure — contrairement à `dcLoss`, déduit de la session du 29 juillet (`index.html:426-428`). Depuis `5e85719`, les pertes allongent la durée : l'hypothèse n'est plus cosmétique. Une analyse de sensibilité exécutée sur le modèle du dépôt, en faisant varier `acLoss` de 5 à 14 %, déplace l'estimation de 28 minutes sur une charge maison 20 → 80 % et de 194 minutes au chalet.

**Bénéfice.** Le CPO branche au chalet le vendredi soir en lisant « prêt dimanche 4 h », et l'auto est prête trois heures plus tôt ou plus tard. Sur une charge de deux jours, c'est la différence entre partir le dimanche matin et devoir attendre l'après-midi.

**Proposition.** Aucun code à écrire pour la mesure : une charge maison relevée de bout en bout — heure de branchement, heure de fin, niveau de départ, niveau d'arrivée, kWh comptés si la borne les affiche — suffit à déduire les pertes. Reporter ensuite la valeur dans `index.html:429` avec le relevé en commentaire, comme cela a été fait pour `dcLoss`.

**Dépend de.** Rien.

**Confiance que ça règle le constat.** 80 % — sur un grand écart de niveau, l'arrondi du pourcentage devient négligeable, mais capacité utile et pertes restent partiellement couplées tant que l'item 8 n'est pas fait.

---

### [5] Donner des raccourcis au niveau actuel

**Constat.** Les cinq boutons de raccourci (`index.html:302-306`) modifient tous `ui.to`, jamais `ui.from` (`index.html:1001` et `index.html:1004`). Le niveau actuel n'a que des boutons ±1 (`index.html:284-285`) et un curseur de 0 à 99 (`index.html:288`), soit 3,4 pixels par point sur un écran de 390 px. Or c'est la seule valeur qui change à chaque session : la cible, elle, est restaurée telle quelle par `restoreUi` (`index.html:735`) et vaut presque toujours 80 ou 90 %.

**Bénéfice.** Le CPO branche à la borne, sort le téléphone, et doit appuyer 38 fois pour passer du 20 % mémorisé au 58 % affiché sur son tableau de bord — debout à côté de l'auto, souvent par mauvais temps. Trois appuis suffiraient.

**Proposition.** Ajouter une rangée de raccourcis sous le curseur du niveau actuel, câblée sur `ui.from`. Des paliers de 10 (`20 · 40 · 60 · 80`) ramènent n'importe quelle valeur à moins de cinq appuis du ±1. Réutiliser le balisage `.quick` (`index.html:96-101`) et l'écouteur existant en le paramétrant sur le champ visé.

**Dépend de.** Rien.

**Confiance que ça règle le constat.** 90 %

---

### [6] Figer le modèle sous tests

**Constat.** Aucun fichier de test dans le dépôt, aucun `package.json`, et `pages.yml:19-45` ne contient qu'un job de publication. Or le modèle a été corrigé trois fois en une seule journée (`500d797`, `980e2fb`, `5e85719`), et `500d797` corrigeait une inversion réelle : le cran « Froid » ressortait plus rapide que « Frais » parce que son plafond de 70 kW ne mordait pas sur une borne de 50 kW. Cette faute a vécu depuis le commit initial sans être vue. Les vérifications qui l'ont attrapée étaient des scripts jetables, jamais versionnés.

**Bénéfice.** Le CPO ajuste `acLoss` après la mesure de l'item 4, et réintroduit sans le voir une inversion entre deux crans de température. Il s'en aperçoit trois semaines plus tard, au chalet, devant une estimation absurde. Un test qui vérifie la monotonie des cinq crans l'aurait arrêté avant le déploiement.

**Proposition.** Un fichier de test en Node pur, sans dépendance, qui extrait `DEFAULTS` de `index.html` et vérifie les invariants : monotonie des crans de température sur chaque borne, absence de division par zéro, cohérence entre durée et kWh, et non-régression sur la session mesurée du 29 juillet (63 → 87 % en 25 min). Ajouter un job `node test.js` dans `pages.yml` avant l'étape de publication.

**Dépend de.** Rien.

**Effort.** Doute entre S et M tranché vers M : trois fichiers touchés (`test.js`, `pages.yml`, et probablement l'extraction de `DEFAULTS` hors du balisage pour rendre le modèle importable).

**Confiance que ça règle le constat.** 85 %

---

### [7] Journaliser les sessions de recharge

**Constat.** Le seul stockage existant couvre la configuration (`index.html:537`) et l'état de l'interface (`index.html:751-756`). Aucune trace des sessions réelles. Tout le recalage du 29 juillet a transité par des photos envoyées à un assistant, qui a modifié des constantes à la main — méthode qui ne survit pas à l'usage courant et dont il ne reste rien dans le produit.

**Bénéfice.** Le CPO termine une charge, saisit quatre valeurs en trente secondes, et l'app conserve de quoi se corriger. Sans ce socle, chaque amélioration du modèle exige de ressortir un appareil photo et de solliciter un tiers.

**Proposition.** Un écran de saisie post-session — borne, niveau de départ et d'arrivée, durée réelle, kWh facturés, cran de température — stocké sous une nouvelle clé `localStorage`, et joint à l'export JSON existant (`index.html:1149-1160`). Point d'entrée : une entrée dans la section *Résultat*, visible seulement quand une estimation est affichée.

**Dépend de.** Rien.

**Effort.** L par la table : nouvelle structure de données persistée.

**Confiance que ça règle le constat.** 80 % — la mécanique est simple, mais la valeur dépend d'une discipline de saisie que le code ne peut pas garantir.

---

### [8] Déduire la capacité utile du journal

**Constat.** `index.html:429` fixe `usableKwh: 75`, valeur de fiche technique de la LR DM. L'auto est de 2021. L'analyse de sensibilité exécutée sur le modèle du dépôt, en faisant varier la capacité de 68 à 77 kWh, déplace l'estimation de 7 min sur une BRCC 50, 34 min sur une charge maison et **235 min au chalet** — la première source d'incertitude du modèle, devant les pertes AC.

**Bénéfice.** Le CPO cesse d'estimer une auto de 2021 avec les chiffres de sa fiche de sortie d'usine. Concrètement, l'estimation au chalet cesse de dériver de plusieurs heures dans le même sens à chaque charge.

**Proposition.** Sur les sessions journalisées, chaque couple (kWh facturés, points de niveau gagnés) donne une estimation de capacité. Ajuster `usableKwh` sur la médiane des sessions récentes, en l'affichant dans les réglages comme valeur déduite et non saisie. Point d'entrée : la section *Configuration* (`index.html:1026`).

**Dépend de.** 7

**Confiance que ça règle le constat.** 70 % — capacité et pertes restent couplées ; il faut au moins une session en alternatif et une en continu pour les séparer, ce que le journal permet mais ne garantit pas.

---

### [9] Afficher l'écart entre estimation et réalité

**Constat.** Rien dans le dépôt ne compare une estimation à un résultat. Le produit ne conserve aucune trace de ce qu'il a annoncé. Le seul contrôle de justesse ayant jamais eu lieu est celui du 29 juillet, déclenché par une impression du CPO et conduit hors du produit.

**Bénéfice.** Le CPO ouvre les réglages et lit « écart moyen sur les 10 dernières charges : 3 min en continu, 22 min en alternatif ». Il sait immédiatement quelle branche du modèle attaquer, au lieu d'attendre qu'une estimation le surprenne assez pour qu'il s'en occupe.

**Proposition.** À la saisie d'une session, conserver l'estimation faite au branchement et calculer l'écart. Afficher l'écart moyen absolu, ventilé par type de borne — une moyenne globale masquerait exactement l'asymétrie entre le continu, calibré, et l'alternatif, qui ne l'est pas. Point d'entrée : un bloc dans la section *Configuration*.

**Dépend de.** 7

**Confiance que ça règle le constat.** 85 %

---

### [10] Résoudre dans le sens inverse

**Constat.** Le produit résout dans un seul sens. `startTime()` (`index.html:726-728`) donne l'instant de branchement et `renderResult` en déduit l'heure de fin (`index.html:857-860`). La question symétrique — partir d'une heure de départ voulue pour obtenir l'heure de branchement — n'existe nulle part dans le code.

**Bénéfice.** Le CPO arrive au chalet le vendredi soir et veut partir dimanche à 14 h avec 80 %. Aujourd'hui il tâtonne : il modifie l'heure de branchement jusqu'à ce que l'heure de fin lui convienne. Une seule saisie remplacerait cette recherche manuelle — et sur une charge de deux jours, se tromper d'heure de branchement se paie en heures d'attente.

**Proposition.** Une bascule dans la section *Résultat* qui inverse les entrées : heure de départ voulue et niveau visé en entrée, heure de branchement en sortie. Le moteur ne change pas — `simulate` donne une durée, il suffit de la soustraire au lieu de l'ajouter. L'heure de branchement modifiable, introduite par `4e2dfac`, fournit déjà le champ de saisie et son formatage.

**Dépend de.** Rien.

**Effort.** Doute entre S et M tranché vers M : la bascule touche le rendu du résultat, la barre ancrée et l'état d'interface persisté, soit trois zones du fichier.

**Confiance que ça règle le constat.** 85 %

---

### [11] Chiffrer le coût de la session

**Constat.** L'avertissement au-delà de 90 % (`index.html:873-889`) parle de tarification pénalisée sans donner un seul montant. `cfg.ceWarnSoc` (`index.html:477`) porte un seuil, jamais un tarif : aucune donnée de prix n'existe dans `DEFAULTS`.

**Bénéfice.** Le CPO lit « surcoût de 90 → 100 % : + 25 min » et n'a aucune idée si l'attente lui coûte 40 ¢ ou 4 $. Avec le montant, la décision de rester ou de partir se prend en une seconde au lieu d'être arbitrée au pif.

**Proposition.** Ajouter une table tarifaire éditable dans la configuration — le Circuit électrique facture au kWh ou à la minute selon le type de borne et le palier de niveau, et c'est cette bascule qui rend le dépassement coûteux. Afficher le montant estimé à côté des kWh, dans le bloc de détail (`index.html:891-905`). Traiter les tarifs comme configuration et non comme constante : ils changent.

**Dépend de.** Rien.

**Confiance que ça règle le constat.** 70 % — la structure tarifaire exacte n'est pas déductible du dépôt et devra être relevée sur un reçu.

---

### [12] Cesser de figer les défauts dans la configuration

**Constat.** `saveConfig` sérialise l'objet de configuration entier (`index.html:537`) et `loadConfig` le superpose aux défauts (`index.html:496`). Conséquence : un unique passage dans le panneau de réglages fige toutes les valeurs, y compris celles jamais touchées, et masque définitivement les défauts ultérieurs. Le commentaire posé en `index.html:482-485` documente le contournement employé — faire passer la clé de stockage à `v2` — et le prix payé : la capacité personnalisée du CPO a été effacée au passage.

**Bénéfice.** Le CPO mesure ses pertes en alternatif (item 4), corrige la valeur, déploie — et son propre téléphone continue d'afficher l'ancien chiffre parce qu'il avait ouvert les réglages six mois plus tôt. Il en conclut que le correctif ne marche pas et perd une soirée à chercher un bug qui n'existe pas.

**Proposition.** Ne stocker que les valeurs qui diffèrent réellement des défauts au moment de l'enregistrement, et les superposer à la lecture. Une valeur jamais touchée suit alors les défauts, une valeur délibérément réglée survit. `mergeConfig` (`index.html:502-530`) accepte déjà des objets partiels : c'est `saveConfig` qui doit changer, plus une migration unique depuis le format `v2`.

**Dépend de.** Rien.

**Effort.** L par la table : changement de la structure de données persistée, avec migration.

**Confiance que ça règle le constat.** 90 %

---

### Écarté

- **Corriger le millésime du véhicule (2021 ou 2022).** `index.html:7` et le manifeste annoncent 2021 ; l'application du Circuit électrique affiche 2022. Non vérifiable ici : le millésime réel n'est pas déductible du dépôt. Devient une question au CPO plutôt qu'un item.
- **Mesurer le palier 75-85 % de la courbe DC.** Non mesuré, documenté comme tel en `index.html:437-439`. L'analyse de sensibilité lui donne 1 minute d'écart sur la plage plausible : intellectuellement insatisfaisant, pratiquement nul.
- **Distinguer température du pack et température ambiante.** Faiblesse conceptuelle réelle — les crans sont libellés en température de l'air, le modèle agit sur la batterie. 6 minutes d'écart maximum, et zéro sur une borne 50 kW, où la borne plafonne sous ce que le pack accepte même froid.
- **Interpoler la courbe de charge au lieu de la découper en paliers.** Le biais est structurel mais absorbé par le recalage des paliers eux-mêmes. Deviendra une conséquence de l'item 8 le jour où le journal fournira assez de couples (niveau, puissance).
- **Ajouter d'autres réseaux de bornes.** Mécaniquement simple, mais chaque facteur réel serait deviné — exactement le 0,85 générique qui faussait le modèle avant le 29 juillet. À faire au fil des visites, avec une mesure, pas d'avance.
- **Compter l'usage de l'interface.** Avec un seul utilisateur, le comptage renseigne moins qu'une question posée directement. À reconsidérer si le produit sort de son unique téléphone.
- **Suivre la dégradation de la batterie dans le temps.** Sous-produit gratuit de l'item 8, mais exige plusieurs mois de sessions avant de valoir quoi que ce soit. À laisser émerger.

---

## 5. Décisions qui appartiennent au CPO

1. **Quel écart d'estimation est acceptable, et à partir de quand ça vaut une soirée de travail ?** Le modèle en continu tient à 20 secondes près sur la session mesurée ; l'alternatif n'a jamais été confronté au réel. Sans seuil déclaré, il n'y a aucun moyen de savoir si les items 4, 8 et 9 sont déjà suffisants ou pas encore commencés.

2. **La saisie manuelle d'une session après chaque charge est-elle réaliste, ou est-ce une corvée qui sera abandonnée après trois fois ?** Les items 7, 8 et 9 — six sessions de travail, soit près du tiers du carnet — reposent entièrement sur cette discipline. Si la réponse est non, il faut les retirer et accepter que le modèle reste calibré à la main.

3. **Le chalet est-il le cas d'usage central ou un cas limite toléré ?** L'analyse de sensibilité montre que les charges en 120 V amplifient chaque incertitude par un facteur trente. Si c'est le cas central, l'item 4 devient urgent et l'item 8 le suit ; si c'est un cas limite, tout le thème alternatif redescend d'une vague.

4. **Le produit doit-il rester utilisable sans réseau et sans compte, même si ça coûte de la précision ?** La frontière posée en section 1 — pas d'API constructeur — est un choix, pas une contrainte technique. Le renverser rendrait le niveau de charge lisible automatiquement et supprimerait la friction de l'item 5, au prix d'un backend, de secrets à stocker et de la fin du fonctionnement hors ligne.

5. **Le dépôt doit-il rester public ?** Il l'est aujourd'hui, ce qui rend l'hébergement gratuit. Le journal des sessions de l'item 7 reste en stockage local, donc rien de personnel ne serait publié — mais l'export JSON de la configuration, s'il est un jour joint au dépôt pour archivage, contiendrait des relevés de recharge horodatés.

---

*Établi à partir du seul contenu du dépôt et des mesures relevées le 29 juillet 2026. Les analyses de sensibilité citées ont été exécutées contre `DEFAULTS` au commit `4e2dfac`.*
