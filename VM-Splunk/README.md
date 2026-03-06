# VM-labb med fokus på Splunk

## 2026-03-06 - Active Directory

Nu är det dags att sätta upp Active Directory. Jag går via Server Manager och väljer Add Roles and Features och följer installations-wizarden.

Min nya forest får heta testdomain.local. Datorn startas om och allt ser ut att vara i ordning.

![TESTDOMAIN-Admin](<Screenshots/Skärmbild 2026-03-06 094958.png>)

Nu lägger vi till Windows 10-maskinen i Active Directory. Börjar med att skapa ett par nya Organizational Units som heter IT och HR. Skapar ett par användare för framtida användning.

Nu ska vi gå med i domänen med Windows10-datorn, så jag använder Advanced System Settings och under Computer Name väljer jag domänen vi skapade (TESTDOMAIN.LOCAL). Innan jag trycker på OK måste jag konfigurera DNS-inställningen, så jag ändrar DNS till min Windows2022-server, i det här fallet 192.168.100.110. Sedan fyller jag i användarnamn och lösenord, det blev min HR-användare.

![Windows10 login](<Screenshots/Skärmbild 2026-03-06 102509.png>)

Allt verkar fungera.

Nu har jag en testmiljö som jag kan leka med senare. Planen är att utföra diverse attacker och sedan undersöka dessa i Splunk.


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