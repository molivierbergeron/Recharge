# Recharge — Polestar 2

Estimateur de temps de recharge pour une **Polestar 2 2021 Long Range Dual Motor
Performance**. Répond à une seule question : *je branche, dans combien de temps je
reviens chercher l'auto ?*

Page HTML autonome, zéro dépendance, installable comme app iPhone et fonctionnelle
hors ligne.

## Utilisation

Ouvrir `index.html`, ou la version publiée sur GitHub Pages. Sur iPhone :
**Partager → Sur l'écran d'accueil**. L'app s'ouvre alors en plein écran et
fonctionne sans réseau.

Trois gestes : le SOC de départ et la cible, la borne et les conditions,
puis la lecture du résultat — durée et **heure de retour**.

## Le modèle

Le temps n'est pas calculé avec une puissance moyenne plate. La courbe
d'acceptation de l'auto est intégrée numériquement par pas de 1 % de SOC :

```
pour soc de départ à cible (pas de 1 %) :
  p_auto      = courbeDC(soc) × facteur_température   # ou 11 kW en AC
  p_borne     = nominal × (partagée ? ratio : 1) × facteur_réel
  p_thermique = plafond(température, préchauffage)
  p_eff       = min(p_auto, p_borne, p_thermique)
  temps      += (capacité × 0,01) / p_eff
```

Courbe de référence (borne ≥ 150 kW, batterie tempérée) :

| SOC | kW acceptés |  | SOC | kW acceptés |
|---|---|---|---|---|
| 0–30 % | 147 |  | 75–85 % | 45 |
| 30–50 % | 120 |  | 85–90 % | 28 |
| 50–65 % | 95 |  | 90–100 % | 12 |
| 65–75 % | 70 |  | | |

Deux quantités distinctes, volontairement :

- l'**énergie dans la batterie** détermine le % de SOC et le temps ;
- l'**énergie tirée de la borne** (batterie + pertes) détermine les kWh facturés.

C'est ce qui explique l'écart observé sur les sessions réelles : un plein
0 → 100 % affiche 75 kWh dans la batterie mais **83,3 kWh facturés**, conforme
aux ~83 kWh constatés.

### Température

Cinq états, du plus chaud au plus froid. Les deux premiers agissent comme un
facteur sur la courbe, les trois suivants comme un **plafond de puissance** :

| Condition | DC | AC |
|---|---|---|
| Été (> 15 °C) | ×1,0 | ×1,0 |
| Frais (5 à 15 °C) | ×0,9 | ×1,0 |
| Froid (−10 à 5 °C) | plafond 70 kW | ×0,9 |
| Très froid (−20 à −10 °C) | plafond 40 kW | ×0,75 |
| Extrême (< −20 °C) | plafond 25 kW | 120 V : charge nette ≈ nulle |

Le préchauffage batterie (navigation lancée 20–30 min avant) remonte l'état
thermique de deux crans côté DC : « très froid » se comporte comme « frais ».
Sans effet en AC.

Par grand froid sur une prise 120 V, l'app **refuse d'estimer** plutôt que
d'afficher un chiffre trompeur : le chauffage de la batterie consomme ce que la
prise fournit.

### Règle Circuit électrique > 90 %

Sur une borne CE rapide, viser plus de 90 % déclenche un bandeau : la
tarification est majorée et la vitesse tombe au minimum. Le temps jusqu'à 90 %
est affiché comme recommandation, et les derniers pourcents comme surcoût
séparé. L'app informe, elle ne bloque pas.

## Réglages

Tout le modèle est éditable derrière l'engrenage : capacité, pertes, table des
bornes (nominal et facteur réel), les sept segments de la courbe DC, les
plafonds thermiques, l'arrondi. Persistance en `localStorage`, avec export et
import JSON, et retour aux défauts.

## Repères validés

| Scénario | Résultat |
|---|---|
| 20 → 80 %, BRCC 100 kW seul, été | 40 min |
| 20 → 80 %, BRCC 50 kW, été | 65 min |
| 20 → 80 %, BRCC 180 kW seul, été | 30 min |
| 80 → 90 %, BRCC 100 kW | + 15 min |
| 90 → 100 %, BRCC 100 kW | + 40 min, avec avertissement |
| 10 → 80 %, maison 11 kW | 5 h 05 |
| 20 → 80 %, BRCC 100 kW, très froid | 1 h 10 (40 min avec préchauffage) |
| Chalet 120 V à −25 °C | aucune estimation |

## Fichiers

| | |
|---|---|
| `index.html` | l'app entière — structure, style et modèle |
| `sw.js` | service worker, cache pour l'usage hors ligne |
| `manifest.webmanifest` | métadonnées d'installation |
| `icon-*.png` | icônes d'écran d'accueil |

## V2 envisagée

Split partiel (l'autre véhicule part en cours de session), météo automatique
selon la géolocalisation, mode calibration à partir des sessions réelles, coût
estimé de la session.
