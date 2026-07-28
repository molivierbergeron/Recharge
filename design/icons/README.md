# Pistes d'icône

Trois propositions pour remplacer `icon-*.png`, dans un registre plus proche de
l'esthétique Polestar : géométrie construite, monochrome dominant, orange réservé
à un seul rôle par piste.

| Piste | Fichier | Fond | Traitement |
|---|---|---|---|
| A — Filaire | `option-a-filaire.svg` | Charbon `#141413` | Éclair évidé, trait 16/512, monochrome |
| B — Instrument | `option-b-instrument.svg` | Papier `#FAFAF8` | Anneau de charge ouvert, orange = portion parcourue |
| C — Tranché | `option-c-decompose.svg` | Papier `#FAFAF8` | Éclair plein coupé par une saignée horizontale |

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

Une fois une piste retenue, exporter en 512 / 192 / 180 px vers `icon-512.png`,
`icon-192.png` et `icon-180.png` à la racine, et ajuster `background_color` /
`theme_color` du manifeste si le fond de l'icône change.
