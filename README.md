# Elävä Bistro – menu

Kaksi staattista sivua GitHub Pagesilla:

- **`index.html`** – lomake, johon liitetään raaka menuteksti. Lähetys POSTaa tekstin Make.com-webhookiin.
- **`menu.html`** – näyttösivu, joka lukee jäsennellyn menun `data.json`-tiedostosta ja renderöi sen Elävä Bistron ulkoasulla (tausta `#D8E9A2`, otsikot Bricolage Grotesque, tuotenimet Manrope).
- **`data.json`** – jäsennelty menu. Make.com-skenaario kirjoittaa tämän tiedoston yli aina kun uusi menu lähetetään lomakkeelta. Repossa on mukana esimerkkidata, jotta `menu.html` ei ole tyhjä ennen ensimmäistä lähetystä.

## Make.com-skenaarion rakenne

```
[Webhook] → [Gemini] → [JSON Parse] → [GitHub: GET sha] → [GitHub: PUT content]
```

### 1. Webhook (Custom webhook)

- Vastaanottaa `index.html`-lomakkeen POST-pyynnön.
- Body on JSON: `{ "text": "<raakateksti>" }`.
- Luo webhook Make.comissa, kopioi sen URL ja liitä se `index.html`-tiedoston `CONFIG.webhookUrl`-muuttujaan (tiedoston lopussa, selkeästi merkitty `TODO`-kommentilla).

### 2. Gemini-moduuli (Google Gemini / Text or JSON generation)

Ottaa webhookilta saadun `text`-kentän sisään ja jäsentää sen JSON:ksi. Esimerkkiprompti:

```
Olet ravintolan menun jäsentäjä. Saat raakaa, mahdollisesti epäsiistiä ja
kirjoitusvirheitä sisältävää menutekstiä. Tehtäväsi:

1. Jaa sisältö kategorioihin. Käytä vain näitä kategorianimiä, jos teksti
   viittaa niihin: "Kuumat juomat", "Suolaista", "Makeaa", "Kylmät juomat".
   Jos tekstissä on muita kategorioita, käytä niitä sellaisenaan.
2. Tunnista jokaisesta kategoriasta tuotteet ja niiden hinnat.
3. Korjaa ilmeiset kirjoitusvirheet tuotenimistä (esim. "kahvii" → "kahvi").
4. Normalisoi hintamuoto aina muotoon "X,XX€" (esim. "2e" → "2,00€",
   "3.5 eur" → "3,50€").
5. Jos tuotteen yhteydessä mainitaan laktoositon (L) tai gluteeniton (G),
   poimi ne badges-listaan, esim. ["L", "G"]. Jos ei mainintaa, badges: [].
6. Vastaa PELKÄSTÄÄN validilla JSON:lla, ei selityksiä, tässä muodossa:

{
  "restaurant": "Elävä Bistro",
  "updated": "<ISO-päivämäärä>",
  "categories": [
    {
      "name": "Kuumat juomat",
      "items": [
        { "name": "Tavallinen kahvi", "badges": [], "price": "2,00€" }
      ]
    }
  ]
}

Raakateksti:
{{1.text}}
```

- Aseta Gemini-moduulin vastauksen muoto/temperature niin, että se palauttaa puhdasta JSON:ia (esim. `response_mime_type: application/json`, jos malli/moduuli tukee sitä, tai puhdista koodilohko Make.comin JSON-parse-moduulilla ennen seuraavaa askelta).
- Lisää heti perään **JSON-parse**-moduuli, joka validoi Geminin vastauksen ennen kuin se menee GitHubiin.

### 3. GitHub-moduulit (GitHub → Other → Make an API Call)

Make.comin GitHub-sovelluksessa ei ole valmista natiivia "Create/Update a file" -moduulia.
Tiedoston kirjoittaminen tehdään geneerisellä **"Make an API Call"** -moduulilla
(löytyy GitHub-appin moduulilistalta kategoriasta "Other"), joka käyttää olemassa
olevaa GitHub-yhteyttä mutta antaa kutsua mitä tahansa GitHub REST -endpointtia.

Koska `data.json` on jo olemassa repossa, GitHub vaatii tiedoston nykyisen `sha`-arvon
mukaan päivityskutsuun — siksi tarvitaan kaksi peräkkäistä "Make an API Call" -askelta.

**3a. Hae nykyinen sha (GET)**

- URL: `/repos/<omistaja>/bistro-menu/contents/data.json`
- Method: `GET`
- Vastauksesta talteen kenttä `sha` — käytetään seuraavassa askeleessa `{{3a.sha}}`
  (moduulinumero voi olla eri, tarkista scenariosta).

**3b. Kirjoita uusi sisältö (PUT)**

- URL: `/repos/<omistaja>/bistro-menu/contents/data.json`
- Method: `PUT`
- Body (raw JSON):

```json
{
  "message": "Menu update {{formatDate(now; \"YYYY-MM-DD HH:mm\")}}",
  "content": "{{base64(<parsoitu-JSON-merkkijonona>)}}",
  "sha": "{{3a.sha}}",
  "branch": "main"
}
```

- `content`-kenttä pitää olla base64-koodattu — käytä Make.comin sisäistä
  `base64()`-funktiota parsitulle JSON-merkkijonolle (esim. JSON-parse-moduulin
  jälkeen `toString()`'illa takaisin merkkijonoksi ja sitten `base64(...)`).
- Autentikointi: sama GitHub-yhteys (Personal Access Token tai OAuth) molemmissa
  kutsuissa, jolla on kirjoitusoikeus repoon (PAT-scope `repo`, tai fine-grained
  tokenilla "Contents: Read and write").

Kun PUT-kutsu (3b) onnistuu, `menu.html` näyttää päivittyneen menun heti
seuraavalla sivulatauksella (GitHub Pages tarjoilee `data.json`-tiedoston
suoraan repositorystä).

## Repon luonti ja GitHub Pages

Tämä ympäristö ei pysty luomaan uutta GitHub-repoa suoraan (istunnon GitHub-yhteys on rajattu jo olemassa oleviin, ennalta valtuutettuihin repoihin), joten aja seuraavat komennot omalla koneellasi, jossa `gh` on kirjautuneena sinun GitHub-tunnuksellesi:

```bash
cd bistro-menu
gh repo create bistro-menu --public --source=. --remote=origin --push

# Ota GitHub Pages käyttöön main-branchilta, root-hakemistosta
gh api -X POST repos/:owner/bistro-menu/pages \
  -f "source[branch]=main" -f "source[path]=/"
```

Sivu on tämän jälkeen osoitteessa `https://<sinun-github-tunnus>.github.io/bistro-menu/menu.html`.

## Webhookin asettaminen

`index.html`-tiedoston lopussa on:

```js
const CONFIG = {
  webhookUrl: 'REPLACE_WITH_YOUR_MAKE_WEBHOOK_URL'
};
```

Korvaa `REPLACE_WITH_YOUR_MAKE_WEBHOOK_URL` Make.com-webhookisi URL:lla ja committaa muutos. Voit myös testata eri webhookilla väliaikaisesti ilman koodimuutosta lisäämällä osoitteeseen `?webhook=...`-parametrin, esim. `index.html?webhook=https://hook.eu2.make.com/xxxxx`.
