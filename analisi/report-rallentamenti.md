# Report — Analisi log router (rallentamento 20:30–21:50)

**Router:** SerComm RHG3006, firmware Vodafone `XF6_4.0.XX.XX`, WAN GPON (eventlog.log:1–6, boot_log:357,835).
**Episodio analizzato:** finestra 20:30–21:50 del **sabato 25 ott 2026** (unica serata coperta dai log; assunzione dichiarata).
**Fuso assunto:** Europe/Rome (CET/CEST). Il cambio ora CEST→CET è avvenuto il 25/10 alle 03:00: la finestra è in CET (UTC+1), coerente con `traffic_speed` (`+01:00` esplicito) e con le date dei log HM.

---

## 1. Sintesi

- **La WAN non mostra alcun guasto** nella finestra né nell'intera copertura: 344/344 LCP echo-reply ricevuti, zero eventi PPP down, errori DNS identici al background notturno [osservato].
- Nella finestra il **Wi‑Fi mostra churn client ~2×** (deauth+disassoc 0,111/min vs 0,054/min fuori) e 4 nuovi DHCP DISCOVER: si riagganciano **44:d3:ad** (Android 8.1), **30:32:35** (dispositivo embedded `udhcp 1.22.1`, con **4‑way handshake timeout** reason 15 durante un passaggio 2,4→5 GHz), fe:ba:bd, 70:56:81 [osservato].
- **Nessun cambio canale, nessun riavvio, nessuna pressione di memoria, nessun provisioning TR‑069** nella finestra [osservato]. Alle 21:32:13 l'ACS di wl0 rimanda la scansione perché **canale occupato** con client associati (eventlog.log:80796) → aria affollata [osservato].
- **Causa più probabile:** degrado lato **Wi‑Fi/LAN** (interferenza/congestione aria in fascia serale + client instabili), **non** lato WAN. Confidenza: **media** — i log non contengono misure d'aria (RSSI, retry, airtime) né velocità durante l'episodio.
- **Da fare subito** (test discriminanti §6): durante il prossimo episodio, speedtest simultaneo **cavo vs Wi‑Fi** + ping continuo a gateway e a 8.8.8.8.

---

## 2. Copertura dei dati

| File | Righe | Intervallo | Note/buchi |
|---|---|---|---|
| eventlog.log | 208.979 | 25/10 10:42:58 → 26/10 14:43:35 | nessun buco (~7.400 righe/h costanti); righe 8–183 = blocco 26/10 10:32–10:41 fuori ordine (pre‑rotazione). Spam `wifi-doctor` = 205.572 righe (99%) |
| dmesg_log/dmesg_log | 12.237 | ≈18/10 14:10 → 26/10 14:30 | tempi **monotoni senza data**: mappati empiricamente (§Appendice); 1 solo boot nel buffer; il boot precedente terminò con `REBOOT_APP_HANGUP` (dmesg_log:744); le date HM indicano un **riavvio servizi** il 19/10 03:46 (acsd INIT, WAN up, cwmp «process not exist» → riavviato) |
| hm_log/internet_check/log | 41.484 | 27/10/2025 → **19/10/2026** | 30 blocchi event‑driven (WAN up/down + ping 8.8.8.8): **non copre l'episodio** |
| hm_log/memory_check/log | 32.629 | 12/07 → 26/10/2026 | 195 campioni ogni 12h (03:46/15:46); più vicini alla finestra: 25/10 14:46 e 26/10 02:46 |
| hm_log/cwmp_check/log | 72.058 | 12/02 → 19/10/2026 | estratti TR‑069 + curl; **termina il 19/10** |
| cwmp_log + cwmp_log_backup | 3.653 + 8.051 | solo 26/10 10:53 → 14:38 | sessioni ogni 15', tutte 200/204 |
| boot_log | 896 | boot kernel (senza data) | modello, radio, GPON |
| wifi/log/acsd_wl0_log, acsd_wl1_log | 271+355 | snapshot al riavvio acsd 19/10 03:46:49 | wl0 = 5 GHz, canale iniziale 52/160 MHz; wl1 = 2,4 GHz, canale 9; **nessun cambio canale poi** |
| traffic_speed | 1 | 26/10 14:13–14:43 | 2×180 campioni da 5 s; **non copre l'episodio** |
| hybrid.log, cwmp_cookies | 1 | epoch 0 / cookie | non pertinenti (cookie ignorato come da specifica) |

**Mappa radio (dai canali negli acsd log):** wl0 = **5 GHz**, wl1 = **2,4 GHz**. Vicini: 12 BSS su 2,4 GHz (4 sul ch1, 2 sul ch6, 2 sul ch11, 2 sul ch12…), 3–4 BSS su 5 GHz (acsd_wl0_log:22–41, acsd_wl1_log:22–34).

---

## 3. Linea del tempo 20:30–21:50 del 25/10 (tutte le voci rilevanti della finestra)

| Ora | Evento | file:riga |
|---|---|---|
| 20:30–21:50 | LCP echo OK ogni ~5' (WAN1/WAN2); nessun errore PPP | eventlog.log (17 coppie, es. :73156–73158) |
| 20:30–21:50 | «DNS relay failed, DNS reply error» ogni ~6' — **background costante 24/7** | eventlog.log:73157 e seguenti |
| 20:41:39–20:44:04 | **44:d3:ad** (Android 8.1) ri‑auth/assoc su wl1 + DHCP; deauth reason4 alle 20:43:35; rientra 20:44:03 | eventlog.log:74577–74725; dmesg_log:11587–11618 |
| ~20:55 | 70:56:81 deautorizzato su wl0 (solo buffer driver) | dmesg_log:11715–11716 |
| 20:57:17–20:58:37 | **30:32:35** (IoT, `udhcp 1.22.1`): assoc wl1 → tentativo su wl0 → **deauth reason15 (4‑way handshake timeout)** + DHCP RELEASE → ritorno su wl1; driver: «Unexpected RX toss reason 33» | eventlog.log:76563–76615; dmesg_log:11621–11670 |
| 21:12:01–21:12:08 | **LAN Port [1] link down**; si disconnettono i dispositivi cablati bc:24:11 e 18:60:24 (stessi che «rimbalzano» insieme dalle 12:46) | eventlog.log:78303, 78316–78317 |
| 21:13:46 | fe:ba:bd disassoc+deauth su wl0 | eventlog.log:78522–78525 |
| 21:32:13 | **wl0: ACS rinvia il cambio/scan canale**: total_frame 1859 > soglia 600, client presenti → canale 5 GHz occupato | eventlog.log:80796 |
| 21:29:28 / 21:36:07 | driver wl1: «tid cleanup … waiting for seq …» (TX stall isolati, sotto baseline) | dmesg_log:11684, 11686 |
| 21:36:01–02 | 0c:c6:fd deauth su wl0 | dmesg_log:11732–11734 |
| 21:50:33–21:51:10 | 70:56:81 (**AirdiLuigi5960**) esce e rientra su wl0, rinnova DHCP | eventlog.log:83055, 83064 sgg. |

Nella finestra **non** compare: reboot, cambio canale effettivo, sessione CWMP, aggiornamento firmware, lease negato, errore NAT/kernel.

---

## 4. Metriche (finestra 81 min vs resto della copertura 25/10 10:43→26/10 14:43 = 1.600 min)

| Metrica | Fuori finestra | Dentro finestra | Variazione |
|---|---|---|---|
| deauth+disassoc Wi‑Fi (eventlog) | 87 → **0,054/min** | 9 → **0,111/min** | **~2×** |
| auth/assoc (nuovi accessi) | 175 → 0,109/min | 14 → 0,173/min | ~1,6× |
| DHCP DISCOVER (nuovi client) | 12 → 0,008/min | 4 → 0,049/min | ~6× (numeri piccoli) |
| DHCP REQUEST | 41 → 0,026/min | 4 → 0,049/min | ~2× |
| «DNS relay failed» | 264 → 0,165/min | 13 → 0,161/min | **invariato** (background ~10/h anche di notte) |
| LCP echo‑reply WAN | 327 → 0,204/min (344/344 ok su tutto il log) | 17 → 0,210/min, 0 persi | invariato |
| IGMP general query | 768 → 0,480/min | 39 → 0,482/min | invariato |
| dmesg: TX stall («tid cleanup») | picco 33/h (24/10 11h); tipico 5–15/h | 5 in 81' (≈3,7/h) | **sotto baseline** |
| dmesg: «no tx attempted» | 20 episodi in 8 giorni (pegg. 22/10, 2,5 min) | 1 episodio singolo (20:42:23) | sotto baseline |
| Memoria libera (campioni 12h) | mediana 105 MiB (min 99, max 219) | n.d. (cadenza 12h; 25/10 14:46 = 101, 26/10 02:46 = 101) | nessun segnale |
| Sessioni CWMP | 26/10: 1 ogni 15', 100% 200/204, nessun push parametri | 0 nella finestra (log non coperto) | n.d. |
| traffic_speed | solo 26/10 14:13–14:43: rx media 5.363 u/5s, p95 28.516, max 33.023 (unità non dichiarate; se KB ≈ picchi ~50 Mbps) | non coperto | n.d. |

Periodicità 20:30–21:50 su più giorni: **non verificabile** — l'eventlog copre una sola serata.

---

## 5. Ipotesi (ordinate per confidenza)

1. **Client problematici su Wi‑Fi (churn) — confidenza MEDIA.** A favore: 44:d3:ad è cronicamente instabile (118 eventi deauth/authorize in 8 giorni in dmesg_log; storm di 17 AUTH/s alle 18:15:02 del 25/10, eventlog.log ~:74xxx); 30:32:35 fallisce l'handshake nel passaggio di banda alle 20:58 (reason 15, eventlog.log:76606). Contro: i numeri assoluti in finestra sono bassi (9 deauth/81') e riguardano 3–4 client. Discrimina: disattivare il band‑steering / fissare il client su una banda e vedere se i deauth calano.
2. **Congestione/interferenza radio serale — confidenza MEDIA‑BASSA.** A favore: ACS di wl0 bloccato da aria occupata alle 21:32 (eventlog.log:80796); 12 BSS vicini su 2,4 GHz e canale 9 in mezzo ai traffici ch6/ch11; churn concentrato su wl1 (66 vs 30 deauth). Contro: nessun cambiamento brusco registrato in finestra (nessun cambio canale, niente RSSI/retry nei log). Discrimina: `chanim_stats`/spettro durante l'episodio.
3. **Problema WAN/operatore — SMENTITA per l'episodio (confidenza alta sul link, media sul throughput).** A favore di nessun problema: 344/344 LCP reply, zero PPP down, DNS invariato. Limite: l'LCP ogni ~5' prova che la portante viva, **non** misura latenza/throughput (nessun speedtest nella finestra). Discrimina: speedtest cablato durante l'episodio.
4. **Esaurimento risorse router — SMENTITA (confidenza alta).** Memoria stabile (390 MiB totali, ~101 liberi, nessun trend su 3,5 mesi); nessun OOM, nessun `conntrack full` (max 12504 sessioni, dmesg), nessun watchdog nel periodo. Gli episodi «no tx» del driver (pegg. 22/10, 2,5 min) sono fuori finestra e isolati.
5. **Provisioning TR‑069 serale — NON DETERMINABILE per la finestra, improbabile.** I log CWMP coprono solo il 26/10 mattina (sessioni regolari, solo `X_EventStruct`, nessun Download/Reboot/setValue di configurazione); nessun cambio di stato coerente con un push nella finestra.
6. **Task pianificati/firmware — SMENTITA per la finestra.** Nessun restart servizio, upgrade o backup tra 20:30 e 21:50; i timer ACS sono solo rimandati (eventlog.log:80796, :15614, :97480, :179098).
7. **Attacco/flood — SMENTITA (confidenza media).** Nessuno storm DHCP/ARP/IGMP; contatori firewall a 0 (dump internet_check); un solo «TCP Scan» bloccato il 22/10 (dmesg_log:6243 sgg.). Limite: i dump firewall non coprono la finestra.
8. **IPv6 RA/ND storm — NON DETERMINABILE.** Nessun messaggio IPv6 nei log disponibili.
9. **Ripetizione quotidiana della finestra — NON DETERMINABILE.** Una sola serata nei log; il pattern in finestra (IGMP/DNS costanti) non mostra attività programmatica ricorrente a quell'ora.

---

## 6. Test discriminanti (da fare durante il prossimo episodio)

1. **Speedtest simultaneo cavo vs Wi‑Fi** (stesso momento, 20:30–21:50). Se cavo OK e Wi‑Fi degradato → problema radio; se entrambi degradati → WAN. Criterio: differenza >30% tra i due.
2. **Ping continuo (1/s) al gateway e a 8.8.8.8/1.1.1.1 da client cablato** durante la finestra. Loss verso WAN >2% o latenza >50 ms con gateway "pulito" = WAN; anche il gateway a 2–3 ms di jitter = LAN/router.
3. **Lettura contatori radio durante l'episodio** (SSH/router): `wl -i wl0 chanim_stats` e `wl -i wl1 chanim_stats` (busy >30–40% = canale saturo), `assoclist` + `rssi` per client (RSSI < −70 dBm = client debole). Questo chiude l'ipotesi 1/2.
4. **Sperimentazione canale:** fissare wl1 su canale 1 o 13 e osservare 2–3 sere i deauth (baseline attuale: ~50/giorno su wl1). Se crollano → interferenza vicini confermata.
5. **Log mirato sul client 30:32:35 e 44:d3:ad:** tenere gliuni su banda fissa (o disattivare steering); se gli episodi scompaiono → fattore scatenante era il passaggio di banda con handshake fallito.

---

## 7. Dati mancanti (ciò che servirebbe)

- **Serie di velocità/latenza durante l'episodio**: `traffic_speed` copre solo 30' del 26/10 alle 14:13–14:43; `internet_check` termina il 19/10.
- **Metriche d'aria per client** (RSSI, PHY rate, retry %, airtime): assenti ovunque; senza di esse il lato Wi‑Fi resta inferenziale.
- **Log CWMP del 25/10 sera** (rotati: restano solo 10:53–14:38 del 26/10).
- **Timestamp assoluti nel dmesg** (qui ricostruiti empiricamente, ±1' nella finestra, peggio verso fine file).
- **Copertura ≥1 settimana** dell'eventlog per verificare la periodicità 20:30–21:50.
- Campione memoria/conntrack **dentro** la finestra (cadenza attuale 12h).

---

## 8. Appendice — estrazioni e aggregazioni (riperibili)

Ambiente: GNU coreutils + Python 3 (sola lettura sui log). Lavorazione in `analisi/`, nessun file di log modificato.

1. **Inventario/ordinamento**: `wc -l` per righe; distribuzione oraria: `grep -oE "2026-10-2[56] [0-9]{2}" eventlog.log | sort | uniq -c`; out‑of‑order: confronto timestamp consecutivi (1 sola inversione: righe 8–183).
2. **Separazione spam**: `grep -vE "\[wifi-doctor\]" eventlog.log | grep -E "^\["` → 2.678 righe utili (`analisi/out/eventlog_no_spam.txt`); breakdown per tag e per ora con `cut|sort|uniq -c`.
3. **Finestra**: `grep -E "2026-10-25 (2[01]):" … | awk` 20:30–21:50 → `analisi/out/window_25ott.txt` (191 righe, letture integrali nel report).
4. **Metriche dentro/fuori**: conteggio regex per categoria con timestamp ∈ finestra vs resto (script Python; risultati §4).
5. **internet_check**: blocchi delimitati da `Start check internet`/date; 30 date + 27 ping 8.8.8.8 (RTT 4,7–6,0 ms; 1 perdita al 100% il 20/06/2026) (`analisi/out/ick_ping.txt`).
6. **memory_check**: 195 campioni (regex data `CES?T` + riga `Mem:`); mediana free 105 MiB, nessun trend per settimana ISO.
7. **CWMP**: sessioni = `FSM----http code` (26/10: 21×200 + 11×204, cadenza 15'); push = `cwmp_setValueByName` (solo `X_EventStruct`); esiti curl storici in cwmp_check (ret 6/28/35/56 → ultimi il 15/10).
8. **Mappa temporale dmesg (empirica)**: estratti tutti i `CONSOLE[wlX]: <t>`; allineati agli eventi datati dell'eventlog per MAC (44:d3:ad, 30:32:35, 0c:c6:fd, fe:ba:bd, 00:45:e2): 5 ancore (es. t=632.837–632.857 ↔ 25/10 20:58:15–27; t=695.218–695.504 ↔ 26/10 14:25–14:30). Residuo ±20 s nella zona della finestra; deriva crescente fino a ~7' a fine buffer (dichiarata). Radio: wl0 = canali 36–140 (5 GHz), wl1 = canali 1–13 (2,4 GHz) dagli acsd log.
9. **dmesg**: conteggi `tid cleanup` (455), `no tx attempted` (66, 20 episodi), `Unexpected RX toss` (135, di cui 124 reason 33), `prep_sdu 802.1x` (1.001 = handshake WPA normali); nessun OOM/watchdog/conntrack‑full nel periodo; `wd_get_reboot_reason = REBOOT_APP_HANGUP` solo all'avvio corrente (dmesg_log:744).
10. **traffic_speed**: parsing `this_period`/`last_period` (180 campioni × 5 s): rx mediana 40, media 5.363, p95 28.516, max 33.023 (u/5s).
11. **Censura**: nessuna `X` trattata come anomalia; MAC ridotti al prefisso produttore; dispositivi seguiti per hostname (HONOR‑200‑Lite, OPPO‑RENO13‑5G, MBPdiPaolo681, AirdiLuigi5960, 21061119DG, Vendor Class `android-dhcp-*`, `udhcp 1.22.1`, `MSFT 5.0`).

Output intermedi salvati in `analisi/out/` (`eventlog_no_spam.txt`, `window_25ott.txt`, `ick_dates.txt`, `ick_ping.txt`, `mck_dates.txt`).
