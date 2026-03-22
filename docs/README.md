# DTM Smoothing

Alat za generisanje novog DTM-a za asfaltni overlay na osnovu geodetskog
snimka izvedenog stanja autoputa.

## Struktura

```
dtm-smoothing/
├── docs/
│   ├── SPEC.md              ← geometrijska specifikacija i constrainti
│   └── IMPLEMENTATION.md    ← tehničke odluke i rationale
├── scripts/
│   ├── s0_pivot_smoothing.py
│   └── s1_dtm_smoothing.py
├── tests/
│   └── test_data/           ← sintetički dataset za validaciju
└── README.md
```

## Workflow

```
1. Pripremi ulazne podatke (vidi docs/SPEC.md → "Zahtjevi prema korisniku")
2. Pokreni Skriptu 0 → pivot_smooth.csv
3. Pokreni Skriptu 1 → novi_dtm.csv + qa_report.csv
4. Pregledaj qa_report.csv — fokus na MANUAL CHECK i UPOZORENJE statuse
5. Vizualna provjera u ProVi CAD za flagirane presjeke
```

## Korisnički parametri

Prije pokretanja definiši u config fajlu ili na vrhu skripte:

| Parametar | Default | Opis |
|-----------|---------|------|
| `KORIDOR_DONJI` | -1.0 cm | Donja granica visinskog koridora |
| `KORIDOR_GORNJI` | +3.0 cm | Gornja granica visinskog koridora |
| `MIN_NAGIB` | 2.5 % | Minimalni poprečni nagib za odvodnju |
| `OUTLIER_PRAG_PRAVAC` | 5.0 cm | Outlier prag na pravcima |
| `OUTLIER_PRAG_KRIVINA` | 8.0 cm | Outlier prag na krivinama |
| `MIN_TACAKA_PRESJEK` | 3 | Min. tačaka po presjeku |

## Zavisnosti

```
numpy pandas scipy cvxpy
```

## Dokumentacija

- **Šta treba postići:** `docs/SPEC.md`
- **Zašto ovako:** `docs/IMPLEMENTATION.md`
