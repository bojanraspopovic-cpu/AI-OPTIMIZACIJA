# DTM Smoothing — Implementacijske Odluke

Ovaj dokument objašnjava *zašto* su donesene ključne tehničke odluke.
Kod živi u .py fajlovima — ovdje nema snippeta za održavanje.

---

## Zašto QP umjesto spline smoothinga (Skripta 0)

`UnivariateSpline` i slični standardni smoothing pristupi ne podržavaju
hard constraints. Spline minimizuje odstupanje od podataka uz kontrolu
glatkoće, ali ne garantuje da rezultat ostaje unutar zadanog koridora.

QP (Quadratic Programming) nativno podržava nejednakosne constrainte:

- Funkcija cilja: minimizacija ukupne krivine (druga derivacija)
- Hard constraints: svaka tačka mora ostati unutar koridora
- Rješava cijelu trasu odjednom — globalno optimalno rješenje
- Implementacija: cvxpy + OSQP solver

Alternativa bi bila iterativni spline sa post-hoc clampingom, ali to
ne garantuje glatkoću nakon clampinga i zahtijeva višestruke iteracije
bez garancije konvergencije.

---

## Zašto 1D interpolacija po presjecima umjesto 2D (Skripta 1)

Traka autoputa je uska (3.5–4 m) i duga (6 km). Globalni 2D interpolator
(npr. griddata, RBF) na ovako ekstremnom aspect ratio-u proizvodi artefakte
— tačke na susjednim presjecima utiču jedna na drugu lateralno iako su
geometrijski nezavisne.

1D interpolacija po presjeku:
- Svaki presjek se obrađuje nezavisno
- Tačke se sortiraju po signed offsetu (udaljenost od pivota u smjeru normale)
- Z se interpolira kao funkcija offseta
- Geometrijski konzistentno sa načinom kako C3D prikazuje poprečne presjeke

---

## Zašto parabolični fit za outlier detekciju na krivinama

Na pravcu, susjedi pivot tačke su približno na istoj visini. Tačka koja
odskače 5 cm od susjeda je vjerovatno greška.

Na krivini, Z se sistematski mijenja — niveleta ide gore ili dolje.
Susjedi nisu ravni. Apsolutno odstupanje od susjeda bi legitimnu
geometriju krivine proglasilo outlierom.

Rješenje: fituj parabolu na ±10 tačaka (lokalni trend), pa mjeri
odstupanje od tog trenda. Parabola aproksimira vertikalnu krivinu.
Tačka koja prati trend ali je 6 cm različita od susjeda — normalna.
Tačka koja odskače 8 cm od paraboličnog trenda — sumnjiva.

Prag na krivinama je povišen (8 cm vs 5 cm na pravcima) jer reziduali
od paraboličnog fita na složenijoj vertikalnoj geometriji mogu biti
veći i bez greške u mjerenju.

---

## Zašto smjer nagiba nije ulazni podatak

Signed nagib se izračunava direktno iz geometrije:
`(Z_spoljna - Z_pivot) / sirina`. Ručno unesena kolona `smjer_nagiba`
u alignment_info.csv bi stvarala rizik nekonzistentnosti — ako se
podatak u CSV-u ne poklapa sa stvarnim nagibom, skripta mora odlučiti
kome vjeruje.

Kolona `tip` (PRAVAC/KRIVINA/PRIJELAZNICA) je legitimna jer govori
skripti kako da obradi dionicu. Smjer nagiba je izvedena veličina.

---

## Transition zone umjesto oštrih granica dionica

Korisnik unosi granice dionica sa preciznošću ±50 m (snimak izvedenog
stanja, ne originalni projekat). Oštar prelaz između tretmana
(rolling average → linearna interpolacija) na pogrešnoj stacionaži
stvara diskontinuitet u nagibima.

Transition zone (30–50 m) oko svake granice blendira oba tretmana.
Ako je granica pogrešna za 30 m, transition zone to apsorbuje.
Rezultat je robustan na neprecizne ulazne stacionaže.

---

## Zašto smoothirana širina umjesto per-presjek mjerenja

Na pravcu i krivini, normala na pivot pogađa spoljnu poliliniju
konzistentno. Na prijelaznici sa promjenom zakrivljenosti, mala greška
u smjeru normale (pola stepena) daje grešku u širini od par centimetara
— a visinski koridor je samo 4 cm.

Smoothirana širina (rolling average ili spline na seriji izmjerenih
širina) eliminuje pojedinačne numeričke greške. Sanity check (> 20 cm
odstupanje od smooth) hvata mjesta gdje normala značajno promaši.

---

## Automatska detekcija razmaka presjeka

Hardkodiran prozor (npr. ±2.5 m) pretpostavlja razmak od 5 m.
Različiti geodetski snimci imaju različite razmake (5 m, 10 m, ili
neregularan). Automatska kalibracija iz podataka:

1. Projektuješ sve tačke na pivot
2. Sortiraš po stacionaži
3. Detektuješ klastere (histogram ili gap analysis)
4. Prosječan razmak → prozor = ± pola razmaka

Fallback: ako presjek nema dovoljno tačaka, proširi prozor na ±1.5×.
Ako i dalje nema — loguj za manualnu provjeru.

---

## Boundary handling kod rolling average-a

Rolling average za smoothing nagiba na PRAVAC/KRIVINA dionicama se
računa samo unutar granica dionice. Na rubovima prozor se skraćuje
(truncation) umjesto da prelazi u susjednu dionicu.

Razlog: susjedna dionica može biti PRIJELAZNICA sa aktivnom promjenom
nagiba. Rolling average koji ulazi u prijelazničnu zonu "uvlači"
promjenu nagiba u pravac — geometrija "krvari" iz jedne dionice u drugu.

---

## QA post-hoc umjesto dodatnih QP constrainta

Neke provjere (vertikalni radijusi, Δnagib na prijelaznicama) su
informativne ali ne trebaju biti QP constrainti:

- **Vertikalni radijusi:** Ne popravljamo vertikalnu geometriju autoputa
  sa 4 cm overlay-a. Radijusi postojećeg puta su kakvi jesu. Ali korisno
  je znati ako su ispod minimuma za projektnu brzinu — to je informacija
  za inženjera, ne zadatak za skriptu.

- **Δnagib na prijelaznicama:** Kratka prijelaznica sa velikim Δnagibom
  je fizička realnost. Upozorenje da, automatska korekcija ne.

- **Prodor u trbuh asfalta:** Linearna interpolacija između ivica može
  proći ispod stare Z na sredini trake. QA check detektuje ovo.
  Korekcija zahtijeva poznavanje profila stare površine između ivica
  — to je za buduću verziju.

---

## Tehnički stack

```
numpy          — numerika, vektori, matrice
pandas         — I/O, CSV handling, QA reporti
scipy.interpolate — interp1d za reparametrizaciju i 1D interpolaciju
scipy.ndimage  — uniform_filter1d za rolling average
cvxpy          — QP formulacija sa OSQP backendom (Skripta 0)
```

Nema KDTree (nepotrebno za ~7200 tačaka), nema griddata (2D artefakti),
nema UnivariateSpline (ne podržava hard constraints).
