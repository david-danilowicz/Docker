# Retro Console Hub

**Retro Console Hub** è un'applicazione web in **Python (Flask)** e **Docker** che permette di sfogliare una libreria di giochi retro (**GBA** e **SNES**) dal browser e avviare ogni partita dentro un container isolato con **RetroArch** e **noVNC**.

Ogni sessione di gioco gira in un ambiente separato, con salvataggi persistenti su disco e chiusura automatica del container quando si esce dal gioco.

Annotato problema di salvataggi, se 2 utenti giocano allo stesso gioco uno dei salvataggi viene sovrascritto con quello dell'altro

## Come funziona

1. **Libreria e orchestrazione (`app.py` + `index.html`)**
   - `app.py` legge le ROM dalle cartelle `GBA` e `SNES`, abbina eventuali copertine in PNG e le mostra nella dashboard web (`index.html`), dove puoi filtrare per console, cercare un gioco e vedere le istanze attive.
   - All'avvio pulisce eventuali container `retroarch-gioco` rimasti appesi e, quando lanci un gioco, assegna una porta libera sull'host e un nome univoco al container (`<titolo>-<id>`).

2. **Ambiente di gioco isolato (`supervisord.conf` + `play.html`)**
   - Dentro ogni container, **Supervisor** avvia in sequenza:
     - **Xvfb**: schermo virtuale (`1280x720x24` sul display `:0`).
     - **Fluxbox**: window manager leggero per gestire finestre e fullscreen.
     - **x11vnc**: espone il display virtuale su VNC (porta `5900`).
     - **noVNC**: traduce il flusso VNC via web (porta `6080`) per mostrarlo nel browser.
     - **RetroArch**: lancia il core adatto (`mgba` per GBA, `snes9x` per SNES) con la ROM scelta, salva progressi e save state in `/saves` e mappa i comandi da tastiera.
   - La pagina `/play` (`play.html`) ospita la sessione di gioco, offre il pulsante per lo schermo intero, mostra la legenda dei tasti e ferma il container quando chiudi la finestra.

## Guida per l'avvio (anche per il docente)

### 1. Clona il repository e prepara le cartelle dati
Dopo aver clonato il progetto da GitHub ed essere entrato nella cartella del repository:

```bash
git clone <URL-DEL-REPOSITORY>
cd <cartella-progetto>
```

Assicurati che **Docker** sia avviato e prepara le cartelle locali (escluse da Git tramite `.gitignore`):
- `roms/GBA` e `roms/SNES` con i file ROM (`.gba`, `.sfc`, `.smc`) per testare il catalogo.
- `saves/` per i salvataggi persistenti su disco.
- `covers/GBA` e `covers/SNES` (opzionale) per le copertine `.png` con lo stesso nome del file ROM.

### 2. Configura `HOST_PATH` per la portabilità
`app.py` usa `HOST_PATH` per montare `roms` e `saves` nei container di gioco avviati dinamicamente. Per evitare percorsi fissi legati a una singola postazione, configura `HOST_PATH` in `app.py` in modo che legga la variabile d'ambiente con fallback automatico alla cartella corrente:

```python
HOST_PATH = os.environ.get("HOST_PATH", os.getcwd()).replace("\\", "/")
```

Se necessario, puoi anche impostare `HOST_PATH` esplicitamente prima dell'avvio (es. su Linux/macOS `export HOST_PATH=/percorso/progetto` oppure su Windows PowerShell `$env:HOST_PATH="C:\percorso\progetto"`).

### 3. Crea l'immagine Docker del gioco
Dalla cartella che contiene il `Dockerfile` e `supervisord.conf`:

```bash
docker build -t retroarch-gioco .
```

### 4. Installa le dipendenze e avvia l'applicazione
Installa i pacchetti Python richiesti e avvia il server Flask:

```bash
pip install flask docker
python app.py
```

### 5. Apri il progetto nel browser e verifica il funzionamento
Vai su:

```text
http://localhost:5000
```

Durante il test puoi verificare:
- Il caricamento della libreria da `/api/giochi`, la ricerca testuale e il filtro per console (`GBA` / `SNES`).
- L'avvio di un gioco su una porta dinamica tramite `/play` con visualizzazione noVNC, schermo intero e legenda comandi.
- La persistenza dei salvataggi in `saves/` e la rimozione automatica del container alla chiusura della scheda di gioco.

## Tasti principali

- **Movimento**: Frecce direzionali (`↑` `↓` `←` `→`)
- **Start / Select**: `Enter` / `Shift Destro`
- **Pulsanti B / A**: `Z` (verifica il tasto `Y`/`Z` su tastiera QWERTZ) / `X`
- **Pulsanti Y / X (SNES)**: `A` / `S`
- **Dorsali L / R**: `Q` / `E`
- **Menu RetroArch**: `F1`
- **Salva / Carica stato**: `F2` / `F4`
