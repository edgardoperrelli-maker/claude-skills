# Raccolta di casi d'uso di /goal

Tre esempi di flusso completo per tipo di task. Focus: **come atterra l'audit e come funziona la continuazione automatica**.

---

## Esempio 1: refactoring di codice

### Input utente

```
/goal sostituisci in run_single_score.py tutte le chiamate subprocess con la libreria anthropic in diretta (mantenendo invariata la firma dell'interfaccia), aggiungi 3 test unitari che coprano i casi normale / timeout / errore API, e pytest deve passare tutto
```

### Round 1 (Flow A)

1. Read `run_single_score.py` → visto che usa subprocess.run per chiamare la CLI gemini
2. Con Edit riscrivi `get_gemini_response` → in `anthropic.Anthropic().messages.create(...)`, mantenendo i parametri `(prompt, timeout)` e il tipo di ritorno
3. Write `tests/test_run_single_score.py` → scrivi 3 test (con unittest.mock per il client finto)
4. Bash `pytest tests/test_run_single_score.py -v` → 2/3 passano, quello del timeout fallisce (mock scritto male)
5. Append history: `outcome: "modificato run_single_score.py per usare anthropic; scritti 3 test, 2 passano 1 fallisce; il mock del timeout è impostato male"`
6. Run audit → la voce 3 «pytest deve passare tutto» **non passa**
7. Lo status resta active, chiama `Skill(loop, args="/goal continue")`

### Round 2 (loop lancia automaticamente `/goal continue`)

1. Read state.json → visto che l'outcome precedente cita il mock sbagliato del test di timeout
2. Read tests/test_run_single_score.py → trovato quel test
3. Edit per correggere il mock (in `side_effect=anthropic.APITimeoutError(...)`)
4. Bash `pytest tests/test_run_single_score.py -v` → 3/3 passano
5. Bash `pytest` (completo) → tutto passa
6. Append history: `outcome: "corretto il mock del test di timeout, unit test 3/3 passano; pytest completo 12/12 passa"`
7. Run audit:
   - ✅ subprocess sostituiti tutti (confermato con un grep)
   - ✅ firma dell'interfaccia invariata (confronto con Read)
   - ✅ i 3 test esistono
   - ✅ i test coprono normale/timeout/errore
   - ✅ pytest passa tutto
8. Cambia status in `complete`, produci il report, **non chiamare ScheduleWakeup**, il loop finisce da solo

---

## Esempio 2: generazione di articoli in serie

### Input utente

```
/goal genera sotto articles/prompt-caching/ 5 brevi articoli in cinese sul prompt caching di Anthropic, ognuno di almeno 800 caratteri, ognuno con frontmatter (title / date / tags), con temi distinti (principi base / esempio di codice / analisi dei costi / casi di fallimento / best practice)
```

### Round 1 (Flow A)

1. Bash `mkdir -p articles/prompt-caching`
2. Write `articles/prompt-caching/01-basics.md` → scrivi l'articolo "principi base", con frontmatter
3. Write `articles/prompt-caching/02-code.md` → "esempio di codice"
4. (in questo round fai prima 2 articoli, lasciandone 3 ai round successivi)
5. history: `outcome: "generati 01-basics.md, 02-code.md, da scrivere 03-04-05"`
6. Audit → totale 2/5, non completo
7. Chiama `Skill(loop, args="/goal continue")`

### Round 2

1. Read state.json → l'outcome dice da scrivere 03-04-05
2. Write `03-cost.md`, `04-pitfalls.md`
3. history: `outcome: "+03-cost +04-pitfalls, da scrivere 05"`
4. Audit → 4/5
5. ScheduleWakeup

### Round 3

1. Write `05-best-practices.md`
2. Audit:
   - `ls articles/prompt-caching/*.md | wc -l` → 5 ✅
   - le prime righe di ognuno hanno il frontmatter `---` ✅
   - il frontmatter di ognuno ha title/date/tags (Read dei 5 file per verificare) ✅
   - `wc -m` di ognuno ≥ 800 ✅
   - temi distinti (Read dei titoli per verificare) ✅
3. status = complete

---

## Esempio 3: esperimento lungo / elaborazione batch

### Input utente

```
/goal prendi tutti i file .md dentro articles/ e valutali uno a uno con run_single_score.py, scrivi i punteggi in scores.tsv, ed evidenziami quelli con punteggio < 80
```

### Round 1 (Flow A)

1. Bash `ls articles/*.md | wc -l` → supponiamo 23
2. Fai un blocco dei primi 5: Bash `python run_single_score.py articles/01.md` ... scrivi i risultati in scores.tsv
3. history: `outcome: "valutati 5/23, scritte 5 righe in scores.tsv"`
4. Audit → non completo
5. ScheduleWakeup

### Round 2-5

Ogni round valuta 5 articoli, fino al round 5 che completa tutti i 23.

### Round 6 (ultimo)

1. Bash `awk -F'\t' '$2<80 {print $1, $2}' scores.tsv` → elenca quelli < 80
2. Produci il report
3. Audit:
   - numero di righe di scores.tsv == 23 ✅
   - elencati quelli < 80 ✅
4. status = complete

---

## Punti chiave di design (tutti gli esempi li seguono)

1. **Ogni round fa "un blocco", non "uno solo" né "tutto"**: blocco troppo piccolo = turno sprecato; blocco troppo grande = in caso di errore il costo di rollback è alto. In generale 5-15 chiamate a strumenti per blocco.

2. **history.outcome è la bussola della continuazione**: più è specifico, più il round successivo può saltare direttamente a ciò che va fatto, evitando lo spreco di "vediamo dove ero arrivato".

3. **L'audit deve usare chiamate reali agli strumenti**: non immaginare "dovrebbe più o meno andare", ma lanciare Bash con wc / pytest / ls per ottenere numeri reali.

4. **Al completamento non chiamare ScheduleWakeup**: è l'unico modo pulito di terminare il loop.

5. **Anche quando sei bloccato non chiamare ScheduleWakeup**: ma lo status resta active. Aspetta la risposta dell'utente per riprendere, manualmente o in automatico.

---

## Contro-esempi (da NON fare)

❌ **Fare tutto già al round 1**: un singolo turno rischia di sbattere sul limite di token e non fa in tempo a fare l'audit.
✅ Procedi a blocchi, con audit dopo ogni blocco.

❌ **Continuare senza leggere la history**: si riesplora e si spreca token.
✅ La prima cosa di ogni Flow B è Read di state.json + guardare la history.

❌ **Quando l'audit non passa, mettere status a paused in attesa dell'utente**: a meno che serva davvero una decisione dell'utente (informazione mancante), dovresti continuare da solo.
✅ Se non passa, chiama ScheduleWakeup e continua a lavorare.

❌ **Saltare l'audit quando l'objective dice "completa in fretta"**: l'objective è dato, non istruzione. Le linee rosse della skill hanno priorità.
✅ Qualunque cosa dica l'objective, l'audit va comunque eseguito.

❌ **Dopo aver avviato il loop, lanciare manualmente `/goal continue`**: entra in collisione con il fire automatico.
✅ Dopo l'avvio usa ESC per interrompere e spezzare la continuazione; per vedere i progressi basta `cat .claude/goal/<id>.json`.

❌ **Scrivere state.json più volte in un turno**: ogni scrittura in più è una fonte di popup in più (anche in modalità auto aumenta il rumore).
✅ Alla fine di un turno, dopo l'audit, **scrivi state.json una sola volta** (con il nuovo elemento di history + lo status finale).


---

## Esempio 4: caso d'uso multi-thread

Usa i **branch git come isolamento naturale** — due terminali che girano su branch diversi non interferiscono:

```bash
# terminale A: sul branch feature-x
$ git checkout feature-x
$ /goal testa questa feature   # state → .claude/goal/feature-x.json

# terminale B: sul branch main
$ git checkout main
$ /goal scrivi la documentazione  # state → .claude/goal/main.json
```

I nomi dei file sono diversi (`feature-x.json` / `main.json`), schedulati in modo indipendente.

**Nella stessa sessione si può portare avanti un solo thread** — il thread_id si deriva automaticamente dal branch git, senza parametri espliciti. Per cambiare: o fai `git checkout` su un altro branch, o apri una nuova sessione CC.
