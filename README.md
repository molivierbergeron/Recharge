# Recharge

Estimateur de temps de recharge pour une **Polestar 2 2021 Long Range Dual Motor Performance**,
pensé pour une seule question : *je branche, dans combien de temps je reviens chercher l'auto ?*

Page HTML autonome, aucune dépendance, déployée sur GitHub Pages et installable comme app iPhone.

## Utilisation

Trois zones sur un seul écran :

1. **Niveau de charge** — « Je suis à » / « Je veux », steppers et sliders, raccourcis `+5` `+10` `+20`
   et cibles directes `80 %` `90 %`.
2. **Borne** — préréglage de borne, plus les bascules *borne partagée* et *préchauffage batterie*.
3. **Résultat** — durée, **heure de retour**, et un détail dépliable (kWh, puissance moyenne,
   temps par tranche de 10 %).

Tant que le résultat n'est pas à l'écran, un rappel ancré en bas garde la durée et l'heure de retour
visibles pendant qu'on ajuste les réglages.

L'engrenage en haut à droite ouvre la configuration ; le même bouton la referme.

## Le modèle

La charge n'est pas calculée avec une puissance moyenne plate. La courbe d'acceptation de l'auto est
intégrée numériquement par pas de 1 % :

```
pour soc de départ à cible, par pas de 1 % :
  p_auto      = courbe DC au SOC courant  (ou le chargeur embarqué en AC)
  p_borne     = nominal × (partagée ? ratio : 1) × facteur réel
  p_thermique = plafond de la condition de température
  p_eff       = min(p_auto, p_borne, p_thermique)
  temps      += (capacité × 1 %) / p_eff
```

La durée affichée est arrondie à la tranche de 5 minutes supérieure, et l'heure de retour vaut
`maintenant + durée affichée`.

### Énergie dans la batterie et énergie facturée

Le modèle distingue les deux :

- l'**énergie dans la batterie** détermine le % de SOC et le temps de charge ;
- l'**énergie tirée de la borne** = énergie batterie ÷ (1 − pertes), et c'est elle qui est facturée.

Les pertes n'allongent donc pas la durée : elles expliquent l'écart entre les deux chiffres. Avec les
défauts, un plein 0 → 100 % en AC stocke 75 kWh et en fait facturer 83,3 — ce qui correspond aux
sessions réelles.

### Courbe DC de référence

Borne ≥ 150 kW, batterie tempérée.

| SOC | Puissance acceptée |
|---|---|
| 0–30 % | 150 kW |
| 30–50 % | 120 kW |
| 50–65 % | 95 kW |
| 65–75 % | 70 kW |
| 75–85 % | 45 kW |
| 85–90 % | 28 kW |
| 90–100 % | 12 kW |

### Préréglages de bornes

| Préréglage | Nominal | Facteur réel |
|---|---|---|
| Maison — niveau 2 (48 A) | 11 kW | 0,95 |
| Chalet — 120 V | 1,2 kW | 0,85 |
| CE — niveau 2 publique | 7,2 kW | 0,95 |
| CE — BRCC 50 kW | 50 kW | 0,85 |
| CE — BRCC 100 kW | 100 kW | 0,85 |
| CE — BRCC 180 kW | 180 kW | 0,85 |

Le facteur réel traduit ce que la borne livre vraiment : tension réseau, câble, plafonnement sous le
nominal. La bascule *borne partagée* divise le nominal de moitié sur les BRCC 100 et 180.

### Température

| Condition | DC | AC |
|---|---|---|
| Été (> 15 °C) | ×1,0 | ×1,0 |
| Frais (5 à 15 °C) | ×0,9 | ×1,0 |
| Froid (−10 à 5 °C) | plafond 70 kW | ×0,9 |
| Très froid (−20 à −10 °C) | plafond 40 kW | ×0,75 |
| Extrême (< −20 °C) | plafond 25 kW | ×0,6 |

Le sélecteur est pré-rempli selon la saison à partir de la date système, puis mémorisé pendant 24 h.

Le **préchauffage batterie** remonte les plafonds de deux crans côté DC : par très grand froid,
l'auto se comporte comme par temps frais. Aucun effet en AC.

En **120 V sous −20 °C**, l'app refuse de donner un chiffre : le maintien en température consomme à
peu près tout ce que la prise fournit, et n'importe quelle durée affichée serait trompeuse.

### Règle Circuit électrique au-delà de 90 %

Viser plus de 90 % sur une borne rapide déclenche un avertissement — tarification pénalisée et
vitesse réduite à ~12 kW. L'app affiche le temps jusqu'à 90 % comme recommandation, le surcoût
90 → cible séparément, et un bouton pour ramener la cible à 90 %. Elle informe, elle ne bloque pas.

Quand le départ est *déjà* au-dessus de 90 %, la recommandation « jusqu'à 90 % » n'a plus de sens :
l'avertissement reste affiché, mais il rappelle plutôt que toute la session se joue au débit réduit
et qu'une borne de niveau 2 ferait le même travail.

## Configuration

Tout est éditable et conservé dans `localStorage` : capacité, chargeur embarqué, pertes AC/DC, table
des bornes, les 7 segments de la courbe DC, facteurs et plafonds de température, seuil
d'avertissement. Export et import JSON, et remise aux défauts.

## En ligne

**https://molivierbergeron.github.io/Recharge/**

Sur iPhone : ouvrir la page dans Safari → Partager → *Sur l'écran d'accueil*. Le service worker met
les fichiers en cache, l'app fonctionne ensuite hors ligne.

## Déploiement

Rien à construire : le site *est* le contenu du dépôt.

GitHub Pages sert la branche `gh-pages`, et le workflow `.github/workflows/pages.yml` y republie la
racine en commit orphelin à chaque push sur `main` ou sur une branche de travail `claude/*` — le
dépôt n'a pas encore de branche par défaut, et le site doit suivre la branche réellement utilisée.
`.github` est retiré du site publié : le jeton du workflow n'a pas le droit d'écrire des fichiers de
workflow sur une autre branche, et le site n'en a pas besoin.

Le service worker sert le cache d'abord et le rafraîchit en arrière-plan : après un déploiement, la
nouvelle version apparaît au chargement suivant. Pour qu'elle s'applique immédiatement, incrémenter
`CACHE` dans `sw.js`.

## Fichiers

| Fichier | Rôle |
|---|---|
| `index.html` | Toute l'app — balisage, styles et modèle |
| `sw.js` | Service worker, cache hors ligne |
| `manifest.webmanifest` | Métadonnées d'installation |
| `icon-*.png` | Icônes d'écran d'accueil |
| `.github/workflows/pages.yml` | Publication sur `gh-pages` |

## Suite

- Partage partiel : l'autre véhicule quitte en cours de session
- Météo automatique par géolocalisation (Environnement Canada)
- Mode calibration à partir des sessions réelles
- Coût estimé de la session

## Réserve

Ce sont des estimations. La courbe de référence, les facteurs réels et les plafonds thermiques sont
des valeurs de départ raisonnables, pas des mesures : l'état de la batterie, la borne et la
température réelle feront toujours varier le résultat.
