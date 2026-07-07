# Design: "Spela mot andra" — enkel realtids-Kahoot

**Datum:** 2026-07-07
**Status:** Godkänd design, väntar på implementationsplan

## Syfte

Låta flera personer på varsin telefon tävla live i Winnerbäck-låtquizet — en enkel
Kahoot-variant. Solo-spelen (flaggspel, gissa-låten) lämnas orörda.

## Val som ligger fast (från brainstorming)

- Varsin telefon, realtid, PIN-join, live-topplista (äkta Kahoot-känsla).
- Poäng: **rätt = 1, fel/inget svar = 0**, med **timer per fråga** (ingen snabbhetsbonus).
- Bara **flerval** (4 alternativ) i multiplayer.
- **Auto-pacing:** timer → facit + delresultat några sek → nästa fråga automatiskt.
- Matchlängd: host väljer **5 / 10 / 15 frågor**.
- Timer: fast **15 sek/fråga** (inte valbart i denna version).

## Arkitektur

**Ephemeral realtid via Supabase Realtime (Broadcast + Presence). Inga DB-tabeller.**

- Frontend förblir en statisk `index.html` (funkar på både Vercel och GitHub Pages).
- `@supabase/supabase-js` v2 laddas från CDN **först när man går in i multiplayer** —
  solo-spelen förblir beroendefria/offline.
- Nytt **dedikerat gratis Supabase-projekt** `winnerback-quiz`, region **eu-north-1
  (Stockholm)**. Endast den publika **anon-nyckeln** bäddas in i `index.html`.
  Isolerat från det befintliga BRF-projektet "gulmaran" (som innehåller GDPR-känslig
  persondata — får inte blandas med ett publikt partyspel).
- **Publik realtidskanal** per match: topic `spel:game:<PIN>`. Inga tabeller → ingen
  RLS att konfigurera (publika Broadcast/Presence-kanaler funkar med anon-nyckel).

### Varför inte databas-backad
En DB-variant (games/players/answers + realtidsändringar) tål omladdning/återanslutning
men kräver schema + RLS-policyer för anonym åtkomst och mer kod. Overkill för en match
som spelas i ett svep. Vald bort medvetet.

## Klient: tillstånd och skärmar

En ny meny-sektion **"Spela mot andra"** bredvid de två befintliga spelkorten.

Klienten genererar ett bestående `playerId` (localStorage) och minns senaste smeknamn.

Skärmar (states):
1. `mp-menu` — välj Skapa spel / Gå med.
2. `mp-lobby-host` — PIN visas stort, live-lista på anslutna (Presence), inställningar
   (album via befintliga filtren, antal frågor 5/10/15), Starta-knapp.
3. `mp-lobby-player` — "Väntar på att värden startar…", live-lista på deltagare.
4. `mp-question` — citat + 4 alternativknappar + nedräkning.
5. `mp-reveal` — rätt svar markerat + delresultat (topplista).
6. `mp-final` — prispall topp 3 + full lista. Knapp: spela igen / tillbaka till meny.

## Protokoll (Broadcast-events på kanalen)

- **Presence**: varje klient trackar `{playerId, nickname, isHost}`. Lobbyns lista =
  presence-state; hanterar join/leave live.
- `start` (host→alla): `{questions:[{quote, options:[4 titlar], correctIndex}], timerSec, total}`.
  Alla klienter får identiska frågor/alternativ.
- `question` (host→alla): `{index}` — gå till frågan; varje klient startar en lokal
  15-sek-nedräkning vid mottagande (lokal start; nätverkslatens <1 s accepteras som
  försumbar orättvisa, undviker klock-synk).
- `answer` (spelare→host+self): `{playerId, index, choiceIndex, correct}`.
- `reveal` (host→alla): `{index, correctIndex, scores:[{playerId, nickname, score}]}`.
- `finished` (host→alla): `{scores:[…slutställning…]}`.

## Spel-loop (hostens enhet är auktoritet)

1. **Skapa:** generera 4-siffrig PIN, gå med i kanalen, sätt presence `isHost=true`.
2. **Starta:** bygg frågeuppsättning ur vald album-pool (återanvänd befintlig
   distraktor-logik: rätt titel + 3 slumpade titlar ur poolen, blanda, spara
   `correctIndex`). Broadcasta `start` + första `question`, starta timer.
3. **Under frågan:** samla inkommande `answer` i en map per frågeindex (dedupe per
   playerId; ignorera dubbletter).
4. **Timer slut (eller alla svarat):** räkna ut vilka som svarade rätt, öka deras
   kumulativa poäng, broadcasta `reveal` med topplista. Vänta ~4 s.
5. Fler frågor → index++, broadcasta nästa `question`, starta timer. Annars broadcasta
   `finished`.

Host är **också spelare**: hostens UI visar frågorna och låter hosten svara som alla andra.

## Frågegenerering (återanvänder befintlig logik)

- Pool = `QUOTES` filtrerad på valda album.
- Välj `numQuestions` unika slumpade citat.
- Per citat: alternativ = `[rätt titel, 3 distraktorer ur poolens titlar]`, blandade;
  spara `correctIndex`. Skicka citattext + alternativ + correctIndex i `start` så att
  alla klienter är identiska.

## Accepterade begränsningar (dokumenterade, medvetna)

1. **Fusk möjligt:** all quiz-data ligger redan i klientens JS, så en tekniskt kunnig
   spelare kan läsa rätt svar. Ofrånkomligt i en ren statisk app; ok bland kompisar.
2. **Host-beroende:** om hosten stänger/laddar om dör matchen (inget tillstånd sparas).
3. **Ingen återanslutning** mitt i en match; join tillåts bara i lobbyn (join efter start
   → "matchen har redan börjat").
4. **Latens-orättvisa** på timern (<1 s) accepteras.
5. **PIN-krock** försumbar (slumpad 4-siffrig); regenereras vid behov.
6. Riktat mot **små sällskap** (~2–10). Presence klarar fler men UI:t optimeras inte för det.

## Teststrategi

- **Manuellt fler-flik-test:** host i en flik, 1–2 spelare i andra flikar; kör en hel match
  och verifiera: lobby-presence uppdateras, synkade frågor, korrekt poäng, prispall.
- **Ren enhet:** frågegenererings-funktionen är ren (pool → frågor) och kan verifieras
  fristående (rätt antal, unika, correctIndex pekar på rätt titel).
- **Regression:** verifiera att solo-spelen är oförändrade och att `supabase-js` INTE
  laddas förrän man går in i multiplayer.

## Icke-mål (YAGNI)

Persistens, återanslutning, konton/inloggning, chatt, avatarer, ljud, snabbhetsbonus,
fler än ~10 spelare, matchhistorik.

## Deployment

Push till GitHub `main` → auto-deploy Vercel + GitHub Pages. Supabase-URL + anon-nyckel
inline i `index.html`. Realtid funkar från båda hostarna.
