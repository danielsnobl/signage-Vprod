# CLAUDE.md — menuTvVb

## Co projekt dělá a pro koho

Digitální signage systém pro bar Vnitroblock. Na TV obrazovce zobrazuje menu a otevírací doby podle aktuálního času a dne v týdnu. Systém tvoří dvě části:

- **`index.html`** — veřejná stránka zobrazovaná na TV, automaticky přepíná obrázky podle rozvrhu
- **`admin.html`** — přístupný pouze správci; umožňuje nahrát nové obrázky menu přes GitHub API bez znalosti gitu

Cílová skupina: obsluha baru a správci systému.

---

## Technický stack

- **HTML5 + CSS3 + Vanilla JS (ES6+)** — žádné frameworky ani balíčky
- **GitHub REST API** — ukládání a nahrazování obrázků v repozitáři
- **GitHub Personal Access Token (PAT)** — autentizace v admin panelu, uložen v `localStorage`
- **Cloudflare Pages** — deployment; statické soubory servírovány přímo z větve `main`

Žádný build systém, žádný `package.json`, žádný backend server.

---

## Struktura projektu

```
menuTvVb/
├── index.html          # TV display — veřejná stránka
├── admin.html          # Admin panel — nahrávání obrázků
├── assets/             # Obrázky menu (JPEG, 4K, 3840×2160 px)
│   ├── snidane.jpg
│   ├── snidane_vikend.jpg
│   ├── obed_tydne.jpg
│   ├── vecere.jpg
│   ├── napoje_a_oteviraci_doba_kuchyne.jpg
│   ├── vypln_pred_otevrenim.jpg
│   ├── vypln_mezi_snidani_a_obedem.jpg
│   ├── vypln_mezi_obedem_a_veceri.jpg
│   └── vypln_po_zavreni.jpg
└── CLAUDE.md
```

---

## Jak funguje TV display (`index.html`)

Stránka každých 15 s zkontroluje čas a den v týdnu a zobrazí odpovídající playlist obrázků. Obrázky v playlistu se střídají každých 15 s. Každé 4 hodiny se stránka plně refreshuje.

**Klíčové proměnné v kódu:**
| Proměnná | Účel |
|---|---|
| `VERSION` | Cache-busting — změnit při každém release (formát `2026wXXXX`) |
| `ROTATE_EVERY_MS` | Interval střídání obrázků v playlistu (default 15 s) |
| `HARD_REFRESH_EVERY_MS` | Interval plného refreshe stránky (default 4 h) |
| `PLAYLISTS` | Mapování klíčů na pole obrázků |
| `SCHEDULE` | Rozvrh — čas → klíč playlistu, zvlášť pro víkend a všední den |

---

## Jak funguje deployment (GitHub → Cloudflare Pages)

1. Změny se pushnou do větve `main`
2. Cloudflare Pages automaticky nasadí novou verzi
3. TV prohlížeč refreshuje stránku každé 4 hodiny (nebo manuálně)

**Repozitář:** `danielsnobl/signage-Vprod` na GitHubu
**Branch pro produkci:** `main`

> Po každém nahrání obrázku přes admin panel je potřeba ručně aktualizovat `VERSION` v `index.html` a pushnou, aby TV stáhla nové obrázky (cache busting).

---

## Jak funguje admin stránka a GitHub API upload

1. Správce otevře `admin.html` a zadá GitHub PAT s `repo` scopem
2. Token se uloží do `localStorage`, ověří se voláním `GET /user`
3. Stránka načte aktuální soubory z `assets/` přes GitHub Contents API
4. Správce klikne na kartu obrázku a nahraje nový JPEG (max 5 MB)
5. Admin panel zakóduje soubor do base64 a pošle `PUT` request na GitHub API
6. GitHub API automaticky vytvoří commit s popisem `Aktualizace obrázku: {název}`

**Konfigurace v `admin.html`:**
- `REPO_OWNER`, `REPO_NAME`, `REPO_BRANCH` — identifikace repozitáře
- `LABELS` — česky pojmenované popisky karet
- `SCHEDULE_TIMES` — zobrazené časy rozvrhu u každé karty
- `PLAYLIST_ORDER` — pořadí karet v gridu

---

## Pravidla pro práci na projektu

### Vždy vytvořit novou větev před změnami
Nikdy nepracovat přímo na `main`. Každá změna dostane vlastní větev:
```
git checkout -b feature/nazev-zmeny
```
Po schválení se merguje do `main` — ten jde rovnou do produkce.

### Nikdy nelinkovat `admin.html` z `index.html`
`admin.html` musí zůstat skrytý před návštěvníky a TV prohlížečem. Nesmí se na něj odkazovat z veřejné stránky žádným způsobem (odkaz, skript, meta tag, apod.).

### Zachovat zpětnou kompatibilitu
- Názvy souborů v `assets/` jsou pevně zakódovány v `index.html` i `admin.html` — přejmenování obrázku rozbije display
- Pokud se musí přejmenovat soubor, je nutné aktualizovat `PLAYLISTS`, `LABELS`, `SCHEDULE_TIMES` i `PLAYLIST_ORDER` najednou
- `VERSION` měnit pouze tehdy, když se mění obsah obrázků — zbytečná změna způsobuje zbytečný cache miss na TV

---

## Plánované funkce

### Promo overlay (feature/promo-overlay)
Globální vrstva zobrazovaná přes stávající playlist.
- Každých 10 minut se zobrazí promo obrázek (`assets/promo.jpg`)
- Viditelný 10 sekund, pak zmizí
- Ostrý přechod (bez prolínání)
- Zobrazuje se celý den
- Nastavení (interval, délka, zapnout/vypnout, obrázek) spravovatelné přes `admin.html`
- Nastavení uloženo v `config.json` v repozitáři
