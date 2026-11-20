# Current-meter v2, specifica del protocollo di comunicazione

Versione: 0.6
Progetto: power profiler USB a doppio INA228, da 1 µA a 1.6 A, ingresso da 1.8 a 6 V
Trasporto: USB CDC (bidirezionale), MCU STM32

Questo documento definisce il formato dei dati scambiati tra il dispositivo e il PC. E' la specifica di riferimento per il firmware, per l'emulatore e per il software lato PC: in caso di divergenza tra implementazioni, vale questo documento.

Le modifiche rispetto alle versioni precedenti sono elencate nella sezione 12.

---

## 1. Architettura della comunicazione

Il dispositivo monta due INA228 continui e indipendenti:

- canale 0: range basso, shunt 10 ohm
- canale 1: range alto, shunt 0.1 ohm

I due canali sono **sempre attivi e sempre trasmessi per intero**. Non esiste multiplexing in funzione dello stato del bypass: i dati del canale non attivo servono a validare lo stitching nel momento della transizione di range, che e' proprio il momento in cui l'informazione conta di piu'.

La comunicazione e' asimmetrica:

- **Device verso PC**: stream binario continuo di frame. Nessun testo.
- **PC verso device**: comandi in ASCII, terminati da `\n`. Le risposte tornano come frame binari di tipo `0x03`.

La scelta dell'ASCII in ingresso serve a poter pilotare lo strumento da un terminale qualunque durante il debug. La banda dei comandi e' irrilevante rispetto allo stream in uscita.

---

## 2. Flussi e cadenze

| Flusso | Contenuto | Cadenza |
|---|---|---|
| Veloce | campioni grezzi di shunt, uno stream per canale | fino a 20 kSPS per canale |
| Lento | telemetria (batteria, bus, temperature, contatori) | 1 - 10 Hz |
| Eventi | risposte ai comandi, header di sessione | su evento; l'header di sessione anche a cadenza fissa ogni 5 s |

Le grandezze lente (tensione di batteria, temperatura di die) non vengono mai inserite nei frame di campioni: replicarle 20.000 volte al secondo triplicherebbe la banda per ritrasmettere numeri che si muovono su scale di secondi.

---

## 3. Formato del frame

### 3.1 Convenzioni di codifica

- Tutti i campi multibyte dell'header, del trailer e dei payload sono **little endian**.
- Fanno eccezione i 3 byte grezzi del registro VSHUNT nei frame `0x01`, che vengono trasmessi **big endian**, cioe' esattamente nell'ordine in cui escono dal registro dell'INA228, senza riallineamento ne' conversione.

### 3.2 Header comune (16 byte)

| Offset | Campo | Tipo | Contenuto |
|---|---|---|---|
| 0 | `sync` | uint16 | `0xA5 0x5A` |
| 2 | `type` | uint8 | tipo di frame, vedi sezione 4 |
| 3 | `channel` | uint8 | 0 = range basso, 1 = range alto, 0xFF = non applicabile |
| 4 | `seq` | uint32 | contatore di frame, indipendente per canale |
| 8 | `t_start_us` | uint32 | timer STM32 in microsecondi, riferito al primo campione del frame |
| 12 | `payload_len` | uint16 | **lunghezza del payload in byte**, valida per tutti i tipi di frame |
| 14 | `flags` | uint16 | configurazione e stato, vedi sezione 5 |

**`payload_len` e' in byte, non in campioni.** Per i frame di tipo `0x01` il numero di campioni si ricava come `payload_len / 3`. Questo permette a un ricevitore di saltare un frame di tipo sconosciuto senza doverne conoscere la struttura: e' il requisito che rende il protocollo estendibile senza rompere i software gia' scritti.

### 3.3 Trailer

| Campo | Tipo | Contenuto |
|---|---|---|
| `crc16` | uint16 | CRC calcolato su header + payload, trasmesso little endian |

Algoritmo: **CRC-16/CCITT-FALSE**. Polinomio `0x1021`, valore iniziale `0xFFFF`, nessuna riflessione di ingresso o uscita, nessuno XOR finale. Vettore di verifica: la stringa ASCII `123456789` deve dare `0x29B1`.

### 3.4 Dimensioni tipiche

Frame di tipo `0x01` con 64 campioni:

```
16 (header) + 192 (payload) + 2 (CRC) = 210 byte
```

Overhead: 8.6%.

Con le cadenze lente il frame e' piu' corto (sezione 4.1): in SLOW contiene un solo campione, `16 + 3 + 2 = 21` byte. L'overhead relativo cresce, ma a 3.75 campioni al secondo la banda e' irrilevante.

---

## 4. Tipi di frame

### 4.1 `0x01` - Blocco campioni

Il flusso veloce. Un frame per canale, emessi in modo indipendente.

**Payload**: `payload_len / 3` campioni da 3 byte, contenuto grezzo del registro VSHUNT dell'INA228 (24 bit, big endian come da sezione 3.1).

**Layout del registro.** I 3 byte sono il registro VSHUNT tal quale, senza riallineamento a bordo. Nel registro il risultato di conversione e' un intero a 20 bit in complemento a due che occupa i **bit 23..4**; i **bit 3..0 sono riservati e valgono 0**. Il firmware non li tocca e il PC li scarta. La procedura corretta, nell'ordine, e':

1. interpretare i 3 byte come intero a **24 bit con segno** (estensione del segno dal bit 23);
2. applicare uno **shift aritmetico a destra di 4** al valore con segno (mai sul valore senza segno: `0xFFFFFF` e' -1 e deve restare -1, non diventare 1048575);
3. applicare l'LSB al risultato a 20 bit: `312.5 nV` con ADCRANGE 0, `78.125 nV` con ADCRANGE 1.

Il fondo scala che ne risulta e' `2^19 x 312.5 nV = 163.84 mV` (ADCRANGE 0), coerente con il campo `adcrange` del frame `0x04`. Un valore a 24 bit pieni darebbe ±2.62 V, che non esiste. La saturazione positiva si legge quindi `0x7FFFF0`, quella negativa `0x800000`.

Formula completa di conversione, con `zero` e `cal` presi dal frame `0x04` della sessione corrente:

```
raw24  = int24_signed(byte0 << 16 | byte1 << 8 | byte2)
raw20  = raw24 >> 4                              (shift aritmetico)
I[A]   = (raw20 - zero[ch]) x LSB(adcrange) / R_shunt[ch] x cal[ch] / 1000000
```

Gli offset `zero_low` e `zero_high` sono in conteggi a 20 bit, cioe' nella stessa scala di `raw20`. Esempi con ADCRANGE 0 e calibrazione unitaria: `0x7FFFF0` sul canale alto (0.1 ohm) vale 1.6384 A; sul canale basso (10 ohm) il fondo scala e' 16.384 mA; 1 mA sul canale basso e' 10 mV = 32000 LSB, cioe' `32000 << 4 = 0x07D000` nel campo trasmesso.

Nessun timestamp per campione. Il rate e' deterministico a partire dai codici VSHCT e AVG e dal bit 11 presenti nei `flags` (sezione 5), quindi il PC ricostruisce i tempi dei campioni successivi a partire da `t_start_us`.

**Chiusura del frame e latenza massima.** Un frame `0x01` si chiude quando e' pieno oppure dopo 100 ms, quello che viene prima. I 100 ms si misurano sui timestamp dei campioni: il frame si chiude dopo un campione se ha raggiunto `samples_per_frame` campioni, oppure se il campione successivo avrebbe un timestamp di 100 ms o piu' dopo il primo. Il numero massimo di campioni di un frame, a un dato periodo di campionamento, e' quindi

```
capacita' = min(samples_per_frame, ceil(100000 us / periodo_us))
```

e l'ultimo campione di un frame cade sempre meno di 100 ms dopo il primo. Con `samples_per_frame` = 64: FAST (50 µs) e DYNAMIC (1102 µs) restano a 64 campioni per frame; SLOW (266.88 ms, sezione 6) scende a 1; AVG 4 con VSHCT 1052 µs (4.408 ms con la conversione VBUS) da' 23 campioni. Quando il periodo supera i 100 ms ogni frame contiene un solo campione e viene emesso appena il campione e' disponibile: la latenza e' quella della conversione, non si puo' scendere sotto.

Un frame puo' contenere **meno** campioni della capacita' quando viene chiuso in anticipo: alla ricezione di un comando che cambia il rate (regola 4), allo `STOP`, e sul canale inattivo prima di una misura singola di tensione (regola 11). In quei casi contiene i campioni gia' acquisiti, e ha i `flags` della configurazione con cui sono stati acquisiti.

Il numero effettivo di campioni di un frame e' **sempre** `payload_len / 3`. Il campo `samples_per_frame` del frame `0x04` e' il valore nominale massimo, non una garanzia sul singolo frame: il PC non deve mai usarlo come numero di campioni di un frame ricevuto. Serve solo a stimare quanti campioni contenevano i frame persi sul link (sezione 10).

### 4.2 `0x02` - Telemetria

Emesso a cadenza fissa tra 1 e 10 Hz. `channel = 0xFF`, `payload_len = 22` (18 fino alla 0.5).

| Offset | Campo | Tipo | Note |
|---|---|---|---|
| 0 | `v_batt_mv` | uint16 | cella dello strumento, letta dall'ADC dell'STM32 |
| 2 | `v_bus_mv` | uint16 | batteria del DUT, dal VBUS dell'INA228 collegato a IN (vedi sotto); `0xFFFF` = mai misurata |
| 4 | `temp_ina0_c16` | int16 | temperatura die INA canale 0, in 1/16 di grado |
| 6 | `temp_ina1_c16` | int16 | temperatura die INA canale 1, in 1/16 di grado |
| 8 | `bypass_state` | uint8 | 0 = bypass inattivo, 1 = attivo |
| 9 | `v_bus_flags` | uint8 | bit 0 = `v_bus_mv` tenuto dall'ultima misura singola (regola 11), non aggiornato; bit 1-7 = 0 |
| 10 | `err_i2c` | uint32 | contatore cumulativo errori I2C |
| 14 | `samples_dropped` | uint32 | contatore cumulativo campioni persi a bordo, vedi regola 7 |
| 18 | `v_bus_t_us` | uint32 | timer STM32 (stessa base di `t_start_us`) dell'istante in cui `v_bus_mv` e' stato misurato |

`v_bus_t_us` serve a collocare la tensione nel tempo senza ambiguita'. Con la misura continua coincide, a meno di un ciclo di conversione, con l'istante della telemetria. Con le misure singole della regola 11 la telemetria ripete per 10 s lo stesso valore: `v_bus_t_us` resta quello della misura, e il PC la disegna una volta sola, nel momento in cui e' stata fatta. Senza questo campo il PC dovrebbe dedurre l'istante dai frame con il bit 12, che possono arrivare prima o dopo la telemetria che porta il valore nuovo.

Le due tensioni hanno fonti diverse:

- `v_batt_mv` e' la cella che alimenta lo strumento, letta dall'ADC dell'STM32 tramite partitore. Serve all'autonomia dello strumento, non dice nulla sul DUT.
- `v_bus_mv` e' la **batteria del DUT**, letta dal registro VBUS dell'INA228. Il pin VBUS di entrambi gli INA228 e' collegato a **IN**, cioe' a monte dello shunt: la misura non contiene la caduta `I x R_shunt` dello strumento, e la piccola corrente assorbita dal pin VBUS arriva dalla sorgente senza passare nello shunt. Collegato a valle dello shunt, quella corrente verrebbe misurata come corrente del DUT: un offset di qualche µA su uno strumento che punta a 1 µA.

La conversione VBUS dell'INA228 non e' gratuita: in modalita' continua ogni ciclo converte shunt e bus in sequenza, e durante la conversione del bus lo shunt non viene misurato. Per questo e' continua solo dove costa poco, e nei preset veloci si fa con misure singole sull'INA del canale inattivo (sezione 5, regola 11). In ogni caso le tensioni restano nel flusso lento a 1 - 10 Hz: nella v2 non si calcolano potenza istantanea o energia.

### 4.3 `0x03` - Risposta a comando

Emesso dopo ogni comando ricevuto. `channel = 0xFF`.

| Offset | Campo | Tipo | Note |
|---|---|---|---|
| 0 | `cmd_echo_len` | uint8 | lunghezza dell'eco |
| 1 | `cmd_echo` | char[] | comando ricevuto, senza terminatore |
| .. | `status` | uint8 | 0 = OK, diverso da 0 = codice di errore |
| .. | `data_len` | uint16 | lunghezza dei dati opzionali |
| .. | `data` | byte[] | payload opzionale |

Payload di `GET CONFIG` (20 byte, nel campo `data`):

| Offset | Campo | Tipo | Note |
|---|---|---|---|
| 0 | `avg_code` | uint8 | |
| 1 | `vshct_code` | uint8 | |
| 2 | `preset` | uint8 | |
| 3 | `bypass_state` | uint8 | |
| 4 | `cal_low` | uint32 | guadagno canale 0 in ppm, vedi 4.4 |
| 8 | `cal_high` | uint32 | guadagno canale 1 in ppm, vedi 4.4 |
| 12 | `zero_low` | int32 | offset di ZERO canale 0, in conteggi grezzi |
| 16 | `zero_high` | int32 | offset di ZERO canale 1, in conteggi grezzi |

### 4.4 `0x04` - Header di sessione

Emesso all'avvio, dopo ogni `ZERO` o `CAL`, e comunque **a cadenza fissa ogni 5 secondi** di tempo device, indipendentemente dalle emissioni su evento e anche a streaming fermo (regola 10). `channel = 0xFF`, `payload_len = 37`.

La riemissione periodica serve al caso normale d'uso: il PC si collega a uno strumento gia' acceso, e un header emesso solo all'avvio non lo riceverebbe mai. Con la cadenza di 5 s un PC che si collega in un istante qualsiasi ha i parametri di conversione entro 5 s.

| Offset | Campo | Tipo | Note |
|---|---|---|---|
| 0 | `fw_version` | uint32 | versione firmware |
| 4 | `proto_version` | uint16 | versione di questo protocollo (6 per la 0.6) |
| 6 | `shunt_low_uohm` | uint32 | valore shunt canale 0, **in micro ohm** (10 ohm = 10000000) |
| 10 | `shunt_high_uohm` | uint32 | valore shunt canale 1, **in micro ohm** (0.1 ohm = 100000) |
| 14 | `cal_low` | uint32 | guadagno di calibrazione canale 0, **in parti per milione** (1000000 = 1.0) |
| 18 | `cal_high` | uint32 | guadagno di calibrazione canale 1, **in parti per milione** (1000000 = 1.0) |
| 22 | `zero_low` | int32 | offset di ZERO canale 0, in conteggi a 20 bit (scala di `raw20`, vedi 4.1) |
| 26 | `zero_high` | int32 | offset di ZERO canale 1, in conteggi a 20 bit |
| 30 | `adcrange` | uint8 | 0 = fondo scala ±163.84 mV, 1 = ±40.96 mV |
| 31 | `preset` | uint8 | preset attivo al momento dell'emissione |
| 32 | `avg_code` | uint8 | averaging attivo al momento dell'emissione |
| 33 | `vshct_code` | uint8 | tempo di conversione attivo al momento dell'emissione |
| 34 | `samples_per_frame` | uint16 | numero **massimo nominale** di campioni per frame `0x01`, non il numero effettivo (vedi 4.1) |
| 36 | `reserved` | uint8 | valore 0 |

L'unita' e' il micro ohm e non il nano ohm perche' 10 ohm in nano ohm sono 1e10, che non entra in un uint32 (massimo 4.295e9). In micro ohm un uint32 arriva a circa 4295 ohm, piu' che sufficiente.

**Semantica della calibrazione.** `cal_low` e `cal_high` sono guadagni moltiplicativi adimensionali espressi in parti per milione: il valore `1000000` corrisponde a un guadagno unitario, `1002300` a +0.23 %, `990000` a -1 %. Il valore `0` non e' una calibrazione valida e il software PC deve rifiutare un frame `0x04` che lo contenga. Il firmware non applica mai la calibrazione ai campioni trasmessi (regola 1): la riporta soltanto, ed e' il software PC a usarla nella conversione in ampere:

```
raw20 = raw24 >> 4                                   (shift aritmetico, vedi 4.1)
I[A]  = (raw20 - zero[ch]) x LSB(adcrange) / R_shunt[ch] x cal[ch] / 1000000
```

con `LSB(0) = 312.5 nV`, `LSB(1) = 78.125 nV`, `R_shunt` in ohm ricavato dai campi in micro ohm, e `zero` in conteggi a 20 bit. Un uint32 in ppm copre guadagni da 1e-6 a circa 4295, molto oltre il necessario; la risoluzione di 1 ppm e' abbondante rispetto alla tolleranza di qualunque shunt.

Questo frame rende ogni registrazione autodescrittiva: un file salvato oggi resta interpretabile anche dopo una ricalibrazione dello strumento.

I campi `preset`, `avg_code` e `vshct_code` sono informativi: il rate con cui interpretare un frame `0x01` e' sempre quello dei suoi `flags` (sezione 5).

---

## 5. Campo `flags`

| Bit | Campo | Valori |
|---|---|---|
| 0-2 | codice AVG | indice nella tabella AVG dell'INA228 |
| 3-5 | codice VSHCT | indice nella tabella dei tempi di conversione |
| 6-7 | preset attivo | 0 = FAST, 1 = DYNAMIC, 2 = SLOW, 3 = custom |
| 8 | stato bypass | 1 = bypass attivo |
| 9 | saturazione ADC | 1 = almeno un campione del frame in saturazione |
| 10 | overflow buffer | 1 = campioni persi immediatamente prima di questo frame |
| 11 | conversione VBUS continua | 1 = ogni ciclo dell'INA228 converte anche VBUS; entra nel periodo (vedi sotto) |
| 12 | pausa per misura di tensione | 1 = prima di questo frame il canale e' stato sospeso per una misura singola di VBUS (regola 11) |
| 13-15 | riservati | 0 |

**Periodo di campionamento.** Con VBUSCT fisso a **50 µs**:

```
periodo = (VSHCT + 50 µs x bit11) x AVG        (forma lineare, vedi regola 8)
```

Il bit 11 e' alzato quando VSHCT >= 540 µs, cioe' quando il tempo cieco dovuto alla conversione del bus resta sotto il 9%: DYNAMIC, SLOW e le combinazioni personalizzate lente. Sotto soglia (FAST e le combinazioni veloci) il bit 11 e' 0 e la tensione arriva dalle misure singole della regola 11.

Ogni frame porta la configurazione con cui e' stato prodotto. Il PC non deve mai dedurre il rate da una risposta a comando ricevuta in precedenza: se quella risposta si perde, il frame resta comunque interpretabile.

---

## 6. Comandi (PC verso device)

ASCII, maiuscolo, terminati da `\n`. Ogni comando produce un frame `0x03`.

| Comando | Argomenti | Effetto |
|---|---|---|
| `SET AVG <n>` | 1, 4, 16, 64, 128, 256, 512, 1024 | imposta l'averaging su entrambi i canali |
| `SET VSHCT <us>` | 50, 84, 150, 280, 540, 1052, 2074, 4120 | imposta il tempo di conversione shunt |
| `SET PRESET <name>` | FAST, DYNAMIC, SLOW | applica una combinazione predefinita |
| `GET CONFIG` | nessuno | restituisce la configurazione corrente, layout in 4.3 |
| `ZERO` | nessuno | azzeramento offset a ingresso aperto, emette poi un frame `0x04` |
| `CAL <ch> <ppm>` | canale (0 o 1), guadagno in parti per milione (1..4294967295, 1000000 = 1.0) | imposta la calibrazione, emette poi un frame `0x04`; 0 viene rifiutato |
| `START` | nessuno | avvia lo streaming |
| `STOP` | nessuno | ferma lo streaming |
| `DFU` | nessuno | riavvia in bootloader |

I preset di riferimento:

| Preset | VSHCT | AVG | Periodo nominale | Campioni per frame (capacita' 4.1, con `samples_per_frame` 64) | Uso |
|---|---|---|---|---|---|
| **FAST** | 50 µs | 1 | 50 µs, 20 kSPS; VBUS a misure singole ogni 10 s | 64 | durata e ampiezza degli spike |
| **DYNAMIC** | 1052 µs | 1 | 1102 µs lineari (con VBUS continua) | 64 | uso generale |
| **SLOW** | 4120 µs | 64 | 266.88 ms, circa 3.75 SPS (con VBUS continua) | 1 | baseline precise: media su molte conversioni |

I periodi indicati sono quelli lineari `(VSHCT + VBUSCT) x AVG` (sezione 5, regola 8).

SLOW era definito come VSHCT 4120 µs con AVG 1024 fino alla versione 0.4: un campione ogni 4.2 s e un frame da 64 campioni ogni quattro minuti e mezzo. AVG 1024 resta disponibile con `SET AVG 1024`, fuori dai preset; con VSHCT 4120 µs il frame contiene un campione e arriva ogni 4.2 s.

---

## 7. Regole vincolanti per il firmware

1. **Solo dati grezzi.** Nessuna conversione in microampere a bordo. Shunt e coefficienti possono cambiare, e una registrazione gia' convertita non e' piu' recuperabile.
2. **Nessun timestamp per campione.** Solo `t_start_us` per frame.
3. **Entrambi i canali sempre trasmessi, sempre completi.** Nessun multiplexing in funzione del bypass.
4. **Cambio di configurazione solo tra un frame e l'altro.** Alla ricezione di un comando che modifica il rate, il firmware chiude il frame in corso, scrive CONFIG, e il frame successivo parte con i nuovi `flags`. Mai due cadenze diverse dentro lo stesso frame. Il frame chiuso in anticipo contiene i soli campioni gia' acquisiti ed esce prima della risposta `0x03` al comando. Lo stesso vale per `STOP`.
5. **Primo campione dopo ogni riconfigurazione scartato**, perche' la conversione in corso al momento della scrittura di CONFIG non e' valida.
6. **`seq` sempre incrementale** per tutta la sessione, mai azzerato. E' il meccanismo con cui il PC rileva i frame persi.
7. **Overflow esplicito e quantitativamente coerente.** Se il buffer di trasmissione si riempie, il firmware scarta campioni, alza il bit 10 dei `flags` sul primo frame valido successivo e incrementa `samples_dropped`. Valgono queste tre condizioni, che devono essere sempre vere insieme:
   - `samples_dropped` conta **campioni**, non frame;
   - il salto di `t_start_us` attraverso un frame con il bit di overflow deve essere pari a `(campioni_del_frame_precedente + campioni_persi) x periodo_campione`;
   - un frame scartato per overflow **non** produce un buco in `seq`, perche' non e' mai stato messo in coda. Un buco in `seq` significa invece perdita sul link, che per contratto non dovrebbe mai accadere.

   La fonte autorevole per il software PC e' il salto di `t_start_us`, perche' localizza la perdita nel tempo. Il contatore in telemetria e' cumulativo e campionato a 1-10 Hz, quindi serve solo come verifica incrociata sul totale.
8. **Rate nominale dalla tabella del datasheet.** Il tempo di ciclo dell'INA228 non e' esattamente `VSHCT x AVG`: il datasheet dichiara circa 868 SPS a VSHCT 1052µs e AVG 1x, mentre la formula lineare darebbe 950 SPS. Il firmware deve usare i valori del datasheet, e il PC la stessa tabella. Un'implementazione che usa l'approssimazione lineare (ad esempio l'emulatore) e' accettabile per lo sviluppo ma deve dichiararlo, perche' introduce un errore di scala del 9% sulla ricostruzione dei tempi.
9. **Latenza massima di frame: 100 ms.** Un frame `0x01` si chiude quando e' pieno oppure quando il campione successivo cadrebbe 100 ms o piu' dopo il primo, quello che viene prima (sezione 4.1). Il firmware calcola la capacita' `min(samples_per_frame, ceil(100 ms / periodo))` a ogni cambio di configurazione, con lo stesso periodo nominale della regola 8, e chiude il frame al campione che la raggiunge. Nessun timer asincrono: la regola si applica ai timestamp dei campioni, quindi il numero di campioni per frame e' deterministico e il PC puo' verificarlo.
10. **Header di sessione periodico.** Il frame `0x04` viene emesso all'avvio, dopo ogni `ZERO` o `CAL`, e comunque ogni 5 s di tempo device a cadenza fissa, anche a streaming fermo. Le emissioni su evento non spostano la cadenza fissa.
11. **Tensione della batteria del DUT.**
   - Con VSHCT >= 540 µs entrambi gli INA228 convertono anche VBUS a ogni ciclo (bit 11, VBUSCT 50 µs), i due canali mantengono lo stesso periodo, e `v_bus_mv` in telemetria e' sempre letta dall'INA del **canale 0**, con `v_bus_flags` = 0. La configurazione cambia solo al cambio di preset, mai al cambio di range.
   - Sotto soglia la conversione VBUS continua e' spenta. Il firmware fa una **misura singola** di VBUS all'ingresso nella configurazione (cambio di preset o `START`) e poi ogni **10 s**, sempre sull'INA del **canale inattivo**: il canale attivo non si interrompe mai. Il frame in corso del canale inattivo si chiude con i campioni gia' acquisiti (come per la regola 4), il canale resta sospeso per la durata della misura, e il suo primo frame dopo la pausa porta il **bit 12**. La durata della pausa dipende dal firmware e il PC non la deve assumere.
   - La misura singola viene **rimandata**, al massimo di 1 s, se c'e' stata una transizione di range negli ultimi 100 ms: subito dopo una transizione i dati del canale inattivo servono integri per verificare il raccordo tra i range.
   - Tra una misura singola e l'altra la telemetria riporta l'ultimo valore misurato con `v_bus_flags` bit 0 = 1 ("tenuto"). Se non c'e' ancora nessuna misura, `v_bus_mv` = `0xFFFF`.
   - Si raccomanda un'isteresi nel comparatore che comanda il bypass: evita commutazioni ripetute con la corrente vicino alla soglia, che moltiplicherebbero le transizioni e quindi i rinvii delle misure.

---

## 8. Bilancio di banda

Caso peggiore, preset FAST su entrambi i canali:

| Voce | Valore |
|---|---|
| Rate per canale | 20 kSPS (VSHCT 50µs, AVG 1x) |
| Rate complessivo | 40 kSPS |
| Payload utile | 120 kB/s |
| Con overhead frame (8.6%) | circa 132 kB/s |
| Telemetria | trascurabile |
| Header di sessione ogni 5 s | 55 byte ogni 5 s, trascurabile |

Un USB CDC full speed regge comodamente questo carico.

Il vincolo vero e' l'I2C. Con registro VSHUNT a 24 bit e puntatore lasciato fermo, una lettura consecutiva costa circa 40 bit:

| Clock I2C | Campioni al secondo per bus |
|---|---|
| 400 kHz | 10 kSPS, insufficiente |
| 1 MHz (Fm+) | circa 25 kSPS |

Quindi **Fast-mode Plus a 1 MHz e' obbligatorio**, con i due INA228 su bus separati. Il margine sopra i 20 kSPS richiesti e' limitato, quindi vanno evitate riletture del puntatore di registro nel percorso critico.

La lettura del registro VBUS (regola 11) richiede di spostare il puntatore e poi riportarlo su VSHUNT: accade al massimo alla cadenza della telemetria (1 - 10 Hz), oppure una volta ogni 10 s sull'INA inattivo, quindi e' trascurabile.

---

## 9. Note per l'emulatore

L'emulatore deve essere indistinguibile dal dispositivo dal punto di vista del software PC:

- genera frame conformi a questa specifica su porta seriale virtuale (com0com su Windows, socat su Linux) o su file
- **riceve i comandi e li applica davvero**, cambiando il rate di generazione e rispondendo con frame `0x03`. Un emulatore che ignora i comandi produce un software PC sviluppato contro uno strumento che e' sempre d'accordo
- legge i comandi con una latenza indipendente dal rate di campionamento: un comando non deve mai aspettare la fine di un frame lento per essere letto
- genera scenari realistici: baseline stabile, spike da circa 0.5ms con ampiezza 0.3 - 0.7A, transizioni di range
- sa iniettare guasti: frame persi sul link (buchi in `seq`), CRC errato, overflow (senza buco in `seq`, con salto temporale coerente), saturazione
- produce un file golden di riferimento per i test di regressione del parser

Per la verifica del timing realistico, l'emulatore va poi portato su una board con USB CDC nativo (Raspberry Pi Pico o STM32F411). Un Arduino Pro Mini con FT232 non e' adatto: non regge la banda e soprattutto emula il trasporto sbagliato, perche' un UART reale con latency timer ha una frammentazione completamente diversa da un CDC.

---

## 10. Requisiti per il parser lato PC

- Deve saltare un frame di tipo sconosciuto usando `payload_len`, senza andare in errore.
- Deve distinguere le tre condizioni anomale: buco in `seq` (perdita sul link), bit di overflow con salto temporale (perdita a bordo), CRC errato (frame corrotto, da scartare senza perdere la sincronizzazione).
- Deve ricostruire i tempi da `t_start_us` e dal rate dichiarato nei `flags`, e segnalare quando il salto temporale non corrisponde ai campioni dichiarati. Il salto atteso tra due frame consecutivi dello stesso canale si calcola dal numero **effettivo** di campioni del frame precedente (`payload_len / 3`), mai da `samples_per_frame`: dalla 0.5 i frame hanno lunghezza variabile. Per i frame persi sul link (buco in `seq`) il numero di campioni non e' noto e si stima con la capacita' della sezione 4.1.
- Deve applicare calibrazione e offset presi dal frame `0x04` della sessione corrente, mai da valori cablati nel codice.
- Deve calcolare il periodo includendo VBUSCT quando il bit 11 e' alzato (sezione 5). Su un frame con il bit 12 il salto di `t_start_us` e' maggiore di quello atteso di una quantita' non prevedibile: non e' un'anomalia, ma il tempo in pausa va contato e reso disponibile. Un salto minore dell'atteso resta un'anomalia anche con il bit 12.
- Deve trattare `v_bus_mv` = `0xFFFF` come "non misurata" e, con `v_bus_flags` bit 0, non presentare come aggiornato un valore tenuto. La tensione va collocata nel tempo con `v_bus_t_us`, non con l'istante della telemetria.
- Finche' non ha ricevuto un frame `0x04` deve dichiarare lo stato **"in attesa dei parametri di sessione"**, visibile all'utente, e contare a parte i frame campioni ricevuti nel frattempo, che non sono convertibili. Non deve accumularli in silenzio tra i non convertibili ne' presentarli come un errore: collegarsi a strumento acceso e' la condizione normale, e l'attesa dura al massimo 5 s (regola 10). Se il `0x04` non arriva entro due periodi (10 s di tempo device) l'attesa diventa un'anomalia da segnalare.

---

## 11. Divisione delle responsabilita'

**Firmware**: campiona, incapsula, spedisce, riceve comandi. Nient'altro.

**Software PC**: stitching dei range, applicazione di calibrazione e offset, conversione in unita' fisiche, decimazione, grafici, analisi degli spike, salvataggio delle registrazioni.

Il motivo e' pratico: ogni modifica al firmware costa un ciclo di flash e un collaudo, mentre lato PC si modifica e si rilancia.

---

## 12. Storia delle modifiche

### Dalla versione 0.5 alla 0.6

1. Sezione 4.2: la telemetria passa a 22 byte con `v_bus_t_us` (offset 18), l'istante della misura di `v_bus_mv`. `v_bus_mv` e' la batteria del DUT letta dal VBUS dell'INA228, con il pin VBUS di entrambi gli INA228 collegato a IN (a monte dello shunt) invece che a valle. Non contiene piu' la caduta sullo shunt, e la corrente del pin VBUS non viene piu' misurata come corrente del DUT. `v_batt_mv` resta la cella dello strumento sull'ADC dell'STM32. Il byte a offset 9 della telemetria, prima riservato, diventa `v_bus_flags` (bit 0 = valore tenuto); `0xFFFF` = mai misurata.
2. Sezione 5: bit 11 (conversione VBUS continua) e bit 12 (pausa per misura di tensione). VBUSCT fisso a 50 µs; periodo `(VSHCT + 50 x bit11) x AVG`; conversione continua solo con VSHCT >= 540 µs. I periodi di DYNAMIC (1102 µs) e SLOW (266.88 ms) cambiano, FAST no.
3. Nuova regola 11: misure singole di VBUS sull'INA del canale inattivo all'ingresso in FAST e ogni 10 s, rinviate dopo una transizione di range; isteresi raccomandata nel comparatore del bypass.
4. Sezione 10: periodo con VBUSCT, pause del bit 12, valore di tensione tenuto o non misurato.
5. Il campo `proto_version` vale 6.

### Dalla versione 0.4 alla 0.5

1. Sezione 4.4 e nuova regola 10: il frame `0x04` viene riemesso a cadenza fissa ogni 5 s, oltre che all'avvio e dopo `ZERO` e `CAL`. Prima usciva solo in quei casi, quindi un PC collegato a strumento gia' acceso non lo riceveva mai e non poteva convertire nessun campione. I campi `preset`, `avg_code` e `vshct_code` riportano la configurazione al momento dell'emissione, non all'avvio.
2. Sezione 10: finche' non ha ricevuto un `0x04` il parser deve dichiarare lo stato "in attesa dei parametri di sessione" invece di accumulare in silenzio frame non convertibili; oltre 10 s di attesa e' un'anomalia.
3. Sezione 6: il preset SLOW diventa VSHCT 4120 µs con AVG 64 (263.68 ms, circa 3.8 SPS). Con AVG 1024 un campione arrivava ogni 4.2 s e un frame da 64 campioni ogni quattro minuti e mezzo. AVG 1024 resta disponibile con `SET AVG`, fuori dai preset. I tre preset sono ora dati in tabella con i valori numerici.
4. Sezione 4.1 e nuova regola 9: un frame `0x01` si chiude quando e' pieno oppure dopo 100 ms, misurati sui timestamp dei campioni: capacita' `min(samples_per_frame, ceil(100 ms / periodo))`. In FAST e DYNAMIC i frame restano da 64 campioni, in SLOW ne contengono uno. `samples_per_frame` del frame `0x04` e' dichiarato valore nominale massimo, non numero effettivo, che resta `payload_len / 3`. La regola 4 precisa che il frame chiuso in anticipo da un cambio di rate, o da `STOP`, contiene solo i campioni gia' acquisiti ed esce prima della risposta.
5. Sezione 10: il salto temporale atteso si calcola dai campioni effettivi del frame precedente; per i frame persi sul link si usa la capacita' della sezione 4.1.
6. Sezione 9: l'emulatore deve leggere i comandi con latenza indipendente dal rate.
7. Il campo `proto_version` vale 5.
8. Sezione 4.2: dichiarato che anche `v_bus_mv` e' letta dall'ADC dell'STM32 tramite partitore e non dal VBUS dell'INA228, con il motivo (la conversione VBUS dimezzerebbe il rate sullo shunt), e che e' misurata a valle dello shunt, quindi include la caduta `I x R_shunt`. Nessun cambiamento del wire format.

### Dalla versione 0.3 alla 0.4

1. Sezione 4.1: definito il layout del registro VSHUNT trasmesso. I 3 byte sono il registro grezzo, il risultato a 20 bit sta nei bit 23..4 e i bit 3..0 sono riservati a zero; lo shift aritmetico di 4, da applicare al valore gia' con segno a 24 bit, spetta al PC. Formula completa di conversione ed esempi numerici (saturazione `0x7FFFF0` = 1.6384 A sul canale alto, fondo scala 16.384 mA sul canale basso). Senza questa precisazione un PC che applicasse l'LSB ai 24 bit pieni leggerebbe correnti 16 volte piu' grandi. Gli offset di ZERO sono dichiarati in conteggi a 20 bit. Il campo `proto_version` vale 4.

### Dalla versione 0.2 alla 0.3

1. Definita la semantica di `cal_low` e `cal_high` (sezione 4.4): guadagni in parti per milione, 1000000 = 1.0, con la formula di conversione che il software PC deve applicare. Prima erano solo "coefficiente di calibrazione", senza dire come si applica ne' quale valore rappresenta il guadagno unitario. Il comando `CAL` (sezione 6) e il payload di `GET CONFIG` (sezione 4.3) sono aggiornati di conseguenza; il valore 0 e' rifiutato. Il campo `proto_version` del frame `0x04` vale 3.

### Dalla versione 0.1 alla 0.2

1. Il campo a offset 12 dell'header era `n_samples` (numero di campioni). Ora e' `payload_len` in byte, valido per tutti i tipi di frame. Senza di esso un ricevitore non puo' saltare un frame di tipo sconosciuto.
2. `shunt_low_nohm` e `shunt_high_nohm` diventano `shunt_low_uohm` e `shunt_high_uohm`, in micro ohm. Il valore di 10 ohm non entrava in un uint32 espresso in nano ohm.
3. L'algoritmo di CRC, prima non specificato, e' ora definito come CRC-16/CCITT-FALSE con vettore di verifica.
4. L'endianness, prima implicita, e' ora dichiarata esplicitamente, inclusa l'eccezione dei 3 byte di VSHUNT.
5. Il payload di `GET CONFIG`, prima non definito, e' ora specificato.
6. Il payload del frame `0x04` e' stato completato con `adcrange`, `avg_code`, `vshct_code` e `samples_per_frame`, e la sua lunghezza e' ora fissata a 37 byte.
7. La regola 7 sull'overflow, prima generica, ora impone la coerenza tra `samples_dropped`, il salto di `t_start_us` e la continuita' di `seq`, e indica quale dei due dati e' autorevole.
8. Aggiunta la regola 8 sul rate nominale da datasheet.
9. Aggiunta la sezione 10 con i requisiti per il parser lato PC.