# Rešerše: hlasové podněty od občanů/uživatelů na Cloudflare

*Stav k 17. 10. 2026*

## Zadání (shrnutí)

- Nejde o chatbota, jen o **mikrofon**: člověk namluví návrh nebo komentář.
- Uživatel se **autentizuje** přes Telegram (preferovaný kanál), e-mail, SMS nebo WhatsApp. Stejný kanál pak slouží pro odpověď.
- Přepis z mluveného slova může být nesouvislý nebo zkomolený. **AI ho přepíše do spisovné, srozumitelné češtiny.**
- Podnět se uloží do **databáze** a zatřídí podle oblasti nebo typu.
- Podobné podněty se **slučují**, ostatní uživatelé hlasují 👍/👎 a administrátor vyřizuje nejdřív ty s nejvyšší hodnotou.
- Administrátor **odpovídá** přímo autorovi přes jeho kanál.

## Hlavní závěr

**Hotové řešení, které by umělo všechno najednou, na GitHubu není.** Pro každou část ale existují použitelné projekty na Cloudflare Workers, ze kterých se dá převzít kód nebo vzor:

| Část systému | Nejlepší kandidát | Proč |
|---|---|---|
| Hlasovací board + admin + e-mail OTP | **[almeidamarcell/marapulse](https://github.com/almeidamarcell/marapulse)** | Hono + D1 + Drizzle + KV + R2, hlasování, kategorie, stavy, e-mailový kód pro uživatele, magic link pro admina |
| Mikrofon → přepis → AI úprava → D1 | **[Oficiální tutoriál Cloudflare „Voice Notes App“](https://developers.cloudflare.com/workers-ai/tutorials/build-a-voice-notes-app-with-auto-transcription)** | Přesně tento tok: nahrávka v prohlížeči, Workers AI Whisper, post-processing LLM, D1 + R2 |
| AI třídění vícejazyčných vzkazů + odpověď zpět | **[idoZ-H/issuary](https://github.com/idoZ-H/issuary)** (MIT) | Telegram → hlasovky → LLM klasifikace → strukturovaný záznam → po vyřízení automatická odpověď autorovi |
| Telegram: obousměrná komunikace s adminem | **[miss82824352235/tg-pm-support-bot](https://github.com/miss82824352235/tg-pm-support-bot)**, **[ruguevara/tg-feedback-bot](https://github.com/ruguevara/tg-feedback-bot)** (MIT) | Workers + D1, každý uživatel má v admin skupině vlastní téma (forum topic), admin odpovídá přímo z Telegramu |
| Autentizace | **[zpg6/better-auth-cloudflare](https://github.com/zpg6/better-auth-cloudflare)** (MIT, ~590 ★) | Better Auth na D1/KV. Better Auth má pluginy pro e-mail OTP, magic link a telefonní číslo (SMS přes vlastního poskytovatele). |

**Doporučení:** postavit vlastní aplikaci na **Hono + D1 + Workers AI** a převzít z výše uvedených projektů vzory a kusy kódu. Kód Marapulse je možný základ boardu (licence O'Saasy povoluje self-hosting, zakazuje jen provozovat konkurenční SaaS). Zbytek (mikrofon, AI pipeline, Telegram) je potřeba doplnit.

---

## 1. Doporučená architektura (vše na Cloudflare)

```
 ┌────────────── Uživatel ──────────────┐
 │ A) Webová stránka s mikrofonem        │  MediaRecorder (webm/opus) + Turnstile
 │ B) Telegram Mini App (stejná stránka) │  autentizace zdarma přes initData
 │ C) Hlasová zpráva přímo botovi v TG   │  bez jakéhokoli UI
 └───────────────┬──────────────────────┘
                 ▼
        Worker (Hono)  ── Better Auth (e-mail OTP / SMS OTP / Telegram)
                 │
                 ├─► R2: originální audio (volitelné, s retencí kvůli GDPR)
                 ▼
        Workflows / Queues (asynchronní pipeline)
          1. STT: @cf/openai/whisper-large-v3-turbo (language: "cs", initial_prompt)
          2. LLM: přepis do spisovné češtiny + JSON {text, shrnutí, oblast, typ, spam?}
          3. Embedding (@cf/baai/bge-m3) → Vectorize → nalezení podobných podnětů
          4. Uložení do D1, notifikace uživateli „Váš podnět jsme přijali“
                 ▼
        D1: users, contacts, submissions, categories, votes, replies
                 ▼
        Admin rozhraní (/admin za Cloudflare Access)
          řazení podle skóre → odpověď → dispatcher kanálů:
          Telegram Bot API │ e-mail (Cloudflare Email Service / Resend) │ SMS (Twilio…) │ WhatsApp Cloud API
```

### 1.1 Mikrofon a přepis řeči

**Možnost A: nahrát a pak přepsat** (doporučeno pro začátek, nejjednodušší)
- V prohlížeči `MediaRecorder` → po skončení se odešle blob do Workeru → Workers AI.
- Model: **`@cf/openai/whisper-large-v3-turbo`**. Podle cenových přehledů stojí zhruba 0,0005 USD/min. Je vícejazyčný a podporuje parametry `language` a `initial_prompt`.
  - `language: "cs"` zabrání špatné detekci jazyka u krátkých nahrávek.
  - Do `initial_prompt` patří slovník domény (názvy ulic, institucí, odborné termíny). Výrazně to omezí komolení.
  - Delší nahrávky je potřeba dělit na části, viz [tutoriál Whisper s chunkingem](https://developers.cloudflare.com/workers-ai/tutorials/build-a-workers-ai-whisper-with-chunking).
- Alternativa: **`@cf/deepgram/nova-3`**. Deepgram v srpnu 2026 dvakrát vylepšil český model Nova-3 (`cs`, `cs-CZ`). **Ověřit:** jestli verze na Workers AI zpřístupňuje parametr jazyka. V Cloudflare API referenci je Nova-3 uvedená jen jako WebSocket endpoint.

**Možnost B: živý přepis během mluvení**
- Balíček **`@cloudflare/voice`** z Cloudflare Agents SDK obsahuje mixin **`withVoiceInput`**. Ten dělá jen přepis, bez odpovědi a bez TTS, a je určený přesně pro diktování. Používá Durable Object a modely Flux nebo Nova-3. Uživatel by viděl text naživo, ale je to složitější a přepis přes Flux je primárně anglický. Pro češtinu proto spíš možnost A.

**Možnost C: hlasovka v Telegramu**
- Uživatel pošle botovi obyčejnou hlasovou zprávu (OGG/Opus). Worker ji stáhne přes `getFile`, přepíše a zpracuje. Pro uživatele Telegramu je to nejjednodušší UX, žádná stránka není potřeba.
- Vzory: [Sovengar/voice-diary](https://github.com/Sovengar/voice-diary), [Sam4n/TelegramVoiceToText](https://github.com/Sam4n/TelegramVoiceToText), [myread02/WorkerWhisper](https://github.com/myread02/WorkerWhisper) (Ogg/Opus na Workeru).

### 1.2 AI přepis do spisovné češtiny

- Jeden požadavek na LLM se strukturovaným JSON výstupem:
  ```json
  {
    "clean_text": "Navrhuji, aby…",
    "summary": "Přechod pro chodce u školy v ulici X",
    "category": "doprava",
    "type": "návrh | stížnost | dotaz | pochvala",
    "is_spam_or_abuse": false,
    "confidence": 0.86
  }
  ```
- **Vždy ukládat obě verze:** surový přepis i upravený text. LLM může posunout význam.
- **Než se podnět uloží, ukázat uživateli upravenou verzi** a nechat ho potvrdit nebo opravit („Odpovídá text tomu, co jste chtěli sdělit?“). Je to nejlevnější ochrana proti zkreslení.
- Model: na Workers AI zvolit co nejsilnější vícejazyčný model (v aktuálním katalogu jsou např. modely řady Llama 4, Gemma, Qwen, gpt-oss nebo Kimi K2.6, který Cloudflare používá ve vlastním voice-agent příkladu). Kvalitu češtiny je potřeba otestovat na reálných nahrávkách.
- Volání vést přes **Cloudflare AI Gateway**: logování, cache, rate-limit a fallback na externího poskytovatele.
- **Alternativa:** Gemini (nebo jiný multimodální model) umí zpracovat audio rovnou a v jednom kroku vrátit hotový spisovný text. Tento přístup používá issuary pro hlasovky.

### 1.3 Autentizace a kontaktní kanál

| Kanál | Jak ověřit | Jak odpovědět | Poznámky |
|---|---|---|---|
| **Telegram** ⭐ | a) **Telegram Login Widget** s `data-request-access="write"`: ověření HMAC hashe ve Workeru a zároveň povolení, aby bot mohl uživateli psát; b) **deep link** `t.me/<bot>?start=<jednorázový_token>`: bot dostane `/start token` a spáruje `chat_id` se sessionou; c) **Mini App**: `initData` podepsaná Telegramem | `sendMessage(chat_id)` | Zdarma. **Bot nemůže uživateli napsat jako první.** Uživatel musí bota spustit nebo udělit povolení ve widgetu, proto je přihlášení přes Telegram zároveň nejlepší způsob, jak získat `chat_id`. |
| **E-mail** | 6místný kód (Better Auth `emailOTP`) | Cloudflare Email Service (binding `send_email`; podle dostupných zdrojů beta na Workers Paid, nutné ověřit) nebo Resend | Levné a spolehlivé |
| **SMS** | OTP přes **Twilio Verify** (REST přes `fetch`, Node SDK na Workers neběží) nebo jiného SMS poskytovatele; Better Auth plugin `phoneNumber` | SMS API poskytovatele | Placené za zprávu. Cloudflare SMS neposílá. |
| **WhatsApp** | OTP šablonou přes WhatsApp Cloud API | Mimo 24hodinové okno **jen schválená šablona** | Nutné Meta Business ověření. **Od 1. 10. 2026** Meta účtuje i odpovědi v 24h okně; podle respond.io je 1000 servisních zpráv měsíčně na číslo zdarma. Nejsložitější kanál, doporučuju až nakonec. |

Ochrana před zneužitím: **Cloudflare Turnstile** na formuláři, rate-limit (KV / Rate Limiting binding), jeden hlas na ověřeného uživatele.

### 1.4 Databáze (D1), návrh schématu

```sql
CREATE TABLE users (
  id TEXT PRIMARY KEY,
  display_name TEXT,
  created_at INTEGER NOT NULL
);

-- jeden uživatel může mít více ověřených kanálů
CREATE TABLE contacts (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  channel TEXT NOT NULL CHECK (channel IN ('telegram','email','sms','whatsapp')),
  address TEXT NOT NULL,           -- chat_id / e-mail / E.164 číslo
  verified_at INTEGER,
  preferred INTEGER DEFAULT 0,
  UNIQUE (channel, address)
);

CREATE TABLE categories (           -- oblasti/směrování (doprava, zeleň, IT…)
  id TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  parent_id TEXT REFERENCES categories(id),
  assignee TEXT                     -- komu se podnět směruje
);

CREATE TABLE submissions (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL REFERENCES users(id),
  audio_key TEXT,                   -- R2 klíč (volitelné, s retencí)
  raw_transcript TEXT NOT NULL,
  clean_text TEXT NOT NULL,
  summary TEXT,
  category_id TEXT REFERENCES categories(id),
  type TEXT,
  status TEXT NOT NULL DEFAULT 'new', -- new / in_review / answered / done / rejected
  duplicate_of TEXT REFERENCES submissions(id), -- sloučení podobných
  up INTEGER NOT NULL DEFAULT 0,
  down INTEGER NOT NULL DEFAULT 0,
  score REAL NOT NULL DEFAULT 0,    -- přepočítat při každém hlasu
  created_at INTEGER NOT NULL
);
CREATE INDEX idx_sub_queue ON submissions(status, category_id, score DESC);

CREATE TABLE votes (
  submission_id TEXT NOT NULL REFERENCES submissions(id),
  user_id TEXT NOT NULL REFERENCES users(id),
  value INTEGER NOT NULL CHECK (value IN (-1, 1)),
  created_at INTEGER NOT NULL,
  PRIMARY KEY (submission_id, user_id)
);

CREATE TABLE replies (
  id TEXT PRIMARY KEY,
  submission_id TEXT NOT NULL REFERENCES submissions(id),
  admin_id TEXT NOT NULL,
  body TEXT NOT NULL,
  channel TEXT NOT NULL,
  delivery_status TEXT,             -- sent / failed / delivered
  created_at INTEGER NOT NULL
);
```

**Řazení podle hodnoty:** prostý rozdíl `up − down` zvýhodňuje staré podněty. Lepší je **Wilsonovo skóre** (dolní mez 95% intervalu spolehlivosti podílu kladných hlasů). Spočítat ho ve Workeru při každém hlasu a uložit do `score`. Volitelně přidat časový útlum, aby nové podněty měly šanci.

**Kumulace podobných podnětů:** embedding upraveného textu (`@cf/baai/bge-m3`, vícejazyčný) uložit do **Vectorize**. U nového podnětu najít nejbližší sousedy. Nad prahem podobnosti nabídnout uživateli „Podobný podnět už existuje, chcete ho podpořit?“. Případně ho sloučí admin (`duplicate_of`) a hlasy se sečtou.

### 1.5 Administrace

- `/admin` chránit přes **Cloudflare Access** (Zero Trust). Pro malý tým je zdarma a nemusí se psát vlastní přihlášení adminů.
- Fronta: filtr podle oblasti a stavu, řazení podle `score`, detail s upraveným textem, surovým přepisem a přehráním audia.
- Tlačítko „Odpovědět“: dispatcher pošle odpověď kanálem, který si uživatel zvolil, a uloží ji do `replies`.
- **Bonus pro Telegram:** podle vzoru tg-pm-support-bot může každý nový podnět založit téma (forum topic) v admin skupině. Admin pak odpovídá přímo z Telegramu a bot zprávu doručí autorovi.

---

## 2. Katalog nalezených repozitářů

### 2.1 Hlas / přepis na Cloudflare Workers

| Repozitář | Co umí | Poznámka |
|---|---|---|
| [Cloudflare tutoriál „Voice Notes App“](https://developers.cloudflare.com/workers-ai/tutorials/build-a-voice-notes-app-with-auto-transcription) | Nahrávání v prohlížeči, Workers AI přepis, volitelný post-processing LLM, D1 + R2, Nuxt | **Nejbližší kostra pro mikrofon → DB.** Komunitní tutoriál, autorem je Rajeev R. Sharma (projekt „Vhisper“). |
| [Cloudflare tutoriál Whisper + chunking](https://developers.cloudflare.com/workers-ai/tutorials/build-a-workers-ai-whisper-with-chunking) | `whisper-large-v3-turbo`, dělení dlouhého audia | Pro delší nahrávky |
| `@cloudflare/voice` ([dokumentace](https://developers.cloudflare.com/agents/communication-channels/voice/), [voice agent příklad](https://developers.cloudflare.com/agents/examples/voice-agent/)) | `withVoiceInput` = jen přepis (diktování), `withVoice` = celý hlasový agent | Pro živý přepis během mluvení |
| [Lqm1/openai-workers-ai](https://github.com/Lqm1/openai-workers-ai) | OpenAI-kompatibilní Whisper API na Workers (Hono, TS) | Hotový endpoint pro přepis |
| [pew/whisper-api-to-cloudflare](https://github.com/pew/whisper-api-to-cloudflare) | OpenAI-kompatibilní proxy na Workers AI Nova-3, streaming, bearer auth | Varianta s Nova-3 |
| [octalpixel/audioflare](https://github.com/octalpixel/audioflare) | Přepis, shrnutí a překlad audia na Workers AI | Šablona; přepisuje jen prvních 30 s |
| [g-guerzoni/poc-audio-recorder-transcription-r2](https://github.com/g-guerzoni/poc-audio-recorder-transcription-r2) | Nahrávání v prohlížeči → R2 → Whisper ve Workeru | Malé PoC |
| [knd8412/cf_ai_echomind](https://github.com/knd8412/cf_ai_echomind) | Whisper + Llama 3.3 shrnutí + Durable Objects | Hlasový deník |
| [aleksbuss/trascribatorV2](https://github.com/aleksbuss/trascribatorV2) | **Telegram Mini App** s živým přepisem (Workers + Groq Whisper + Web Audio API) | Vzor pro mikrofon uvnitř Telegramu |
| [Sovengar/voice-diary](https://github.com/Sovengar/voice-diary), [Sam4n/TelegramVoiceToText](https://github.com/Sam4n/TelegramVoiceToText), [myread02/WorkerWhisper](https://github.com/myread02/WorkerWhisper) | Telegram hlasovka → Whisper na Workeru | Vzor pro možnost C |
| [SammyLin/yanroku](https://github.com/SammyLin/yanroku) | Worker + D1/R2 pro hlasové poznámky (přepis běží na Macu) | Spíš inspirace pro datový model |

### 2.2 AI třídění podnětů

| Repozitář | Co umí | Poznámka |
|---|---|---|
| [idoZ-H/issuary](https://github.com/idoZ-H/issuary) (MIT) | Klienti píšou nebo mluví v Telegramu ve svém jazyce, LLM klasifikuje (chyba/požadavek/kecání), vytvoří strukturovaný záznam, slučuje zprávy do 60 s, po vyřízení automaticky odpoví v jazyce klienta. Workers + KV + Vectorize, hlasovky přes Gemini. | **Koncepčně nejbližší projekt.** Prompt je laděný na hebrejštinu a výstupem jsou GitHub issues, obojí by se upravilo. |
| [althea-yuquan-chen/Cloudflare-Feedback-Analyzer](https://github.com/althea-yuquan-chen/Cloudflare-Feedback-Analyzer) | D1 + Workers AI (Llama 3 8B): sentiment, kategorie, týdenní souhrn | Jednoduchý vzor; licence neuvedena, jen jako inspirace |

### 2.3 Telegram boti na Workers (obousměrná komunikace)

| Repozitář | Co umí | Poznámka |
|---|---|---|
| [ruguevara/tg-feedback-bot](https://github.com/ruguevara/tg-feedback-bot) (MIT) | Inbox podnětů, admin vyřizuje nebo zamítá, notifikace uživateli; Python Workers + D1 | Čisté, otestované, skripty pro nasazení |
| [miss82824352235/tg-pm-support-bot](https://github.com/miss82824352235/tg-pm-support-bot) | Forum topic na uživatele, obousměrné přeposílání, captcha, D1 + KV, nasazení na jeden příkaz | README v čínštině, licence neuvedena (jen jako inspirace) |
| [AkkilMG/Feedback-Bot-CF](https://github.com/AkkilMG/Feedback-Bot-CF) | Klon LivegramBotu na Workers | Jednoduchý |
| [tbxark/telegram-bot-workers-template](https://github.com/tbxark/telegram-bot-workers-template), [kevindaffaarr/grammy_cloudflare_dev_template](https://github.com/kevindaffaarr/grammy_cloudflare_dev_template) | Šablony bota (TS, grammY) | Kostra pro vlastního bota |

### 2.4 Hlasovací boardy (👍/👎, kategorie, stavy)

| Repozitář | Stack | Hodnocení |
|---|---|---|
| [almeidamarcell/marapulse](https://github.com/almeidamarcell/marapulse) | **Workers, Hono JSX, Alpine.js, D1 + Drizzle, KV, R2, Resend** | Hlasování, komentáře, kategorie, stavy (New → Done), uživatelé přes 6místný e-mailový kód, admin přes magic link, vložitelný widget. Licence O'Saasy. **Nejlepší základ na Cloudflare.** |
| [echoomegaprime/echo-feedback-board](https://github.com/echoomegaprime/echo-feedback-board) | Workers + D1 + KV | Hlasování, roadmapa, AI shrnutí; velmi raná fáze, autentizace jen API klíčem |
| [getfider/fider](https://github.com/getfider/fider) (~4,6k ★) | Go + PostgreSQL, AGPL, open-core | Nejzralejší open-source board, **na Workers ale nepoběží**. Inspirace pro UX. |
| [logchimp/logchimp](https://github.com/logchimp/logchimp) (~1,1k ★) | TypeScript + PostgreSQL, Apache-2.0 | Totéž: vzor, ne základ |
| [decidim/decidim](https://github.com/decidim/decidim) (~1,8k ★), [consuldemocracy/consuldemocracy](https://github.com/consuldemocracy/consuldemocracy) (~1,5k ★) | Ruby on Rails | Platformy pro **občanskou participaci** (návrhy, hlasování, participativní rozpočty). Pokud jde o obec nebo instituci, jsou dobrou inspirací pro procesy a kategorie. |

### 2.5 Autentizace a fullstack kostry

| Repozitář | Co umí |
|---|---|
| [zpg6/better-auth-cloudflare](https://github.com/zpg6/better-auth-cloudflare) (MIT, ~590 ★) | Better Auth na D1/KV/R2, CLI pro generování projektu (Hono, Next.js) a migrace |
| [johanguse/cloudflare-fullstack-boilerplate](https://github.com/johanguse/cloudflare-fullstack-boilerplate) | Hono + React 19 + D1 + Drizzle + Better Auth + Cloudflare Email |
| [qinsong77/cloudflare-workers-d1-hono-drizzle-better-auth-template](https://github.com/qinsong77/cloudflare-workers-d1-hono-drizzle-better-auth-template) | Minimální šablona Hono + D1 + Drizzle + Better Auth |

---

## 3. Rizika a na co nezapomenout

1. **GDPR:** hlas, telefon i e-mail jsou osobní údaje. Je potřeba informovat o zpracování (včetně AI), nastavit retenci audia v R2 (např. smazat po 30 dnech pomocí lifecycle pravidla) a umožnit smazání účtu. Pro veřejné zobrazení podnětů ukazovat jen upravený text bez jmen (LLM může zároveň anonymizovat).
2. **Zkreslení významu AI:** ukládat surový přepis, uživatel potvrzuje upravenou verzi, admin vidí obojí.
3. **Kvalita češtiny:** před výběrem modelu otestovat Whisper i Nova-3 (a případně Gemini) na 20–30 reálných nahrávkách s místními názvy. Spolehlivé srovnání WER pro češtinu jsem nenašel.
4. **Telegram:** bot nemůže začít konverzaci sám. Přihlášení proto navrhnout tak, aby `chat_id` vzniklo už při autentizaci (deep link nebo widget s `request_access=write`).
5. **WhatsApp:** od 1. 10. 2026 se mění ceník, odpovědi mimo 24h okno jdou jen šablonou a je nutné Meta Business ověření. Doporučuju až jako poslední kanál.
6. **Spam a manipulace s hlasy:** Turnstile, rate-limit a hlasovat jen s ověřeným kanálem (jeden hlas na uživatele, `PRIMARY KEY (submission_id, user_id)`).

## 4. Co se nepodařilo ověřit

- Stránky `developers.cloudflare.com` a `blog.cloudflare.com` nebyly z tohoto prostředí dostupné (blokované DNS/proxy). Údaje o modelech Workers AI, cenách a `@cloudflare/voice` proto pochází z výsledků vyhledávání a sekundárních zdrojů. Před implementací je potřeba ověřit aktuální katalog modelů.
- Zda Workers AI verze `@cf/deepgram/nova-3` přijímá parametr jazyka `cs`.
- Zda je Cloudflare Email Service už mimo betu.

## 5. Navržený postup (MVP → plná verze)

1. **MVP:** web s mikrofonem + Turnstile, přihlášení e-mailem (OTP) a Telegramem, Whisper + LLM úprava s potvrzením uživatelem, D1, jednoduchý admin za Cloudflare Access s odpovědí přes Telegram/e-mail.
2. **Hlasování a kategorie:** veřejný seznam podnětů, 👍/👎, Wilsonovo skóre, filtry podle oblasti.
3. **Slučování duplicit:** Vectorize + embeddingy, nabídka „podpořit existující podnět“.
4. **Telegram nativně:** hlasovky přímo botovi, admin odpovídá z forum topics.
5. **SMS a WhatsApp:** až bude jasná poptávka (oba jsou placené a administrativně náročnější).

---

## Zdroje

- Cloudflare Workers AI: [Voice Notes App tutoriál](https://developers.cloudflare.com/workers-ai/tutorials/build-a-voice-notes-app-with-auto-transcription), [Whisper s chunkingem](https://developers.cloudflare.com/workers-ai/tutorials/build-a-workers-ai-whisper-with-chunking), [changelog nových modelů 03/2025](https://developers.cloudflare.com/changelog/2025-03-17-new-workers-ai-models/), [cena whisper-large-v3-turbo (cloudprice.net)](https://cloudprice.net/models/%40cf/openai/whisper-large-v3-turbo)
- Cloudflare Agents / hlas: [voice agent příklad](https://developers.cloudflare.com/agents/examples/voice-agent/), [voice kanál](https://developers.cloudflare.com/agents/communication-channels/voice/), [blog „Add voice to your agent“](https://blog.cloudflare.com/voice-agents/), [Deepgram Flux changelog](https://developers.cloudflare.com/changelog/post/2025-10-02-deepgram-flux/index.md)
- Cloudflare Email: [Email Service Workers API](https://developers.cloudflare.com/email-service/api/send-emails/workers-api/index.md), [Email Routing konfigurace](https://developers.cloudflare.com/email-routing/configuration/)
- Deepgram čeština: [Nova-3 +11 jazyků](https://deepgram.com/learn/deepgram-expands-nova-3-with-11-new-languages-across-europe-and-asia), [changelog 7. 8. 2026](https://developers.deepgram.com/changelog/2026/8/7), [changelog 27. 8. 2026](https://developers.deepgram.com/changelog/2026/8/27)
- Srovnání Whisper vs. Nova: [CallMissed](https://www.callmissed.com/blog/deepgram-nova-vs-whisper-large-v3-turbo), [mikr/whisper-large-v3-czech-cv13](https://huggingface.co/mikr/whisper-large-v3-czech-cv13)
- Telegram: [Login Widget](https://core.telegram.org/widgets/login-legacy), [omezení „bot nemůže psát první“ (Sinch)](https://support.sinch.com/hc/en-us/articles/54629170811027-Is-it-possible-to-use-a-Telegram-Bot-to-send-campaign-or-regular-updates-to-Telegram-users)
- SMS: [Twilio Verify z Cloudflare Workeru (Webflow)](https://webflow.com/blog/twilio-sms-2fa-webflow-cloud), [Twilio serverless phone verification](https://www.twilio.com/en-us/blog/serverless-phone-verification)
- WhatsApp: [respond.io: změna ceníku 2026](https://respond.io/blog/whatsapp-pricing-change-2026.md)
- Feedback boardy: [Fider vs LogChimp](https://openalternative.co/compare/fider/vs/logchimp), [Astuto vs Fider](https://openalternative.co/compare/astuto/vs/fider), [Cloudron fórum](https://forum.cloudron.io/post/52838)
- Vhisper: [DEV článek](https://dev.to/ra_jeeves/building-vhisper-voice-notes-app-with-ai-transcription-and-post-processing-1609)

---

## Doplnění 24. 10. 2026: odpovědi z webu obce Příšovice

Provozovatelem je obec Příšovice. Odpovědi se mají primárně hledat na webu obce, který je zdrojem pravdy. Každá odpověď musí odkazovat na konkrétní článek nebo dokument. Když se odpověď nenajde, dotaz převezme správce.

- **Web obce:** [prisovice.cz](https://www.prisovice.cz). Podle adres dokumentů (`/wp-content/uploads/…`) zřejmě běží na WordPressu a dokumenty zveřejňuje jako PDF, například [jednací řád zastupitelstva 2022–2026](https://prisovice.cz/wp-content/uploads/2024/01/jednaci-rad-2022-2026.pdf).
- **Úřední deska:** samostatný systém e-obec, [prisovice.e-obec.cz](https://prisovice.e-obec.cz/urednideska.php?soubor=378).
- **Stahování webu:** Cloudflare Browser Rendering má od 10. 3. 2026 v otevřené betě endpoint [`/crawl`](https://developers.cloudflare.com/changelog/2026-03-10-br-crawl-endpoint). Projde web od zadané adresy a vrátí stránky jako HTML, Markdown nebo JSON. Umí opakované běhy jen nad změněnými stránkami (`modifiedSince`) a respektuje robots.txt. Druhá možnost je WordPress REST API, pokud ho web má zapnuté.
- **Vyhledávání:** vestavěné procházení webu v Cloudflare AI Search funguje jen pro doménu ve stejném Cloudflare účtu ([dokumentace](https://developers.cloudflare.com/fundamentals/reference/cloudflare-site-crawling/)). Web obce tam pravděpodobně není, proto vlastní stahování do R2 a AI Search nad R2.
- **Neověřeno:** web obce i dokumentace Cloudflare byly z tohoto prostředí blokované. Údaje pochází z výsledků vyhledávání.

---

## Doplnění 24. 10. 2026: český a evropský hosting

### Povinnost pro obce: katalog cloud computingu

Když aplikaci provozuje obec, smí podle zákona o informačních systémech veřejné správy (§ 6l a násl. ZoISVS) používat jen cloud zapsaný v **katalogu cloud computingu**, který vede Digitální a informační agentura (DIA). Přechodné období skončilo 31. 12. 2023. Od 1. 1. 2026 platí pro zápis vyhláška 505/2025 Sb. Bezpečnostní úrovně jsou nízká, střední, vysoká a kritická (vyhláška 411/2025 Sb.).

- Zapsaný musí být poskytovatel i konkrétní nabídka služby. Nevyhovující cloud musí úřad přestat používat do 12 měsíců.
- **Cloudflare:** zápis v katalogu se mi nepodařilo potvrdit.
- **Potvrzený český poskytovatel:** MasterDC je v katalogu na bezpečnostní úrovni 3 (srpen 2025), s datovými centry v ČR.
- **Neověřeno:** WEDOS, Zerops, vshosting a evropské clouds (Scaleway, OVHcloud, Hetzner, IONOS).
- Katalog na dia.gov.cz byl z tohoto prostředí blokovaný. Před ostrým provozem je potřeba ověřit stav přímo v katalogu a s DIA nebo právníkem obce.

Zdroje: [výklad § 6l ZoISVS (DIA)](https://dia.gov.cz/media/2233/download/Vyklad-ustanoveni-%C2%A7-6l-ZoISVS_231223.pdf), [workshop DIA k zápisu do katalogu 2025](https://dia.gov.cz/media/2221/download/2025-03-05_Workshop_zapis-do-katalogu-CC_DIA.pdf), [nabídky v katalogu](https://www.dia.gov.cz/cs/nase-cinnosti/na-cem-pracujeme/egovernment-cloud/katalog-cloud-computingu/nabidky-cloud-computingu), [metodika DIA pro poskytovatele](https://metodiky.dia.gov.cz/egov-cloud/metodika-pro-poskytovatele), [Lupa: MasterDC na úrovni 3](https://www.lupa.cz/aktuality/masterdc-je-prvni-poskytovatel-z-ceska-s-vysokou-urovni-bezpecnosti-na-dodavky-cloudu-statu/), [MasterDC pro veřejnou správu](https://www.masterdc.com/cloud-for-government/).

### Hosting aplikace a databáze

| Poskytovatel | Země | Co nabízí | Orientační cena | Katalog |
|---|---|---|---|---|
| [Zerops](https://www.zerops.io) | ČR (Praha, vznikl ve vshosting) | PaaS: aplikace (Bun, Python…), spravovaný PostgreSQL, S3 úložiště, cron | Plán 10 USD měsíčně + služby po minutách; ukázková aplikace s DB a úložištěm 12,83 USD ([getdeploying](https://getdeploying.com/zerops)) | Neověřeno |
| [MasterDC](https://www.masterdc.com/cloud-servers/) | ČR (Praha, Brno) | IaaS a PaaS, vlastní cloudová platforma | Na poptávku | Ano, úroveň 3 |
| [WEDOS](https://www.wedos.cz) | ČR (Hluboká nad Vltavou) | VPS a cloud, správa serveru je na nás | SSD VPS od cca 12 USD měsíčně za 4 GB RAM ([WHTop](https://www.whtop.com/plans/wedos.com/144763)) | Neověřeno |
| Scaleway / OVHcloud | Francie | Kontejnery, spravovaný PostgreSQL, úložiště i AI v jednom účtu | Podle využití | Neověřeno |

pgvector v PostgreSQL na Zerops se mi nepodařilo potvrdit.

### AI: přepis řeči a jazykový model

Český poskytovatel jazykového modelu jsem nenašel. Pro přepis češtiny existuje česká služba Beey.

| Služba | Země | Co nabízí | Cena |
|---|---|---|---|
| [Beey API](https://docs.beey.io/) (Newton Technologies) | ČR (Praha) | Přepis řeči, čeština je hlavní jazyk | Na poptávku |
| [Mistral AI](https://mistral.ai/cloud/inference/) | Francie | Jazykové modely (Mistral Small, Large) a přepis (Voxtral), EU infrastruktura podle zprostředkovatelů | Mistral Large 0,50 / 1,50 USD za milion tokenů (vstup / výstup), řeč se účtuje po minutách |
| [OVHcloud AI Endpoints](https://www.ovhcloud.com/de/public-cloud/ai-endpoints/catalog/whisper-large-v3/) | Francie | Whisper large v3, jazykové modely (např. gpt-oss-120b), OpenAI-kompatibilní API | Oficiální ceník jsem neověřil |
| Scaleway Generative APIs | Francie | Whisper large v3, jazykové modely, OpenAI-kompatibilní API | Whisper cca 0,0035 USD za minutu ([computeprices](https://computeprices.com/providers/scaleway/models/whisper-v3-large)) |

### Dopad na architekturu

Bez Cloudflare se změní stavební kameny, ale ne logika aplikace:

- Workers → Node.js/TypeScript (Hono běží i mimo Cloudflare)
- D1 → PostgreSQL
- Vectorize + AI Search → pgvector v PostgreSQL
- R2 → S3-kompatibilní úložiště
- Workers AI → OpenAI-kompatibilní API evropského poskytovatele
- Browser Rendering `/crawl` → vlastní jednoduché stahování podle mapy webu a převod PDF na text

Když se aplikace postaví přenositelně (Node.js, PostgreSQL, S3, OpenAI-kompatibilní API), poběží na Zerops, WEDOS VPS, MasterDC i u Scaleway bez přepisování.
