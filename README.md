# Gestione Contratti di Locazione

Applicazione web **in un unico file** (`index.html`) per gestire i contratti di locazione dello studio:

- **importazione automatica da cartella** (anche di rete) di PDF, DOCX, TXT e immagini, con **OCR in italiano** per i contratti scansionati;
- riconoscimento di **locatore e conduttore** (nome, codice fiscale, P.IVA; persone fisiche e società), **immobile** (indirizzo, categoria catastale, categoria fiscale: abitativo, non abitativo, strumentale, pertinenza, terreno…), **soggettività IVA**, **tipologia** (4+4, 3+2, transitorio, studenti, 6+6, 9+9, libero), **regime fiscale** (ordinario, cedolare secca 21%/10%, IVA imponibile/esente), **date** di stipula, inizio e scadenza, canone, deposito, estremi di registrazione;
- **maschera di revisione**: i dati rilevati sono evidenziati, si vede il testo estratto a fianco e si corregge a mano prima di creare il contratto (oppure si inserisce tutto manualmente);
- **scadenzario**: registrazione entro 30 giorni, imposta di registro delle annualità successive (con stima dell'importo), aggiornamento ISTAT, termine ultimo di disdetta, scadenza, comunicazioni RLI; esportazione CSV e calendario `.ics` (Outlook/Google);
- **rinnovi, proroghe, disdette e risoluzioni** con storico, ricalcolo delle nuove scadenze e annullamento dell'ultimo evento;
- **controlli di coerenza** (es. cedolare secca con soggetto IVA, aliquota 10% fuori dal canone concordato, CF/P.IVA non validi);
- **import/export** JSON (completo) e CSV (contratti, scadenze, anagrafiche), copia del database `.sqlite`;
- **registro errori** consultabile, filtrabile, copiabile e svuotabile, più un'autodiagnosi integrata (37 controlli).

Versione corrente: **1.2.0** (indicata nel titolo della pagina e in Impostazioni → Informazioni).

---

## Scelte tecniche

| Aspetto | Scelta | Motivazione |
|---|---|---|
| Linguaggio | **JavaScript (ES2020)** | È l'unico linguaggio eseguito nativamente da tutti i browser richiesti: niente compilazione, niente installazione. |
| Framework | **Nessuno (vanilla JS)** con un piccolo registro di moduli | Un framework come React o Vue richiederebbe una fase di build o un runtime pesante. Per 5 utenti e un solo file, i template con escape automatico e un registro di moduli con dipendenze esplicite bastano e restano leggibili. |
| Database | **SQLite** tramite [sql.js](https://sql.js.org) 1.10.3 (SQLite compilato in WebAssembly) | Si usa un vero file `.sqlite`, apribile con qualsiasi strumento SQLite. Lo schema e le migrazioni sono applicati automaticamente all'avvio. |
| Condivisione tra le 5 postazioni | File `contratti-locazione.sqlite` in una **cartella del server locale scelta dall'utente**, letto e scritto con la File System Access API | Non serve un server applicativo: prima di ogni salvataggio l'app ricarica il file se un collega l'ha modificato, controlla gli aggiornamenti ogni 15 secondi e ogni giorno salva una copia in `backup/` (ultime 30). |
| Lettura documenti | pdf.js 3.11.174, mammoth 1.6.0, tesseract.js 5.1.0 (lingua `ita`) | Sono caricati dal CDN solo quando si importa. **I documenti non lasciano mai il PC**: dalla rete arrivano solo le librerie e il modello OCR. |

### Compatibilità dei browser

| Funzione | Chrome / Edge | Firefox / Safari |
|---|---|---|
| Consultazione, inserimento, scadenzario, import/export | ✅ | ✅ |
| Import da cartella con OCR | ✅ (la cartella resta memorizzata) | ✅ (va riselezionata a ogni sessione) |
| **Archivio condiviso sul server** (scrittura diretta del `.sqlite`) | ✅ | ❌ si lavora su una copia locale, da caricare e riscaricare a mano |
| App **allegata al sito Giovannigor** (area applicativi) | apri/salva il file a mano (vedi sotto) | apri/salva il file a mano |

> **Per l'uso in ufficio (5 postazioni) si usa Chrome o Edge.** Firefox e Safari non permettono a una pagina web di scrivere su un file del disco.

### Limiti noti, da conoscere

- **Concorrenza**: SQLite viene caricato in memoria e il file viene riscritto intero a ogni salvataggio. Prima di scrivere l'app ricarica il file, quindi due colleghi che salvano a pochi secondi di distanza non si sovrascrivono. Due salvataggi nello stesso istante, invece, possono far vincere l'ultimo. Se un collega ha modificato lo stesso contratto mentre lo avevi aperto, l'app avvisa prima di sovrascrivere. Con 5 utenti il rischio è basso, ma conviene comunque fare un **backup periodico** (Import / Export → *Salva copia .sqlite*, oppure il backup del server).
- **Riconoscimento automatico**: è euristico. Funziona bene sui contratti con struttura classica ("Il Sig. … C.F. … di seguito denominato locatore", "LOCATORE: …"). Ogni documento passa comunque dalla revisione manuale prima di diventare un contratto.
- **OCR**: per default legge le prime 8 pagine di ogni PDF scansionato (valore modificabile). La prima volta scarica circa 15 MB tra motore e lingua italiana. Su PC poco potenti può richiedere 10–30 secondi a pagina.
- **Calcoli fiscali**: l'imposta di registro è una **stima** (2%, 1% per gli strumentali dei soggetti IVA, 0,5% per i terreni agricoli, base al 70% per il canone concordato). Gli importi definitivi vanno verificati.
- Serve la **connessione Internet** al primo caricamento, per scaricare sql.js e le librerie dal CDN jsDelivr.

---

## Requisiti

- Un browser recente: Chrome o Edge (consigliati), Firefox o Safari.
- Per l'archivio condiviso: una cartella del server locale raggiungibile da tutte le postazioni (es. unità di rete `Z:\`).
- Per lo sviluppo e il deploy: [Git](https://git-scm.com/) e, facoltativo, [Node.js](https://nodejs.org/) 18+ per la Vercel CLI.

**Non ci sono dipendenze da installare**: niente `npm install`, niente build.

## Avvio in locale

**Opzione 1 – doppio clic.** Apri `index.html` con Chrome o Edge. Di solito funziona anche da `file://`; se il browser blocca qualcosa, usa l'opzione 2.

**Opzione 2 – server locale** (consigliato, perché si comporta come la versione pubblicata):

```bash
python -m http.server 8080
```

Poi apri http://localhost:8080. In alternativa, con Node.js:

```bash
npx serve .
```

Al **primo avvio** l'app crea il database, applica le migrazioni e carica 5 contratti di esempio (codici `ES-001`…`ES-005`), che si eliminano da *Impostazioni → Elimina dati di esempio*.

## Configurazione dell'archivio condiviso (una volta per ufficio)

1. Sulla **prima postazione**, in Chrome o Edge: *Impostazioni → Elimina dati di esempio* (facoltativo), poi **Scegli la cartella del server…** (il pulsante è anche nel Riepilogo) e seleziona la cartella condivisa, ad esempio `Z:\Contratti`. Il browser chiede di consentire la modifica dei file della cartella: conferma. L'app crea `contratti-locazione.sqlite` con i dati attuali.
2. Sulle **altre postazioni**: **Scegli la cartella del server…** e seleziona la stessa cartella. L'app trova il file esistente e lo apre.
3. Nella sottocartella `backup` l'app salva automaticamente, alla prima modifica di ogni giorno, una copia `contratti-locazione_AAAA-MM-GG.sqlite`, conservando le ultime 30. Per ripristinare un backup: *Import / Export → Carica da file .sqlite*.
4. In alternativa puoi scegliere liberamente **cartella e nome del file**: *Impostazioni → Crea archivio con nome e percorso a scelta…* apre la finestra "Salva con nome" e crea lì l'archivio con i dati attuali, per esempio `Z:\Studio\Archivio contratti.sqlite`. Le altre postazioni lo collegano con *Apri un file .sqlite specifico…*. In questo caso l'app conosce solo il file e non la cartella, quindi **non fa i backup giornalieri**.
5. Su ogni postazione indica il **nome dell'utente** (Impostazioni → Utente). Viene registrato su modifiche, rinnovi e adempimenti completati.
6. A ogni riapertura del browser, Chrome potrebbe chiedere di riconfermare l'accesso alla cartella. In quel caso, in alto compare **"Ricollega …"**: un clic e si riparte. Con Chrome 122 o successivo si può scegliere "Consenti sempre" per non doverlo più ripetere.

Lo stato dell'archivio è sempre visibile in alto a destra: verde = condiviso, giallo = copia locale, rosso = da ricollegare (o, nel sito, modifiche da salvare).

## Uso come applicativo del sito Giovannigor

`index.html` si può allegare così com'è nell'area applicativi del sito: *Admin → Applicativi → carica file HTML* (è un file singolo da circa 230 KB, sotto il limite di 10 MB). Le librerie arrivano dal CDN, che la sandbox del sito consente.

Il sito però esegue gli applicativi in un **iframe sandbox senza `allow-same-origin`**, una scelta di sicurezza per evitare che un'app legga la sessione degli utenti. In quella sandbox il browser **non permette di scrivere nelle cartelle del server** e non conserva dati (niente IndexedDB). L'app lo riconosce da sola e passa alla **modalità memoria**:

1. *Apri archivio .sqlite…* e seleziona `contratti-locazione.sqlite` dalla cartella del server;
2. lavora normalmente: in alto compare "modifiche da salvare";
3. *Salva archivio*: il browser scarica il file nella cartella Download (oppure chiede dove salvarlo, se in Chrome/Edge è attiva l'opzione "Chiedi dove salvare ciascun file"). Il file va **rimesso nella cartella del server sostituendo il precedente**. Se chiudi la pagina con modifiche non salvate, il browser chiede conferma.

In questa modalità i colleghi non vedono le modifiche finché il file non viene rimesso sul server, e due persone che lavorano insieme possono sovrascriversi. Per il lavoro quotidiano in 5 conviene quindi aprire l'app **fuori dal sito**, dove salva direttamente nella cartella e sincronizza da sola: dall'indirizzo Vercel oppure dal file `index.html` copiato sul server. Sul sito può restare come accesso di consultazione, oppure come link a quella versione.

### Copie dell'archivio

*Import / Export → Salva copia .sqlite (scegli cartella e nome)…* apre la finestra "Salva con nome": scegli dove salvare la copia e con quale nome. Il nome proposto è `contratti-locazione_AAAA-MM-GG.sqlite`. Non si può sovrascrivere per errore il file dell'archivio in uso. Dove il browser non lo permette (sito in sandbox, Firefox, Safari) la copia viene scaricata nella cartella Download.

## Importazione dei contratti da cartella

1. *Importa da cartella → Seleziona cartella…* e scegli la cartella dei contratti (vengono incluse anche le sottocartelle).
2. L'app calcola l'impronta SHA-256 di ogni file: i file già elaborati vengono saltati, quindi *Ricontrolla cartella* elabora solo i nuovi.
3. Per ogni documento: **Verifica e importa**. Si apre la maschera con i campi rilevati (etichetta "rilevato", passando il mouse si vede il testo di origine) e il testo estratto a fianco, con ricerca. Correggi i campi e conferma.
4. Per i documenti non leggibili (es. file `.doc` vecchi): **Inserisci a mano**, con il link all'originale.

## Struttura del progetto

```
contratti-di-locazione/
├── index.html      ← l'intera applicazione (HTML + CSS + JavaScript inline)
├── vercel.json     ← configurazione del deploy statico su Vercel (header di sicurezza)
├── README.md
└── .gitignore      ← esclude database, esportazioni e documenti (dati personali)
```

### Moduli in `index.html`

Ogni modulo è registrato con `Modules.define(nome, [dipendenze], factory)` e riceve **solo** le dipendenze dichiarate. Il registro risolve l'ordine di inizializzazione e segnala le dipendenze circolari.

| Modulo | Responsabilità |
|---|---|
| `Config` | Versione, URL delle librerie, costanti |
| `Log` | Registro errori persistente (localStorage) e intercettazione degli errori globali |
| `Util` | Date ISO, numeri in formato italiano, CSV, SHA-256, download, appunti |
| `Ambiente` | Riconosce sandbox (sito), disponibilità di file e cartelle |
| `Store` | Chiave/valore su IndexedDB: copia di lavoro del DB, handle di file e cartelle (in memoria se IndexedDB manca) |
| `Prefs` | Preferenze della singola postazione (utente, OCR, tema) |
| `Migrations` | Schema versionato e migrazioni (tabella `schema_version`) |
| `Db` | sql.js, query, transazioni serializzate, archivio locale / cartella del server / memoria, backup giornalieri, sincronizzazione |
| `Dominio` | Tipologie, regimi, categorie, calcolo delle scadenze, stima del registro, controlli di coerenza |
| `Scadenzario` | Generazione e gestione delle scadenze, esportazione `.ics` |
| `Repo` | Accesso ai dati di contratti, parti, immobili e documenti |
| `Rinnovi` | Rinnovi, proroghe, disdette, risoluzioni e annullamento |
| `Seed` | Dati di esempio |
| `Extract` | Estrazione del testo (pdf.js, OCR Tesseract, mammoth) |
| `Parser` | Riconoscimento euristico dei campi, validazione di CF e P.IVA |
| `FolderImport` | Selezione e scansione della cartella, coda di elaborazione |
| `ImportExport` | JSON (sostituisci/unisci) e CSV |
| `UI`, `Archivio`, `ContrattoForm`, `ContrattoDettaglio`, `ScadenzeTable`, `ScadenzaForm` | Componenti di interfaccia (`Archivio`: scelta cartella, ricollega, apri/salva file) |
| `*View` | Le sezioni dell'app (Riepilogo, Contratti, Scadenzario, Importa, Import/Export, Impostazioni, Registro errori) |
| `SelfTest` | Autodiagnosi (Registro errori → *Esegui autodiagnosi*) |
| `App` | Avvio, navigazione, barra di stato, controllo periodico delle modifiche |

### Schema del database

Il codice completo è nel modulo `Migrations` e in *Impostazioni → Informazioni → Migrazioni*.

| Versione | Contenuto |
|---|---|
| 1 | `soggetti`, `immobili`, `contratti`, `contratto_parti` (N locatori / N conduttori con quota), `scadenze`, `rinnovi` |
| 2 | `documenti` (file importati, impronta univoca, testo estratto, dati rilevati, stato), `impostazioni` |
| 3 | colonne `modificato_da`, `completata_da`, `registrato_da` per tracciare l'utente |

**Aggiungere una migrazione.** Si aggiunge un oggetto `{ version: 4, name, sql }` in coda all'array `MIGRATIONS`, senza mai modificare quelle già rilasciate, e si incrementa `APP_VERSION`. Al primo avvio ogni postazione aggiorna da sola il file condiviso. Se un database è più recente dell'app, viene mostrato un avviso.

---

## Repository GitHub: primo push

Il repository esiste già (https://github.com/Giova2024/contratti-di-locazione). Dalla cartella del progetto:

```bash
git init -b main
git remote add origin https://github.com/Giova2024/contratti-di-locazione.git
git add index.html README.md vercel.json .gitignore
git commit -m "Gestione contratti di locazione v1.0.0"
git push -u origin main
```

Se la cartella è già stata clonata da GitHub, `git init` e `git remote add` non servono: bastano gli ultimi tre comandi.

A ogni modifica successiva:

```bash
git add -A
git commit -m "Descrizione della modifica"
git push
```

## Deploy su Vercel

L'app è un sito statico: Vercel pubblica `index.html` così com'è, senza build.

### Dal sito di Vercel (consigliato)

1. Accedi a https://vercel.com con l'account GitHub.
2. **Add New… → Project** e importa il repository `Giova2024/contratti-di-locazione`.
3. Nella configurazione:
   - **Framework Preset**: `Other`
   - **Root Directory**: `./`
   - **Build Command**: lasciare vuoto (oppure "Override" con campo vuoto)
   - **Output Directory**: `.` (punto)
   - **Install Command**: lasciare vuoto
4. **Deploy**. Dopo circa 30 secondi l'app è online su `https://contratti-di-locazione.vercel.app` (o un indirizzo simile).
5. Da quel momento ogni `git push` su `main` ripubblica in automatico. I push sugli altri branch generano anteprime separate.

### Dalla riga di comando (alternativa)

```bash
npm install -g vercel
vercel login
vercel
vercel --prod
```

`vercel` crea un'anteprima; `vercel --prod` pubblica in produzione.

### Variabili d'ambiente

**Nessuna.** L'app non ha un backend: non ci sono chiavi, password né stringhe di connessione da configurare. I dati non passano da Vercel, che ospita solo il file HTML. Restano nel file `.sqlite` sul server dello studio e nel browser di ogni postazione.

### Note su sicurezza e privacy

- L'indirizzo Vercel è pubblico, ma chi lo apre vede un'app **vuota**: i dati sono solo sul server dello studio. Per limitare anche l'accesso all'app si può attivare *Vercel Authentication / Password Protection* (Settings → Deployment Protection), oppure servire `index.html` direttamente dal server locale.
- Il file `vercel.json` imposta gli header `X-Content-Type-Options`, `X-Frame-Options: DENY`, `Referrer-Policy: no-referrer` e `Permissions-Policy`.
- Non caricare mai su GitHub database, esportazioni o contratti: il `.gitignore` li esclude già.
