# REFIT — ohje sovelluksesta ja sen muokkaamisesta

Tämä tiedosto selittää mikä REFIT on, miten se toimii teknisesti, ja miten treenejä
tai niiden osia muutetaan/lisätään. Tarkoitettu luettavaksi niin ihmiselle kuin
Claudelle (tai muulle avustavalle mallille) ennen muutosten tekemistä.

## 1. Mikä tämä on

REFIT on henkilökohtainen, itse hostattava saliohjelman seurantasovellus.
Ei backendia, ei build-prosessia, ei riippuvuuksia — kaksi itsenäistä
HTML-tiedostoa, jotka voi avata suoraan selaimessa tai hostata missä tahansa
staattisen sivuston palvelussa (Netlify Drop, GitHub Pages, jne.).

Kaksi versiota, molemmat samassa kansiossa:

| Tiedosto | Nimi käyttöliittymässä | Ominaisuudet |
|---|---|---|
| `index.html` | **REFIT+TRACK** | Kaikki perusversion ominaisuudet + painojen (kg) merkitseminen painoharjoitteissa, painot tallentuvat treenihistoriaan |
| `index-0.5.html` | **REFIT** | Perusversio ilman painoseurantaa — alkuperäinen, säilytetty rinnalla |

Molemmista pääsee toiseen "Vaihda versioon" -linkillä sivun alalaidasta
(treenihistorian jälkeen). Tiedostot ovat rakenteeltaan hyvin samanlaisia,
mutta **eivät jaa koodia** — ne on synkronoitava käsin kun tehdään
molempiin päteviä muutoksia (ks. kohta 6).

## 2. Käyttäjän näkökulma: mitä sovellus tekee

1. Käyttäjä valitsee kumman salitreenin tekee tänään: **Treeni A** tai **Treeni B**.
2. Treeni jakautuu vaiheisiin (aina samat, riippumatta A/B-valinnasta):
   Alkuvenyttely → Alkuverryttely → Painoharjoitteet (A tai B) → Vatsalihakset → Loppuverryttely.
3. Käyttäjä rastittaa liikkeet tehdyiksi. Yläreunan palkki näyttää kokonaisedistymän
   ja tekstinä missä vaiheessa mennään.
4. Treeni A:n supersarjoissa on lisäksi kierrosmerkintä (1/2/3) koko liikeparin
   yhteydessä, koska ohjelma tehdään 1–3 kierroksena kiertoharjoitteluna.
5. REFIT+TRACK:ssa käyttäjä voi kirjata käytetyn painon (kg) jokaiseen
   painoharjoitteeseen. Paino jää näkyviin seuraavalle kerralle (ei nollaudu
   automaattisesti päivän vaihtuessa), jotta progressiivista kuormaa on helppo
   seurata. "Tyhjennä"-nappi nollaa halutessa myös painot (varmistetaan
   erikseen dialogilla).
6. "Merkitse treeni tehdyksi" tallentaa treenin (+ REFIT+TRACK:ssa käytetyt painot)
   historialistaan ja nollaa päivän rastit.
7. Treenihistoria näkyy sivun alaosassa, siitä voi poistaa yksittäisiä merkintöjä,
   ja koko data voidaan viedä/tuoda JSON-tiedostona (varmuuskopio).

## 3. Tekninen rakenne

Yhden tiedoston sovellus: `<style>` + HTML-runko + `<script>` samassa
`.html`-tiedostossa. Ei kääntämistä, ei paketteja — muokkaus tapahtuu suoraan
tiedostoon ja tallennus näkyy heti selaimen päivityksellä.

### 3.1 Datamalli — `PROGRAM`-olio

Kaikki treenisisältö on yhdessä JS-oliossa `PROGRAM` skriptin alussa.
Rakenne:

```js
const PROGRAM = {
  <blockKey>: {
    title: "Otsikko",
    note: "Valinnainen huomio (näkyy vaiheen alla, esim. kiertomäärä)",
    items: [
      [nimi, toistoteksti, ryhmäId, kuvaus],
      ...
    ]
  },
  ...
};
```

`blockKey`-arvot: `common1` (alkuvenyttely), `common2` (alkuverryttely),
`A` ja `B` (painoharjoitteet, valittavissa käyttäjän toimesta), `abs`
(vatsalihakset), `cooldown` (loppuverryttely).

Yksittäinen liike on **taulukko, ei olio** — järjestys on tarkka:

1. `nimi` (string) — liikkeen nimi, näytetään lihavoituna.
2. `toistoteksti` (string) — esim. `"8–10 ×2"` tai `"×20"`. Tyhjä string `""`
   piilottaa toistomerkinnän kokonaan (CSS: `.reps:empty{display:none}`).
3. `ryhmäId` (number tai `undefined`) — supersarjan tunniste. Peräkkäiset
   liikkeet samalla ryhmäId:llä ryhmittyvät "Supersarja N" -otsikon alle.
   **Tärkeää:** käytä nimenomaan `undefined` (ei `null`!) kun liike ei kuulu
   mihinkään ryhmään — koodi tarkistaa `group !== undefined`, ja `null` laukaisisi
   virheellisesti uuden ryhmäotsikon.
4. `kuvaus` (string) — pieni harmaa suoritusohje liikkeen nimen alla. Pidä
   1 virke, ytimekäs, oikea liiketekniikka (ei yleisluontoista täytettä).

Esimerkki (Treeni A, kaksi liikettä samassa supersarjassa):

```js
["Penkkipunnerrus (kp tai levytanko)", "8–10 ×2", 2, "Punnerra paino penkillä maaten rinnalta ylös."],
["Vipunosto maaten", "8–10 ×2", 2, "Selin makuulla nosta käsipainot suorin käsin sivulta ylös."],
```

### 3.2 Vaihejärjestys ja treenin kokoonpano

```js
function blockOrder(){ return ["common1","common2", state.plan, "abs","cooldown"]; }
```

`state.plan` on `"A"` tai `"B"` — vain painoharjoiteosuus vaihtuu, muut vaiheet
ovat kaikille yhteisiä. `PHASE_LABELS`-taulukko määrittää tekstin, joka näkyy
"Nyt: ..." -ilmaisimessa, ja **sen pitää pysyä samassa järjestyksessä** kuin
`blockOrder()` palauttaa.

`PHASE_META`-olio määrittää Material Symbols -ikonin per `blockKey`
(esim. `fitness_center`, `local_fire_department`). Nimet ovat Google
Material Symbols -kirjaston ikoninimiä (ladataan Google Fontsista
`<link>`-tagilla `<head>`:ssä).

### 3.3 Tila ja tallennus (localStorage)

Ei backendiä — kaikki data on selaimen `localStorage`:ssa, laitekohtaista.

| Avain | Sisältö | Nollautuu |
|---|---|---|
| `refit(-track)-state` | `{plan, checks, rounds, weights?, date}` — päivän tilanne | `checks`+`rounds` nollautuu automaattisesti kun `date` ≠ tämä päivä. `weights` (vain REFIT+TRACK) **ei nollaudu automaattisesti** päivän vaihtuessa — se on tarkoituksella pysyvä referenssi viimeksi käytetylle painolle. Manuaalinen "Tyhjennä"-nappi nollaa REFIT+TRACK:ssa `checks`+`rounds`+`weights` kaikki kolme (käyttäjä vahvistaa tämän erikseen, koska se pyyhkii myös painot). |
| `refit(-track)-history` | Taulukko valmiiksi merkityistä treeneistä | Ei nollaudu itsestään; poistetaan yksittäin roskakori-ikonista. |

REFIT käyttää avainten etuliitettä `refit-`, REFIT+TRACK `refittrack-` —
tarkoituksella eri, jotta kahden version data ei mene sekaisin samalla
origin/pathilla.

`checks`-avainten muoto: `"<blockKey>:<itemIndex>"`, esim. `"A:3"`.

`rounds`-avainten muoto **muuttuu ryhmän mukaan** (REFIT+TRACK:ssa):
supersarjan kierrospallot on kiinnitetty koko ryhmään, ei yksittäiseen
liikkeeseen, avain on `"<blockKey>:g<ryhmäId>"` esim. `"A:g2"`.

`weights`-avainten muoto (vain REFIT+TRACK): sama kuin `checks`,
`"<blockKey>:<itemIndex>"` → merkkijono (kg).

### 3.4 Renderöintilogiikka

`render()`-funktio piirtää koko `#blocks`-listan uudestaan aina kun tila
muuttuu (yksinkertainen re-render, ei virtuaali-DOM:ia). Avoinna olevat
osiot muistetaan `dataset.key`:n avulla ennen uudelleenpiirtoa, jotta
käyttäjän auki klikkaama osio ei sulkeudu.

Supersarjaotsikko (`groupHead`) generoidaan kun `items`-taulukossa
`ryhmäId` muuttuu edelliseen liikkeeseen verrattuna — siksi samaan ryhmään
kuuluvat liikkeet **on pakko** olla peräkkäin taulukossa.

### 3.5 Vienti/tuonti (JSON-varmuuskopio)

`exportData()` kirjoittaa `{app, exported, history, state}`-olion tiedostoksi.
`app`-kentän arvo (`"refit"` tai `"refit-track"`) estää väärän version
tiedoston tuonnin toiseen sovellukseen. `importData()` yhdistää historiat
(pudottaa täydelliset duplikaatit) ja palauttaa päivän tilan **vain** jos
varmuuskopio on tämän päivän päivämäärältä eikä tämän päivän merkintöjä ole
vielä tehty — ei ylikirjoita kesken olevaa treeniä.

## 4. Prosessi: liikkeen sisällön muuttaminen

1. Etsi oikea `blockKey` ja rivi `PROGRAM`-oliosta (`grep -n "<liikkeen nimi>"`).
2. Muuta haluttu kenttä taulukosta (nimi/toistot/ryhmä/kuvaus).
3. **Tee sama muutos molempiin tiedostoihin**, jos muutos ei liity
   nimenomaan painoseurantaan (ks. kohta 6).
4. Tarkista terminaalissa sulkumerkkien tasapaino ennen selaimeen viemistä:
   ```bash
   python3 -c "
   c=open('index.html').read()
   s=c.split('<script>')[1].split('</script>')[0]
   print(s.count('{')-s.count('}'), s.count('(')-s.count(')'))
   "
   ```
   Molempien lukujen pitää olla `0`.
5. Avaa selaimessa ja tarkista visuaalisesti (rastita, avaa/sulje osio,
   tarkista supersarjaotsikko jos ryhmää muutettiin).

## 5. Prosessi: uuden liikkeen/vaiheen lisääminen

**Uusi liike olemassa olevaan vaiheeseen:**
Lisää uusi rivi oikean `blockKey`:n `items`-taulukkoon oikeaan kohtaan
(järjestys = suoritusjärjestys). Jos liike kuuluu supersarjaan, anna sille
sama `ryhmäId` kuin parinsa, ja varmista että se on taulukossa suoraan
parinsa vierellä.

**Uusi supersarja Treeni A/B:hen:**
Kasvata `ryhmäId` seuraavaan vapaana olevaan numeroon (esim. jos suurin
käytössä on `5`, uusi ryhmä on `6`). Lisää molemmat parin liikkeet
peräkkäin samalla `ryhmäId`:llä.

**Kokonaan uusi vaihe (esim. "Jäähdyttely 2.0"):**
1. Lisää uusi avain `PROGRAM`-olioon (`title`, valinnainen `note`, `items`).
2. Lisää vastaava teksti `PHASE_LABELS`-taulukkoon **samaan kohtaan** kuin
   se ilmestyy `blockOrder()`:ssa.
3. Lisää `PHASE_META`-olioon `{icon: "<material-symbol-nimi>"}` — käytä
   olemassa olevaa [Google Material Symbols](https://fonts.google.com/icons)
   -ikoninimeä (esim. `spa`, `directions_run`).
4. Lisää `blockKey` `blockOrder()`-funktion palauttamaan taulukkoon oikeaan
   kohtaan.
5. Jos vaihe on vain toisessa versiossa (esim. painoseurantaan liittyvä),
   toista **vain** siihen tiedostoon.

**Painoseuranta uuteen painoharjoiteosaan (vain REFIT+TRACK):**
Lisää uusi `blockKey` `WEIGHT_BLOCKS`-taulukkoon (`index.html`:n alussa,
`const WEIGHT_BLOCKS = ["A","B"];`). Tämä riittää — kg-kenttä ilmestyy
automaattisesti kaikkiin sen osion liikkeisiin.

## 6. Kahden version synkronointi

`index.html` (REFIT+TRACK) ja `index-0.5.html` (REFIT) jakavat suurimman
osan CSS:stä, `PROGRAM`-datasta ja renderöintilogiikasta, mutta ovat
**erilliset tiedostot** — ei importteja, ei jaettua moduulia.

| Muutostyyppi | Molempiin | Vain `index.html` |
|---|---|---|
| Liikkeen nimi/toistot/kuvaus/ryhmä | ✅ | |
| Uusi vaihe/liike, joka ei koske painoja | ✅ | |
| Visuaaliset/tyyliparannukset (värit, fontit, kontrasti) | ✅ | |
| Painojen syöttökenttä, painojen tallennus historiaan | | ✅ |
| Kierrospallojen (1-2-3) sijainti/logiikka | ✅ (jaettu ominaisuus, molemmissa samanlainen) | |

Kun teet muutoksen jonka pitää näkyä molemmissa: muokkaa ensin yhtä
tiedostoa, varmista se toimii, ja **toista sama muutos** toiseen (ei
kopioida koko tiedostoa päälle — se hävittäisi version-spesifiset erot).

## 7. Tunnetut sudenkuopat

- `ryhmäId`: käytä `undefined`, ei `null`. Katso kohta 3.1.
- Rounds-avaimen muoto muuttui liikeriveiltä (`bk:i`) ryhmätasolle
  (`bk:g<ryhmäId>`) kun kierrosmerkintä siirrettiin liikeriviltä
  supersarjaotsikkoon. Jos lisäät uuden ryhmätason ominaisuuden, käytä
  samaa `bk:g<id>`-muotoa, ei liikekohtaista `key`-muuttujaa.
- `localStorage`-avainten etuliitteet (`refit-` vs `refittrack-`) on
  pidettävä erillisinä — jos niitä sekoittaa, kahden version data menee
  päällekkäin ja vienti/tuonti-validointi (`data.app`) alkaa hylkiä
  tiedostoja virheellisesti.
- Material Symbols -ikonit latautuvat Google Fontsista — sovellus vaatii
  verkkoyhteyden ensimmäisellä latauksella (fontti jää selaimen cacheen).
  Tämä on hyväksytty poikkeus "täysin offline" -periaatteesta, koska
  sivu on tarkoitettu hostattavaksi julkiseen osoitteeseen.
- Ei buildia, ei minifiointia — muutokset näkyvät suoraan kun tiedosto
  tallennetaan ja selain päivitetään. Ei tarvetta `npm install`/`build`-
  komennoille.
