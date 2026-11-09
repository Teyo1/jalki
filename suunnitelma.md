# Harjoitustyön suunnitelma: Jälki

**Tekijä:** [Teijo Raiskio]
**Ohjelman nimi:** Jälki
**Harjoitustyön osoite gitissä:** https://github.com/Teyo1/jalki
**Työn tyyppi:** C#-komentoriviohjelma (ei peli)
**Käyttäjien lukumäärä:** 1

---

## Taustatarina ja kuvaus teemasta

Tietoturvatyössä kysytään usein: "Tapahtuiko koneella jotain heti sen jälkeen, kun joku pääsi sisään?"
Jälki on pieni tutkintatyökalu, joka auttaa vastaamaan tähän kysymykseen. Se yhdistää kaksi tietolähdettä:

1. SSH-kirjautumisloki (kuka yritti kirjautua ja mistä IP-osoitteesta)
2. Tiedostojen eheys (mitkä tiedostot ovat muuttuneet, lisääntyneet tai kadonneet)

Kun molemmat on tutkittu, ohjelma etsii yhteyden: jos IP-osoite yritti ensin monta kertaa väärää salasanaa ja pääsi sitten sisään, ohjelma näyttää, mitkä tiedostot muuttuivat pian sen jälkeen.

Ohjelma on tarkoitettu ensimmäiseksi tutkintavaiheeksi (triage). Se antaa vihjeitä, ei todista mitään.

## Idea ja tavoitteet

- Käyttäjä lataa lokitiedoston ja valitsee tarkkailtavan kansion.
- Ohjelma tunnistaa **raa'an voiman hyökkäyksen** (monta epäonnistunutta kirjautumista samasta IP-osoitteesta lyhyessä ajassa).
- Ohjelma tunnistaa, jos hyökkääjä lopulta **onnistui** kirjautumaan.
- Ohjelma vertaa tiedostojen tiivisteitä (SHA-256) tallennettuun perustilaan ja listaa muutokset.
- Ohjelma **korreloi** löydökset: näyttää tiedostomuutokset, jotka tapahtuivat onnistuneen kirjautumisen jälkeen tietyssä aikaikkunassa.

**Tavoite:** käyttäjä näkee yhdestä raportista, mitkä IP-osoitteet ovat epäilyttäviä ja mitä koneella tapahtui niiden jälkeen.

## Hahmotelma ohjelmasta

Valikko:

```
=== JÄLKI ===
1. Lataa kirjautumisloki
2. Näytä epäilyttävät IP-osoitteet
3. Tallenna kansion perustila
4. Tarkista kansion muutokset
5. Näytä korrelaatio (kirjautuminen -> muutokset)
6. Muuta asetuksia (kynnys, aikaikkuna)
0. Lopeta
Valinta:
```

Esimerkkitulostus korrelaatiosta:

```
Epäilyttävä: 203.0.113.5
  Epäonnistuneita kirjautumisia: 12
  Onnistunut kirjautuminen: 10:42 (käyttäjä: admin)
  Muuttuneita tiedostoja 15 minuutin sisällä:
    - config/sshd_config  (muokattu 10:44)
    - bin/backup.sh       (muokattu 10:47)
    - uusi: tmp/x.sh      (lisätty 10:49)
```

(Hahmotelmakuvat lisätään suunnitelma-kansioon: valikko ja raportti.)

## Miten ohjelma etenee

1. Ohjelma käynnistyy ja näyttää valikon.
2. Käyttäjä valitsee toiminnon ja antaa tarvittavat tiedot (tiedostopolku, kansio).
3. Ohjelma käsittelee tiedot ja tulostaa tuloksen.
4. Käyttäjä palaa valikkoon ja jatkaa toisella toiminnolla tai lopettaa.

Ohjelma ei kaadu virheellisestä syötteestä (väärä tiedostopolku, tyhjä loki, väärin muotoiltu rivi). Se antaa selkeän virheilmoituksen ja palaa valikkoon.

## Ohjelman oliot ja niiden suhteet

| Luokka | Tehtävä |
|---|---|
| `Ohjelma` | Valikko ja käyttäjän syötteet. Kutsuu muita luokkia. |
| `LokiMerkinta` | Yksi lokirivi: aika, käyttäjä, IP, onnistuiko. |
| `LokiAnalysaattori` | Lukee lokin, jäsentää rivit, tunnistaa raa'an voiman ja onnistuneet kirjautumiset. |
| `TiedostoMuutos` | Yksi löydetty muutos: polku, tyyppi (muuttunut/lisätty/poistettu), aika. |
| `EheysTarkastaja` | Laskee tiedostojen tiivisteet, tallentaa perustilan ja vertaa nykytilaa siihen. |
| `Korrelaattori` | Yhdistää epäilyttävät kirjautumiset ja tiedostomuutokset aikaikkunan avulla. |

Suhteet: `Ohjelma` käyttää `LokiAnalysaattoria`, `EheysTarkastajaa` ja `Korrelaattoria`. `LokiAnalysaattori` tuottaa `LokiMerkinta`-listan, `EheysTarkastaja` tuottaa `TiedostoMuutos`-listan, ja `Korrelaattori` ottaa vastaan molemmat listat.

## Vaatimusten täyttyminen

- **Aliohjelmat:** useita, esim. `JasennaRivi`, `LueLoki`, `LaskeHash`, `VertaaPerustilaan`, `EtsiKorreloivat`, `TulostaValikko`.
- **Lista:** lokimerkinnät, tiedostomuutokset ja löydökset tallennetaan listoihin, joita käytetään raportissa.
- **Silmukka:** käydään läpi lokirivit, tiedostot ja löydökset.
- **Funktio:** esim. `string LaskeHash(string polku)` palauttaa tiivisteen, `List<TiedostoMuutos> EtsiKorreloivat(...)` palauttaa löydökset.
- **Vakiot (`const`):** epäonnistuneiden kirjautumisten kynnys, aikaikkuna minuutteina, lokirivin tunnisteteksti.
- **Ei `public static` -muuttujia.** Vain `Main` on static.
- **Testit:** yksikkötestit aliohjelmille (ks. alla).
- **Dokumentaatio:** luokat, aliohjelmat ja attribuutit dokumentoidaan `summary`-tageilla. Luokan alussa `@author` ja `@version`.
- **Tyyli:** kurssin tyyliopas, tasan kaksi tyhjää riviä aliohjelman `}` jälkeen, ei turhia literaaleja.

## Testaus

Koska työ ei ole peli, testaan aliohjelmia yksikkötesteillä:

- `JasennaRivi` palauttaa oikean IP:n, ajan ja tuloksen tunnetusta rivistä, ja käsittelee virheellisen rivin hallitusti.
- Raa'an voiman tunnistus: 5 epäonnistumista kynnyksen sisällä tunnistetaan, 4 ei.
- `LaskeHash` antaa saman tiivisteen samalle sisällölle ja eri tiivisteen muuttuneelle.
- `EtsiKorreloivat` palauttaa vain aikaikkunassa tapahtuneet muutokset kiinteillä aikaleimoilla.

## Esimerkkiaineisto

Repossa on **keinotekoinen** esimerkkiloki ja esimerkkikansio, joten ohjelma toimii millä tahansa koneella. Aineisto ei sisällä oikeita tunnuksia, oikeita IP-osoitteita tai oikeita järjestelmätiedostoja.

## Rajaukset

- Aikaleimojen vertailu on yksinkertaista. Korrelaatio on vihje tutkintaan, ei todiste hyökkäyksestä.
- Ohjelma ei skannaa verkkoa, ei muuta järjestelmää eikä lähetä dataa minnekään.
- Ohjelma lukee vain annettua lokia ja annettua kansiota.

## Toteutuksen suunnitelma

Työaika yhteensä noin 27 tuntia.

**Vaihe 1: Suunnitelma (noin 2 h)**
- Luodaan GitHub-etävarasto, asetetaan Git-tiedot, haetaan varasto koneelle.
- Esitys jos pakko, mutta asian voi lukea kooditasolla ja ymmärtää asian suunnitelmasta.

**Vaihe 2: Työ 50 % valmis (noin 12 h)**
1. Luo solution ja projektirakenne, lisää testiprojekti (noin 1 h).
2. `LokiMerkinta` ja `JasennaRivi` testeineen (noin 3 h).
3. `LokiAnalysaattori`: lokin luku, raa'an voiman tunnistus ja onnistuneet kirjautumiset (noin 4 h).
4. Valikko ja tulostukset (noin 2 h).
5. Esimerkkiloki ja päivitys etävarastoon (noin 2 h).

**Vaihe 3: Työ 100 % valmis (noin 13 h)**
1. `EheysTarkastaja`: tiivisteet, perustila, vertailu testeineen (noin 5 h).
2. `Korrelaattori` testeineen (noin 3 h).
3. Virheenkäsittely ja asetukset (noin 2 h).
4. Dokumentaatio, tyylin tarkistus ja Riderin varoitusten korjaus (noin 2 h).
5. Lopputarkistus: lataa etävarasto uuteen kansioon ja varmista, että ohjelma toimii (noin 1 h).

**Jos aika loppuu kesken:** loki-osa ja eheys-osa toimivat myös erikseen, joten vain korrelaatio jätettäisiin pois.
