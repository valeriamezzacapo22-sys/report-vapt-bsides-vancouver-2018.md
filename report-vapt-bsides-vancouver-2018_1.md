# Report di Penetration Test — BSides Vancouver 2018 (Workshop)

**Autore:** Valeria Mezzacapo
**Corso:** Master in Cybersecurity — Epicode
**Data:** 2 novembre 2026

> Ho svolto questa attività solo in laboratorio, su una macchina volutamente vulnerabile
> scaricata da VulnHub e di mia proprietà, a scopo di studio. Nessun sistema di terzi è stato toccato.

---

## 1. In breve

Ho eseguito un penetration test sulla macchina **BSides Vancouver 2018**, seguendo le fasi
classiche: ho prima esplorato la macchina, poi ho cercato i suoi punti deboli, e infine ho
individuato **come un attaccante potrebbe entrarci**.

Sono riuscita a mappare tutti i servizi esposti e a trovare la **via d'ingresso**: un pannello
di login di WordPress senza protezioni, attaccabile indovinando la password dell'utente
amministratore. L'ultimo passo (indovinare davvero la password) non l'ho completato per un
**limite del mio ambiente di laboratorio**, che spiego alla fine — non perché la via non ci fosse.

---

## 2. Con cosa ho lavorato

- **Macchina attaccante:** Kali Linux
- **Macchina bersaglio:** BSides Vancouver 2018 (Ubuntu), indirizzo `192.168.64.5`
- **Strumenti:** Nmap, Gobuster, WPScan, Hydra, browser

---

## 3. Fase 1 — Ho trovato la macchina sulla rete

Ho cercato quali dispositivi erano accesi sulla rete:
```bash
nmap -sn 192.168.64.0/24
```
Ho individuato il bersaglio all'indirizzo **`192.168.64.5`**.

---

## 4. Fase 2 — Ho visto quali porte e servizi erano aperti

```bash
nmap -sV 192.168.64.5
```
Ho trovato tre servizi aperti:

| Porta | Servizio | Versione |
|-------|----------|----------|
| 21 | FTP | vsftpd 2.3.5 |
| 22 | SSH | OpenSSH 5.9p1 |
| 80 | HTTP (sito web) | Apache 2.2.22 |

Tre possibili vie d'ingresso, tutte su versioni vecchie.

---

## 5. Fase 3 — Ho esplorato il sito web

Aprendo `http://192.168.64.5` ho visto solo la pagina di default di Apache ("It works!"),
cioè una pagina vuota. Ma sapevo che potevano esserci cartelle nascoste, quindi le ho cercate:
```bash
gobuster dir -u http://192.168.64.5 -w /usr/share/wordlists/dirb/common.txt
```
Tra i risultati ho notato il file **`robots.txt`**. Aprendolo ho trovato:
```
Disallow: /backup_wordpress
```
Il sito nascondeva una cartella: **`/backup_wordpress`**.

Andando su `http://192.168.64.5/backup_wordpress` ho trovato un **sito WordPress**
("Deprecated WordPress blog"). Leggendo i post, ho scoperto **due nomi utente**:
- **`john`** (descritto come amministratore IT)
- **`admin`**

---

## 6. Fase 4 — Ho cercato come entrare

**SSH (porta 22):** ho provato a indovinare la password via SSH con Hydra, ma il server
**accetta solo chiavi, non password**. Quindi questa via era chiusa.

**Login di WordPress:** il login di WordPress invece accetta le password e **non ha nessuna
protezione** contro i tentativi ripetuti. Questa era la via giusta. Ho quindi avviato un attacco
per indovinare la password dell'utente `john`:
```bash
wpscan --url http://192.168.64.5/backup_wordpress --usernames john \
       --passwords /usr/share/wordlists/rockyou.txt
```

---

## 7. Dove mi sono fermata e perché

Non ho completato l'ultimo passo (indovinare la password) per un **limite hardware**, non della
metodologia:

- La macchina bersaglio è fatta per computer Intel (x86), mentre il mio è un Mac con chip ARM.
  Per farla girare ho dovuto usare l'**emulazione**, che rende il bersaglio molto lento.
- Ogni tentativo di password sul login di WordPress richiede un calcolo pesante sul bersaglio:
  in emulazione diventa lentissimo, quindi provare milioni di password richiederebbe troppo tempo.
- Ho provato a spostare la macchina su un server cloud per averla a velocità piena, ma essendo
  una macchina di laboratorio non pensata per il cloud, lì non è partita correttamente.

**In sintesi:** ho individuato e confermato la via d'ingresso; completarla richiede un computer
Intel vero o un laboratorio dedicato.

---

## 8. I problemi di sicurezza che ho trovato (e come si risolvono)

| Problema | Gravità | Come si risolve |
|----------|---------|-----------------|
| Il file robots.txt rivela una cartella nascosta (`/backup_wordpress`) | Media | Non scrivere percorsi sensibili in robots.txt; togliere i backup dal sito |
| I nomi utente di WordPress sono visibili dai post | Media | Nascondere gli autori; bloccare l'elenco utenti |
| Il login di WordPress non blocca i tentativi ripetuti | Alta | Limitare i tentativi, bloccare l'account, usare 2FA e password forti |
| Servizi vecchi (vsftpd, Apache, SSH del 2012) | Alta | Aggiornare tutti i servizi |
| Backup del sito accessibile dal web | Alta | Rimuovere i backup dalla cartella pubblica |

---

## 9. Conclusioni

Ho seguito tutte le fasi di un penetration test fino a trovare e confermare la via d'ingresso.
La macchina ha una catena di debolezze chiara (cartella nascosta rivelata → nomi utente esposti
→ login senza protezioni) che, su un computer adatto, porterebbe a prendere il controllo
dell'account amministratore di WordPress e poi dell'intero sistema.

Con questa attività ho imparato a usare in pratica gli strumenti principali del settore
(Nmap, Gobuster, WPScan, Hydra) e a capire l'importanza di ogni fase, oltre a gestire i limiti
reali di un ambiente di laboratorio.

---
*Report svolto nell'ambito del Master in Cybersecurity — Epicode.*
