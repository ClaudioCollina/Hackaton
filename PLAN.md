## Piano: StrikeMate dai wireframe a una PWA con Web API .NET

Le pagine HTML esistenti diventano il frontend. Un progetto ASP.NET Core 8 ospita le API REST, il database SQLite (EF Core) e SignalR per gli aggiornamenti in tempo reale. Al posto dei dati scritti nell'HTML e dei `location.href` fissi ci saranno chiamate `fetch` alle API. Integrazioni QubicaAMF, pagamenti e login Google/Apple sono simulati. Perimetro: solo il flusso **Play Now**.

### Fase 1: Struttura del progetto
1. Creare `src/StrikeMate.Api` (ASP.NET Core 8 Web API) con i pacchetti:
   - `Microsoft.EntityFrameworkCore.Sqlite`
   - SignalR
   - Swagger
   - `QRCoder`
   - JWT Bearer
2. In `Program.cs` attivare `UseDefaultFiles` e `UseStaticFiles`. Copiare il contenuto di `wireframes/` in `wwwroot/` e lasciare intatti gli originali. Frontend e API hanno la stessa origine, quindi non serve CORS.

### Fase 2: Modello dati e seed (dipende dalla 1)
3. Entità:
   - `User`: nome, avatar, livello, reputazione, ranking.
   - `Center`: nome, indirizzo, prezzo a persona, attesa massima.
   - `Lane` (`CenterId`).
   - `Game`: `CenterId`, tipo (Bowling/Pool/Cornhole), livello, numero massimo di giocatori (2–6), `StartAt`, note (max 100 caratteri), prezzo, `LaneId` facoltativo, `Status` (Open, Ready, CheckedIn, Completed, Cancelled, Expired).
   - `GameParticipant`.
4. `AppDbContext` con SQLite: il database si crea all'avvio con `EnsureCreated`, senza migrazioni. Seed iniziale: il centro "QubicaAMF Bologna" con le sue corsie, 4–5 utenti demo e 2–3 partite aperte.

### Fase 3: API (dipende dalla 2)
5. Login demo: `POST /api/auth/login` riceve provider ed email e restituisce un JWT per un utente esistente o nuovo. Il token si salva in `localStorage`.
6. Endpoint:
   - `GET /api/me`
   - `GET /api/centers/{id}`: persone presenti, corsie libere, partite aperte
   - `GET /api/centers/{id}/games`
   - `POST /api/games` (pagina 03)
   - `GET /api/games/{id}` (pagine 08 e 06)
   - `PUT /api/games/{id}` e `DELETE /api/games/{id}`, solo per chi ha creato la partita (pagina 02B)
   - `POST /api/games/{id}/join`: in una transazione; risponde 409 se la partita è piena o l'utente è già dentro
   - `GET /api/games/{id}/qr`: QR in SVG (pagina 07)
7. `ILaneService` / `MockLaneService` assegna una corsia libera quando la partita è al completo e la porta in stato Ready. In futuro si potrà sostituire con Conqueror.

### Fase 4: Tempo reale (in parallelo con la 3)
8. Hub SignalR `/hubs/center` con gruppi `center-{id}` e `user-{id}`. Eventi:
   - `GameCreated`, `GameUpdated`, `GameRemoved`
   - `PlayerJoined`
   - `GameReady`
   - `CenterStatsChanged`
9. "Persone presenti" = valore di partenza nel seed + connessioni SignalR attive.

### Fase 5: Frontend (dipende dalle 3 e 4)
10. Moduli condivisi:
    - `wwwroot/js/api.js`: wrapper di `fetch` che aggiunge il JWT e rimanda al login in caso di 401.
    - `wwwroot/js/realtime.js`: client SignalR.
11. Uno script per ogni pagina:
    - le card hardcoded diventano elementi `<template>` riempiti con i dati delle API;
    - la navigazione passa l'id della partita nell'URL (`08-join.html?id=…`);
    - la pagina 03 invia un `POST` con i valori di stepper, radio e note.
12. La pagina 05 diventa un banner in-app mostrato quando arriva `PlayerJoined`. Percorso: banner → 06 → 07.

### Fase 6: PWA (in parallelo con la 5)
13. `manifest.webmanifest` con icone e `display: standalone`.
14. `sw.js`: mette in cache solo i file statici; le chiamate `/api` passano sempre dalla rete.

### File coinvolti
- `wireframes/`: sorgente da copiare in `wwwroot/`. Vanno modificati i `onclick="location.href=…"` di `01-login.html`, `02-community.html`, `02B-community.html`, `03-create.html` e `08-join.html`.
- `wireframes/03-create.html`: mantenere ed estendere gli script inline (stepper, contatore caratteri) aggiungendo l'invio del `POST`.
- `index.html`: aggiornare il redirect al nuovo percorso.
- Nuovi file in `src/StrikeMate.Api/`:
  - `Program.cs`
  - `Data/AppDbContext.cs`, `Data/Seed.cs`
  - `Entities/*`
  - `Endpoints/*`
  - `Hubs/CenterHub.cs`
  - `Services/MockLaneService.cs`

### Verifica
1. `dotnet build` e `dotnet run`, poi provare gli endpoint da Swagger (`/swagger`).
2. Due browser (uno normale e uno in incognito) con due utenti demo diversi:
   - A crea una partita e B la vede comparire senza ricaricare;
   - B entra nella partita e ad A compare il banner 05;
   - dal banner si va alla 06, poi alla 07 con corsia e QR reali.
3. Caso limite: un terzo utente che entra in una partita già piena riceve 409.
4. Lighthouse (Chrome DevTools): la PWA risulta installabile.

### Scelte e perimetro
- La risorsa si chiama `Center` (non `Venue`) in tutte le API, le entità e gli eventi SignalR.
- Frontend in vanilla JS, senza framework e senza build.
- Fuori perimetro: profilo 04 (resta statico), chat, Challenge, recensioni, Web Push reali, OAuth reali, pagamenti, integrazione con Conqueror e scoring.
