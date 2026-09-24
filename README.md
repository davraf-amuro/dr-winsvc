# dr-winsvc

Pacchetto di linee guida dr-* per i progetti Windows Service e Worker Service su .NET 10, con uno o più worker in background.

## 🧩 Cosa contiene

| File | Tipo | A cosa serve |
|------|------|--------------|
| `.github/instructions/windows-service.instructions.md` | Istruzione | Regole obbligatorie per i Windows Service .NET 10: un file per worker in `Workers/<Name>Worker.cs` che eredita `BackgroundService`, `AddWindowsService` in `Program.cs`, configurazione `IOptions<T>` dedicata per ogni worker. Vieta `Thread.Sleep`, loop senza controllo di `stoppingToken`, eccezioni non gestite in `ExecuteAsync`, ASP.NET Core, IRepository, AutoMapper e MediatR. |
| `.github/prompts/card-worker-service.prompt.md` | Prompt | Genera o aggiorna `docs/card-<nome_progetto>.md`, la scheda riassuntiva del servizio: worker e ciclo di elaborazione, dipendenze, database, servizi esterni, hosting, configurazione dei worker. Ricava i dati da `.csproj`, `appsettings*.json`, `Program.cs`, `Workers/` e `Workers/Options/`, senza inventare e senza scrivere valori sensibili. |

Punti chiave di `windows-service.instructions.md`:

| Tema | Cosa impone |
|------|-------------|
| Stack | .NET 10 Worker Service senza ASP.NET Core, `Microsoft.Extensions.Hosting.WindowsServices`, `Microsoft.Extensions.Hosting`, Serilog. EF Core 10 solo dopo averlo chiesto. |
| Struttura | `src/<project>/` con `Workers/`, `Infrastructure/Provider/`, `Properties/`, `Program.cs`; poi `test/` e `docs/`. |
| `Program.cs` | `Host.CreateApplicationBuilder`, `AddWindowsService` con `ServiceName` letto da `Service:Name`, un `AddHostedService<T>` per ogni worker. |
| Ciclo del worker | `while (!stoppingToken.IsCancellationRequested)` con attesa `await Task.Delay(TimeSpan.FromSeconds(IntervalSeconds), stoppingToken)`. |
| Errori | `catch` separati: `OperationCanceledException` chiude il ciclo, `Exception` viene loggata e il ciclo prosegue. |
| Codice | Primary constructors obbligatori, `CancellationToken` ultimo parametro nei metodi privati, logging strutturato con placeholder. |
| Configurazione | Sezione `Workers:<NomeJob>` in `appsettings.json` con `IntervalSeconds`, una classe options per ogni job. `appsettings.local.json` caricato in `Program.cs` e ignorato da git. |
| Commenti | `///` su ogni worker e metodo non banale, più commenti inline sulle operazioni rilevanti. |

## 🔗 Dipendenze e domini

- Dipende da: `dr-dotnet-backend` (installato in automatico se manca dal manifest: comportamento dell'installer, non ancora provato sul campo).
- Richiesto da: nessun pacchetto.
- Dominio del catalogo: nessun dominio lo elenca direttamente; il dominio `dotnet-backend` elenca solo `dr-dotnet-backend`. `appliesTo`: `dotnet`.
- Tipologia che lo suggerisce: `worker-service` — Windows Service / Worker (.NET 10), insieme a `dr-dotnet-backend`; `dr-efdb` è opzionale. Template `dotnet new worker`, guida di scaffolding `docs/scaffolding-windows-service.md` nel core.
- `-Update` non si propaga alle dipendenze: aggiornare solo questo pacchetto lascia `dr-dotnet-backend` com'è. `/dr-get-latest` aggiorna anche la dipendenza, che ha una propria voce nel manifest.
- La skill `/dr-audit-api` di `dr-dotnet-backend` usa questa istruzione nella fase di conformità quando trova `Workers/*.cs`.

File di altri pacchetti citati dall'istruzione. Non vengono installati in automatico:

| File citato | Pacchetto che lo porta |
|-------------|------------------------|
| `sensitive-data.instructions.md`, `code-organization.instructions.md` (Regola 6, commenti) | `dr-guidelines` (core) |

## 🚀 Come si installa

Di solito non serve installarlo a mano. Per una soluzione nuova segui la guida del core [Creare una soluzione da zero](https://github.com/davraf-amuro/dr-guidelines/blob/main/docs/guida-nuova-soluzione.md): `/dr-scaffold` installa da solo i pacchetti giusti. Il flusso completo non è ancora stato provato sul campo.

A mano. Conviene installare prima il core `dr-guidelines`, che porta `CLAUDE.md`, configurazione e skill; l'installer però non lo impone. Prerequisiti: PowerShell 7, git, `gh auth status` autenticato (l'installer si scarica con `gh api`; i repo sono Public dal 2026-09-21). L'installer clona il repo da `github.com` con `git clone --depth 1`: con i repo Public non servono credenziali; su un repo Private anche git deve poterlo leggere (`gh auth setup-git`).

Lancia i comandi dalla **root del repository host**: l'installer usa la cartella corrente come destinazione e non avvisa se sbagli cartella.

Via core, un solo installer:

```powershell
Set-Location <root-del-progetto-host>
& ([scriptblock]::Create((gh api repos/davraf-amuro/dr-guidelines/contents/dr-guidelines-install.ps1 -H "Accept: application/vnd.github.raw" | Out-String))) -Package dr-winsvc
```

Oppure con l'installer del pacchetto:

```powershell
& ([scriptblock]::Create((gh api repos/davraf-amuro/dr-winsvc/contents/dr-winsvc-install.ps1 -H "Accept: application/vnd.github.raw" | Out-String)))
```

Nota: la forma breve `irm https://raw.githubusercontent.com/... | iex` funziona solo a repo Public; oggi risponde 404.

Se `dr-dotnet-backend` manca dal manifest, l'installer lo installa prima di `dr-winsvc` e lo segnala con questa riga:

```text
Dipendenza mancante: dr-dotnet-backend -> installazione automatica
```

## 📦 Cosa finisce nel progetto host

| Percorso nel progetto host | Contenuto |
|----------------------------|-----------|
| `.github/instructions/windows-service.instructions.md` | Regole di architettura Windows Service |
| `.github/prompts/card-worker-service.prompt.md` | Prompt per la scheda del servizio |
| `.ai/dr-guidelines-packages.json` | Voce `dr-winsvc` con data (`installedAt`) e commit installato (`commit`) |

Nessuna modifica a `CLAUDE.md` né ai file di configurazione del core (`.editorconfig`, `.gitignore`, `.gitattributes`, `.claude/settings.json`, `.mcp.json`).

Non vengono copiati `README.md`, `LICENSE`, `.github/ISSUE_TEMPLATE/` e l'installer.

Senza `-Update` un file già presente resta com'è e l'installer stampa `[SKIP]`.

Se la dipendenza viene installata in automatico, arrivano anche i file di `dr-dotnet-backend`: vedi la sezione "Cosa finisce nel progetto host" del README di [dr-dotnet-backend](https://github.com/davraf-amuro/dr-dotnet-backend).

## 🔄 Aggiornare

Tutti i pacchetti del progetto: `/dr-get-latest`.

Solo questo pacchetto: stesso comando dell'installer del pacchetto, con `-Update` in coda:

```powershell
& ([scriptblock]::Create((gh api repos/davraf-amuro/dr-winsvc/contents/dr-winsvc-install.ps1 -H "Accept: application/vnd.github.raw" | Out-String))) -Update
```

Nota: `-Update` sovrascrive le copie locali. Si installa sempre l'ultimo `main` pushato su GitHub: le modifiche a questo repo non pushate su `main` non arrivano nei progetti host.

## 🐞 Segnalare un problema o una miglioria

Non correggere la copia nel progetto host: si perde al primo `-Update`.

Dal progetto host usa `/dr-segnala-miglioria <descrizione>` (su Copilot il prompt `.github/prompts/dr-segnala-miglioria.prompt.md` del core). La issue si apre in questo repo, dopo la tua conferma esplicita di titolo e corpo.

| Modello del repo | Quando usarlo |
|------------------|---------------|
| `.github/ISSUE_TEMPLATE/miglioria.md` | Richiesta evolutiva: regola nuova, precisazione, estensione del pacchetto |
| `.github/ISSUE_TEMPLATE/problema.md` | Malfunzionamento: regola sbagliata, ambigua o che porta l'agente fuori strada |

---

*Documento aggiornato: Settembre 2026 — Revisione v1.0 — 2026-09-16 — claude-opus-5*
