# SIPp per test carico di freeswitch
Semplici scenari di chiamata per test di carico di  freeswitch

## Funzionamento e Architettura
Negli scenari proposti, SIPp agisce come **UAC (User Agent Client)**, ovvero è lui che "prende l'iniziativa" (invia REGISTER o INVITE).


#  REGISTRAZIONE - settare con -m quante register inviare
docker run -it --network host \\
  -v /home/grigiu/sipp/scenari:/scenarios \\
  -w /scenarios \\
  ghcr.io/grigiu/sipp:26.050.92 \\
  172.16.2.99:5060 \\
  -sf /scenarios/register.xml \\
  -inf /scenarios/register-accounts.csv \\
  -i 192.168.255.14:5060 \\
  -r 1  \\
  -rp 1 \\
  -m 100 \\
  -aa \\
  -trace_msg -trace_err \\
  -message_file /scenarios/register.log


# CHIAMATA SENZA AUTH (se FreeSwitch risponde subito 200 OK)
docker run -it --network host \\
  -v /home/grigiu/sipp/scenari:/scenarios \\
  -w /scenarios \\
  ghcr.io/grigiu/sipp:26.050.92 \\
  172.16.2.99:5060 \\
  -sf /scenarios/invite-without-auth.xml \\
  -inf /scenarios/invite-accounts.csv \\
  -i 192.168.255.14:5060 \\
  -m 1 -r 1 \\
  -trace_msg -trace_err \\
  -message_file /scenarios/invite-noauth.log


- **Durata Chiamata**: Per cambiare quanto dura una chiamata, apri il file `.xml` e cerca la riga `<pause milliseconds="30000"/>`. Cambia il valore (es. 60000 per un minuto).

# TEST DI CARICO (es. 500 chiamate,  4 chiamate al secondo, max 50 simultanee)
docker run -it --network host \\
  -v /home/grigiu/sipp/scenari:/scenarios \\
  -w /scenarios \\
  ghcr.io/grigiu/sipp:26.050.92 \\
  172.16.2.99:5060 \\
  -sf /scenarios/invite-without-auth.xml \\
  -inf /scenarios/invite-accounts.csv \\
  -i 192.168.255.14:5060 \\
  -m 500 \\
  -r 4 -rp 1000 \\
  -l 50 \\
  -trace_msg -trace_err \\
  -message_file /scenarios/loadtest.log

# TEST P2P (Chiamate tra interni 7000 -> 7001, ecc.) (TBD)
docker run -it --network host \\
  -v /home/grigiu/sipp/scenari:/scenarios \\
  -w /scenarios \\
  ghcr.io/grigiu/sipp:26.050.92 \\
  172.16.2.99:5060 \\
  -sf /scenarios/invite-auth.xml \\
  -inf /scenarios/invite-p2p.csv \\
  -i 192.168.255.14:5060 \\
  -m 1 -r 1 \\
  -trace_msg -trace_err \\
  -message_file /scenarios/p2p.log


#### Comandi per Scenario Distribuito (P2P)

**Sulla VM-B (UAS - Destinatari):**
```bash
# 1. Registra i numeri PARI (7000, 7002...)
docker run -it --network host -v /home/grigiu/sipp/scenari:/scenarios -w /scenarios ghcr.io/grigiu/sipp:26.050.92 172.16.2.99:5060 -sf /scenarios/register.xml -inf /scenarios/register-uas.csv -m 5 -i 192.168.255.15:5060

# 2. Resta in ascolto per ricevere chiamate
docker run -it --network host -v /home/grigiu/sipp/scenari:/scenarios -w /scenarios ghcr.io/grigiu/sipp:26.050.92 -sf /scenarios/uas.xml -i 192.168.255.15:5060 -p 5060
```

**Sulla VM-A (UAC - Chiamanti):**
```bash
# 1. Registra i numeri DISPARI (7001, 7003...)
docker run -it --network host -v /home/grigiu/sipp/scenari:/scenarios -w /scenarios ghcr.io/grigiu/sipp:26.050.92 172.16.2.99:5060 -sf /scenarios/register.xml -inf /scenarios/register-uac.csv -m 5 -i 192.168.255.14:5060

# 2. Lancia le chiamate verso i numeri PARI
docker run -it --network host -v /home/grigiu/sipp/scenari:/scenarios -w /scenarios ghcr.io/grigiu/sipp:26.050.92 172.16.2.99:5060 -sf /scenarios/invite-auth.xml -inf /scenarios/invite-distributed.csv -i 192.168.255.14:5060 -m 5 -r 1
```
# troubleshooting
### Perché `invite-without-auth.xml`?
Nello scenario di test verso l'Echo Test (`*9196`), usiamo `invite-without-auth.xml` perché:
1. **Fiducia del Server**: Spesso FreeSwitch/FusionPBX, se riconosce l'IP del mittente o se l'utente si è appena registrato con successo, non richiede una seconda sfida di autenticazione (`407 Proxy Authentication`) per la chiamata.
2. **Risposta Immediata**: L'Echo Test è un'applicazione "interna" che risponde subito con un `200 OK`. Lo scenario `invite-without-auth.xml` è ottimizzato per questo flusso: `INVITE -> 200 OK -> ACK`.

### Perché il test P2P "puro" fallisce con un solo SIPp?
Se chiami un altro interno (es. 7000 chiama 7001) e sono entrambi registrati dallo stesso SIPp:
1. SIPp invia l'INVITE a FreeSwitch.
2. FreeSwitch prova a far squillare il 7001 inviando un INVITE *indietro* a SIPp.
3. SIPp, essendo configurato solo come "Chiamante" (UAC), non sa cosa fare con una chiamata in entrata e la scarta.
4. FreeSwitch va in timeout (`408`) perché nessuno risponde al 7001.

**Per i test di carico e performance, l'uso di `invite-without-auth.xml` verso l'Echo Test è il metodo più affidabile e preciso.**

### Test P2P Distribuito (Avanzato)
Se vuoi testare chiamate tra interni senza usare l'Echo Test, devi dividere gli utenti tra due istanze di SIPp:

1. **VM-B (UAS - Ricevente)**:
   - Deve registrare gli utenti "destinatari" (es. 7001, 7003...).
   - Resta in ascolto con `uas.xml`.
   - FreeSwitch sa che questi utenti sono sulla VM-B perché il `REGISTER` è partito da lì.
2. **VM-A (UAC - Chiamante)**:
   - Registra gli utenti "mittenti" (es. 7000, 7002...).
   - Effettua le chiamate verso i destinatari.
   - Quando FreeSwitch riceve la chiamata per il 7001, "guarda" nei suoi database di registrazione e vede che il 7001 è sulla **VM-B**, quindi gli gira l'INVITE.
3. **Il Ruolo del UAS**: Il SIPp sulla VM-B riceve l'INVITE, risponde con `180 Ringing` e poi `200 OK` (simulando una persona che risponde). Questo permette alla chiamata di instaurarsi correttamente tra le due VM attraverso FreeSwitch.

  
