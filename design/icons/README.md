# Pistes d'icône

Trois propositions pour remplacer `icon-*.png`, dans un registre plus proche de
l'esthétique Polestar : géométrie construite, monochrome dominant, orange réservé
à un seul rôle par piste.

| Piste | Fichier | Fond | Traitement |
|---|---|---|---|
| A — Filaire | `option-a-filaire.svg` | Charbon `#141413` | Éclair évidé, trait 16/512, monochrome |
| **B — Instrument** ✅ | `option-b-instrument.svg` | Papier `#FAFAF8` | Anneau de charge ouvert, orange = portion parcourue |
| C — Tranché | `option-c-decompose.svg` | Papier `#FAFAF8` | Éclair plein coupé par une saignée horizontale |

**B est la piste retenue** : c'est la source de `icon-512.png`, `icon-192.png` et
`icon-180.png` à la racine. A et C sont conservées comme archive de la recherche.

Tout le dessin tient dans un rayon de 200 px autour du centre (256, 256), soit la
zone sûre des icônes masquables Android.

## Régénérer les PNG

```sh
pip install cairosvg
python3 - <<'PY'
import cairosvg
for n in ['option-a-filaire', 'option-b-instrument', 'option-c-decompose']:
    for s in (512, 192, 64):
        cairosvg.svg2png(url=f'{n}.svg', write_to=f'{n}-{s}.png',
                         output_width=s, output_height=s)
PY
```

## Réexporter les icônes de l'app

```sh
python3 - <<'PY'
import cairosvg
src = 'design/icons/option-b-instrument.svg'
for s, out in [(512, 'icon-512.png'), (192, 'icon-192.png'), (180, 'icon-180.png')]:
    cairosvg.svg2png(url=src, write_to=out, output_width=s, output_height=s)
PY
```

À lancer depuis la racine du dépôt. Penser à incrémenter `CACHE` dans `sw.js`,
sinon les appareils déjà installés gardent les anciennes icônes en cache.

Le fond de la piste B est le même `#FAFAF8` que `background_color` et
`theme_color` du manifeste — rien à ajuster de ce côté. Une piste sur fond
charbon (A) demanderait de revoir ces deux valeurs.
