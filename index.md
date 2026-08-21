---
title: "Uno stack AI interamente locale su 16 GB di VRAM"
description: "Cosa entra davvero in una GPU consumer, e cosa si impara provandoci: misure, decisioni e dieci trappole diagnostiche."
---

# Uno stack AI interamente locale su 16 GB di VRAM

Questa pagina racconta la costruzione di un ambiente AI completo — conversazione, voce,
trascrizione, immagini, video — che gira **interamente su una macchina desktop**, senza API a
pagamento e senza che un byte di dato esca dal computer.

Non è un tutorial. È il resoconto di cosa è entrato davvero nella scheda, quali decisioni sono
state prese e perché, e soprattutto delle **dieci trappole diagnostiche** che hanno richiesto
più tempo dell'installazione stessa. Ogni numero qui è misurato su quella macchina, non copiato
da una scheda tecnica.

---

## La macchina, e l'unico vincolo che conta

| Componente | Valore |
|---|---|
| GPU | NVIDIA RTX 5070 Ti — **16 GB VRAM**, Blackwell sm_120 |
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

| Modello | Dimensione | Velocità misurata |
|---|---|---|
| `qwen3:8b` | 7.7 GB | **126.3 tok/s** — 100% GPU |
| `qwen3:14b` | 11 GB | 75.5 tok/s — 100% GPU |
| `qwen3:30b-a3b` | 19 GB | 92 tok/s a contesto 4k, 78.9 a 32k |
| `qwen3-coder:30b-a3b` | 20 GB | 77 tok/s |

Il risultato controintuitivo: **il modello da 8 miliardi è il più rapido dei quattro**, più del
30B a mistura di esperti. Non perché sia intrinsecamente più veloce, ma perché è l'unico che
entra in VRAM lasciando spazio al motore vocale. È il primo esempio del principio di sopra.

Il contesto è stato fissato a **32768 token** su misura, non a intuito:

| Contesto | Velocità | Costo |
|---|---|---|
| 4096 | 96.6 tok/s | — |
| 16384 | 88.3 tok/s | −9% |
| **32768** | **78.9 tok/s** | **−18%** |
| 65536 | 64.4 tok/s | −33% |

Ottuplicare il contesto costa il 18%. I 65536 sono stati scartati non per la velocità, ma per
lasciare margine: il perché sta nella lezione 3.

### Trascrizione

**Whisper `large-v3-turbo`** attraverso CTranslate2, con una divisione dei compiti deliberata:

| Percorso | Device | Velocità | Uso |
|---|---|---|---|
| Riga di comando | GPU, float16 | **30x il tempo reale** — un'ora in 2 minuti | file, sottotitoli |
| Dentro l'interfaccia | CPU, int8 | 2.6x — un'ora in 23 minuti | dettatura dal microfono |

La versione su CPU sembra una rinuncia e non lo è: elimina alla radice il conflitto di memoria
con il modello linguistico, che si manifesterebbe nel momento peggiore — premere il microfono
mentre stai conversando, con la GPU già piena. Per una frase dettata di dieci secondi la CPU
risponde in quattro, che basta.

Un dettaglio con conseguenze pratiche: `int8` **perde la punteggiatura**. Stesso file, stesse
opzioni, solo la quantizzazione diversa:

- `float16` → `Va bene, allora vediamo se funziona questo coso, dammi la mano a fare un test.`
- `int8` → `va bene allora vediamo se funziona questo coso`

Per questo i file veri passano dalla riga di comando, non dall'interfaccia.

Accetta in ingresso qualunque cosa ffmpeg sappia decodificare — verificato su `mp3`, `m4a`,
`ogg`, `flac` e sui contenitori video `mp4`, `mkv`, `webm`.

### Voce

**Chatterbox Multilingual**, 23 lingue, in un container dedicato. Sintetizza 6.2 secondi di
audio in 2.6, quindi più veloce del tempo reale.

Una scoperta che cambia il modo di usarlo: **l'accento non viene dal codice lingua, viene dalla
voce di riferimento**. Il parametro `language` governa fonetica e prosodia; l'accento —
americano, britannico — dipende da quale campione vocale si usa. Il corollario è potente: la
stessa voce parla tutte le lingue, quindi dieci secondi della propria voce diventano voice
cloning multilingua.

### Immagini e video

**Flux.1-schnell** in GGUF Q4: 1024×1024 in 4 passi, 10.6 GB di VRAM. La variante quantizzata è
stata scelta al posto della fp8 all-in-one da 17 GB perché quest'ultima non sta in scheda e
verrebbe continuamente paginata.

**LTX-Video 2B distilled**: quattro secondi di video, 97 fotogrammi a 768×512, generati in
**2.1 secondi** a modello caldo. Più veloce del tempo reale del filmato prodotto. La stima
iniziale era "decine di secondi": sbagliata di un ordine di grandezza, in meglio.

---

## Le interfacce

**Open WebUI** è il posto di lavoro quotidiano: conversazione, ricerca web con DuckDuckGo senza
chiavi API, RAG sui documenti, dettatura, esecuzione di codice. Espone anche chiavi API proprie,
generate localmente.

**ComfyUI** è lo studio per le immagini e i video: seed riproducibili, img2img, inpainting,
upscaling, ControlNet, LoRA, batch. Ci si va quando l'immagine *è* il prodotto.

**Una GUI scritta su misura** per il doppiaggio di registrazioni: si carica un audio o un video,
si scelgono lingua di partenza, lingua di arrivo e cartella di destinazione, e si ottengono
trascrizione, traduzione e audio ridetto nella lingua scelta. Ottanta righe di libreria standard
Python — nessun pip, nessuna dipendenza da mantenere — perché deve invocare gli eseguibili sul
filesystem e da dentro un container non li raggiungerebbe.

**Un proxy di rilevamento lingua**, il pezzo più interessante del progetto. Open WebUI, parlando
con un endpoint TTS compatibile OpenAI, invia solo il testo e la voce. Chatterbox ha bisogno di
sapere *in che lingua* leggere, e non lo deduce da sé. Senza intermediario, il giapponese viene
letto con fonetica inglese.

Il proxy si mette in mezzo, deduce la lingua e **segmenta il testo misto**. Perché il caso d'uso
reale è didattico: il modello risponde in giapponese e aggiunge la traduzione italiana tra
parentesi, e ogni parte va pronunciata nella sua lingua.

Il rilevamento lavora su due livelli. Le lingue con alfabeto proprio — cirillico, kana, hangul,
arabo, greco, devanagari — sono inequivocabili dal sistema di scrittura. Per quelle in alfabeto
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

## Il modello per il codice

`qwen3-coder:30b-a3b` è post-addestrato per l'uso agentico sul codice: navigare file, invocare
strumenti, applicare patch su più turni. Dichiara `capabilities: [completion, tools]` — supporta
il tool calling, che è il prerequisito per usarlo come agente.

Ma non ha il **thinking**, che il generalista possiede. Su una traduzione idiomatica o su un
problema tortuoso quella differenza si sente, e per questo è la scelta *peggiore* per tradurre —
un difetto che si era insinuato come default nell'interfaccia e che è stato corretto.

Il limite realistico va detto: questi modelli sono forti su autocomplete, boilerplate, singole
funzioni, spiegazione di codice e generazione di test. Su refactoring multi-file e bug non ovvi
non tengono il passo. La collocazione giusta è affiancarli a un IDE, non sostituire un modello di
frontiera.

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
striscia sul bus. Nel caso reale il colpevole era un videogioco aperto in un'altra finestra.

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
assente, e un formato audio sbagliato — WAV etichettato mp3, che il browser non decodifica.
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

## Cosa contiene il repository

```
CLAUDE.md                    documentazione operativa completa e dieci lezioni
docs/index.md                questa pagina
start-stack.bat              accende tutto, attende ogni servizio, stampa gli indirizzi
stop-stack.bat               spegne tutto e libera VRAM, RAM e macchina virtuale
trascrivi.bat                trascrizione da riga di comando, anche drag and drop
gui.bat + gui/               interfaccia web per il doppiaggio di registrazioni
tts-proxy/                   proxy di rilevamento lingua, sola libreria standard
tests/                       22 verifiche eseguibili senza GPU né servizi
scripts/                     diagnostica VRAM, liberazione GPU, riavvii puliti
comfyui/                     workflow testati per immagini e video
open-webui/                   configurazione del container
```

Due script gestiscono l'intero ciclo di vita. Lo spegnimento libera **tutte** le risorse — GPU,
RAM e macchina virtuale — perché il presupposto è che chi lo lancia abbia bisogno del computer
per altro.

---

## Cosa non è stato fatto

La **diarizzazione** per i meeting a più voci, che richiede un componente aggiuntivo e
distinguerebbe chi parla. I **sottotitoli tradotti** con timestamp preservati: è un problema
meno banale di quanto sembri, perché tradurre riscrive la segmentazione delle frasi. E il
supporto per modelli video di qualità superiore, che scambia secondi con minuti.

---

## La conclusione onesta

Su una GPU consumer si può costruire un ambiente AI locale sorprendentemente completo, e per una
fascia di lavoro reale — volume, privacy, costo marginale zero, funzionamento offline — è
genuinamente competitivo.

Ma va detto con chiarezza dove sta il confine. Questi modelli non sostituiscono quelli di
frontiera su ragionamento complesso, coerenza su contesto lungo e cicli agentici estesi. La
divisione sensata non è *"locale invece che remoto"* ma *"locale per ciò che è ripetitivo e
riservato, frontiera per ciò che richiede giudizio"*.

E la scoperta più utile non riguarda i modelli: riguarda il fatto che su hardware al limite la
diagnostica conta più della configurazione. Sette delle dieci trappole qui sopra producevano
sistemi che rispondevano `200 OK` mentre non funzionava niente.
