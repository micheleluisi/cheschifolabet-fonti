# Metodologia di generazione quote per IFHL (Che Schifo la Bet)

## 1. Contesto del progetto

IFHL è una lega di fantahockey (fantasy NHL) in cui ogni "fantasquadra" possiede un roster di giocatori NHL reali. Ogni giornata IFHL corrisponde a un periodo di calendario NHL reale (tipicamente lun-giov o ven-dom). Il punteggio di una fantasquadra in una giornata è la somma dei fantapunti ottenuti dai suoi titolari nelle partite NHL realmente giocate in quel periodo.

L'obiettivo è generare quote scommesse (stile bookmaker) per ogni partita della giornata IFHL, seguendo un template JSON fisso con questi mercati per ogni incontro:

- `winnerAway`, `winnerHome`: vincente 1X2 (esito diretto, senza pareggio)
- `under90`, `over90`, `under125`, `over125`, `under150`, `over150`, `under175`, `over175`: soglie sul totale combinato punti delle due squadre
- `spreadNeg10`, `spread10_20`, `spread20_30`, `spread30Plus`: fasce sul margine di scarto assoluto tra le due squadre

Il formato di output è un JSON con struttura:
```json
{
  "giornata": N,
  "tipo_giornata": "lunga" | "corta",
  "partite": [
    {
      "awayTeam": "...", "homeTeam": "...",
      "quote": { "winnerAway": 0, "winnerHome": 0, ... }
    }
  ]
}
```

## 2. Regole di scoring rilevanti (da regolamento IFHL)

- Formazione titolare: 1 portiere, 6 difensori, 12 attaccanti (4 LW, 4 C, 4 RW), su 3 linee di difesa e 4 di attacco.
- **Tetto sulle partite conteggiate (MAX GP)**: nel periodo di gioco si contano al massimo le prime 2 partite NHL giocate da ogni skater titolare, e **solo la prima** per il portiere. Questo è cruciale per il calcolo delle proiezioni: se una squadra NHL gioca 3 partite nel periodo, i suoi giocatori valgono comunque al massimo 2 partite (1 per i portieri).
- Scoring per categoria (punti per evento): gol, assist primari/secondari, tiri in porta parati, plus/minus, PIM, ecc., con pesi diversi per ruolo e linea (es. gol di un difensore di 4a linea vale più di un gol di un attaccante di 1a linea, per bilanciare la scarsità).

## 3. Fonti dati

Tutte le fonti risiedono in una repository GitHub (`cheschifolabet-fonti`), organizzata così:

```
/
├── Fonti fisse/
│   ├── Estratto REGOLAMENTO IFHL [stagione].docx
│   └── Stats giocatori stagione [anno].xlsx   <- fantapunti/game stagione NHL PASSATA, per giocatore
├── Roster automatici/
│   └── [NomeFantasquadra].csv                  <- roster completo (titolari + riserve) con Pos, Player, Team NHL, Fantasy Points, Average FP/Game
├── _Classifica IFHL.xlsx                        <- classifica fantalega corrente
├── classifica NHL.csv                           <- classifica NHL reale corrente (GP, W, L, OT, PTS, P%, GF, GA, ecc.)
├── giornata [N].csv                             <- calendario NHL reale per il periodo della giornata IFHL N (colonne: Away, Home, Orario, raggruppate per data)
├── risultati IFHL.csv                           <- punteggi reali di ogni fantasquadra per ogni giornata IFHL già disputata
└── giornata_[N]_quote_template.json             <- template vuoto da popolare (quote a 0)
```

Il flusso operativo previsto: l'utente aggiorna questi file su GitHub prima di ogni richiesta; il modello legge sempre la versione più recente al momento della richiesta, senza bisogno di ricaricare manualmente i dati.

## 4. Evoluzione della metodologia (importante per capire le scelte finali)

### 4.1 Versione 1 — Solo storico IFHL
Prima iterazione: forza della fantasquadra = media dei punteggi ottenuti nelle giornate IFHL già giocate, con deviazione standard blended (50% std specifica squadra + 50% std di lega) per compensare il campione ridotto (2 giornate). Limite riconosciuto: con solo 2 partite di storico, il segnale è troppo rumoroso e non distingue chi ha un roster oggettivamente forte da chi ha avuto un calendario NHL favorevole per puro caso.

### 4.2 Versione 2 — Blend con talento roster (pesi fissi)
Introdotto un punteggio di "talento" = media aritmetica dei fantapunti/game (stagione NHL passata) dei 18 titolari + portiere di ogni fantasquadra. Combinato con:
- Talento roster: 70%
- Storico IFHL: 20%
- Difficoltà media avversari NHL (calendario reale incrociato con classifica NHL): 10%

Ciascuna componente normalizzata a z-score prima di combinarla. Limite riconosciuto dall'utente: pesi fissi arbitrari, aggregazione troppo precoce a livello di fantasquadra, non catturano l'effetto "giornata specifica" (quale roster ha un calendario NHL favorevole *in questo turno* rispetto alla propria norma).

### 4.3 Versione 3 (finale, corrente) — Proiezione bottom-up giocatore per giocatore
Approccio riprogettato su richiesta esplicita dell'utente, per rispecchiare come un vero analista valuterebbe la giornata: proiettare ogni giocatore titolare individualmente, poi sommare.

## 5. Modello finale: proiezione giocatore per giocatore

### Passo 1 — Determinare partite nel periodo per ogni giocatore
Per ogni squadra NHL, dal file `giornata [N].csv` si contano le partite disputate nel periodo (lun-giov o ven-dom), determinando anche gli avversari di ciascuna partita. Si applica poi il cap di regolamento:
- Skater: `n_partite_contate = min(partite_reali_nel_periodo, 2)`
- Goalie: `n_partite_contate = min(partite_reali_nel_periodo, 1)`

Se una squadra gioca 3 volte nel periodo, si prendono solo le prime 2 (skater) o la prima (goalie) in ordine cronologico, e si registrano i relativi avversari.

### Passo 2 — Baseline individuale
Per ogni titolare: `fp_baseline = Average Fantasy Points per Game` dalla stagione NHL passata (colonna già presente nei roster CSV / file Stats giocatori).

### Passo 3 — Fattore avversari (matchup specifico)
Per ogni giocatore, si calcola la forza media degli avversari NHL affrontati nel periodo (usando il Punti% dalla classifica NHL corrente):

```
opp_strength = media(Punti% delle squadre avversarie nelle partite conteggiate)
opp_factor = 1 + (media_lega_Punti% - opp_strength) * 1.2
```

Interpretazione: se il giocatore affronta avversari più forti della media lega, `opp_strength > media_lega`, quindi `opp_factor < 1` (penalizzazione). Se affronta avversari deboli, `opp_factor > 1` (bonus). Il coefficiente 1.2 è un moltiplicatore di sensibilità, scelto empiricamente e regolabile.

### Passo 4 — Fattore forma propria squadra NHL
Si usa il Punti% della squadra NHL del giocatore stesso come proxy di "quanto sta girando bene" quel contesto offensivo:

```
own_pct = Punti% della squadra NHL del giocatore
own_form_factor = 1 + (own_pct - media_lega_Punti%) * 0.5
```

Coefficiente 0.5, più basso di quello avversari (1.2) perché è un effetto di secondo ordine rispetto alla difficoltà diretta dell'avversario.

### Passo 5 — Proiezione individuale finale
```
proiezione_giocatore = fp_baseline * opp_factor * own_form_factor * n_partite_contate
```

### Passo 6 — Somma a livello di fantasquadra
```
proiezione_grezza_squadra = somma(proiezione_giocatore per tutti i 19 titolari, incluso portiere)
```

### Passo 7 — Calibrazione di scala (correzione empirica IMPORTANTE)
Nella pratica, la somma grezza delle proiezioni individuali (Passo 6) tende a sovrastimare sistematicamente i punteggi reali osservati storicamente (nel test con la giornata 3, la proiezione grezza media era ~65-70% più alta del livello storico reale). Causa probabile: i fantapunti/game da stagione NHL "intera" non tengono conto di fattori di attrito del fantagioco (bench/infortuni, variabilità di formazione, differenze tra scoring system stagione regolare NHL vs scoring IFHL specifico). 

Per correggere, si applica un fattore di scala calcolato sui dati storici disponibili:

```
scale_factor = media(punteggi_storici_IFHL_tutte_squadre) / media(proiezioni_grezze_tutte_squadre)
proiezione_scalata_squadra = proiezione_grezza_squadra * scale_factor
```

**Nota per l'LLM che riceve questo documento**: questo passo è un patch empirico, non un principio teorico solido. Andrebbe sostituito da una calibrazione più rigorosa (es. regressione lineare tra proiezione grezza e punteggio reale storico, magari con intercetta diversa da zero) non appena si avrà a disposizione un campione di giornate più ampio (attualmente solo 2 giornate storiche).

### Passo 8 — Blend con media storica della fantasquadra
```
proiezione_finale_squadra = 0.75 * proiezione_scalata_squadra + 0.25 * media_storica_IFHL_squadra
```

Il peso 0.25 (residuo) serve a catturare eventuali effetti sistematici della fantasquadra non colti dal modello giocatore-per-giocatore (es. qualità delle riserve/bench strength, tendenza a schierare formazioni sub-ottimali). Anche questo peso è una scelta euristica, non derivata da ottimizzazione statistica formale.

### Passo 9 — Deviazione standard per la simulazione (CORREZIONE CRITICA)
**Errore iniziale da non ripetere**: calcolare la deviazione standard della fantasquadra come combinazione delle varianze individuali dei singoli giocatori (es. `std_squadra = sqrt(sum((fp_i * 0.55)^2))`) produce una std artificialmente bassa (nell'ordine di 6-13 punti), perché tratta i punteggi dei giocatori come statisticamente indipendenti. In realtà sono fortemente correlati (stessa sera, stesso contesto NHL, effetti di squadra), quindi la varianza non si "media via".

**Soluzione adottata**: usare direttamente la deviazione standard osservata empiricamente sui punteggi storici reali delle fantasquadre:

```
std_reale = deviazione_standard(tutti i punteggi storici IFHL osservati, tutte le squadre, tutte le giornate)
```

Nel test con 2 giornate disponibili, `std_reale ≈ 15.5` punti, applicata uniformemente a tutte le fantasquadre per mancanza di campione sufficiente a stimarne una specifica per squadra. 

**Nota per l'LLM**: con più giornate storiche disponibili, si dovrebbe evolvere verso una `std` specifica per fantasquadra (alcune squadre mostrano più "volatilità" di risultato di altre, es. una squadra con top-player star-dipendenti vs una con distribuzione uniforme di talento), invece di una std di lega uniforme.

## 6. Simulazione Monte Carlo e conversione in quote

Per ogni partita (away, home):

```python
N = 200_000  # iterazioni
away_scores = normal(media=proiezione_finale_away, std=std_reale, size=N)
home_scores = normal(media=proiezione_finale_home, std=std_reale, size=N)
away_scores = clip(away_scores, min=0)  # punteggi non negativi
home_scores = clip(home_scores, min=0)

total = away_scores + home_scores
margin_assoluto = abs(home_scores - away_scores)

p_home_win = mean(home_scores > away_scores)
p_away_win = 1 - p_home_win

p_under_X = mean(total < X)   # per X in [90, 125, 150, 175]
p_over_X  = mean(total >= X)

p_spreadNeg10  = mean(margin_assoluto < 10)
p_spread10_20  = mean(10 <= margin_assoluto < 20)
p_spread20_30  = mean(20 <= margin_assoluto < 30)
p_spread30Plus = mean(margin_assoluto >= 30)
```

Conversione probabilità → quota decimale con margine bookmaker:

```python
def prob_to_odds(p, margin=0.06, min_odds=1.01, max_odds=15.0):
    p_clipped = clip(p, 1/(max_odds*(1-margin)), 1/(min_odds*(1-margin)))
    fair_odds = 1 / p_clipped
    quota_finale = fair_odds * (1 - margin)
    return round(max(min_odds, quota_finale), 2)
```

Il margine del 6% è una scelta arbitraria in linea con la prassi bookmaker reale (over-round complessivo sul mercato). I tetti (1.01 minimo, 15.0 massimo prima del margine) evitano quote assurde (es. 0.98, che sarebbe matematicamente scorretta, o quote a migliaia per eventi quasi impossibili).

## 7. Interpretazione dei mercati nel template

- **winnerAway / winnerHome**: probabilità che la squadra away/home totalizzi più punti fantahockey nella giornata.
- **under/over [soglia]**: soglia sulla somma dei punti fantahockey delle due squadre nella singola partita.
- **spreadNeg10 / 10-20 / 20-30 / 30Plus**: fasce sul valore assoluto della differenza di punti tra le due squadre a fine giornata (non hanno un segno; rappresentano "quanto sarà risicato o ampio" il divario, indipendentemente da chi vince).

## 8. Bug noti e lezioni apprese (utili per validazione da parte di un altro LLM)

1. **Quote sotto 1.01 o sopra i tetti**: risolto applicando `max(1.01, quota)` e clip sulle probabilità prima della conversione.
2. **Sovrastima sistematica dei totali dal modello bottom-up**: la somma grezza delle proiezioni individuali era ~35% più alta dei livelli storici osservati. Risolto con `scale_factor` empirico (Passo 7). Soluzione temporanea, da rivedere con più dati.
3. **Std sottostimata da aggregazione di varianze indipendenti**: bug più grave, produceva quote quasi tutte uguali/estreme (1.01 o 13.25 ripetuti) perché gli z-score dei totali rispetto alle soglie fisse (90/125/150/175) erano quasi sempre enormi (fino a ±4.5). Risolto sostituendo la std "sommata" con la std osservata empiricamente sui dati storici reali (~15.5). Questo è il fix più importante e va sempre verificato quando si ricalibra il modello: **controllare sempre lo z-score `(soglia - media_totale) / std_totale` per ogni soglia e ogni partita; se è sistematicamente oltre ±3 per la maggior parte delle partite, la std è troppo bassa.**

## 9. Domande aperte per iterazione futura (da proporre all'LLM che riceve questo documento)

- Come stimare in modo più rigoroso il `scale_factor` del Passo 7 quando si avrà un campione di 5-10+ giornate storiche (es. regressione lineare invece di rapporto tra medie)?
- Come costruire una `std` specifica per fantasquadra invece di quella uniforme di lega, con dati limitati (shrinkage verso la media di lega, tipo stima bayesiana empirica)?
- I fattori `opp_factor` (coefficiente 1.2) e `own_form_factor` (coefficiente 0.5) sono scelti euristicamente: esiste un modo per calibrarli sui dati storici (es. verificare se effettivamente i giocatori con calendario favorevole hanno sovraperformato la loro media nelle giornate passate)?
- Il modello attualmente non pesa diversamente i vari ruoli/linee (1a-2a linea vs 3a-4a linea), nonostante il regolamento IFHL preveda scoring diverso per linea: andrebbe raffinato includendo il ruolo esatto (C1 vs C3 vs C4 ecc.) invece di trattare tutti i titolari come equivalenti.
- Attualmente il margine bookmaker (6%) è fisso e uguale per tutti i mercati: un bookmaker reale spesso applica margini diversi per mercato (es. più alto su mercati esotici come spread30Plus, più basso su 1X2).
