# Audit CerCollettiva – modulo `core/`

- **Repository:** `lucamenini/CerCollettiva` (fork di `atomozero/CerCollettiva`), branch `main`, stato al 21/11/2026
- **File analizzati:** `core/models.py` (1363 righe), indice di `core/views/` e `core/services/`, `tests/test_models_core.py`, `.gitignore`
- **Confronto con:** Regolamento dei Benefici della Fondazione Energia Solidale ETS (CdA del 23/07/2025), comunicazioni GSE CR6605 / CR7038
- **Stato:** solo lettura, nessun file di codice modificato

---

## A. Difetti tecnici bloccanti

1. **Le migrazioni non sono nel repository.** Il `.gitignore` (righe 102-103) esclude `migrations/` e tiene solo `__init__.py`. Ogni installazione genera il proprio schema con `makemigrations` e non esiste uno storico condiviso. Così una PR che modifica i modelli non si può revisionare né portare in produzione in modo affidabile. Va risolto prima di toccare i modelli.
2. **I test di `core` non corrispondono ai modelli.** `test_models_core.py` usa campi che non esistono:
   - su CERConfiguration: `vat_number`, `admin`;
   - su Plant: `code`, `type`, `power_kw`, `pod`, `commissioning_date`, `cer`;
   - su CERMembership: `share_percentage` e il ruolo `VIEWER`.

   Questi test non possono passare. Di fatto `core` è senza test.
3. **Codice rotto o duplicato in `Plant`:**
   - `get_total_system_power` è definito due volte. La prima versione usa `DeviceMeasurement`, `Max` e `Sum` senza importarli.
   - `energy/views/plant_views.py:59` lo chiama con `user_id=`, un parametro che non esiste, quindi va in `TypeError`.
   - `get_energy_today`, `get_current_power` e `get_daily_energy_data` usano `self.mqtt_data`, ma nessun modello definisce quel `related_name`, quindi danno `AttributeError`.
   - C'è un `test_mqtt_connection` fuori dalle classi (righe 1229-1288), che è codice morto.
   - `Alert` dichiara due volte `status`, `severity`, `message` e `created_at`.
4. **Problemi di sicurezza e robustezza:**
   - `mqtt_password` è salvata in chiaro, mentre CLAUDE.md promette la cifratura a livello di campo.
   - `Plant.save()` chiama Nominatim in modo sincrono a ogni salvataggio senza coordinate, anche durante i test e gli import.
   - `verify_gaudi` copia con `setattr` qualunque chiave ricevuta, senza una lista di campi ammessi.
5. **Numerazione delle tessere e del registro soci soggetta a race condition.** `generate_card_number` usa `count()+1` e `register_member` usa `last+1`, senza lock. Con due iscrizioni contemporanee si ottengono numeri duplicati o un `IntegrityError`.

## B. Lacune di dominio CACER

| Aspetto | Cosa fa il repository | Cosa richiedono la norma e il caso della Fondazione |
|---|---|---|
| CER e configurazioni | `CERConfiguration` ha un solo `primary_substation`, quindi tratta 1 CER = 1 cabina primaria | Un soggetto giuridico (la Fondazione) può avere più configurazioni, una per cabina primaria, ciascuna con il suo codice GSE (es. CR6605, CR7038) |
| Cabina primaria | Campo di testo libero (`CharField(100)`) a livello CER | Va collegata al singolo POD, con un codice preso dalla mappa delle aree convenzionali del GSE (D.Lgs. 199/2021, art. 31; DM 414/2023, art. 2, comma da verificare) |
| POD | Esiste solo come `Plant.pod_code`, quindi solo per gli impianti | I POD dei consumatori, cioè i punti di prelievo, non vengono registrati. Senza di essi non si calcola l'energia condivisa |
| Produttore terzo | `Plant.clean()` obbliga il proprietario dell'impianto a essere socio attivo | Il Regolamento dei Benefici, art. 3 c. 1, ammette i Produttori Terzi, come previsto dal TIAD (Del. ARERA 727/2022, Allegato A, art. 3.4 lett. f). Oggi il modello li blocca |
| Ruolo nella configurazione | `member_type` su CERMembership, e di nuovo `plant_type` su Plant | Il ruolo produttore/consumatore dipende dal POD, non dalla persona |
| Data di esercizio | `installation_date` ed `expected_operation_date` (la data "presunta" di GAUDÌ) | Per l'ammissione al premio conta la data effettiva di entrata in esercizio e se viene prima o dopo la costituzione della CER (DM 414/2023, art. 3, comma da verificare). Il campo non c'è |
| Potenza | Cinque campi FloatField senza dire quale vale | Serve la potenza incentivabile per impianto o UP, che determina la fascia del TP_base (≤200 / ≤600 / >600 kW). Va anche controllato il limite di 1 MW (D.Lgs. 199/2021, art. 31, c. 2 lett. b) |
| Accumulo | Solo il booleano `has_storage` | Mancano potenza e capacità dell'accumulo e la sua posizione (lato produzione o lato post-produzione). Come incide sull'energia condivisa: riferimento normativo da verificare (Regole operative GSE, Allegato 1 al DD MASE n. 22 del 23/02/2024 e successivi aggiornamenti) |
| Ripartizione economica | Percentuali fisse 45/30/20/3/2 | Vedi il punto C |
| Incassi GSE | `GSEIncomeTracking` per CER, con un solo importo e una sola tariffa | Le voci vanno separate per configurazione (codice CR): TIP in acconto e a conguaglio, corrispettivo di valorizzazione ARERA, contributo PNRR, costi amministrativi GSE trattenuti. Inoltre `reference_month` e `reference_year` sono ridondanti |

## C. L'ipotesi più a rischio

Il modello economico del repository dà per scontato che l'incentivo venga ripartito tra i soci secondo percentuali fisse. Il Regolamento dei Benefici della Fondazione segue un'altra logica:

- **Art. 3 c. 2:** ai produttori (membri o terzi) va una quota decisa dal CdA, divisa in proporzione ai kWh immessi e conteggiati per la condivisione.
- **Art. 4 c. 1:** il resto, tolti i costi GSE e le spese di gestione, rimane alla Fondazione, perché i membri vi rinunciano.
- **Art. 4 c. 2, in applicazione del DM 414/2023, art. 3 c. 2 lett. g:** la parte di TIP oltre la soglia percentuale di energia condivisa va solo ai consumatori non imprese o a finalità sociali, con contabilità separata.

Il modello attuale non può rappresentare né la ripartizione proporzionale ai kWh per singolo impianto né l'eccedenza oltre soglia con contabilità separata. Per la Fondazione la seconda è un obbligo di legge.

## D. Interventi proposti, in ordine

1. Versionare le migrazioni (togliere l'esclusione dal `.gitignore`) e fissare una baseline.
2. Riscrivere `test_models_core.py` sui modelli reali prima di modificarli.
3. Correggere i difetti del punto A: metodi duplicati o rotti, `Alert`, cifratura di `mqtt_password`, geocoding asincrono, numerazioni dentro una transazione con lock.
4. Ristrutturare il dominio con un modello nuovo: `SoggettoCER → Configurazione (codice GSE, cabina primaria) → PuntoPOD (prelievo/immissione, cabina primaria) → Impianto/UP (potenza incentivabile, data di esercizio, accumulo, produttore titolare o terzo)`. Da progettare dopo l'audit di `energy/`.
5. Sostituire `CERDistributionConfiguration` con regole di ripartizione parametriche: quota produttori in proporzione ai kWh, trattenuta della Fondazione, eccedenza oltre soglia separata.

## Prossimi moduli da analizzare

`energy/` (misure, calcolatore dell'energia condivisa, intervallo orario o quartorario, accumuli), poi `documents/` (parser GAUDÌ), `monitoring/`, `users/` con `tests/`, infine la sessione trasversale su settings, dipendenze e CI.
