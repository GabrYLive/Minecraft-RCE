# Minecraft Server RCE - TRIPALDI Gabriele
## Preambolo:  cos'è un server Minecraft?
Minecraft è un videgioco sandbox molto popolare che da vari anni è stato acquisito dalla Microsoft Il gioco ha due versioni differenti, la "Java Edition" scritta appunto in Java, e la "Bedrock Edition" una versione multipiattaforma scritta in C++, un server dell abedrock edition non supporta client Java e viceversa.
Questo progetto si baserà sui server e client basati sulla versione Java, eseguibile solamente su sistemi Windows, Linux e Mac.

----------
### La vulnerabilità del server
Nella versione 1.8.0/1.8.8 del gioco è stata trovata la vulnerabilità zero-day > CVE-2021-44228, meglio nota come Log4Shell.
Microsoft ha successivamente rilasciato una patch per prevenirne gli abusi direttamente sostituendo la repository ufficiale. Questi server sono chiamati "Vanilla", ossia senza modifiche del server stesso, tuttavia non sono in grado di prevenire l'abuso di cheat all'interno del gioco e per questo motivo sono stati creati dalla community versioni di terze parti (es: Bukkit, Spigot, Paper) che consentono maggior sicurezza e l'installazione di plugins scritti in Java.
Visto che il server Vanilla vulnerabile è di difficile reperibilità, si è proceduto tramite risorse esterne a scaricare un server di terze parti molto popolare "Paper" che ha successivamente patchato il server ma la versione unpatched è più facile da reperire. 

**Log4j**: La vulnerabilità si basa sul framework Log4j che consente di gestire il loggin dell'applicazione o dei servizi online ed ha anche la possibilità di comunicare con altri servizi sul sistema. Minecraft implementa questo framework che sarà la causa della vulnerabilità.




