# devdb

> Database remoti, piccoli e condivisi per lo sviluppo, gestiti da riga di comando.

**Stato:** idea / proposta. Al momento non esiste ancora nessuna implementazione:
questo documento descrive come dovrebbe funzionare.

## L'idea

Durante lo sviluppo ogni sviluppatore di solito si tira su un database in locale
(Docker, installazione diretta, copie dei dati passate a mano...). Il risultato sono dati diversi su
ogni macchina, configurazioni da ripetere e difficoltà nel lavorare sugli stessi
dati con i colleghi.

`devdb` è uno strumento da riga di comando che usa un **fornitore** cloud per creare al volo **piccoli
database remoti dedicati allo sviluppo**. Ogni database viene creato con un
comando e restituisce subito una **stringa di connessione completa** (il cosiddetto DSN:
indirizzo, utente, password e nome del database in un'unica riga), pronta da
incollare nel proprio file `.env`, nel `docker-compose.yml` o nella
configurazione dell'editor.

Dato che il database è remoto, la stessa stringa di connessione può essere condivisa: più persone
possono lavorare sullo **stesso database di sviluppo**, senza copie del database da
passarsi a mano e senza doversi allineare ogni volta.

## Fornitore

Lo strumento non è legato a un singolo servizio: si configura con un fornitore (provider) che si
occupa di ospitare i database. Il primo fornitore previsto è
**[pizzacloud.com](https://pizzacloud.com)**.

```shell
devdb provider set pizzacloud.com
devdb login   # accesso al proprio account
```

## Concetti

`devdb` ragiona con tre entità:

| Entità    | Cosa rappresenta                                                       |
|-----------|------------------------------------------------------------------------|
| **app**   | Un concetto logico: il progetto o il servizio su cui stai lavorando.   |
| **env**   | Un ambiente (es. `sviluppo`, `collaudo`, `nuova-funzione`, `mario`).   |
| **db**    | Un database reale ospitato dal fornitore, con la sua stringa di connessione. |

Il punto chiave è che **le tre entità si creano in modo indipendente** e poi si
**agganciano** tra loro. La gerarchia non è fissata a monte:

- puoi creare un db "sciolto" e solo dopo decidere a quale ambiente appartiene;
- puoi agganciare lo stesso db a più ambienti (o a più app) per condividerlo;
- puoi riorganizzare app e ambienti senza ricreare i database.

```
app01 ──┬── sviluppo ───────┬── db: app01-principale
        │                   └── db: app01-cache
        └── nuova-funzione ─┬── db: app01-principale  (stesso db condiviso con sviluppo)
                            └── db: nuova-funzione-prova
```

## Esempi d'uso

> I comandi sono indicativi: servono a descrivere il flusso, non sono ancora definitivi.

### Creare le entità separatamente

```shell
devdb app create app01
devdb env create sviluppo
devdb db create app01-principale --engine mysql
```

Alla creazione del db lo strumento mostra subito la stringa di connessione
completa da copiare:

```text
✔ Database "app01-principale" creato su pizzacloud.com

  motore      mysql
  server      francescobianco.devdb.pizzacloud.com
  database    app01_principale

  connessione mysql://root:********@francescobianco.devdb.pizzacloud.com:3306/app01_principale
```

### Agganciare le entità

```shell
devdb env attach sviluppo --app app01
devdb db attach app01-principale --env sviluppo
```

### Consultare e recuperare le stringhe di connessione

```shell
devdb ls                               # albero di app, ambienti e db
devdb db dsn app01-principale          # mostra la stringa di connessione di un db
devdb env dsn sviluppo --app app01     # mostra quelle di tutti i db dell'ambiente
```

### Usarlo nel proprio progetto

La stringa di connessione si incolla direttamente nella configurazione dell'ambiente di sviluppo,
ad esempio in un `docker-compose.yml`:

```yaml
services:
  app:
    build: .
    environment:
      - MYSQL_HOST=francescobianco.devdb.pizzacloud.com
      - MYSQL_USER=root
      - MYSQL_PASSWORD=${PIZZACLOUD_DEVDB_API_KEY}
      - MYSQL_DATABASE=app01_principale
```

## Clonare e copiare i database

Oltre a creare database vuoti, `devdb` permette di lavorare sui dati già
esistenti.

### Clonare un database

Crea un nuovo database identico a uno esistente, con la stessa struttura e gli
stessi dati. È utile, ad esempio, per provare una modifica senza toccare il
database che usano anche gli altri:

```shell
devdb db clone app01-principale nuova-funzione-prova
```

Il clone è un database a tutti gli effetti: ha la sua stringa di connessione e
si può agganciare a qualsiasi ambiente.

### Spostare il contenuto da un database a un altro

Copia tutto il contenuto di un database dentro un altro database già esistente,
sostituendone i dati. È utile, ad esempio, per riportare in `sviluppo` i dati
preparati in un altro ambiente, o per riallineare un database alla versione di
un collega:

```shell
devdb db copy nuova-funzione-prova app01-principale
```

Dato che i dati del database di destinazione vengono sostituiti, lo strumento
chiede sempre una conferma prima di procedere.

## Database condivisi

Un database creato con `devdb` può essere condiviso con altri utenti, così che
un gruppo di lavoro usi gli stessi dati di sviluppo:

```shell
devdb db share app01-principale --user collega@example.com
```

Il collega, dopo aver fatto l'accesso allo stesso fornitore, vede il db nel
proprio terminale e può recuperarne la stringa di connessione.

## Obiettivi

- **Nessuna installazione locale**: niente database da installare sul proprio computer.
- **Connessione subito pronta**: ogni db creato è immediatamente utilizzabile.
- **Struttura flessibile**: app, ambienti e db sono indipendenti e componibili.
- **Collaborazione**: database di sviluppo condivisi tra più persone.
- **Dati sempre a portata di mano**: clonare un database o spostarne il contenuto in un altro richiede un solo comando.
- **Fornitori intercambiabili**: si parte da pizzacloud.com, ma il modello è aperto ad altri fornitori.

## Cosa non vuole essere

`devdb` non vuole essere una soluzione per database di produzione: i database sono
pensati per essere piccoli, economici e usa-e-getta, a supporto dello sviluppo.

## Licenza

Vedi [LICENSE](LICENSE).
