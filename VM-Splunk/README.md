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