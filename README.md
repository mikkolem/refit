# REFIT

Henkilökohtainen saliohjelman seurantasovellus. Ei backendia, ei
build-vaihetta — kaksi itsenäistä HTML-tiedostoa, jotka voi avata
suoraan selaimessa tai hostata missä tahansa staattisen sivuston
palvelussa.

## Versiot

| Tiedosto | Nimi | Kuvaus |
|---|---|---|
| [`index.html`](index.html) | **REFIT+TRACK** | Saliohjelman seuranta + painojen (kg) merkitseminen (yhteispaino tai per käsi) ja tallennus historiaan |
| [`index-0.5.html`](index-0.5.html) | **REFIT** | Perusversio ilman painoseurantaa |

Molemmista pääsee toiseen sivun alalaidan "Vaihda versioon" -linkistä.

## Ominaisuudet

- Saliohjelma jaettuna vaiheisiin: alkuvenyttely → alkuverryttely → painoharjoitteet (Treeni A / B) → vatsalihakset → loppuverryttely
- Liikkeiden rastitus ja koko treenin edistymän seuranta
- Supersarjojen dynaaminen kierrosmerkintä — "+ Lisää kierros" avaa 2. ja 3. kierroksen tarvittaessa
- Vapaa tekstikenttä liikkeille joilla ei ole kiinteää sisältöä (esim. "Vapaa painoharjoittelu")
- Treenihistoria, josta merkintöjä voi myös poistaa
- Data tallentuu selaimen `localStorage`:iin — ei tiliä, ei palvelinta
- Varmuuskopion vienti/tuonti JSON-tiedostona

## Käyttö

Avaa `index.html` (tai `index-0.5.html`) suoraan selaimessa, tai hosta
kansio staattisena sivustona esim. [Netlify Dropissa](https://app.netlify.com/drop)
tai GitHub Pagesissa (Settings → Pages → Deploy from branch → `main` / `/ (root)`).

## Muokkaaminen

Katso [OHJE.md](OHJE.md) — kuvaa sovelluksen datamallin, tilan
tallennuksen ja vaiheittaisen prosessin treenien/liikkeiden
muuttamiseen ja lisäämiseen.
