# 📊 Olist E-Commerce Analytics — Report Power BI End-to-End & Analisi Strategica di Business

## Indice

1. [Introduzione & Contesto del Dataset](#-1-introduzione--contesto-del-dataset)
2. [Definizione dei Requisiti & Domande di Business](#-2-definizione-dei-requisiti--domande-di-business)
3. [Data Understanding & ETL (Power Query)](#️-3-data-understanding--etl-power-query)
4. [Data Modeling (Star Schema)](#-4-data-modeling-star-schema--executive-design-system)
5. [Dashboard & Analisi Strategica (Pagina per Pagina)](#-5-dashboard--analisi-strategica-pagina-per-pagina)
6. [Conclusioni e Roadmap Strategica](#-6-conclusioni-e-roadmap-strategica-esecutiva)

---

## 📌 1. Introduzione & Contesto del Dataset

Il presente progetto consiste in un'analisi end-to-end basata sul **dataset reale e anonimizzato** generato da Olist, la principale piattaforma e-commerce marketplace in Brasile.

Il dataset copre oltre **100K ordini** effettuati tra il **2016 e il 2018**, fornendo una visione completa dell'intero ciclo di vita della transazione e-commerce: dall'acquisto alla fase di evasione logistica, fino al pagamento, alla consegna e alla recensione finale del cliente.

**Panoramica del Dataset**

L'architettura nativa del dataset si articola su 9 tabelle relazionali che coprono l'intera catena del valore:

- **Transazioni & Logistica** (*Orders*, *Order Items*): stato degli ordini, milestone temporali (acquisto, spedizione, consegna), prezzi unitari e costi di trasporto (*Freight Value*).
- **Anagrafiche** (*Customers*, *Sellers*, *Geolocation*): dati anagrafici e geolocalizzazione (città, stato, CAP, coordinate) di clienti e venditori.
- **Catalogo & Traduzioni** (*Products*, *Product Category Name Translation*): specifiche fisiche dei prodotti e traduzione delle categorie dal portoghese all'inglese.
- **Finance & Recensioni** (*Order Payments*, *Order Reviews*): modalità di pagamento, frazionamento in rate (*Payment Installments*), punteggio delle recensioni (1-5 stelle) e commenti dei clienti.



## 🎯 2. Definizione dei Requisiti & Domande di Business

Prima della fase di sviluppo, il progetto è stato strutturato definendo le domande analitiche fondamentali.

**💼 Vendite & Revenue**
- **Trend & Stagionalità**: come varia il fatturato mese per mese/trimestre nel periodo 2016-2018? Esistono picchi stagionali di vendita?
- **Mix di Categoria**: quali categorie di prodotto generano il maggior fatturato? Quali hanno volumi inferiori ma un prezzo medio più alto?
- **Average Order Value (AOV)**: qual è il valore medio dell'ordine e come oscilla nel tempo o tra i diversi Stati brasiliani?

**🚚 Logistica, Spedizioni & Ritardi**
- **Tempi di Consegna**: qual è il tempo medio effettivo tra l'acquisto e la consegna al cliente?
- **Tasso di Ritardo (Late Delivery Rate %)**: quanti ordini vengono consegnati oltre la data stimata (`order_estimated_delivery_date`)?
- **Impatto del Nolo**: quanto incide mediamente il costo di spedizione (`freight_value`) sul valore finale del prodotto?

**💳 Pagamenti, Prodotti & Comportamento Cliente**
- **Metodi di Pagamento**: qual è la distribuzione delle vendite tra carta di credito, Boleto Bancário (è un popolare sistema di pagamento in contanti o tramite voucher regolamentato dalla Banca Centrale del Brasile), voucher e carta di debito?
- **Leva Finanziaria delle Rate**: il numero di rate (`payment_installments`) correla positivamente con l'AOV?
- **Retention Rate**: quanti clienti effettuano più di un riacquisto sulla piattaforma?

**⭐ Soddisfazione Cliente & Performance Marketplace**
- **Impatto Logistico sul Rating**: in che misura il ritardo nella consegna compromette il punteggio delle recensioni (`review_score`)?
- **Concentrazione dei Venditori**: chi sono i Top Seller per fatturato e dove sono localizzati sul territorio rispetto ai clienti?


## 🛠️ 3. Data Understanding & ETL (Power Query)

![Power Query](/Images/Power%20Query.png)

### 3.1 Pulizia Avanzata e Normalizzazione Geografica (Geolocation)

La tabella *Geolocation* nativa presentava marcate incoerenze dovute all'inserimento non standardizzato dei nomi delle città brasiliane. Per uniformare i dati ed eliminare i duplicati testuali, è stato applicato un flusso di pulizia in più passaggi:

- **Tipizzazione dei dati**: conversione del codice postale (`zip_code_prefix`) nel tipo Numero Intero e definizione dei campi città e stato come Testo.
- **Normalizzazione del testo**:
  - `Text.Trim` per eliminare spazi vuoti a inizio/fine stringa
  - `Text.Clean` per rimuovere caratteri non stampabili o invisibili
  - `Text.Lower` per evitare duplicati causati dall'alternanza di maiuscole/minuscole
- **Rimozione dei caratteri speciali** (accenti): funzione M personalizzata per sostituire i caratteri accentati (es. á, â, ã, ç) con i corrispondenti caratteri ASCII standard.
- **Correzione degli errori**: sostituzione mirata di anomalie (es. correzione manuale da "sa£o paulo" a "sao paulo").
- **Standardizzazione dei separatori & spazi**: sostituzione di apostrofi (`'`) e trattini (`-`) con spazi singoli, seguita dalla duplica degli spazi doppi o multipli.
- **Riduzione dati superficiali**: eliminazione delle colonne `geolocation_lat` e `geolocation_lng`, in quanto la visualizzazione cartografica è basata sulle aggregazioni a livello di Stato e Città.

### 3.2 Diagnosi sulla Qualità dei Dati Geografici (Data Profiling)

Per verificare l'integrità dei dati territoriali tra l'anagrafica clienti (*Customers*) e il database di geolocalizzazione (*Geolocation*), è stata eseguita un'analisi diagnostica approfondita.

**Metodologia**: esecuzione di un *Full Outer Join* temporaneo basato sulla colonna della chiave geografica (`zip_code_prefix`), usato come strumento di Data Profiling per identificare sia le corrispondenze esatte sia i record univoci su entrambe le tabelle.

**Risultato della diagnosi**:
- Identificate **968.299 corrispondenze esatte** su un totale di 1.000.163 record esaminati nella tabella di geolocalizzazione.
- Tasso di copertura pari al **96,8%**, a conferma dell'ottima qualità complessiva del dato geografico.
- Il restante 3,2% rappresenta codici postali non mappati nel dataset originale.

**Decisione architetturale**: a seguito della diagnosi, il passaggio di merge è stato rimosso per preservare la struttura dello Star Schema ed evitare la duplicazione delle righe clienti (*fan-out*). La tabella *Geolocation* è stata disabilitata dal caricamento nel modello (`Enable Load = False`), affidando la geocodifica diretta agli attributi di stato e città già presenti e puliti nelle dimensioni *Customers* e *Sellers*.

### 3.3 Gestione delle Date e Impostazioni Locali (in Orders Deliveries)

I campi temporali rappresentano l'infrastruttura fondante per tutte le metriche di Time Intelligence.

- **Risoluzione dell'ambiguità sintattica (locale)**: il dataset contiene timestamp grezzi nel formato statunitense (MM/DD/YYYY). Su sistemi/tenant con impostazioni regionali europee (DD/MM/YYYY), Power Query confonde i mesi con i giorni o genera errori sulle date con giorno superiore a 12 (es. 05/30/2018). Applicando esplicitamente la cultura *English (United States)* durante il cambio tipo di dato, è stata forzata la corretta interpretazione sintattica.
- **Standardizzazione al tipo Date puro**: tutti i campi temporali (`order_purchase_timestamp`, `order_delivered_customer_date`, `order_estimated_delivery_date`, ecc.) sono stati convertiti al tipo puro *Date*.
- **Rimozione della componente oraria & abbattimento della cardinalità**: la rimozione dell'orario ha allineato la granularità permettendo al motore di comprimere i dati in RAM con la massima efficienza.
- **Gestione dei dati anomali**: identificati 8 casi isolati con stato ordine `delivered` ma privi di una data di consegna coerente (`order_delivered_customer_date` nullo o discordante). Record documentati e isolati per evitare distorsioni nel calcolo dei tempi medi di consegna.

### 3.4 Trattamento dei Valori Mancanti nella Dimensione Prodotti (Products)

- **Gestione dei valori nulli**: rilevati circa 600 record con attributi fisici o categoria mancanti. Anziché eliminare le righe — operazione che avrebbe causato perdita di fatturato e ordini orfani nella Fact Table *Order Items* — i null sono stati sostituiti con la stringa di fallback `"unknown"`, garantendo l'integrità referenziale.
- **Eliminazione delle colonne ridondanti**: rimosse `product_name_lenght`, `product_description_lenght` e `product_photos_qty`, prive di valore analitico dimostrato per gli obiettivi di business.
- **Integrazione delle traduzioni**: la traduzione dei nomi categoria dal braziliano all'inglese è stata eseguita direttamente dentro la dimensione *Products* tramite merge con la tabella *Product Category Name Translation*. La tabella delle traduzioni originale è stata poi disabilitata dal caricamento (`Enable Load = False`) ridurre l'impronta di memoria del file .pbix.



## 📐 4. Data Modeling (Star Schema)

![Star Schema](/Images/Star%20Scheme.png)

### 4.1 Architettura Star Schema

Il modello dati è strutturato secondo il classico formato a stella per ottimizzare le prestazioni:

- **Fact Tables operative** (livello inferiore): *Orders Products*, *Orders Transactions* e *Orders Reviews*, contenenti il dettaglio transazionale, dei pagamenti e delle valutazioni.
- **Tabella perno centralizzata** (*Orders Deliveries*): posizionata al centro del modello con granularità di 1 riga per ordine (`order_id`), gestisce lo stato dell'ordine.
- **Dimensioni** (*Customers*, *Products*, *Sellers*): collegate alla tabella centrale tramite relazioni 1 a molti (1:*) con direzione del filtro **Singola**.
- **Time Intelligence** (`Dim_Date`): tabella calendario continua generata in DAX, collegata tramite relazione attiva su `order_purchase_timestamp` e relazioni inattive sulle date logistiche (`order_delivered_customer_date`, `order_estimated_delivery_date`).

### 4.2 Master Grid & Layout System (SaaS UI/UX)

Tutte le 4 pagine della dashboard seguono un layout basato su coordinate pixel-precise per garantire leggibilità ed equilibrio visivo:

**Top Bar & Navigation (Y: 24–110 px)**
- Titolo report: X 24 px, Y 24 px, W 600 px, H 40 px
- Slicer sincroni: 3 filtri cromatici coerenti (Year, Product Category, Customer State), Y 110 px, H 82 px
- KPI Header Container: angolo superiore destro, X 968 px, Y 24 px, W 928 px, H 168 px

**Griglia Analitica 2×2**
- Quadranti sinistra (alto/basso): X 24 px, Y 208 px / 636 px, W 928 px, H 412 px
- Quadranti destra / colonna allungata: X 968 px, Y 208 px / 636 px (o fusione verticale H 840 px per elenchi geografici e treemap estese)


## 📈 5. Dashboard & Analisi Strategica (Pagina per Pagina)

![Page 1](/Images/Page%201.png)

### 📄 Pagina 1 — Sales & Revenue (*Executive Sales Overview*)

**KPI principali**

| Metrica | Valore |
|---|---|
| Total Orders | 99K |
| Total Revenue | $13.59M |
| AOV (Average Order Value) | $136.68 |
| GMV (Gross Merchandise Value) | $15.84M |

**Visual della pagina**
- *Monthly Revenue Trend & MoM Growth*: evoluzione temporale dei ricavi — inizio analisi nel 2016, forte crescita nel 2017 con picco a novembre (Black Friday, $1M), stabilizzazione nel 2018.
- *Revenue & Average Product Price by Category*: il fatturato è trainato da categorie ad alto volume e prezzo contenuto (es. `health_beauty`, `bed_bath_table`), mentre settori come `computers` compensano i bassi volumi con importi medi elevati.
- *Average Order Value Trend*: esclusa l'anomalia di dicembre 2016, l'AOV risulta costante tra $130 e $150.
- *Revenue Distribution by Customer Location*: concentrazione netta dei ricavi negli stati del Sud-Est, São Paulo in primis.

**La storia dei ricavi: crescita 10x e stallo**

L'andamento mensile del fatturato racconta un'evoluzione in due atti:
- **Atto 1 — La Scalata** (gennaio 2017 – gennaio 2018): crescita di circa **10x** del fatturato mensile, confermata dalla media mobile a 3 mesi (non semplice trend stagionale). Massimo storico a novembre 2017 ($1.00M/mese), spinto dal Black Friday.
- **Atto 2 — La Maturazione** (marzo – agosto 2018): stabilizzazione importi costanti tra $0.85M e $1.00M al mese, segnale di saturazione della base clienti raggiunta.

**Note tecniche sui dati storici**
- *Assenza dati novembre 2016*: l'azzeramento delle vendite è dovuto al fermo temporaneo dello store per la nuova versione della piattaforma (confermato dal creatore del dataset).
- *Crollo dell'ultimo mese*: il calo verticale nell'ultimo punto del grafico è un artefatto tecnico dei dati (record incompleti vicini alla data di chiusura del dataset), non un declino reale del business.

**Performance dei prodotti**
- *Best seller attuali*: `health_beauty`, `watches_gifts` e `bed_bath_table` trainano fatturato e volume.
- *Prodotti ad alto potenziale futuro*: `computers_accessories` e `furniture_decor` mostrano un prezzo medio per pezzo superiore alla media — spingerli con campagne mirate può rompere il plateau e alzare l'AOV oltre i $150.

**Raccomandazioni strategiche**
- **Superare lo stallo**: il mercato è entrato in fase di maturazione — la crescita futura dipenderà da Cross-Selling (es. pacchetti `health_beauty` + `watches_gifts`) e programmi di fidelizzazione, non solo da acquisizione organica.
- **Pianificazione stagionale**: con il Black Friday confermato come principale driver, scorte e capacità logistica vanno saturate entro fine settembre.
- **Espansione geografica**: promuovere la spedizione agevolata nel Nord-Est per intercettare nuova domanda e ridurre la dipendenza dal mercato saturo del Sud-Est.

---

![Page 2](/Images/Page%202.png)

### 📄 Pagina 2 — Logistica & Spedizioni (*Executive Logistics Overview*)

**KPI principali**

| Metrica | Valore |
|---|---|
| Late Delivery Rate | **6.77%** 🔴 |
| Tempo medio di consegna nazionale | 12.5 giorni |
| Costo medio nolo (Freight Value) | $22.80 / ordine |

**Visual della pagina**
- *Monthly Delivery Time & Delay Trend* : giorni medi di consegna e giorni medi di ritardo effettivo nel tempo.
- *Average Delivery Time by State* : São Paulo consegna in ~8-9 giorni, gli stati remoti del Nord (es. Roraima) arrivano quasi a 30 giorni.
- *Average Freight Value by State* : costo medio di spedizione per tutti i 27 stati — il nolo cresce allontanandosi dal Sud-Est.

Su circa 99.000 ordini, oltre **6.600 spedizioni** hanno superato la data stimata di consegna — causa primaria delle insoddisfazioni e delle recensioni negative.

**Evoluzione temporale dei tempi di consegna**
- *Fase d'innesco* (inizio 2017): picchi critici tra 40 e 50 giorni, legati all'assestamento dei vettori partner.
- *Fase di regime* (2017–2018): l'integrazione di nuove tratte e il consolidamento dei venditori hanno stabilizzato il tempo medio a 10-14 giorni, mantenendo i ritardi sotto la soglia.

**Disparità territoriali — Top 5 vs Bottom 5**

| Top 5 — Eccellenze 🟢 | Giorni | Bottom 5 — Criticità 🔴 | Giorni |
|---|---|---|---|
| São Paulo (SP) | 8.7 | Roraima (RR) | 29.3 |
| Paraná (PR) | 11.9 | Amapá (AP) | 27.2 |
| Minas Gerais (MG) | 11.9 | Amazonas (AM) | 26.4 |
| Distrito Federal (DF) | 12.9 | Alagoas (AL) | 25.5 |
| Santa Catarina (SC) | 14.9 | Pará (PA) | 23.7 |

**Insight chiave**: esiste un divario di oltre 20 giorni tra la consegna più veloce (São Paulo, 8.7 gg) e quella più lenta (Roraima, 29.3 gg) — le regioni amazzoniche superano di oltre tre volte la media della capitale finanziaria.

**Impatto dei costi di spedizione**

Correlazione diretta e proporzionale tra tempi di spedizione e incidenza dei costi di trasporto:
- *Zone a basso costo*: São Paulo, nolo medio ~$15.00/ordine → alta marginalità sui carrelli medio-bassi.
- *Zone ad alto costo*: Roraima, Pará, Amapá, Acre → nolo oltre $40.00–$45.00/spedizione.
- *Impatto sul prodotto*: nelle tratte ad alto costo il trasporto arriva a incidere fino al 35-50% del valore del carrello, scoraggiando il riacquisto e riducendo il conversion rate nelle aree periferiche.

**Analisi delle Cause**
- *Concentrazione geografica dei venditori*: oltre il 70% dei seller attivi si trova nel Sud-Est (São Paulo e Rio de Janeiro). Le spedizioni verso il Nord richiedono molteplici snodi logistici e transiti fluviali/aerei non ottimizzati.
- *Algoritmo SLA inadeguato*: il 6.77% di ritardo non dipende solo dai vettori, ma da stime di consegna troppo aggressive mostrate al checkout negli stati del Nord — promettere 20 giorni per una destinazione che richiede strutturalmente 27 giorni genera un falso ritardo percepito.

**Raccomandazioni strategiche**
- **Decentralizzazione tramite hub logistici**: aprire un centro di stoccaggio/smistamento partner nel Nord-Est (es. Bahia o Pernambuco). L'allocazione preventiva dei top-seller in questi hub può ridurre i tempi di consegna del Nord-Est da 25 a meno di 10 giorni.
- **Ricalibrazione algoritmo delle consegne**: aggiornare le stime di consegna al checkout per gli stati del Bottom 5 (es. dichiarare 30 giorni reali invece di 22 per Roraima/Amapá), abbattendo il Late Delivery Rate reale sotto il 2% e riducendo drasticamente le recensioni a 1 stella.
- **Spedizione gratuita differenziata**: soglia di carrello minimo più alta per la spedizione gratuita nelle tratte ad alto nolo, proteggendo i margini dall'erosione dei costi di trasporto.

---

![Page 3](/Images/Page%203.png)

### 📄 Pagina 3 — Prodotti, Pagamenti & Clienti (*Products, Payments & Customers*)

**KPI principali**

| Metrica | Valore |
|---|---|
| Total Unique Customers | 96K |
| Customer Retention Rate | **3.12%** |
| Average Installments Count | 2.85 rate/transazione |

**Visual della pagina**
- *Total Payment Value by Payment Type*: transato complessivo $16.01M — carta di credito ~73.9-78.3%, Boleto braziliano ~17.8-17.9%, voucher e carta di debito a seguire.
- *AOV by Payment Installments* : l'AOV sale da ~$80-100 (1-3 rate) fino a $400-600+ per frazionamenti da 10 a 24 rate.
- *Total Revenue Share by Product Category* : quota di fatturato generata dai singoli comparti del catalogo.

**Analisi dei metodi di pagamento**
- **Carta di credito**: modalità dominante, ~73.9% del valore totale (oltre $11.8M).
- **Boleto bancario braziliano**: 17.8% del fatturato — modalità essenziale per intercettare la popolazione non bancarizzata o priva di carta di credito in Brasile.
- **Voucher & carta di debito**: coprono la quota rimanente (~2.5% e <2%).

**Relazione tra rateizzazione e scontrino medio**: correlazione diretta e positiva tra numero di rate e AOV — da ~$80-100 in un'unica soluzione fino a $400-600+ con 10-24 rate. La rateizzazione a lungo termine è la leva fondamentale che permette l'acquisto di prodotti di fascia alta (elettronica, orologi, arredo).

**Top 5 categorie per fatturato (Treemap)**

| Categoria | Fatturato |
|---|---|
| health_beauty | $1.26M |
| watches_gifts | $1.21M |
| bed_bath_table | $1.04M |
| sports_leisure | $0.99M |
| computers_accessories | $0.91M |

Le prime 5 categorie generano da sole oltre **$5.4M** — più di un terzo del fatturato globale della piattaforma. Le categorie consumabili/ricorrenti (`perfumery` $0.40M, `baby` $0.41M) mostrano volumi ancora contenuti ma rappresentano la leva chiave per incrementare la retention.

**Analisi Cause — perché la retention è solo al 3.12%?**
- *Predominanza di beni durevoli*: le categorie principali (`watches_gifts`, `bed_bath_table`, `furniture_decor`) hanno cicli di riacquisto pluriennali.
- *Assenza di retargeting post-vendita*: nessun programma di loyalty o campagna e-mail basata sui tempi di esaurimento dei prodotti consumabili.
- *Fattore logistico*: i ritardi di consegna nelle zone periferiche (fino a 29 giorni, vedi Pagina 2) compromettono la fiducia dei primi acquirenti, impedendo un rapporto ricorrente.

**Raccomandazioni strategiche**
- **Leva finanziaria (Buy Now, Pay Later)**: incentivare le rate (10-12) su categorie ad alto margine come `computers_accessories` e `watches_gifts`, mostrando la rata mensile già in scheda prodotto (es. "Tuo a partire da $25/mese") per spingere l'AOV oltre $150.
- **Retention sui prodotti consumabili**: campagne di remarketing/abbonamento su `health_beauty` e `perfumery`. Aumentare la retantion rate aumenterebbe anche il fatturato senza costi aggiuntivi di acquisizione.
- **Ottimizzazione checkout Boleto**: incentivare chi paga con Boleto (17.8% dei volumi) a registrare una carta tramite sconto sul secondo ordine, riducendo i tempi di attesa dell'accredito e migliorando l'esperienza utente.

---

![Page 4](/Images/Page%204.png)

### 📄 Pagina 4 — Recensioni, Soddisfazione e Venditori (*Reviews, Satisfaction & Sellers*)

**KPI principali**

| Metrica | Valore |
|---|---|
| Average Review Score | 4.09 / 5.00 ★ |
| Incidenza recensioni 1 stella | **11.51%** 🔴 |
| Active Sellers | 3.10K |

**Correlazione diretta tra logistica e rating**

Il grafico *Average Review Score by Delivery Status* dimostra che l'insoddisfazione è prevalentemente un problema logistico, non di prodotto:

| Stato consegna | Rating medio |
|---|---|
| On Time | ~4.20 ★ |
| Late | ~2.25 ★ (-46%) |
| Not Delivered | ~1.70 ★ |

**Distribuzione dei voti**
- *Dominanza dei promotori*: le 5 stelle rappresentano la maggioranza (oltre 57.000 recensioni), seguite dalle 4 stelle (~19.000).
- *Anomalia delle 1-star*: le recensioni a 1 stella (~11.500) superano numericamente sia le 3 stelle (~8.000) sia le 2 stelle (~3.000) — il giudizio negativo non deriva da una modesta insoddisfazione sul prodotto, ma da attese prolungate e consegne mancate.

**Analisi della rete venditori**
- *Concentrazione del fatturato*: i primi 10 venditori generano una quota sproporzionata del fatturato, con il primo seller che supera $200.000. La dipendenza da pochi grandi venditori espone la piattaforma a rischi operativi in caso di ritardi nella loro catena di evasione.
- *Squilibrio geografico*: oltre l'85% dei 3.100 seller attivi si trova fisicamente nello Stato di São Paulo e nella regione Sud-Est. ⚠️ *(verifica coerenza con il dato "oltre il 70%" riportato a Pagina 2 prima di pubblicare)*. La quasi totale assenza di seller nel Nord e Nord-Est costringe a spedizioni a lungo raggio, generando i ritardi e i costi elevati visti a Pagina 2.

**Raccomandazioni strategiche**
- **Ricalibrazione promessa servizio consegne al checkout**: estendere la data stimata di consegna per le regioni periferiche (da 20 a 28 giorni) eviterà che gli ordini finiscano in ritardo "percepito", spingendo i commenti positivi ad aumentare.
- **Notifiche proattive di tracciamento**: comunicazioni automatiche via e-mail/SMS in caso di rallentamento nel transito, per ridurre la percezione negativa prima della consegna.
- **Onboarding seller regionali**: incentivi per reclutare venditori locali nel Nord e Nord-Est (es. Bahia, Pernambuco, Ceará), riducendo drasticamente tempi di spedizione e incidenza dei ritardi.



## 🎯 6. Conclusioni e Roadmap Strategica Esecutiva

L'analisi integrata dei dati dell'e-commerce (2016–2018) evidenzia un modello di business con un'infrastruttura commerciale solida, capace di generare **$15.84M di GMV** e servire **96K clienti unici**. Tuttavia, per superare la fase di stallo e saturazione (a ~$1.00M/mese) e garantire una crescita sostenibile nel lungo termine, l'azienda deve intervenire su tre punti critici.

**Sintesi dei 3 punti di criticità**

1. **Decentramento logistico** (vera causa delle recensioni a 1 stella): divario di oltre 20 giorni tra São Paulo (8.7 gg) e Roraima (29.3 gg); il ritardo fa crollare il rating da 4.20★ a 2.25★, alimentando l'11.51% di recensioni a 1 stella.
2. **Il modello d'acquisto** (retention al 3.12%): quasi il 97% degli utenti acquista una sola volta, sostenendo costi di acquisizione continui senza valorizzare la base clienti accumulata.
3. **La dipendenza finanziaria dalle rateizzazioni**: la carta di credito copre il 73.9% del valore transato, e la rateizzazione fino a 24 rate è il driver principale che spinge l'AOV sopra i $400-600 sui beni ad alto valore.

**Roadmap strategica di intervento**

| Orizzonte | Azioni |
|---|---|
| **Breve termine** | Aumentare di 5-7 giorni le stime di consegna al checkout per gli stati remoti · Notifiche automatiche di tracciamento via e-mail |
| **Medio termine** | Campagne di remarketing sui prodotti consumabili (`health_beauty`, `perfumery`) · Promuovere la rateizzazione (10-12 rate) sui beni ad alto valore |
| **Lungo termine** | Programma di onboarding per venditori locali nel Nord/Nord-Est, per ridurre distanze di spedizione, costi di trasporto e tempi di consegna |

**Risultati attesi (Expected Impact)**

- **Customer satisfaction**: calo significativo delle recensioni a 1 stella (attualmente 11.51%) grazie a stime di consegna più trasparenti.
- **Tasso di riacquisto**: dal 3.12% a un target di 5.00–6.00% spingendo sui prodotti consumabili, aumentando il valore per cliente senza costi di acquisizione aggiuntivi.
- **AOV**: dagli attuali $136.68 a oltre $150.00, incentivando gli acquisti rateizzati sui prodotti ad alto costo.
- **Costi di spedizione e ritardi**: tempi di consegna più brevi e nolo più contenuto nel Nord/Nord-Est grazie all'attrazione di venditori locali.

---

*Dataset: [Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) — dati reali e anonimizzati, 2016-2018.*

