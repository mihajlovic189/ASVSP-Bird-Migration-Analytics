# BirdWatch: istorijska analiza migracija ptica i praćenje osmatranja u realnom vremenu

Projekat iz predmeta **Arhitektura sistema velikih skupova podataka (ASVSP)**.

| | |
|---|---|
| **Student** | Marko Mihajlović |
| **Tema** | Bird Migration Analytics: istorijski obrasci osmatranja ptica (iNaturalist) i praćenje osmatranja u realnom vremenu (eBird) |
| **Paketni (primarni) skup podataka** | iNaturalist Open Data: https://registry.opendata.aws/inaturalist-open-data/ |
| **Skup podataka u realnom vremenu** | eBird API 2.0: https://documenter.getpostman.com/view/664302/S1ENwy59 |

---

## KT1: Specifikacija projekta

### 1. Domen

Domen projekta je **biodiverzitet, a posebno ornitologija**: praćenje gde se, kada i u kom broju ptice pojavljuju. Danas većinu ovih podataka skupljaju volonteri, odnosno „građanska nauka” (engl. *citizen science*). Milioni ljudi preko aplikacija kao što su **iNaturalist** i **eBird** beleže osmatranja sa vremenom, GPS lokacijom i vrstom. Tako nastaju ogromni skupovi prostorno-vremenskih podataka koje ornitolozi, zaštitari prirode i turistički sektor koriste za praćenje migracija, ugroženih vrsta i promena u ekosistemima.

### 2. Motivacija

- **Migracije se menjaju.** Zbog klimatskih promena mnoge vrste u proleće stižu ranije, a na jug odlaze kasnije. Na ovakvim podacima se te promene mogu izmeriti.
- **Ugrožene i invazivne vrste.** Rano uočavanje retke vrste van njenog uobičajenog areala, ili širenja invazivne vrste, važno je za zaštitu prirode.
- **Volumen i brzina.** iNaturalist ima stotine miliona osmatranja (arhiva metapodataka ima oko 36 GB), a na eBird stižu nova osmatranja svakog minuta. To prirodno odgovara kombinaciji **paketne obrade** (istorijski obrasci) i **obrade tokova** (šta se dešava sada i da li odstupa od istorijske norme).

### 3. Persone

#### Persona 1: dr Jelena Marković, ornitološkinja u instituciji za zaštitu prirode

- **Pozadina:** Ima 15 godina iskustva u terenskom praćenju ptica i vodi program monitoringa migratornih vrsta. Piše izveštaje za ministarstvo i međunarodne organizacije (npr. BirdLife).
- **Interesovanja:** fenologija migracija (vreme dolaska i odlaska), rasprostranjenost vrsta, ugrožene vrste, uticaj klime.
- **Potrebe u svakodnevnom radu:**
  - želi da kvantifikuje promene u vremenu migracija tokom poslednjih 10 i više godina;
  - želi da zna koje vrste i regioni pokazuju trend opadanja broja osmatranja;
  - želi da bude **odmah obaveštena** kada se ugrožena ili retka vrsta pojavi van uobičajenog areala ili u neuobičajeno vreme.
- **Skupovi podataka od interesa:** istorijska osmatranja ptica (iNaturalist) i osmatranja u realnom vremenu (eBird).

#### Persona 2: Nikola Petrović, vlasnik agencije za birdwatching ture i fotograf prirode

- **Pozadina:** Organizuje višednevne ture za posmatrače ptica i fotografe. Klijenti dolaze sa „listom želja”, odnosno vrstama koje žele da vide.
- **Interesovanja:** najbolje lokacije i periodi za određene vrste, „hotspot” lokacije, retke vrste koje privlače klijente.
- **Potrebe u svakodnevnom radu:**
  - želi da planira ture (gde i kada) na osnovu istorijske verovatnoće da se vrsta vidi;
  - želi da tokom ture u realnom vremenu vidi gde su se upravo pojavile zanimljive vrste, da bi mogao da promeni rutu;
  - želi da zna koji hotspot je trenutno „najživlji”.
- **Skupovi podataka od interesa:** istorijski hotspotovi i sezonalnost (iNaturalist), aktuelna i značajna (*notable*) osmatranja (eBird).

### 4. Ciljevi

1. Izgraditi skladište tipa **data lakehouse** sa medaljonskom arhitekturom (raw → transformisana → curated zona) za istorijska osmatranja ptica iz iNaturalist skupa.
2. Paketnom obradom izvući **istorijske obrasce**: sezonalnost, trendove, hotspotove, raznovrsnost vrsta i promene u vremenu migracija.
3. Obradom toka eBird osmatranja u **realnom vremenu** detektovati neuobičajena osmatranja tako što se tok poredi sa istorijskom normom iz paketnog sloja (spajanje toka sa paketnim podacima) i agregirati aktivnost po vremenskim prozorima.
4. Rezultate prikazati u BI alatu (dashboard za obe persone).

### 5. Skupovi podataka

#### 5.1 Primarni (paketni) skup: iNaturalist Open Data

- **Izvor:** https://registry.opendata.aws/inaturalist-open-data/ (GitHub: https://github.com/inaturalist/inaturalist-open-data)
- **Direktno preuzimanje (S3, bez naloga):** `https://inaturalist-open-data.s3.amazonaws.com/metadata/inaturalist-open-data-latest.tar.gz` (~36 GB)
- **Format:** tab-separated CSV fajlovi:
  - `observations.csv`: `observation_uuid, observer_id, latitude, longitude, positional_accuracy, taxon_id, quality_grade, observed_on`
  - `taxa.csv`: `taxon_id, ancestry, rank_level, rank, name, active` (taksonomsko stablo; ptice = klasa *Aves*, taxon_id 3)
  - `observers.csv`: `observer_id, login, name`
  - `photos.csv`: `photo_uuid, photo_id, observation_uuid, observer_id, extension, license, width, height, position`
- **Obim:** stotine miliona osmatranja svih taksona; nakon filtriranja na ptice (*Aves*) i ocenu kvaliteta `research` ostaju desetine miliona zapisa, što je daleko više od zahtevanih 300 MB.
- **Planirano obogaćivanje u transformisanoj zoni:** mapiranje koordinata na državu ili region (npr. Natural Earth granice), izvođenje godine, nedelje u godini i sezone, kao i taksonomske hijerarhije (red, porodica, rod, vrsta) iz polja `ancestry`.

#### 5.2 Tok podataka: eBird API 2.0 (Cornell Lab of Ornithology)

- **Dokumentacija:** https://documenter.getpostman.com/view/664302/S1ENwy59
- **Pristup:** REST API sa besplatnim ključem (https://ebird.org/api/keygen) koji se šalje u zaglavlju `X-eBirdApiToken`.
- **Krajnje tačke koje se koriste (periodično dovlačenje, npr. na 1–5 minuta, pa slanje u Kafka topic):**
  - `GET /v2/data/obs/{regionCode}/recent`: najnovija osmatranja u regionu;
  - `GET /v2/data/obs/{regionCode}/recent/notable`: značajna i retka osmatranja (eBird ih sam označava);
  - `GET /v2/ref/hotspot/{regionCode}`: šifarnik hotspot lokacija.
- **Struktura (JSON):** `speciesCode, comName, sciName, locId, locName, obsDt, howMany, lat, lng, obsValid, obsReviewed, locationPrivate, subId`
- **Zašto se razlikuje od primarnog skupa:** drugi izvor (Cornell Lab i eBird, nasuprot iNaturalist-u), drugačija struktura (JSON zapisi vezani za kontrolne liste (*checklist*), sa brojem jedinki i hotspot lokacijama, nasuprot CSV osmatranjima pojedinačnih jedinki sa fotografijom) i druga zajednica posmatrača. Logička veza su ista domena i iste vrste: spajanje ide preko naučnog imena vrste (`sciName` ↔ `taxa.name`) i preko lokacije i regiona.

### 6. Pitanja za paketnu obradu (iNaturalist, istorijski podaci)

| # | Pitanje | Persona |
|---|---|---|
| 1 | Kako se tokom godina menja **prosečna nedelja prvog prolećnog osmatranja** za 20 najčešćih migratornih vrsta po regionu, i da li postoji trend ranijeg dolaska? (window: `MIN` / `FIRST_VALUE` po vrsti, regionu i godini, `LAG` za razliku u odnosu na prethodnu godinu) | Jelena |
| 2 | Koje vrste imaju najveći **pad ili rast broja osmatranja** (normalizovanog ukupnim brojem osmatranja ptica u regionu) u poslednjih 10 godina? (linearni trend po vrsti, rangiranje sa `RANK`) | Jelena |
| 3 | Kako izgleda **sezonski profil** svake vrste, odnosno u kojim nedeljama u godini je verovatnoća osmatranja najveća? (udeo vrste u ukupnim osmatranjima po nedelji, klizni prosek preko 3 nedelje) | Nikola |
| 4 | Koje su **top 10 lokacija (ćelije mreže ~10×10 km) po raznovrsnosti vrsta** za svako godišnje doba? (`COUNT DISTINCT` vrsta, `DENSE_RANK` po sezoni) | Nikola |
| 5 | Koje vrste su **retke** u određenom regionu (manje od 1 % osmatranja), a ipak se tamo javljaju svake godine, kao kandidati za „target” vrste na turama? | Nikola |
| 6 | Za koliko se godišnje **pomera težište areala** (srednja geografska širina osmatranja) kod pojedinih vrsta, odnosno da li se vrste pomeraju ka severu? (`AVG(latitude)` po vrsti i godini, `LAG`) | Jelena |
| 7 | Koje **porodice ptica** čine najveći udeo osmatranja po regionu i kako se taj udeo menjao kroz godine? (udeo preko `SUM() OVER (PARTITION BY region, year)`) | Jelena |
| 8 | Ko su **najaktivniji posmatrači** po regionu, koliki je njihov doprinos ukupnom broju osmatranja i koliko vrsta su zabeležili? (kumulativni udeo, Pareto 80/20) | Jelena / Nikola |
| 9 | U kom periodu **jesenje migracije** (pomeranje osmatranja na jug) se beleži najveći broj vrsta po regionu, i koliko traje „prozor” migracije (od 10. do 90. percentila datuma osmatranja)? (`PERCENTILE_APPROX` po vrsti i godini) | Jelena / Nikola |
| 10 | Koje kombinacije vrsta se najčešće **pojavljuju zajedno** na istoj lokaciji istog dana (ko-okurencija), kao preporuka „ako vidiš X, verovatno ćeš videti i Y”? | Nikola |
| 11 | Kako se **kvalitet podataka** (udeo osmatranja ocenjenih kao `research`, preciznost GPS koordinata) menja po regionima i godinama? | Jelena |
| 12 | Koji meseci u godini daju **najveći broj različitih vrsta** po regionu (kumulativna lista vrsta po mesecima sa `SUM() OVER (ORDER BY month)`)? | Nikola |

### 7. Pitanja za obradu u realnom vremenu (eBird tok + istorijska norma)

| # | Pitanje | Persona |
|---|---|---|
| 1 | Da li je upravo osmotrena vrsta **neuobičajena za ovaj region i ovu nedelju u godini** u odnosu na istorijsku normu iz iNaturalist-a? (spajanje toka sa paketnim podacima; alarm ako istorijska verovatnoća padne ispod praga) | Jelena |
| 2 | Koliko je **različitih vrsta i ukupno jedinki** (`howMany`) osmotreno po regionu u poslednjih 30 minuta, sa ažuriranjem na svakih 5 minuta? (sliding window agregacija) | Nikola |
| 3 | Koji su **trenutno najaktivniji hotspotovi** po broju osmatranja i broju vrsta u poslednjem satu (tumbling window od 1 h), obogaćeni nazivom i koordinatama iz šifarnika hotspotova? | Nikola |
| 4 | Da li se **značajna (notable) osmatranja** iste vrste ponavljaju na više lokacija u kratkom periodu (npr. 3 i više lokacija u 2 h), što ukazuje na talas migracije ili invaziju vrste? (spajanje toka `notable` sa tokom `recent`, session ili sliding window) | Jelena |
| 5 | Kako se **trenutni broj osmatranja** neke vrste u regionu odnosi prema istorijskom proseku za istu nedelju u godini (indeks aktivnosti = tekuće / istorijsko), tj. da li je migracija „ranija” ili „jača” nego obično? | Jelena / Nikola |
| 6 | Koje vrste sa **liste želja** klijenata su osmotrene u krugu od X km od trenutne lokacije ture u poslednja 2 sata? | Nikola |

---

## Plan za naredne kontrolne tačke (okvirno)

- **KT2:** Docker Compose okruženje: HDFS ili MinIO (raw, transformisana i curated zona), Apache Spark (paketna obrada), Apache Kafka i Spark Structured Streaming ili Flink (tokovi), Airflow (orkestracija), skladište za rezultate (npr. PostgreSQL/Citus ili Elasticsearch) i Metabase ili Superset za vizualizaciju.
- **KT3:** implementacija upita iz poglavlja 6 i 7, vizualizacija i orkestracija.
