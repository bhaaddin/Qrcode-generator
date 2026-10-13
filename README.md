# Modul QR

Samostatný modul, který se dá vložit do jakéhokoli webu. Umí jen čtyři věci:

| Tlačítko | Co dělá |
|---|---|
| **Platební QR** | QR pro platbu rezervace (formát SPAYD, variabilní symbol z kódu rezervace) |
| **QR lístku** | QR s tokenem lístku (jen token, nic jiného) |
| **Skener** | čte QR z kamery a ověří ho přes nastavitelný adaptér |
| **Banka** | načte výpis z banky a spáruje platby s rezervacemi |

Nic dalšího v něm není — žádné přihlášení, databáze ani prodej.

## Rychlá integrace (jeden skript, jeden tag)

```html
<script src="modul-qr.iife.js"></script>
<modul-qr config="config.json"></modul-qr>
```

Soubor `dist/modul-qr.iife.js` nemá žádné závislosti. Konfiguraci lze dát
i přímo do tagu (JSON uvnitř), nebo z kódu: `el.konfigurace = { … }`.
Pro ES moduly: `import { vytvorModul } from './modul-qr.js'`.

## Nastavení pro třídu (`config.example.json`)

- `platba.iban` — účet třídy; kontroluje se kontrolní součet a překlep QR nevyrobí.
- `platba.predpona` nebo `platba.datumPlesu` — začátek variabilního symbolu (MMDD z data);
  `platba.delkaCisla` — kolik číslic má pořadí rezervace (výchozí 4).
- `platba.zprava` — text zprávy pro příjemce, `{kod}` se nahradí kódem rezervace.
- `banka.adapter` — `vypis` (soubor CSV/JSON z bankovnictví, funguje u každé banky)
  nebo `fio` (token jen pro čtení, `banka.token`). Jiné názvy sloupců u `vypis`:
  `banka.sloupce: { "Datum": "Datum zaúčtování", "Objem": "Částka", "VS": "Variabilní symbol" }`.
- `system.adapter` — jak se ověřuje lístek: `kod` (jen vrátí přečtený text), `4thebeat`
  (`POST {system.url}/api/kontrola`), nebo vlastní funkce `system.overit(kod)`.

Další banku nebo způsob ověření přidáš bez zásahu do modulu:
`registrujBanku('nazev', nastaveni => ({ nacti: async vstup => [pohyby] }))`,
`registrujOvereni('nazev', nastaveni => async kod => ({ stav: 'ok' }))`.

## Kód místo tlačítek

```js
const m = ModulQr.vytvorModul(konfigurace);
m.platba.qr({ kod: 'R-0098', castka: 350 }); // { spayd, vs, svg, adresa }
m.listek.qr(token);                          // { svg, adresa }
m.skener.spust(videoEl, { po: (kod, vysledek) => … });
m.banka.zkontroluj(souborBajty, rezervace);  // sparovano, castkaNesedi, nejednoznacne, nenalezeno, nezaplaceno
```

Události na tagu: `modulqr:platba`, `modulqr:sken`, `modulqr:banka`.
Vzhled: CSS proměnné `--mqr-akcent`, `--mqr-pozadi`, `--mqr-okraj`, `--mqr-text`, `--mqr-chyba`.

## Párování plateb — pravidla

Spáruje se jen při shodě symbolu **a** přesné částky. Symbol u víc rezervací nebo
víc plateb se nehádá (`nejednoznacne`). Banky zahazují úvodní nuly symbolu, proto
se porovnává bez nich. Odchozí platby se ignorují; cizí měna jde do `nenalezeno`.

## Omezení: ověření lístku z jiné domény

Adaptér `4thebeat` funguje jen ze **stejné domény** jako systém lístků (přihlášení je
cookie `SameSite=Lax` a server odmítá cizí POST). Z cizího webu je potřeba
tokenové ověřovací API — viz `FLAGS.md` F-009. Do té doby použij `system.overit`.

## Sestavení a zkouška

```
node modul-qr/sestavit.mjs          # přegeneruje dist/
node scripts/zkouska-modul-qr.mjs   # zkouška (běží v CI)
```
