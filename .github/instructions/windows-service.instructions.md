---
applyTo: "**"
---

# Windows Service Design Rules (AI Agent)

Scopo: regole obbligatorie per progetti Windows Service .NET 10. Segui sempre. Testo ottimizzato per token.

## Stack (obbligatorio)
- .NET 10 Worker Service (no ASP.NET Core)
- `Microsoft.Extensions.Hosting.WindowsServices` (UseWindowsService)
- `Microsoft.Extensions.Hosting` (BackgroundService / IHostedService)
- Serilog — fonte unica `serilog-logging.instructions.md` (pacchetto `dr-dotnet-backend`): pacchetti, sezione `Serilog` di `appsettings.json`, enricher, soggetto del log. Su `HostApplicationBuilder` si registra con `builder.Services.AddSerilog(...)`; `UseSerilog` sulla proprietà `Host` esiste solo su `WebApplicationBuilder`: qui non compila
- Entity Framework Core 10: chiedere prima di aggiungere il pacchetto
- Aggiungi sempre il file appsettings.local.json, aggiungi la chiamata in Program.cs, e ignora il file in .gitignore
- Dati sensibili: segui sempre `sensitive-data.instructions.md`

## Vietato
- `Thread.Sleep` → usare `await Task.Delay`
- Loop infiniti senza `stoppingToken.IsCancellationRequested`
- Exception non gestite in `ExecuteAsync` (crash silenzioso del servizio)
- ASP.NET Core Minimal API / MVC Controllers
- IRepository pattern, AutoMapper, MediatR

## Struttura progetto
- src/<project>/
  - Workers/
  - Infrastructure/Provider/{Entities,Filters,*DbContext.cs,*Provider.cs}
  - Infrastructure/Logging/ (richiesta da `serilog-logging.instructions.md`)
  - Properties/
  - Program.cs
- test/
- docs/

## Regole core (sempre)
1) Ogni worker in file separato: `Workers/<Name>Worker.cs`
2) Eredita `BackgroundService`, override di `ExecuteAsync(CancellationToken stoppingToken)`
3) Primary constructors obbligatori
4) CancellationToken sempre ultimo parametro nei metodi privati
5) Loop: `while (!stoppingToken.IsCancellationRequested)`
6) Timer: `await Task.Delay(TimeSpan.FromSeconds(options.Value.IntervalSeconds), stoppingToken)`
7) Logging strutturato con placeholder (no string interpolation nei log)
8) `Program.cs`: `Host.CreateApplicationBuilder` + `AddWindowsService` + `AddHostedService<T>`
9) `ServiceName` in `AddWindowsService` da `appsettings.json` (`Service:Name`)
10) Graceful shutdown: `ExecuteAsync` con try/catch su `OperationCanceledException`
11) Multi-worker: ogni job ha `IOptions<T>` dedicato con `IntervalSeconds`, registrati separatamente
12) Configurazione strongly-typed per worker in `appsettings.json` sezione `Workers:<NomeJob>`
13) Elementi indipendenti nello stesso ciclo (`concurrency.instructions.md` del core): preferisci l'esecuzione parallela con limite da configurazione (`fan-out-parallelism.instructions.md` o `producer-consumer-queue.instructions.md` di `dr-dotnet-backend`); se resti in serie, scrivi nel codice `// Resto in serie: <motivo>`

## Pattern richiesti (copiabili)

### Program.cs (singolo worker)
```csharp
var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddWindowsService(options =>
    options.ServiceName = builder.Configuration["Service:Name"]!);

builder.Services.Configure<SyncOrdersOptions>(builder.Configuration.GetSection("Workers:SyncOrders"));
builder.Services.AddHostedService<SyncOrdersWorker>();

// Serilog: registrazione con AddSerilog su IServiceCollection, come in serilog-logging.instructions.md ("Program.cs — registrazione")

var host = builder.Build();
host.Run();
```

### Program.cs (multi-worker)
```csharp
var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddWindowsService(options =>
    options.ServiceName = builder.Configuration["Service:Name"]!);

builder.Services.Configure<SyncOrdersOptions>(builder.Configuration.GetSection("Workers:SyncOrders"));
builder.Services.Configure<CleanupOptions>(builder.Configuration.GetSection("Workers:Cleanup"));

builder.Services.AddHostedService<SyncOrdersWorker>();
builder.Services.AddHostedService<CleanupWorker>();

// Serilog: registrazione con AddSerilog su IServiceCollection, come in serilog-logging.instructions.md ("Program.cs — registrazione")

var host = builder.Build();
host.Run();
```

### BackgroundService con timer
```csharp
public class SyncOrdersWorker(ILogger<SyncOrdersWorker> logger, IOptions<SyncOrdersOptions> options) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        logger.LogInformation("Worker {Worker} started", nameof(SyncOrdersWorker));

        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await DoWorkAsync(stoppingToken);
            }
            catch (OperationCanceledException)
            {
                break;
            }
            catch (Exception ex)
            {
                logger.LogError(ex, "Error in worker {Worker}", nameof(SyncOrdersWorker));
            }

            await Task.Delay(TimeSpan.FromSeconds(options.Value.IntervalSeconds), stoppingToken);
        }

        logger.LogInformation("Worker {Worker} stopped", nameof(SyncOrdersWorker));
    }

    // Campo scelto dall'utente come soggetto del log (serilog-logging.instructions.md)
    private const string OrderSubjectField = "OrderCode";

    private async Task DoWorkAsync(CancellationToken ct)
    {
        logger.LogInformation("Executing job {Worker}", nameof(SyncOrdersWorker));
        // Lettura degli elementi da elaborare: provider del progetto (es. dr-efdb)
        IReadOnlyList<Order> pending = [];

        // Resto in serie: un ordine alla volta perché <motivo: ordine di elaborazione, DbContext condiviso, rate limit del servizio>
        foreach (var order in pending)
        {
            // Ogni log dell'elemento, anche dai componenti chiamati, porta S:OrderCode=<codice>
            using var _ = LogSubject.Begin(logger, OrderSubjectField, order.Code);
            try
            {
                logger.LogInformation("Sync started");
                // logica business...
                await Task.CompletedTask;
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                // Dentro lo scope: anche l'errore dell'elemento si isola col filtro del soggetto
                logger.LogError(ex, "Sync failed");
            }
        }
    }
}
```

### Configurazione strongly-typed per worker
```csharp
public class SyncOrdersOptions
{
    public int IntervalSeconds { get; set; } = 60;
}
```

### appsettings.json (struttura)
```json
{
  "Service": {
    "Name": "MyWindowsService"
  },
  "Workers": {
    "SyncOrders": {
      "IntervalSeconds": 60
    },
    "Cleanup": {
      "IntervalSeconds": 3600
    }
  }
}
```
La sezione `Serilog` di `appsettings.json` non si ricopia qui: copiala da `serilog-logging.instructions.md` (fonte unica). Differenza del worker: l'override `Microsoft.AspNetCore` non serve (niente ASP.NET Core); resta `Microsoft.EntityFrameworkCore` se il worker usa EF. Il file di log è uno solo (`logs/app-.log`), non uno per worker: i worker si distinguono dal nome riportato nei messaggi (`{Worker}`).

## Errori comuni (rapidi)
- `Thread.Sleep` => blocca shutdown SCM, usare `Task.Delay` con `stoppingToken`
- `ExecuteAsync` senza try/catch su `Exception` => eccezione silenziosa, servizio si ferma
- `Task.Delay` senza `stoppingToken` => delay non interrompibile allo stop
- `AddWindowsService` non chiamato => non funziona come servizio SCM (solo console)
- `OperationCanceledException` non catturata separatamente => log errato come errore
- `IOptions<T>` non registrato => `Cannot resolve service` a runtime

## ✅ Checklist Post-Generazione
- [ ] Ogni worker in `Workers/<Name>Worker.cs`, eredita `BackgroundService`
- [ ] Primary constructors usati in tutti i worker
- [ ] Loop con `while (!stoppingToken.IsCancellationRequested)`
- [ ] `Task.Delay` con `stoppingToken` come secondo argomento
- [ ] try/catch per `OperationCanceledException` e `Exception` separati
- [ ] `AddWindowsService` con `ServiceName` da configurazione
- [ ] Ogni worker ha `IOptions<T>` dedicato con `IntervalSeconds`
- [ ] `appsettings.json` con sezione `Workers:<NomeJob>` per ogni worker
- [ ] Commenti `///` su ogni worker e metodo non banale + inline su operazioni rilevanti (code-organization Regola 6)
- [ ] Logging strutturato con placeholder, nessuna string interpolation
- [ ] Serilog conforme a `serilog-logging.instructions.md`: sezione `Serilog` completa in `appsettings.json`, enricher in `Program.cs`
- [ ] Soggetto del log indicato dall'utente: `LogSubject.Begin` per ogni elemento elaborato
- [ ] `appsettings.local.json` aggiunto e ignorato in `.gitignore`
- [ ] `appsettings.json` contiene solo valori fake/placeholder per dati sensibili, mai credenziali reali

## 🎯 Criteri di successo (verificare prima di iniziare)

Prima di iniziare, chiediti:
- [ ] So esattamente quali worker creerò/modificherò?
- [ ] Ho letto le istruzioni modulari pertinenti al task?
- [ ] Ho verificato che il worker non esista già in `Workers/`?

Se una risposta è NO → chiedi chiarimenti all'utente prima di procedere.

*Template v1.5 - .NET 10 - Token-optimized for AI agents* - Last Update 2026-10-20 — claude-opus-5-5 — audit suite: registrazione Serilog e sezione `appsettings` non più ricopiate (rimando alla fonte unica), `Infrastructure/Logging/` nella struttura, regola 13 sulla dichiarazione "Resto in serie". v1.4: Serilog: enricher in `Program.cs` e fonte unica `serilog-logging.instructions.md` (`dr-dotnet-backend`); soggetto del log nel worker, errore dell'elemento loggato dentro lo scope. v1.3: Serilog registrato con `builder.Services.AddSerilog` (davraf-amuro/dr-winsvc#1)
