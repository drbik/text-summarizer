# Hlasové podněty – Nezávislí Příšovice

Webová aplikace, kde občané Příšovic namluví návrh nebo dotaz. AI ho přepíše do spisovné češtiny a přiřadí k existujícímu tématu. Pak hledá odpověď na webu obce, a když ji nenajde, odpoví zastupitel. Nejde o chatbota: AI s uživatelem nevede dialog, jen zpracuje jeden vzkaz.

Provozovatel je uskupení Nezávislí Příšovice (9 z 15 zastupitelů). Aplikace bude součástí jeho webu nezavisli-prisovice.cz a není úředním kanálem obce.

## Dokumenty

- `docs/PRD.md` – požadavky, verze 0.7 (export). Živá verze je v Claude Docs: https://claude.ai/artifact/1uqzdxoYxj8jR45McMdo3Z. Při rozporu platí živá verze.
- `docs/research-hlasove-podnety.md` – rešerše hotových řešení, hostingu a všech zdrojů.

Požadavky mají ID (F-01 až F-83) a prioritu MVP / Později / Zvážit. V commitech a kódu na ně odkazuj.

## Rozhodnutí (24. 10. 2026)

- Zdrojem pravdy pro odpovědi je celý obsah webu prisovice.cz včetně PDF. Úřední deska na prisovice.e-obec.cz se nesleduje.
- Odpověď z webu se autorovi ukáže hned, bez schválení. Autor potvrdí „stačí“, nebo se doptá. Doptání je poddotaz ke stejnému tématu.
- Stejné dotazy se slučují do jednoho tématu. O shodě rozhoduje autor: totéž, poddotaz, nebo jiné téma.
- Web se zaindexuje jednou celý, pak se stahují jen změny (výchozí denně, lze týdně). Stahuje se bez spolupráce správce webu.
- Témata a odpovědi jsou veřejné bez přihlášení. Hlasovat smějí ověření občané; v MVP se pobyt neověřuje a hlasuje každý přihlášený.
- Věková hranice je 15 let.
- Oblasti vychází z aktuálních a budoucích projektů na nezavisli-prisovice.cz.
- U každé odpovědi musí být upozornění, že nejde o úřední vyjádření obce.

## MVP (prezentační verze)

1. Stránka s mikrofonem a záložním textovým polem.
2. Přihlášení a odpovědi přes Telegram.
3. Přepis řeči, AI úprava do spisovné češtiny, potvrzení textu autorem.
4. Hledání podobných témat: připojit se, poddotaz, nebo nové téma.
5. Odpověď z webu prisovice.cz s odkazem na zdroj; index se aktualizuje ručně.
6. Bez odpovědi ze zdroje převezme dotaz správce.
7. Veřejný seznam témat a odpovědí, hlasování palcem.
8. Jednoduchá administrace: fronta podle skóre a odpověď.

Do MVP nepatří e-mail, GDPR souhlasy a moderace, zakrytí osobních údajů, ověření pobytu ani pravidelný index. Nerozšiřuj MVP bez souhlasu.

## Otevřené otázky, které blokují start

- Hosting: Cloudflare (Workers, D1, Vectorize, AI Search), nebo český Zerops + Mistral. Rozhodne hlavně to, na čem běží web nezavisli-prisovice.cz.
- Repozitář webu nezavisli-prisovice.cz zatím není připojený.
- Kdo bude správce osobních údajů podle GDPR.

## Zásady pro kód

- Dokud není rozhodnutý hosting, piš přenositelně: TypeScript a Hono, přístup k databázi, úložišti a AI schovaný za vlastní rozhraní.
- Automatická odpověď smí obsahovat jen informace z nalezených úryvků. Každé tvrzení má citaci (URL, název, datum). Bez opory ve zdroji jde dotaz ke správci, nikdy si AI nic nedomýšlí.
- Ukládej surový přepis i upravený text.
- Texty pro uživatele piš spisovnou češtinou.
- Osobní údaje sbírej v minimu. V MVP neukládej audio.
