---
title: "Uno stack AI interamente locale su 16 GB di VRAM"
description: "Cosa entra davvero in una GPU consumer, e cosa si impara provandoci: misure, decisioni e dieci trappole diagnostiche."
---

# Uno stack AI interamente locale su 16 GB di VRAM

Questa pagina racconta la costruzione di un ambiente AI completo, conversazione, voce,
trascrizione, immagini, video, che gira **interamente su una macchina desktop**, senza API a
pagamento e senza che un byte di dato esca dal computer.

Non è un tutorial. È il resoconto di cosa è entrato davvero nella scheda, quali decisioni sono
state prese e perché, e soprattutto delle **dieci trappole diagnostiche** che hanno richiesto
più tempo dell'installazione stessa. Ogni numero qui è misurato su quella macchina, non copiato
da una scheda tecnica.

---

## La macchina, e l'unico vincolo che conta

| Componente | Valore |
|---|---|
| GPU | NVIDIA RTX 5070 Ti, **16 GB VRAM**, Blackwell sm_120 |
| CPU | AMD Ryzen 7 9800X3D (8 core / 16 thread) |
| RAM | 64 GB DDR5 |
| OS | Windows 11, WSL2 in modalità *mirrored* |

Sedici gigabyte di VRAM sono il vincolo che determina ogni scelta architetturale che segue. Non
la potenza di calcolo, non la RAM, non il disco: **la memoria video**. La lezione centrale
dell'intero progetto è che su hardware consumer la domanda giusta non è *"quanto è buono questo
modello"* ma *"cosa gli sta accanto"*.

---

## Cosa gira, e quanto va veloce

### Linguaggio

Tre modelli, uno per ruolo. Il parco è passato da cinque a tre dopo una prova che ha misurato
cosa **sanno** invece di quanto corrono:

| Modello | Ruolo | Dimensione | Velocità misurata |
|---|---|---|---|
| MoE da 35B, 3 attivi | il predefinito, tutti i giorni | 23 GB | 74 tok/s |
| MoE da 21B, 3.6 attivi | il codice | 13 GB | **161 tok/s**, interamente in GPU |
| denso da 8B | la voce | 5.2 GB | 126.3 tok/s |

Il risultato controintuitivo: **il modello da 8 miliardi è il più rapido di quelli grandi che
c'erano prima**, non perché sia intrinsecamente veloce, ma perché è l'unico che entra in VRAM
lasciando spazio al motore vocale. È il primo esempio del principio di sopra.

Il secondo esempio è la scelta del predefinito, che non è il più rapido: è quello che **usa gli
strumenti**. Davanti a un indirizzo che non conosce va ad aprirlo, mentre il candidato più veloce
rispondeva a memoria e negava. Con il tool calling nativo, un modello che non chiama lo strumento
rende inutile tutta la catena della ricerca.

Il contesto è stato fissato a **32768 token** su misura, non a intuito:

| Contesto | Velocità | Costo |
|---|---|---|
| 4096 | 96.6 tok/s | - |
| 16384 | 88.3 tok/s | −9% |
| **32768** | **78.9 tok/s** | **−18%** |
| 65536 | 64.4 tok/s | −33% |

Ottuplicare il contesto costa il 18%. I 65536 sono stati scartati non per la velocità ma per
lasciare margine, e una misura successiva ha dato ragione a quella scelta: a 65536 il modello non
sta più in scheda, e il monitor dichiara il 42% del lavoro sulla CPU.

C'è un corollario che vale per qualunque documento lungo: **un testo più lungo del contesto viene
troncato in silenzio.** Una trascrizione da 42000 token, con il contesto a 32768, ne ha lasciati
passare 16386: il modello ha risposto sul 40% del testo, tenendo la fine e buttando l'inizio, e
la risposta era buona. Buona e parziale, che è il modo peggiore di essere sbagliata. Il numero da
guardare è quello dei token che il motore dichiara di aver letto, non la qualità della
risposta.

### Trascrizione

**Whisper `large-v3-turbo`** attraverso CTranslate2, con una divisione dei compiti deliberata:

| Percorso | Device | Velocità | Uso |
|---|---|---|---|
| Riga di comando | GPU, float16 | **da 30 a 46x il tempo reale**: due ore in tre minuti | file, sottotitoli |
| Dentro l'interfaccia | CPU, int8 | 2.6x: un'ora in 23 minuti | dettatura dal microfono |

La versione su CPU sembra una rinuncia e non lo è: elimina alla radice il conflitto di memoria
con il modello linguistico, che si manifesterebbe nel momento peggiore: premere il microfono
mentre stai conversando, con la GPU già piena. Per una frase dettata di dieci secondi la CPU
risponde in quattro, che basta.

Un dettaglio con conseguenze pratiche: `int8` **perde la punteggiatura**. Stesso file, stesse
opzioni, solo la quantizzazione diversa:

- `float16` → `Va bene, allora vediamo se funziona questo coso, dammi la mano a fare un test.`
- `int8` → `va bene allora vediamo se funziona questo coso`

Per questo i file veri passano dalla riga di comando, non dall'interfaccia.

Accetta in ingresso qualunque cosa ffmpeg sappia decodificare, verificato su `mp3`, `m4a`,
`ogg`, `flac` e sui contenitori video `mp4`, `mkv`, `webm`.

### Dalla riunione registrata al dialogo scritto

Il caso d'uso che ha guidato più scelte di tutti: una riunione di due ore, registrata con OBS
su più tracce audio, che deve diventare un testo leggibile con la frase giusta accanto alla
voce giusta. Non è una funzione sola: è una catena di cinque passaggi, ognuno nato da un
problema incontrato davvero.

**Le tracce si riconoscono misurandole, non leggendo i metadati.** Una registrazione OBS
arriva con cinque tracce e nessuna etichetta affidabile. Un file da nove gigabyte non si
carica nel browser, quindi si indica dov'è: l'interfaccia le ascolta e dichiara cosa sono, il
mix (la più forte), le sorgenti distinte, le vuote, e i doppioni identici bit per bit che
nascono dalle assegnazioni multiple nel software di registrazione.

**Quale traccia è il microfono lo dice il silenzio, non il volume.** Una traccia di
applicazione scende nel silenzio digitale quando nessuno parla; un microfono aperto non ci
arriva mai. È un criterio più robusto di qualunque confronto di livelli, e quando non basta,
su una registrazione corta dove nessuno tace mai, l'interfaccia dichiara di non saperlo invece
di indovinare. Sbagliare qui vorrebbe dire depurare la traccia sbagliata, cancellando proprio
la voce da tenere.

**Senza cuffie il microfono riprende anche l'audio delle casse**, quindi metà delle frasi
degli altri finisce nella propria traccia, e attribuirle a chi registra è sbagliato. Su due
ore di riunione erano 1199 battute su 1390: senza questo filtro, l'86% delle frasi altrui
risultava in bocca a chi registrava. Si tolgono confrontando il livello delle due tracce
battuta per battuta: dove domina quella delle applicazioni, la voce arriva da loro. Sui tratti
lunghi funziona; su battute di due parole sovrapposte sbaglia in entrambe le direzioni, e lì
il rimedio non è un parametro migliore, sono le cuffie.

**Chi parla quando, dentro la traccia degli altri.** I partecipanti che stanno tutti in una
traccia sola vengono distinti e etichettati da un modello di diarizzazione che gira in locale
come il resto: sessanta volte il tempo reale, e i quasi quattro gigabyte di memoria video che
occupa tornano liberi appena finisce, perché gira come processo separato che muore a lavoro
concluso. I pesi vengono da modelli ad accesso controllato su Hugging Face: gratuiti, ma
dietro consensi da accettare a mano, e i consensi necessari sono **tre e non due**, perché la
versione recente della pipeline prende un componente da un deposito che nessuna guida nomina.
La trappola annessa: un token senza il permesso giusto risponde `200` alle richieste di
metadati e `403` al download dei file, che è il modo perfetto di sembrare configurati senza
esserlo.

**La composizione finale è un dialogo.** Le battute della propria voce, depurate dal rientro,
si intrecciano con quelle degli altri, etichettate per voce, in un testo solo con gli orari:

    [00:13:22] IO: ...
    [00:13:31] VOCE 2: ...

Le battute consecutive della stessa voce diventano un turno, perché il botta e risposta si
legge nei cambi di voce, non in cinquemila righe da tre parole. Due ore di riunione diventano
un documento così in tre minuti di macchina, senza che un secondo di audio esca dal computer:
per una riunione di lavoro vera non è un dettaglio, è il requisito.

### Voce

**Chatterbox Multilingual**, 23 lingue, in un container dedicato. Sintetizza 6.2 secondi di
audio in 2.6, quindi più veloce del tempo reale.

Una scoperta che cambia il modo di usarlo: **l'accento non viene dal codice lingua, viene dalla
voce di riferimento**. Il parametro `language` governa fonetica e prosodia, mentre l'accento (americano,
britannico) dipende da quale campione vocale si usa. Il corollario è potente: la
stessa voce parla tutte le lingue, quindi dieci secondi della propria voce diventano voice
cloning multilingua.

### Immagini e video

**Flux.1-schnell** in GGUF Q4: 1024×1024 in 4 passi, 10.6 GB di VRAM. La variante quantizzata è
stata scelta al posto della fp8 all-in-one da 17 GB perché quest'ultima non sta in scheda e
verrebbe continuamente paginata.

**Qwen-Image** accanto a Flux, non al suo posto: sei volte più lento (95 secondi contro 15) e
molto più fedele a quello che hai scritto. Sullo stesso prompt Flux ha inquadrato una finestra e
ignorato il resto della scena richiesta, mentre Qwen-Image ha costruito la stanza. La sorpresa è
che sul testo dentro l'immagine ha vinto Flux, cioè esattamente l'aspettativa contraria. Si tiene
Flux per iterare su un'idea e Qwen-Image quando la composizione conta.

**LTX-Video**: quattro secondi di video, 97 fotogrammi a 768×512, generati in **12 secondi** con
il modello da 2 miliardi di parametri. Il 13B in fp8 costa il doppio del tempo e ha sei volte i
parametri: la differenza non è che il video sia più bello, è che **capisce cosa gli hai chiesto**.
Il prompt diceva "piallare" e solo il grande ha messo in mano una pialla riconoscibile, con i
riccioli di legno invece di granuli sparsi. Restano entrambi, per lo stesso motivo delle
immagini.

---

## Le interfacce

**Open WebUI** è il posto di lavoro quotidiano: conversazione, RAG sui documenti allegati e
dettatura. La ricerca web passa da **un motore di ricerca in casa**, in un container, non da un
servizio esterno: sulla stessa domanda il connettore pubblico dava tre risultati, di cui due
video, mentre l'aggregatore locale ne dà trenta con le recensioni delle testate in cima. Non
vuole chiavi API e la query non esce verso qualcuno che la registra.

Il resto delle integrazioni è stato **scorporato**, e il motivo è misurabile: con
l'autocompletamento attivo ogni pausa di digitazione genera una richiesta al modello, e titoli,
tag e domande di completamento ne aggiungono altre tre per messaggio. Su una GPU dove il modello
sta a filo, quelle richieste si mettono in fila davanti alla tua: in una sessione si leggono
dodici chiamate, con una da 34 minuti. Dopo la pulizia: una sola chiamata per messaggio.

**ComfyUI** è lo studio per le immagini e i video: seed riproducibili, img2img, inpainting,
upscaling, ControlNet, LoRA, batch. Ci si va quando l'immagine *è* il prodotto.

**Una GUI scritta su misura** per il doppiaggio di registrazioni: si carica un audio o un video,
si scelgono lingua di partenza, lingua di arrivo e cartella di destinazione, e si ottengono
trascrizione, traduzione e audio ridetto nella lingua scelta. Sola libreria standard Python,
nessuna dipendenza da mantenere, e gira sull'host perché deve invocare gli eseguibili sul
filesystem: da dentro un container non li raggiungerebbe.

**Un proxy di rilevamento lingua**, il pezzo più interessante del progetto. Open WebUI, parlando
con un endpoint TTS compatibile OpenAI, invia solo il testo e la voce. Chatterbox ha bisogno di
sapere *in che lingua* leggere, e non lo deduce da sé. Senza intermediario, il giapponese viene
letto con fonetica inglese.

Il proxy si mette in mezzo, deduce la lingua e **segmenta il testo misto**. Perché il caso d'uso
reale è didattico: il modello risponde in giapponese e aggiunge la traduzione italiana tra
parentesi, e ogni parte va pronunciata nella sua lingua.

Il rilevamento lavora su due livelli. Le lingue con alfabeto proprio, cirillico, kana, hangul,
arabo, greco, devanagari, sono inequivocabili dal sistema di scrittura. Per quelle in alfabeto
latino serve un punteggio sulle parole funzionali, con una soglia di confidenza: senza,
`(Konnichiwa, minna-san!)` finiva classificato **portoghese**, perché le sillabe `o` e `a`
combaciano con gli articoli portoghesi. Sotto soglia si ripiega, e la direzione del ripiego
dipende da chi domina il messaggio:

- `こんにちは（Konnichiwa）` → domina il giapponese: il latino è romaji, va letto giapponese
- `La parola ありがとう significa grazie` → domina il latino: la parola isolata è un prestito, il resto è italiano

Il conteggio dei caratteri distingue i due casi meglio di qualunque euristica sul testo.
**Ventidue test** coprono questi comportamenti, ognuno nato da un difetto osservato e non da
un'ipotesi.

---

## Il modello per il codice, scelto due volte

La prima scelta è stata fatta sull'etichetta: un modello post-addestrato per l'uso agentico sul
codice, con il punteggio più alto su SWE-bench Verified fra quelli che entrano in 16 GB. Sembrava
ovvio.

Messo alla prova su dodici domande da sviluppatore senior, non ha vinto **nessuna** delle dodici.
Su Terraform rinominava una risorsa creandone una seconda e chiamando la CLI del provider da un
`null_resource`, dove la risposta era il blocco `moved`. Su un test di proprietà scriveva le
funzioni annidate dentro un'altra funzione di test, che pytest non esegue mai: una suite che
passa sempre senza provare niente.

Il benchmark non era sbagliato: misurava un'altra cosa. Valuta la risoluzione di issue in
repository Python con i test come giudice, e nessuna di quelle aree è quel compito. Un punteggio
alto là non dice niente su quanto è vecchia l'API di un provider che il modello ricorda.

Adesso il modello per il codice è un MoE da 21 miliardi di parametri con 3.6 attivi, che sta
interamente in GPU lasciando 2.3 GB di margine e genera a 161 token al secondo. Ha prodotto la
migliore risposta dell'intera prova, sull'unica domanda in cui contava conoscere una parte del
linguaggio invece di ricordare una libreria.

Il limite realistico resta, e vale per tutti: su refactoring multi-file e bug non ovvi non
tengono il passo. E su tre framework di orchestrazione molto diffusi nessuno dei modelli locali
provati è utilizzabile: producono codice dalla forma perfetta con import che non esistono, e il
difetto non si vede leggendo, si vede quando parte.

---

## Dieci trappole diagnostiche

Questa è la parte che vale il progetto. Reinstallare tutto richiede mezz'ora; ricostruire queste
diagnosi ha richiesto un'intera sessione. Sono ordinate per quanto tempo hanno rubato.

**1. Il pavimento di VRAM decide tutto, e le metriche mentono.** Stesso prompt, stesso modello,
unica variabile la memoria occupata *prima* del caricamento:

| VRAM occupata a riposo | Velocità |
|---|---|
| ~1150 MiB | **96.6 tok/s** |
| 4879 MiB | **10.3 tok/s** |
| 5181 MiB | **2.4 tok/s** |

In tutti e tre i casi lo strumento di monitoraggio dichiarava `77% GPU`. La causa è il driver
WDDM di Windows, che non rifiuta un'allocazione più grande della memoria disponibile: la pagina
in RAM di sistema attraverso il PCIe, in silenzio. Il modello crede di essere in GPU e invece
striscia sul bus. Nel caso reale il colpevole era un'applicazione 3D aperta in un'altra finestra.

**Se un modello sembra improvvisamente lento o stupido, non è il modello.**

**2. I processi figli sopravvivono ai padri.** Terminare il server di inferenza lascia vivo il
processo che tiene effettivamente i pesi: un orfano ha trattenuto 14.6 GB di VRAM per trenta
minuti, producendo un errore CUDA che *sembrava* un'incompatibilità con l'architettura della
GPU. Non lo era.

**3. Un errore CUDA di inizializzazione è quasi sempre memoria, non architettura.** Il messaggio
puntava a un kernel di flash attention e ha portato a disattivare due ottimizzazioni sane. Con
memoria libera funzionano perfettamente.

**4. `localhost` non è `127.0.0.1`.** Con WSL in modalità *mirrored*, `localhost` risolve prima
su `::1` e quel percorso non risponde. Inoltre gli strumenti di sistema non vedono i listener dei
container, perché vivono in un altro namespace: sembra che il port forwarding sia rotto, e non lo
è.

**5. Il database vince sulle variabili d'ambiente.** Open WebUI conserva la configurazione scritta
al *primo* avvio e da quel momento ignora l'ambiente. Il motore vocale restava quello del browser
nonostante fosse configurato altrimenti, e la dettatura girava sul modello peggiore senza dirlo.

**6. Quattro problemi distinti sulla stessa funzione, tutti con risposta `200 OK`.** Far
funzionare la voce ha richiesto di risolvere in sequenza: il parametro lingua non trasmesso, un
modello che era solo inglese e *ignorava quel parametro in silenzio*, una libreria fonetica
assente, e un formato audio sbagliato: WAV etichettato mp3, che il browser non decodifica.
Ognuno mascherava il successivo: senza risolvere il primo non si vedeva il secondo.

**7. La verbosità è un problema di prestazioni.** Una richiesta di *una frase* generava una
risposta con la lettura in kana, una variante alternativa, una domanda finale e un'emoji: 13.6
secondi di audio da sintetizzare prima di riprodurre qualsiasi cosa. L'attesa faceva sembrare
rotta la riproduzione. Meno testo non è solo più pulito, è quattro volte più rapido.

**8. Uno spegnimento sporco lascia socket non eliminabili.** Dopo un blackout restano voci del
filesystem che puntano a oggetti kernel morti: non leggibili, non cancellabili, e sufficienti a
impedire l'avvio di Docker. La cartella si può però *rinominare*, e il servizio la ripopola.

**9. Gli script batch di Windows falliscono in modi silenziosi.** `timeout` non funziona quando
lo stdin è rediretto e i cicli di attesa girano a vuoto; `start` con due argomenti quotati li
fonde; `Start-Process` con uno stile di finestra richiede una window station interattiva. Ogni
attesa va verificata, non assunta.

**10. Misurare, non dedurre.** Le stime iniziali sono state sbagliate tre volte: la velocità di
un modello per un fattore 4, la generazione video per un fattore 10, l'occupazione di memoria del
motore vocale per un fattore 2. In ogni caso la misura ha smentito il ragionamento. Su hardware
al limite, l'unico dato affidabile è quello preso sulla propria macchina.

---

## Gli obiettivi raggiunti

Il progetto e' partito da una domanda sola, cosa entra davvero in sedici gigabyte, e questi sono
i punti in cui si e' fermato:

**Indipendenza completa.** Conversazione, ricerca web, trascrizione, traduzione, sintesi vocale,
immagini e video girano senza una chiave API e senza che un dato esca dalla macchina. La ricerca
passa da un motore in casa, quindi nemmeno la query viene registrata da qualcun altro.

**Una catena, non sette strumenti.** Da un file audio o video si arriva a trascrizione,
sottotitoli tradotti battuta per battuta e capitoli, in un passaggio. I timestamp restano
identici al millesimo attraverso la traduzione, perche' il modello sceglie il numero della battuta
e l'orario lo copia il codice: la differenza fra un sottotitolo che regge e uno che slitta.

**Le registrazioni di riunioni vere.** Tracce audio separate riconosciute e trascritte una per
una, la voce di chi registra separata da quella degli altri anche quando il microfono ha ripreso
le casse, e i partecipanti distinti fra loro dalla diarizzazione. Due ore di audio in tre minuti.

**Modelli locali che toccano i file, dentro regole precise.** Un agente confinato in una
cartella puo' elencare, leggere e cercare, con un registro di ogni accesso e una lista bianca dei
comandi. Legge i PDF veri, compresi quelli da ufficio con le codifiche che rendono il testo
illeggibile agli estrattori ingenui, i documenti Office, e **le tabelle incollate come
screenshot**, che per qualunque ricerca testuale non esistono: quelle le trascrive un modello che
vede. E quando non puo' verificare qualcosa lo dichiara, invece di confondere il non trovato con
il non esserci.

**Un solo posto da cui accendere.** Una pagina mostra quanta memoria video resta prima di
decidere, accende le dipendenze da se', avvisa quando una scelta manderebbe in paginazione un
modello gia' caricato, e libera la GPU senza spegnere niente. Spegnere significa liberare tutto,
macchina virtuale compresa, perche' chi lo fa ha bisogno del computer per altro.

**La scelta dei modelli fatta su misure, non su classifiche.** Il modello con il punteggio piu'
alto fra quelli che entrano in scheda non ha vinto nessuna delle dodici domande di una prova di
competenza, ed e' stato rimosso. Quello che resta e' stato scelto perche' usa gli strumenti
invece di rispondere a memoria.

**Le trappole documentate mentre costavano tempo**, che e' la parte che vale piu' del codice: chi
riprende il progetto non paga due volte la stessa diagnosi.

---

## La conclusione onesta

Su una GPU consumer si può costruire un ambiente AI locale sorprendentemente completo, e per una
fascia di lavoro reale, volume, privacy, costo marginale zero, funzionamento offline, è
genuinamente competitivo.

Ma va detto con chiarezza dove sta il confine. Questi modelli non sostituiscono quelli di
frontiera su ragionamento complesso, coerenza su contesto lungo e cicli agentici estesi. La
divisione sensata non è *"locale invece che remoto"* ma *"locale per ciò che è ripetitivo e
riservato, frontiera per ciò che richiede giudizio"*.

E la scoperta più utile non riguarda i modelli: riguarda il fatto che su hardware al limite la
diagnostica conta più della configurazione. Sette delle dieci trappole qui sopra producevano
sistemi che rispondevano `200 OK` mentre non funzionava niente.
