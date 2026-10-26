# Piano d'azione — risalire alla fonte dei rallentamenti (finestra 20:30–21:50)

Base: `analisi/report-rallentamenti.md`. Quadro: WAN viva (344/344 LCP ok), sospettati lato radio/LAN:
client **44:d3:ad** (Android 8.1, cronico su 2,4 GHz), **30:32:35** (IoT `udhcp`, fallisce il band‑steering),
aria 2,4 GHz affollata (12 BSS vicini, noi su canale 9), **LAN Port 1** che rimbalza (bc:24:11 + 18:60:24
si disconnettono insieme più volte al giorno e alle 21:12 del 25/10).

Regola d'oro: **un cambiamento per sera**, misurazioni sempre uguali, altrimenti i risultati non sono
confrontabili.

---

## 1. Kit di misurazione (preparare PRIMA della serata)

| # | Strumento | Dove | Comando / uso |
|---|---|---|---|
| 1 | Ping con timestamp | PC cablato | Linux/macOS: `ping -i 1 -D 192.168.1.1 \| tee ping-gw.txt` e `ping -i 1 -D 1.1.1.1 \| tee ping-wan.txt`. Windows: `ping -t 1.1.1.1 > ping-wan.txt` |
| 2 | Ping con timestamp | Client Wi‑Fi (uno per banda, se possibile) | stesso comando, file `ping-wifi24.txt` / `ping-wifi5.txt` |
| 3 | Speedtest ripetuto | PC cablato + client Wi‑Fi | app Speedtest/fast.com ogni ~15 min dalle 20:25 alle 21:55; annotare **ora + Mbps down/up + ping** in un foglio |
| 4 | Analizzatore Wi‑Fi | Android: *WiFiAnalyzer*; iOS: *AirPort* alternativi; PC: *inSSIDer* | screenshot/log dei canali ogni 15': canali dei vicini, livello segnale (dBm), % occupazione |
| 5 | (opzionale ma ideale) iperf3 | Server su un PC sempre acceso (es. il mini‑PC/VM dietro bc:24:11): `iperf3 -s`; client: `iperf3 -c <IP> -t 30 -i 5` | misura la rete **interna** senza dipendere da Internet: separa LAN da WAN |

Le ore locali vanno sempre annotate; il router ha l'ora corretta, così possiamo incrociare col suo log.

## 2. Durante il prossimo episodio (checklist 20:25–21:55)

1. Avviare i ping (1, 2) e lasciarli girare fino a fine episodio.
2. Speedtest ogni 15' in questo ordine: **PC cablato → client 2,4 → client 5** (50–60 s ciascuno).
3. Screenshot/lettura dell'app o pagina «Client» del router: chi è collegato, su quale banda, segnale.
4. WiFiAnalyzer: registrare il canale 2,4 GHz (chi c'è, quanto è occupato) e il nostro 5 GHz.
5. Appuntare **l'ora esatta** dei momenti peggiori (per l'incrocio con eventlog.log).

## 3. Esperimenti A/B (una variabile per sera, in quest'ordine)

| Sera | Azione | Se migliora → |
|---|---|---|
| 1 | **Baseline**: solo misure (sez. 2), nessun cambiamento | — (è il riferimento) |
| 2 | **44:d3:ad offline** (modalità aereo 20:30–21:50) | il client cronico era fattore o co‑fattore |
| 3 | **30:32:35 offline** (IoT `udhcp 1.22.1`) | il fallimento di band‑steering era il fattore |
| 4 | **Canale 2,4 fisso su 1 o 13** (niente auto) | interferenza dei vicini (oggi siamo sul 9, tra i traffici dei ch 6 e 11) |
| 5 | **Band‑steering OFF** (SSID separati 2,4/5) e i due client fissati su una banda | il passaggio di banda con handshake fallito era il fattore |
| 6 | **Sostituzione cavo/porta di LAN 1** (bc:24:11 e 18:60:24) | la portante cablata rimbalzante era il fattore (indipendente dal Wi‑Fi, si può fare subito) |

Nota: bc:24:11 è un prefisso tipicamente usato da macchine virtuali Proxmox [inferito, da verificare
a casa]: se bc:24:11 è una VM/host sempre acceso, è il posto perfetto per il server iperf3 e per il
monitoraggio del punto 5.

## 4. Albero decisionale (come leggere i risultati)

- **Cablato OK, Wi‑Fi degradato (>30% in meno)** → problema radio.
  - Se peggiora solo il 2,4 → interferenza vicini (conferma col punto 4 del kit: canale condiviso con AP potente). Rimedio: canale fisso 1/13.
  - Se peggiora anche il 5 → client lontani o canale 5 GHz affollato (i nostri client 5 GHz sono su canale DFS 52: eventuali radar lo bloccano, ma nei log non è mai accaduto).
- **Wi‑Fi: ping verso il gateway che perde pacchetti** (dal client Wi‑Fi, mentre il cablato no) → collegamento client↔router, non WAN.
- **Anche il cablato degradato** → WAN/operatore. I log provano che la portante non cade (LCP sempre ok), quindi sarà **congestione**: portare a Vodafone l'elenco di speedtest con ora precisa.
- **iperf3 interno OK ma Internet lento** → WAN (come sopra). iperf3 lento già in LAN → router/radio.
- **Le deauth nell'eventlog spariscono nelle sere 2/3** → colpevole individuato: quel client.
- Le «DNS relay failed» (~10/h, 24/7) sono rumore di fondo costante: NON sono la causa dei rallentamenti serali, ma se vuoi toglierle prova DNS 1.1.1.1/9.9.9.9 nelle impostazioni WAN (igiene, non cura).

## 5. Monitoraggio permanente (per non ripartire da zero la prossima volta)

1. **Ping/speedtest programmati** dal mini‑PC/VM sempre acceso: cron ogni 10–15 min dalle 20:00 alle 22:30 (ping a 1.1.1.1 con timestamp + speedtestcli se disponibile), salvati su file con data → copertura duratura come `internet_check`, che purtroppo **si è fermato al 19/10**.
2. **Svuotare il log dallo spam**: l'eventlog si riempie in ~1 giorno per il 99% di righe `wifi-doctor`; se la GUI Vodafone lo permette, disattiva la telemetria/diagnostica Wi‑Fi → la cronologia dura giorni invece che ore.
3. **Backup di configurazione** dopo ogni cambiamento (attenzione: il backup contiene tutto, va censurato prima di condividerlo).
4. Tenere questo repo aggiornato: aggiungere i nuovi log (censurati) dopo ogni episodio.

## 6. Cosa NON fare

- Non cambiare più parametri contemporaneamente (perdi la capacità di attribuire la causa).
- Non giudicare una sola sera: il segnale è debole (9 deauth/81'), servono 2–3 serate per variabile.
- Non cancellare/riavviare il router durante l'episodio: il log è la prova; il riavvio la cancella.
