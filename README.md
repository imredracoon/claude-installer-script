# claude-installer-script

Installa **Claude Desktop** su **Fedora** e su **Arch Linux** (e derivate: EndeavourOS, Manjaro, CachyOS…), solo x86_64, partendo dalla build Linux ufficiale di Anthropic. Poi lo tiene aggiornato da solo con un timer di systemd.

> Anthropic pubblica Claude Desktop per Linux solo come pacchetto `.deb` per Ubuntu e Debian. Su Fedora e Arch l'app non è supportata ufficialmente: funziona, ma per eventuali problemi il supporto Anthropic non aiuta.

## Come funziona

1. **`build.sh`** scarica la build ufficiale e la trasforma in un pacchetto nativo della tua distro: un RPM su Fedora, un pacchetto pacman su Arch. La distro viene riconosciuta da sola (`dnf` o `pacman`). Lo script:
   - scarica solo da `downloads.claude.ai`, il repository ufficiale di Anthropic;
   - verifica la firma dell'indice con la chiave di Anthropic incorporata nello script (fingerprint `31DDDE24DDFAB679F42D7BD2BAA929FF1A7ECACE`, la stessa pubblicata nella [documentazione ufficiale](https://code.claude.com/docs/en/desktop-linux));
   - controlla lo SHA256 del `.deb` scaricato;
   - reimpacchetta l'app così com'è, senza modificarla.
2. **`claude-desktop-autoupdate`** confronta la versione installata con quella pubblicata da Anthropic. Se ce n'è una nuova, costruisce il pacchetto e lo installa. Download, verifica e costruzione girano come utente `nobody` (senza privilegi): solo l'installazione finale (`dnf` o `pacman`) gira come root.
3. **Un timer systemd** lancia l'aggiornamento una volta al giorno, con un ritardo casuale fino a un'ora. Se il computer era spento, recupera al primo avvio. Gira a priorità minima, quindi non rallenta il lavoro.

Il tutto non dipende da repository o pacchetti di terzi: gli script vengono copiati in una cartella di sistema e cambiano solo quando li aggiorni tu.

## Installazione

Lo stesso comando vale per Fedora e per Arch:

```bash
git clone https://github.com/<tuo-utente>/claude-installer-script.git
cd claude-installer-script
sudo bash install.sh
```

Usa `sudo bash install.sh` e non `./install.sh`: se i file sono stati caricati su GitHub dal browser perdono il permesso di esecuzione, e così funziona comunque.

L'installer:
- installa gli strumenti necessari (su Fedora `rpm-build`, `binutils`, `file`, `xz`, `curl`, `gnupg2`; su Arch `base-devel`, `binutils`, `file`, `xz`, `curl`, `gnupg`, `libarchive`);
- copia gli script in `/usr/local/libexec/claude-desktop/` e `/usr/local/sbin/`;
- crea e attiva il timer;
- fa subito un primo controllo: se Claude non è installato lo scarica (circa 165 MB) e lo installa.

Dopo l'installazione la cartella della repo si può cancellare.

### Note per Arch

- Le dipendenze dell'app (GTK, NSS, Mesa…) le installa `pacman` dai repository ufficiali al primo aggiornamento.
- Il pacchetto si chiama `claude-desktop`. Se avevi già un pacchetto con lo stesso nome preso dall'AUR, verrà sostituito; se ne avevi uno con un nome diverso (per esempio `claude-desktop-bin`), toglilo prima per evitare conflitti.
- Le funzioni opzionali (sandbox di Claude Code, icona nella barra, portachiavi…) usano pacchetti consigliati: `pacman -Qi claude-desktop` li elenca alla voce *Dipendenze opzionali*.

## Comandi utili

```bash
# forzare un controllo adesso
sudo systemctl start claude-desktop-autoupdate.service

# vedere cosa ha fatto l'ultima volta
journalctl -u claude-desktop-autoupdate -n 20

# disattivare gli aggiornamenti automatici
sudo systemctl disable --now claude-desktop-autoupdate.timer
```

Altri comandi:

```bash
# controllare se c'è un aggiornamento senza installarlo
sudo claude-desktop-autoupdate --check

# ricostruire e reinstallare anche se la versione è già l'ultima
sudo claude-desktop-autoupdate --force

# vedere quando è previsto il prossimo controllo
systemctl list-timers | grep claude
```

Dopo un aggiornamento chiudi e riapri Claude per usare la nuova versione.

## Disinstallazione

```bash
# toglie timer e script, Claude resta installato
sudo bash install.sh --uninstall

# toglie anche Claude
sudo dnf remove claude-desktop      # Fedora
sudo pacman -R claude-desktop       # Arch
```

## Se l'aggiornamento si ferma con un errore

Se nel log compare un errore sulla **firma** o sulla **chiave**, non aggirarlo: significa che qualcosa non torna con l'indice scaricato, oppure che Anthropic ha cambiato chiave o formato del repository. In quel caso controlla la [documentazione ufficiale](https://code.claude.com/docs/en/desktop-linux) per eventuali novità, aggiorna `build.sh` e rilancia `sudo bash install.sh`.

## Icona nella barra di GNOME (Fedora)

Fedora non include l'estensione per le icone nella barra. Installa **AppIndicator Support** da [extensions.gnome.org](https://extensions.gnome.org/extension/615/appindicator-support/) o dall'app *Gestore estensioni*, poi esci e rientra nella sessione. Il pacchetto `gnome-shell-extension-appindicator` di Fedora può risultare incompatibile se hai una versione di GNOME più recente di quella per cui è stato compilato.

## Crediti e licenza

- Parte di `build.sh` (verifica dell'indice firmato e conversione del `.deb` in RPM) è ispirata a [dewzor/claude-desktop-fedora](https://github.com/dewzor/claude-desktop-fedora).
- Questa repo è rilasciata con licenza MIT (vedi `LICENSE`).
- Claude Desktop è software proprietario di Anthropic e **non** è incluso in questa repo: viene scaricato dai server di Anthropic al momento dell'installazione.
