# Note su Tamarin Prover

## 1. Idea centrale

Tamarin è uno strumento simbolico per la modellazione e l'analisi di protocolli di sicurezza. Funziona descrivendo il comportamento del protocollo come un insieme di **regole di riscrittura di multinsiemi** (multiset rewriting rules), le capacità dell'attaccante e le proprietà che devono essere verificate. Lo stato del protocollo è rappresentato simbolicamente, insieme ai messaggi di rete, ai valori freschi (*fresh*) e alla conoscenza dell'avversario.

Tamarin supporta la ricerca di prove sia automatica che interattiva. Quando l'automazione non è sufficiente, l'interfaccia grafica (GUI) aiuta a ispezionare gli stati della prova, i grafi degli attacchi e i vincoli intermedi.

---

## 2. Primitive crittografiche e Built-ins

```tamarin
builtins: hashing, asymmetric-encryption, symmetric-encryption, signing, diffie-hellman
```

Questa riga dichiara le primitive crittografiche integrate utilizzate nel modello. Tamarin gestisce automaticamente le teorie equazionali per queste:
* **Hashing:** Una funzione unaria `h(x)`.
* **Crittografia asimmetrica:** `aenc(m, pk)`, `adec(c, sk)` e `pk(sk)` per la chiave pubblica.
* **Crittografia simmetrica:** `senc(m, k)` e `sdec(c, k)`.
* **Firma digitale:** `sign(m, sk)` e `verify(sig, m, pk(sk))`. La funzione `true` viene spesso utilizzata per rappresentare una verifica riuscita.
* **Diffie-Hellman:** Abilita l'elevamento a potenza simbolico (es. `g^~x`), fondamentale per protocolli di scambio chiavi come TLS o IKE.

Un'equazione chiave della teoria della crittografia asimmetrica integrata è che la decrittazione di un testo cifrato con la chiave privata corretta restituisce il testo in chiaro originale.

### Teorie Equazionali Personalizzate (Custom Equational Theories)
A volte i built-ins non sono sufficienti. Puoi definire le tue equazioni matematiche personalizzate in cima al file usando la parola chiave `equations:`.
```tamarin
equations: f(g(x)) = x
```
Tuttavia, per la stragrande maggioranza dei protocolli standard, le primitive integrate bastano e avanzano.

---

## 3. Modellare una Public Key Infrastructure (PKI)

In Tamarin, il protocollo e il suo ambiente sono modellati con regole sui fatti (*facts*). Ad esempio:
* `Out(m)` significa che il messaggio `m` viene inviato sul canale pubblico.
* `In(m)` significa che il messaggio `m` viene ricevuto dal canale pubblico.

Una tipica regola PKI è:

```tamarin
rule Register_pk:
  // Generate a fresh long-term private key
  [ Fr(~ltk) ]
-->
  // Add persistent facts to the state
  [ !Ltk($A, ~ltk), !Pk($A, pk(~ltk)) ]
```

Ciò significa che quando viene generata una nuova chiave privata a lungo termine `~ltk`, Tamarin aggiunge allo stato i fatti persistenti `!Ltk($A, ~ltk)` e `!Pk($A, pk(~ltk))`. I fatti persistenti rimangono disponibili per future applicazioni delle regole.

---

## 4. Notazione e Sintassi dei Messaggi

* `~x` indica `x:fresh` (valore fresco)
* `$x` indica `x:pub` (valore pubblico)
* `%x` indica `x:nat` (numero naturale)
* `#i` indica `i:temporal` (punto temporale)
* `m` indica `m:msg` (messaggio generico)

**Tuple di messaggi:**
Quando è necessario raggruppare più dati in un unico messaggio (es. invio di un ID, un nonce e un testo cifrato), si usano le parentesi angolari per creare una tupla: `<A, ~n, aenc(m, pkB)>`.

**Intuizione utile:**
* `~x` è un valore fresco, solitamente creato da `Fr(~x)`.
* `$x` è un valore pubblico.
* `%x` è un numero naturale.
* `#i` è una variabile del punto temporale.
* `m` è un termine di messaggio generale.

**Inoltre:**
* `!` prima di un fatto significa che il fatto è persistente.
* `@ #i` significa che un fatto o un'azione è valida nella posizione della traccia `#i`.
* `#i < #j` significa che l'evento `i` accade prima dell'evento `j`.

---

## 5. Sort e costanti

Tamarin utilizza un sort principale `msg`, con subsort come `fresh`, `pub` e `nat`. Una costante come `'c'` indica un nome pubblico. Le variabili temporali appartengono al sort `temporal` e sono usate per ragionare sull'ordinamento delle tracce.

---

## 6. Fact, term, sostituzione, matching, unificazione, rewriting

### Definizioni

* **Fact**: Un *fact* è come un "contenitore di stato" o un predicato simbolico. Serve per memorizzare informazioni durante l’esecuzione del protocollo: ad esempio chi ha una chiave, cosa è stato inviato, cosa è stato ricevuto, o quali condizioni valgono in un certo momento. I fatti possono essere lineari (consumabili) oppure persistenti.
* **Term**: Un *term* è un messaggio simbolico costruito con costanti, variabili e funzioni. Per esempio, `h(k)`, `pk(~ltk)` e `aenc(m, pk(sk))` sono termini. In Tamarin, i messaggi non sono numeri o stringhe concrete, ma strutture simboliche.
* **Sostituzione**: Una sostituzione di variabili con termini. Ad esempio, se `x ↦ aenc(m, pk(sk))`, ogni occorrenza di `x` viene rimpiazzata con quel termine. Le sostituzioni servono per istanziare regole, fatti e vincoli.
* **Matching**: Il processo di verificare se un pattern può essere reso uguale a un termine tramite una sostituzione appropriata, senza dover cambiare liberamente entrambe le parti. In pratica si cerca di adattare un pattern a un termine.
* **Unificazione**: Più generale del matching: cerca una sostituzione che renda uguali due termini. Se il matching è "pattern contro termine", l'unificazione è "termine contro termine" e cerca una soluzione comune. Se esistono più variabili, la soluzione può non essere unica.
* **Rewriting**: La trasformazione di un termine o di uno stato usando una regola. Le regole descrivono come lo stato evolve: se i fatti nel lato sinistro della regola sono presenti, la regola può scattare e produrre i fatti del lato destro.
* **Multiset rewriting**: Il meccanismo di base di Tamarin: lo stato è un multinsieme di fatti, non un insieme ordinato. Applicare una regola significa consumare alcuni fatti e produrne altri.

### Versione breve da ricordare
* **facts** = stato
* **terms** = messaggi simbolici
* **substitutions** = variabili rimpiazzate da termini
* **matching** = adattare un pattern a un termine
* **unification** = trovare una sostituzione che renda uguali due espressioni
* **rewriting** = applicare una regola per trasformare lo stato

---

## 7. Regole: State Facts vs. Action Facts

Le regole descrivono le transizioni di stato e sono scritte come:

```tamarin
rule RuleName:
  [ Premises ] // State Facts (Fatti di Stato)
--[ Labels ]->   // Action Facts (Fatti di Azione)
  [ Conclusions ] // State Facts (Fatti di Stato)
```

**La differenza fondamentale:**
* **State Facts (Premesse/Conclusioni):** Stanno dentro le parentesi quadre `[ ]`. Esistono nello stato e vengono consumati (se lineari) o restano (se persistenti). *Non puoi usarli direttamente nei lemmi.*
* **Action Facts (Etichette):** Stanno dentro i trattini `--[ ]->`. Non fanno parte dello stato, ma vengono registrati nella "traccia" (la storia del protocollo). *I lemmi possono leggere SOLO questi fatti.*

```tamarin
rule Send_Message:
  [ Fr(~n) ]                    // State Fact (consumed)
--[ Send($A, ~n) ]->            // Action Fact (logged in the trace)
  [ Out(< $A, ~n >), State(~n) ] // State Facts (produced)
```

---

## 8. Restrizioni

Le regole definiscono cosa *può* succedere, le restrizioni definiscono cosa è *permesso* che esista in una traccia valida. Sono fondamentali per filtrare comportamenti indesiderati che non hanno senso logico. Se una traccia viola una restrizione, Tamarin la scarta.

**Esempio classico (Verifica di uguaglianza):**
Spesso usato per verificare firme, MAC o pattern matching rigidi.

```tamarin
restriction Equality:
  "All x y #i. Eq(x,y) @i ==> x = y"
```
*(Questa restrizione richiede che ci sia una regola con l'Action Fact `--[ Eq(a, b) ]->`)*

### Embedded Restrictions (Restrizioni incorporate)
Oltre alle restrizioni globali, le versioni moderne di Tamarin permettono di usare **restrizioni incorporate** direttamente all'interno degli Action Fact di una regola tramite la parola chiave speciale `_restrict()`. Questo evita di dover definire una restrizione globale e un fatto `Eq` per controlli semplici.

```tamarin
rule B_receive_embedded:
  [ In(<m, sig>), !Pk(A, pkA) ]
--[ 
    Recv(m),
    _restrict(verify(sig, m, pkA) = true) // Restrizione incorporata!
  ]->
  [ State(m) ]
```
Se la condizione all'interno di `_restrict()` risulta falsa, la regola semplicemente non può scattare e la traccia viene scartata, esattamente come una restrizione globale, ma con un codice molto più compatto.

---

## 9. Lemmi e Induzione

Un lemma enuncia una proprietà che Tamarin deve provare o confutare. I lemmi sono proprietà delle tracce: parlano di ciò che deve o non deve accadere in qualsiasi traccia di esecuzione.

**Simboli logici tipici:**
* `Ex` = esiste
* `All` = per ogni
* `&` = and
* `|` = or
* `not(...)` = negazione
* `==>` = implicazione

**Induzione:**
Quando scrivi lemmi su fatti persistenti o su protocolli con cicli infiniti (es. chiavi aggiornate continuamente), il prover di base spesso va in loop. Aggiungendo `[use_induction]`, Tamarin proverà a dimostrare il lemma per induzione.

```tamarin
lemma My_Security_Property [use_induction]:
  "All x #i. Action(x) @ #i ==> (Ex #j. OtherAction(x) @ #j & #j < #i)"
```

---

## 10. Modello dell'avversario

Per impostazione predefinita, Tamarin assume un avversario di tipo Dolev–Yao: l'attaccante controlla la rete e può intercettare, eliminare, modificare e iniettare messaggi, pur essendo limitato dalle equazioni crittografiche simboliche del modello.

---

## 11. Frecce e colori della GUI

Nella GUI, Tamarin utilizza diversi stili e colori di frecce per visualizzare il grafo della prova.

### Frecce solide (continue)
* **Frecce solide nere o grigie:** origini dei fatti del protocollo, per fatti lineari o persistenti.
* **Frecce solide rosse/arancioni:** passaggi in cui l'avversario estrae valori da un messaggio ricevuto.

### Frecce tratteggiate
Le frecce tratteggiate rappresentano vincoli di ordinamento tra le azioni:
* **Tratteggio nero:** vincolo derivante dalle formule, ad esempio da un lemma o da una restrizione.
* **Tratteggio blu scuro:** vincolo causato da un valore fresco (*fresh*).
* **Tratteggio rosso:** vincolo derivante dai passaggi di composizione dell'avversario.
* **Tratteggio arancione scuro:** vincolo implicato dalle condizioni di forma normale di Tamarin.
* **Tratteggio viola:** vincolo originato da un'istanza di un fatto iniettivo.

> *Nota: Una freccia può avere più colori se si applicano contemporaneamente diversi vincoli.*

### Frecce punteggiate
* **Frecce punteggiate verdi:** passaggi di deduzione dell'avversario incompleti durante la ricerca della prova.

### Semplificazione del grafo
Tamarin può semplificare i grafi nascondendo alcune regole/frecce. La GUI offre diversi livelli di semplificazione e il menu "Options" permette di modificare il livello di dettaglio mostrato.

---

## 12. Workflow utile

1. Definire le primitive crittografiche (`builtins`).
2. Modellare lo stato e il comportamento del protocollo con le regole.
3. Definire le `restrictions` per filtrare gli stati non validi.
4. Enunciare le proprietà desiderate come `lemmas`.
5. Eseguire il prover automatico:
   ```bash
   tamarin-prover Protocol.spthy --prove
   ```
6. Ispezionare la GUI se la prova si blocca o se viene trovato un attacco.

---

## 13. Suggerimenti e Best Practices (Extra)

* **Sanity Checks (Lemmi Esistenziali):** Prima di provare la sicurezza, prova che il protocollo *possa funzionare*. Scrivi un lemma con `exists-trace`. Se Tamarin non trova una traccia, le tue regole potrebbero bloccarsi a vicenda.
  ```tamarin
  lemma executable:
    exists-trace
    "Ex A B m #i #j. Send(A, m) @ #i & Receive(B, m) @ #j & #i < #j"
  ```
* **I 4 Livelli di Autenticazione (Gerarchia di Lowe):** Quando modelli l'autenticazione, devi sapere a quale livello di sicurezza punti.
  1. **Aliveness (Presenza):** Se B finisce il protocollo, sa che A è "viva" e ha eseguito il protocollo di recente. *Problema:* A potrebbe aver parlato con C, non con B!
  2. **Weak Agreement (Accordo Debole):** B sa che A è viva e che A intendeva parlare proprio con B. *Problema:* Potrebbero non essere d'accordo su quale sia il ruolo di ciascuno o sui dati scambiati.
  3. **Non-injective Agreement (Accordo Non-Iniettivo):** A e B sono d'accordo sulle reciproche identità e su tutti i dati scambiati (es. la chiave di sessione). *Problema:* Non protegge dai *replay attack* (un messaggio di A potrebbe essere inviato 5 volte a B).
  4. **Injective Agreement (Accordo Iniettivo):** Il livello massimo. Aggiunge l'unicità (1-a-1). A ogni sessione completata da B corrisponde *esattamente una e una sola* sessione iniziata da A.
* **Chiavi e Freshness:** Usa sempre il prefisso `~` per chiavi di sessione e nonce (es. `~k`). Se usi variabili pubbliche `$k`, l'avversario ne avrà immediatamente il controllo.
* **Evita regole troppo grandi:** Se una regola fa troppe cose (riceve, decritta, genera chiavi e invia contemporaneamente), il prover fatica. Mantieni i passaggi atomici.
* **Action Facts per la Segretezza:** Emetti un Action Fact `--[ Secret(x) ]->` quando viene calcolata una chiave, e usa un lemma per verificare che l'avversario (`K(x)`) non possa mai conoscerla.
  ```tamarin
  lemma secrecy:
    "All x #i. Secret(x) @ i ==> not (Ex #j. K(x) @ j)"
  ```

---

## 14. Let Bindings e Macro Globali

Quando un termine complesso si ripete più volte all'interno della stessa regola, puoi usare il costrutto `let ... in` per creare delle macro locali, migliorando la leggibilità.

```tamarin
rule MyRuleName:
  let
    foo = h(~x, y)
  in
  [ In(foo) ] --> [ Out(foo) ]
```

Se desideri utilizzare le stesse macro in più regole, lemmi o restrizioni, puoi definirle a livello globale tramite la keyword `macros:`.

---

## 15. Annotazioni dei Lemmi ([reuse] e [sources])

Oltre a `[use_induction]`, esistono altre annotazioni fondamentali:
* **`[reuse]`**: Un lemma con questa annotazione verrà utilizzato nelle dimostrazioni di tutti i lemmi successivi. Utile per dimostrare proprietà intermedie.
* **`[sources]`**: Essenziale quando Tamarin fallisce a causa di loop infiniti durante la risoluzione delle origini dei fatti (*partial deconstructions*). I *sources lemmas* aiutano a raffinare le origini dei fatti durante la precomputazione.

---

## 16. Equivalenza Osservazionale (Privacy)

I lemmi classici ragionano sulle singole tracce. Per proprietà di privacy o indistinguibilità, Tamarin supporta l'*Observational Equivalence*. Permette di provare che un intruso non può distinguere tra due istanze di un sistema. Si modella usando l'operatore `diff(x, y)` per i termini che differiscono tra le istanze e richiede l'avvio con il flag `--diff`.

---

## 17. Altri Built-in Utili

Il manuale fornisce altri built-in essenziali per la modellazione avanzata:
* **`xor`**: Modella l'operazione di OR esclusivo (XOR) con le relative equazioni di cancellazione.
* **`multiset`**: Introduce l'operatore `++` per modellare i multinsiemi.
* **`natural-numbers`**: Definisce la costante `%1` e l'operatore `%+` per implementare contatori di stato.

---

## 18. Calcolo dei Processi (SAPIC+)

Invece di scrivere regole di riscrittura di multinsiemi frammentate, SAPIC+ (Stateful Applied PI-Calculus) ti permette di modellare un protocollo come un flusso sequenziale, in modo simile alla programmazione classica. Tamarin compilerà poi automaticamente il processo nelle regole di riscrittura di multinsiemi sottostanti.

Questa è la sintassi di base per definire i processi (`P`, `Q`):

### Costrutti Base
* `new ~n; P` : Genera un valore fresco `~n`.
* `out(t); P` : Invia il termine `t` sul canale pubblico.
* `in(x); P` : Riceve un messaggio dal canale pubblico e lo lega alla variabile `x`.
* `out(c, t); P` : Invia `t` su un canale privato/autenticato `c`.
* `in(c, x); P` : Riceve da un canale `c` e lo lega a `x`.
* `if cond then P else Q` : Ramo condizionale (es. per verificare se due valori corrispondono).
* `let x = t in P else Q` : Pattern matching o assegnazione.
* `P | Q` : Esecuzione parallela (i due processi procedono simultaneamente).
* `!P` : Replicazione (genera infinite istanze concorrenti di `P`, essenziale per modellare sessioni multiple).
* `0` : Processo nullo (termina il ramo).

### Gestione dello Stato Globale (Stateful)
A differenza del tradizionale Pi-Calculus, SAPIC+ permette di gestire database chiave-valore condivisi tra i processi, il che è fondamentale per modellare logiche complesse come le revoche dei certificati o i contatori:
* `insert k, v; P` : Inserisce o aggiorna il valore `v` associato alla chiave `k`.
* `lookup k as x in P else Q` : Cerca la chiave `k`. Se esiste, lega il suo valore a `x` e continua in `P`, altrimenti esegue `Q`.
* `delete k; P` : Rimuove la chiave `k` dallo stato globale.

### Eventi e Sincronizzazione
* `event F; P` : Emette un Action Fact `F` esattamente nel momento in cui il processo raggiunge quel punto (necessario per i lemmi e il tracciamento della cronologia).
* `lock t; P` e `unlock t; P` : Meccanismo di mutua esclusione per evitare *race conditions* quando più processi tentano di accedere allo stesso stato globale in parallelo.

### Esempio Pratico

```tamarin
process:
!(
  // Avvia una nuova sessione con una chiave fresca
  new ~k;
  
  // Emette un evento per i lemmi di sicurezza
  event SessionStarted(~k);
  
  // Salva la chiave nel database globale con un identificatore pubblico
  insert <'session_key', $A>, ~k;
  
  // Invia la chiave cifrata con la chiave pubblica di B
  out(aenc(~k, pkB));
  
  // Termina l'istanza di questo processo
  0
)