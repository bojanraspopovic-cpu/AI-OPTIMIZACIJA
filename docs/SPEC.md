# DTM Smoothing — Specifikacija

## Svrha projekta

Geodetski snimak as-built stanja autoputa (≈6 km). Stari asfalt ima deformirane
poprečne nagibe. Cilj: predložiti novi DTM za overlay sloj sa konzistentnim
poprečnim nagibima, unutar zadanog visinskog koridora.

Skripta ne rekonstruiše originalni projekat — predlaže novu, glatku geometriju
koja poštuje fizičke i funkcionalne constrainte.

---

## Korisnički parametri

Korisnik mora definisati sljedeće parametre prije pokretanja:

| Parametar | Opis | Primjer |
|-----------|------|---------|
| `KORIDOR_DONJI` | Donja granica visinskog koridora (cm) | -1.0 |
| `KORIDOR_GORNJI` | Gornja granica visinskog koridora (cm) | +3.0 |
| `MIN_NAGIB` | Minimalni poprečni nagib za odvodnju (%) | 2.5 |
| `OUTLIER_PRAG_PRAVAC` | Prag za outlier detekciju na pravcima (cm) | 5.0 |
| `OUTLIER_PRAG_KRIVINA` | Prag za outlier detekciju na krivinama (cm) | 8.0 |
| `MIN_TACAKA_PRESJEK` | Minimalan broj tačaka po presjeku | 3 |

---

## Geometrija trake

Skripta obrađuje jednu traku odvojeno.

**Terminologija (primjer: desna traka autoputa):**

- **Pivot (unutrašnja ivica)** — lijeva ivica desne trake, sredina kolnika.
  Rotacijska tačka od koje se računa poprečni nagib.
- **Spoljna ivica** — desna ivica desne trake.
- Poprečni nagib je uvijek upravan na tangentu trase u toj tački.
- Širina trake nije konstantna — varira na krivinama i prijelaznicama.

**Signed nagib:**

```
nagib (%) = (Z_spoljna - Z_pivot) / sirina × 100

Pozitivan: Z pada od pivota prema spoljnoj ivici
Negativan: Z pada od spoljne ivice prema pivotu

Odvodnja check: abs(nagib) >= MIN_NAGIB
```

**Kritično:** Svi nagibi se računaju i smoothiraju u procentima (%), nikad kao
apsolutna visinska razlika. Time se eliminišu vještački skokovi na mjestima
gdje se širina trake mijenja.

---

## Visinski koridor (constraint)

Novi Z mora biti unutar koridora u odnosu na originalne Z vrijednosti:

```
Z_original + KORIDOR_DONJI <= Z_novi <= Z_original + KORIDOR_GORNJI
```

Provjerava se na:
1. **Pivot Z** — nakon smoothinga (Skripta 0)
2. **Spoljna Z** — nakon primjene novog nagiba (Skripta 1)
3. **Srednje tačke** — QA check za "prodor u trbuh asfalta" (Skripta 1)

Tačke između ivica dobijaju Z linearnom interpolacijom po signed offsetu.
QA check na srednjim tačkama: ako je nova Z ispod stare Z + KORIDOR_DONJI,
loguj upozorenje za taj presjek.

---

## Hijerarhija constrainta

Kad dođe do konflikta, prioritet je:

1. **Visinski koridor** — apsolutni prioritet, nikad se ne krši
2. **Odvodnja (≥ MIN_NAGIB)** — koriguje se automatski ako ne krši koridor
3. **Glatkoća** — žrtvuje se prva ako bi kršila gornja dva

---

## Koordinatni sistem i stacionaže

**Sve stacionaže i projekcije su 2D (XY ravnina).**

Kumulativna 3D dužina polilinije varira sa nagibom terena i stvara
kumulativno odstupanje u stacionažama. Na 6 km trase sa prosječnim nagibom
od 5%, razlika između 2D i 3D dužine iznosi ≈12 cm na 100 m.

Normale na pivot poliliniju se također računaju u XY ravnini.

---

## Ulazni podaci

### Zahtjevi prema korisniku

Korisnik mora pripremiti sljedeće fajlove iz Civil 3D:

**Za Skriptu 0 (pivot smoothing):**

| Fajl | Sadržaj | Kako dobiti |
|------|---------|-------------|
| `pivot_raw.csv` | 3D pivot polilinija (X, Y, Z) | C3D: desni klik na poliliniju → Export CSV |
| `pivot_minus1.csv` | Pivot offsetovana za KORIDOR_DONJI | C3D: vertikalni offset polilinije |
| `pivot_plus3.csv` | Pivot offsetovana za KORIDOR_GORNJI | C3D: vertikalni offset polilinije |

**Za Skriptu 1 (DTM smoothing):**

| Fajl | Sadržaj | Kako dobiti |
|------|---------|-------------|
| `tacke.csv` | Sve XYZ tačke trake (geodetski snimak) | Export iz geodetskog softvera |
| `pivot_smooth.csv` | Smoothirana pivot linija | Output Skripte 0 |
| `spoljna.csv` | 3D polilinija spoljne ivice (X, Y, Z) | C3D: Export CSV |
| `alignment_info.csv` | Definicija dionica trase | Ručno od korisnika |

**Format svih CSV fajlova:** X, Y, Z po redu, sa ili bez zaglavlja.
Skripta automatski detektuje zaglavlje.

### alignment_info.csv — format i pravila

```
od_stat, do_stat, tip
0,       420,     PRAVAC
420,     580,     PRIJELAZNICA
580,     1200,    KRIVINA
1200,    1320,    PRIJELAZNICA
```

- `tip`: PRAVAC | KRIVINA | PRIJELAZNICA
- Stacionaže su aproksimativne (±50 m je prihvatljivo)
- **Smjer nagiba se ne unosi** — računa se automatski iz geometrije
- Dionice moraju pokriti kompletnu stacionažu bez rupa
- Skripta dodaje transition zone (30–50 m) oko svake granice dionice

### Pretpostavke o kvaliteti ulaznih podataka

- Pivot i spoljna polilinija pokrivaju istu dužinu (±tolerancija)
- Tačke u tacke.csv pripadaju jednoj traci (nema tačaka sa susjedne trake)
- Geodetski snimak ima konzistentan razmak presjeka (5 m, 10 m, ili drugi)
- Skripta automatski detektuje razmak presjeka iz podataka

---

## Obrada — Skripta 0 (Pivot Smoothing)

### Problem

Pivot linija je longitudinalni profil snimljenog terena. Ima blagi šum oko
stvarnog trenda i povremene grube greške (vegetacija, pomaknut reflektor).

### Tri situacije

| Situacija | Opis | Tretman koridora |
|-----------|------|-----------------|
| Normalna dionica | Blagi šum oko trenda | Koridor oko izmjerene Z |
| Outlier | Nagli skok koji nije stvarna geometrija | Koridor oko interpolirane Z iz susjeda |
| Vertikalna krivina | Sistematska promjena Z | QP automatski prati kroz podatke |

### Outlier detekcija

Na svim dionicama (uključujući krivine):

1. Fituj lokalni trend (parabolični fit na ±10 tačaka)
2. Izračunaj rezidual = odstupanje tačke od lokalnog trenda
3. Ako rezidual > prag → tačka je outlier

Pragovi:
- PRAVAC / PRIJELAZNICA: `OUTLIER_PRAG_PRAVAC` (default 5 cm)
- KRIVINA: `OUTLIER_PRAG_KRIVINA` (default 8 cm, jer sistematska promjena Z
  na krivini može izgledati kao outlier bez trenda)

Za outlier tačke: koridor se gradi oko interpolirane Z iz susjednih normalnih
tačaka, ne oko izmjerene Z.

### QP optimizacija

```
Minimizuj:  ukupnu krivinu (drugu derivaciju) pivot linije
Uz uslove:  Z_smooth[i] >= Z_koridor_donja[i]  za svaki i
            Z_smooth[i] <= Z_koridor_gornja[i]  za svaki i
```

Hard constraints — koridor se nikad ne krši. Rješava se cijela trasa odjednom.

---

## Obrada — Skripta 1 (DTM Smoothing)

### Pipeline

**Korak 1 — Učitavanje i validacija:**
- Učitaj sve fajlove
- Provjeri da pivot i spoljna pokrivaju istu dužinu
- Provjeri da alignment_info pokriva kompletnu stacionažu
- Provjeri da nema tačaka u tacke.csv sa udaljenošću > 2× širine trake od pivota

**Korak 2 — Reparametrizacija:**
- Obje polilinije na jednake intervale (1 m) u XY ravnini
- Kontrolisana gustoća neovisno o originalnom C3D exportu

**Korak 3 — Provjera orijentacije:**
- Spoljna ivica mora biti u konzistentnom smjeru normale kroz cijelu trasu
- Provjera jednom na početku

**Korak 4 — Geometrija presjeka (na svakih 5 m duž pivot polilinije):**
- Tangenta: centrirana diferenca ±3 tačke na reparametriziranoj liniji
- Normala: ortogonalna na tangentu u XY ravnini
- Širina: udaljenost do spoljne polilinije u smjeru normale
- Tip dionice: iz alignment_info.csv (sa transition zone na granicama)

**Korak 5 — Smoothiranje širine:**
- Rolling average ili spline na seriji izmjerenih širina
- Sanity check: ako sirova širina odskače > 20 cm od smoothirane → upozorenje

**Korak 6 — Projekcija tačaka:**
- Za svaku tačku iz tacke.csv: stacionaža + signed offset
- Signed offset: pozitivan = spoljna strana, negativan = unutra
- Projekcija u XY ravnini, segment-by-segment

**Korak 7 — Automatska detekcija razmaka presjeka:**
- Iz projektovanih stacionaža odredi klastere (prosječan razmak)
- Prozor za tačke po presjeku = ± pola razmaka
- Ako presjek ima < MIN_TACAKA_PRESJEK tačaka → proširi prozor na ±1.5×
- Ako i dalje nema dovoljno → loguj MANUAL CHECK

**Korak 8 — 1D interpolacija po presjeku:**
- Za svaki presjek: tačke unutar prozora, sortirane po signed offsetu
- 1D interpolacija Z(offset)
- Izračunaj signed nagib: nagib = (Z_spoljna - Z_pivot) / sirina × 100

**Korak 9 — Smoothing nagiba po dionicama:**
- PRAVAC / KRIVINA: rolling average, prozor = min(max(duzina/3, 50 m), 150 m)
  - Rolling average se računa samo unutar granica dionice (truncation na rubovima)
  - Trend koji ostaje iznad MIN_NAGIB nije šum — ne eliminisati
- PRIJELAZNICA: linearna interpolacija nagiba između susjednih dionica
- Transition zone (30–50 m) oko svake granice: blending oba tretmana

**Korak 10 — Nove Z vrijednosti:**

Za PRAVAC / KRIVINA:
1. Nova pivot Z = iz pivot_smooth.csv
2. Nova spoljna Z = Z_pivot_novi - (nagib_smooth / 100 × sirina)
3. Ako abs(nagib_smooth) < MIN_NAGIB:
   - Postavi nagib na sign(nagib) × MIN_NAGIB
   - Prilagodi spoljnu Z
   - Status = AUTO-ADJUSTED
4. Constraint provjera spoljne Z:
   - OK: unutar koridora i abs(nagib) ≥ MIN_NAGIB
   - AUTO-ADJUSTED: van koridora → prilagodi nagib prema granici
   - MANUAL CHECK: ni granični nagib ne zadovoljava oba constrainta
5. Srednje tačke: linearna interpolacija po signed offsetu

Za PRIJELAZNICA:
1. Nova pivot Z = iz pivot_smooth.csv
2. Nagib = linearna interpolacija između susjednih dionica
3. Constraint provjera spoljne Z (bez odvodnja checka)
4. Status = PRIJELAZNICA
5. Srednje tačke: linearna interpolacija po signed offsetu

**Korak 11 — QA check za srednje tačke:**
- Za svaku srednju tačku: nova Z vs stara Z
- Ako nova Z < stara Z + KORIDOR_DONJI → upozorenje "prodor u trbuh asfalta"

---

## Output

### Skripta 0

| Fajl | Sadržaj |
|------|---------|
| `pivot_smooth.csv` | X, Y, novi Z za pivot liniju |
| `pivot_qa.csv` | Po tački: orig_Z, novi_Z, delta, status (NORMALNO / OUTLIER / KORIDOR_GRANICA) |

**QA upozorenja u pivot_qa.csv:**
- Tačke gdje smooth linija leži na granici koridora 3+ uzastopne tačke
  → moguće da je ulazni podatak loš na toj lokaciji

### Skripta 1

| Fajl | Sadržaj |
|------|---------|
| `novi_dtm.csv` | Originalne XY, nove Z za sve tačke |
| `qa_report.csv` | Po presjeku: stacionaža, tip_dionice, nagib_stari, nagib_novi, delta_pivot, delta_spoljna, odvodnja_ok, prodor_check, status |

**Status vrijednosti:** OK | AUTO-ADJUSTED | MANUAL CHECK | PRIJELAZNICA

**odvodnja_ok:** DA | NE | N/A (za prijelaznice)

**prodor_check:** OK | UPOZORENJE (ako srednja tačka prodire ispod starog DTM-a)

---

## Validacija ulaznih podataka (error handling)

Skripta provjerava na startu i javlja grešku sa lokacijom problema:

| Provjera | Akcija ako ne prođe |
|----------|---------------------|
| Pivot i spoljna iste dužine (±5%) | STOP + poruka |
| alignment_info pokriva cijelu stacionažu | STOP + poruka sa rupama |
| alignment_info nema preklapajuće dionice | STOP + poruka |
| Tačke predaleko od pivota (>2× širina) | Izbaci + loguj u QA |
| Presjek sa < MIN_TACAKA_PRESJEK tačaka | Proširi prozor, ako ne uspije → MANUAL CHECK |
| Širina odskače > 20 cm od smoothirane | Loguj upozorenje sa stacionažom |

---

## Prijelaznice (Δnagib check)

Nakon interpolacije nagiba na prijelaznici, provjeri promjenu nagiba između
uzastopnih presjeka. Ako Δnagib prelazi 0.5% po presjeku → loguj upozorenje.

Ovo je QA check, ne automatska korekcija — kratka prijelaznica sa velikim
Δnagibom je fizička realnost, ne greška skripte.

---

## Buduće stavke (van prve verzije)

- Visual QA: dijagram nagiba (%) starog vs novog stanja po stacionaži
- QA check: vertikalni radijusi pivot linije nakon smoothinga
  (informativno, ne kao QP constraint — ne popravljamo vertikalnu
  geometriju autoputa sa 4 cm overlay-a)
