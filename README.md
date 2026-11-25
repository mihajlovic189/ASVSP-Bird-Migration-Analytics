# Analiza migracija ptica (iNaturalist + eBird)

Projekat iz predmeta Arhitektura sistema velikih skupova podataka (ASVSP).

**Student:** Marko Mihajlović, E2 110/2026

**Tema:** Bird Migration Analytics: Historical Bird Observation Patterns (iNaturalist) and Real-Time Sighting Monitoring (eBird)

**Skupovi podataka:**
- paketni (istorijski): iNaturalist Open Data - https://registry.opendata.aws/inaturalist-open-data/
- real-time: eBird API 2.0 - https://documenter.getpostman.com/view/664302/S1ENwy59

## KT1 - Specifikacija projekta

### Domen i motivacija

Tema projekta su osmatranja ptica. Veliki deo podataka o tome gde se i kada pojavljuju ptice danas ne dolazi od naučnika, nego od običnih ljudi koji preko aplikacija kao što su iNaturalist i eBird beleže šta su videli. Svako osmatranje ima vrstu, datum i GPS lokaciju, pa se iz toga mogu dobiti dosta korisne stvari, npr. kada ptice dolaze na proleće, gde ih ima najviše, koje vrste su sve ređe itd.

Zašto mi je ova tema zanimljiva:
- zbog klimatskih promena mnoge vrste dolaze ranije nego pre, a to se može videti iz podataka kroz godine
- retka vrsta koja se pojavi tamo gde je inače nema je bitna informacija za zaštitu prirode, ali i za ljude koji se bave posmatranjem ptica
- podataka ima mnogo (iNaturalist arhiva ima oko 36 GB), a na eBird nova osmatranja stižu stalno, tako da se tema dobro uklapa i u paketnu obradu i u obradu u realnom vremenu

### Persone

**1. Jelena Marković, ornitolog u zavodu za zaštitu prirode**

Već dugo radi na terenskom praćenju ptica i vodi program monitoringa migratornih vrsta, za koji piše godišnje izveštaje. Zanima je kako se menja vreme dolaska i odlaska ptica, koje vrste su u opadanju i da li se neke vrste pomeraju ka severu. Pored istorijske analize, želi da odmah sazna kada se neka retka ili ugrožena vrsta pojavi na mestu ili u periodu kada se to ne očekuje.

**2. Nikola Petrović, organizator birdwatching tura**

Vodi malu agenciju koja organizuje ture za posmatrače ptica i fotografe. Klijenti obično imaju spisak vrsta koje žele da vide, pa mu je bitno da zna gde i kada su šanse za to najveće. Tokom same ture bi mu koristilo da vidi gde su se upravo pojavile zanimljive vrste i koje lokacije su trenutno najaktivnije, da bi po potrebi promenio rutu.

### Ciljevi

- napraviti data lakehouse sa medaljonskom arhitekturom (raw, transformisana i curated zona) za istorijska osmatranja ptica
- paketnom obradom dobiti istorijske obrasce: sezonalnost, trendove, najbolje lokacije, raznovrsnost vrsta, promene u vremenu migracija
- u realnom vremenu pratiti eBird osmatranja i porediti ih sa istorijskim podacima, kako bi se uočila neuobičajena osmatranja
- rezultate prikazati kroz dashboard u nekom BI alatu

### Skupovi podataka

**iNaturalist Open Data (primarni, paketni skup)**

- link: https://registry.opendata.aws/inaturalist-open-data/
- preuzimanje bez naloga: `https://inaturalist-open-data.s3.amazonaws.com/metadata/inaturalist-open-data-latest.tar.gz` (oko 36 GB)
- sadrži nekoliko CSV fajlova (tab separated):
  - `observations.csv` - observation_uuid, observer_id, latitude, longitude, positional_accuracy, taxon_id, quality_grade, observed_on
  - `taxa.csv` - taxon_id, ancestry, rank_level, rank, name, active
  - `observers.csv` - observer_id, login, name
  - `photos.csv` - podaci o fotografijama uz osmatranja
- arhiva sadrži sve žive organizme, a ja ću koristiti samo ptice (klasa Aves) i osmatranja sa ocenom `research`. I nakon tog filtriranja ostaju desetine miliona zapisa.
- u transformisanoj zoni planiram da dodam državu/region na osnovu koordinata, godinu, nedelju i sezonu, kao i red, porodicu i rod vrste (iz polja `ancestry`)

**eBird API 2.0 (tok podataka)**

- dokumentacija: https://documenter.getpostman.com/view/664302/S1ENwy59
- potreban je besplatan API ključ (https://ebird.org/api/keygen)
- podaci će se dovlačiti periodično (na par minuta) i slati u Kafku. Koristiću sledeće endpointe:
  - `/v2/data/obs/{regionCode}/recent` - najnovija osmatranja u regionu
  - `/v2/data/obs/{regionCode}/recent/notable` - retka/značajna osmatranja
  - `/v2/ref/hotspot/{regionCode}` - spisak hotspot lokacija
- jedan zapis izgleda otprilike ovako: speciesCode, comName, sciName, locId, locName, obsDt, howMany, lat, lng, obsValid, obsReviewed, subId

Ova dva skupa dolaze iz različitih izvora (iNaturalist i Cornell Lab of Ornithology) i imaju različitu strukturu (CSV tabele sa pojedinačnim osmatranjima nasuprot JSON-a sa brojem jedinki i hotspot lokacijama). Povezuju se preko naučnog naziva vrste (`sciName` i `taxa.name`) i preko lokacije.

### Pitanja za paketnu obradu

1. Da li ptice na proleće dolaze ranije nego ranijih godina? Za najčešće migratorne vrste gledam prvo osmatranje u godini po regionu i poredim ga sa prethodnim godinama. *(Jelena)*
2. Koje vrste imaju najveći pad ili rast broja osmatranja u poslednjih 10 godina, ako se uzme u obzir da ukupan broj osmatranja raste? *(Jelena)*
3. U kojim nedeljama u godini je najveća šansa da se vidi određena vrsta? *(Nikola)*
4. Koje su top 10 lokacija (ćelije oko 10x10 km) po broju različitih vrsta za svako godišnje doba? *(Nikola)*
5. Koje vrste su retke u nekom regionu (manje od 1% osmatranja), ali se ipak pojavljuju svake godine? *(Nikola)*
6. Da li se areal nekih vrsta pomera ka severu, tj. kako se kroz godine menja prosečna geografska širina njihovih osmatranja? *(Jelena)*
7. Koje porodice ptica imaju najveći udeo u osmatranjima po regionu i kako se to menjalo kroz godine? *(Jelena)*
8. Ko su najaktivniji posmatrači po regionu i koliki deo ukupnih osmatranja oni čine? *(Jelena, Nikola)*
9. Koliko traje jesenja migracija za pojedine vrste, tj. period između 10. i 90. percentila datuma osmatranja? *(Jelena, Nikola)*
10. Koje vrste se najčešće viđaju zajedno, na istoj lokaciji i istog dana? *(Nikola)*
11. Kako se kvalitet podataka (udeo `research` osmatranja, preciznost lokacije) razlikuje po regionima i godinama? *(Jelena)*
12. U kojim mesecima se po regionu može videti najviše različitih vrsta, i kako raste ukupan broj viđenih vrsta kroz godinu? *(Nikola)*

### Pitanja za obradu u realnom vremenu

1. Da li je vrsta koja je upravo osmotrena neuobičajena za taj region i to doba godine, u poređenju sa istorijskim podacima iz iNaturalist-a? *(Jelena)*
2. Koliko je različitih vrsta i ukupno jedinki viđeno po regionu u poslednjih 30 minuta (ažurira se na svakih 5 minuta)? *(Nikola)*
3. Koji hotspotovi su trenutno najaktivniji, po broju osmatranja i vrsta u poslednjih sat vremena? *(Nikola)*
4. Da li se ista retka vrsta pojavljuje na više lokacija u kratkom periodu (npr. na 3 ili više lokacija u roku od 2 sata), što može značiti da je u toku talas migracije? *(Jelena)*
5. Koliko se trenutan broj osmatranja neke vrste razlikuje od istorijskog proseka za istu nedelju u godini, tj. da li je migracija ove godine ranija ili jača nego obično? *(Jelena, Nikola)*
6. Koje vrste sa liste klijenata su viđene u blizini trenutne lokacije ture u poslednja 2 sata? *(Nikola)*

## Plan za KT2 i KT3

- KT2: postavljanje okruženja u Docker Compose-u. Za skladište HDFS ili MinIO, za paketnu obradu Spark, za tokove Kafka i Spark Structured Streaming (ili Flink), za orkestraciju Airflow, za rezultate PostgreSQL ili Elasticsearch, a za vizualizaciju Metabase ili Superset.
- KT3: implementacija upita iz prethodnih poglavlja, vizualizacija i automatsko pokretanje obrade.
