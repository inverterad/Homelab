# 2025-03-01

Jag har lagt upp lite media och bara testat via telefonen så att allt fungerar korrekt. Jag laddade upp några skivor jag äger på Bandcamp, och det fungerar ju hur bra som helst såhär lokalt, nästa steg borde bli att göra det här tillgängligt ute på stan?

![Jellyfin på mobilen](Screenshots/Screenshot_20260301-085645.png)

# 2025-02-05

En liten uppdatering, jag blev tvungen att resetta lösenordet då jag hade glömt bort det, måste vara in i servern oftare så jag inte glömmer bort.

Det löste jag med hjälp av AI, fick instruktionerna:

        # Reboota, håll in shift/esc för att komma in i GRUB
        Advanced Mode
        Recovery Mode
        root (drop to root shell prompt)
        mount -o remount,rw /

        passwd username

        reboot

Det är inte exakt som jag minns det, men ungefär korrekt. Spännande var det i alla fall.

Efter det har jag justerat docker-container och samba-inställningarna, jag har lagt till en ny katalog för musik och av någon anledning så kan jag streama Mad God utan problem nu. Så det är kul!

Uppdaterat docker-kommando


        docker run -d --name jellyfin -p 8096:8096/tcp -p 7359:7359/udp --volume jellyfin-config:/config --volume jellyfin-cache:/cache --mount type=bind,source=/srv/share,target=/media --mount type=bind,source=/srv/musik,target=/media2 --restart=unless-stopped jellyfin/jellyfin

Fortsättning följer.

# 2025-12-01
## Docker
Nu följer jag den här instruktionen för att installera Docker: https://docs.docker.com/engine/install/ubuntu/ <br>
Vi får se hur bra det går. <br><br>
<code>
sudo apt update<br>
sudo apt install ca-certificates curl<br>
sudo install -m 0755 -d /etc/apt/keyrings<br>
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc<br>
sudo chmod a+r /etc/apt/keyrings/docker.asc<br>
<br>
sudo tee /etc/apt/sources.list.d/docker.sources << EOF <br>
Types: deb<br>
URIs: https://download.docker.com/linux/ubuntu<br>
Suites: \$(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")<br>
Components: stable<br>
Signed-By: /etc/apt/keyrings/docker.asc<br>
EOF<br>
<br>
sudo apt update<br>
</code>

Det verkar ha fungerat helt ok. Då ska vi se om vi kan använda det till något.<br>
Vidare kommer jag följa https://docker-curriculum.com/ så långt det känns rimligt.<br><br>
Det verkar som att jag ska lägga till min användare i en viss grupp så att jag slipper skriva sudo före allting. Så jag fixade så att min användare också tillhör gruppen docker, och nu verkar allt vara frid och fröjd på den fronten.<br><br>
Ett viktigt kommando: <br>
<code>docker container prune</code><br><br>
Allt verkar fungera, så nu ett faktiskt test.. <br>

## Jellyfin via Docker
Då följer jag instruktionerna på: https://jellyfin.org/docs/general/installation/container/?method=docker-cli

Först tar vi hem jellyfin, sedan skapar vi volymer vi kommer behöva

        docker pull jellyfin/jellyfin
        docker volume create jellyfin-config
        docker volume create jellyfin-cache

Sedan kör vi igång alltihop

        docker run -d --name jellyfin -p 8096:8096/tcp -p 7359:7359/udp --volume jellyfin-config:/config --volume jellyfin-cache:/cache --mount type=bind,source=/srv/share,target=/media --restart=unless-stopped jellyfin/jellyfin


Det fungerar, jag kan koppla upp mig mot min jellyfin från min stationära (den här) datorn. Men den kan inte spela upp video åt mig, direkt jag försöker så fryser bilden och ingenting händer. Det verkar potentiellt ha med hardware acceleration att göra, men jag är osäker. Får titta närmare så snart jag kan.

https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html Blir guiden jag följer för att installera möjligheten att använda mig av mitt grafikkort när jag ska spela upp från jellyfin.

# 2025-11-29
## UFW
Idag ska jag starta upp den klassiska brandväggen ufw. Antagligen bara göra lite enklare inställningar, i dagsläget vill jag t.ex. bara att man ska komma åt SSH med lokal anslutning. Nu är det så att jag är bakom en CG-NAT så ingen skulle nog försöka utifrån i vilket fall som helst men jag vill ställa in det ändå. <br>

<code>sudo ufw allow from 192.168.1.0/24 to any port 22</code> <br>

Det får räcka för stunden då jag tror att ufw ska neka alla ingående utom de jag nu sagt ja till. Vi märker väl om det börjar bli problem framöver så får man justera brandväggen.<br>

## Samba
Jag behöver ha ett smidigt sätt att flytta filer mellan min stationära och min server, så nu ska jag installera Samba och försöka förstå hur det fungerar.<br>
<code>sudo apt install samba -y</code><br>
<code>sudo mkdir -p /srv/share</code><br>

<code>sudo smbpasswd -a USERNAME</code><br>
<code>sudo smbpasswd -e USERNAME</code><br>

<code>sudo nano /etc/samba/smb.conf</code><br>

För in följande längst ner: <br>
<code>[Shared]<br>
        path = /srv/share<br>
        browseable = yes<br>
        read only = no<br>
        valid users = USERNAME<br>
        force user = USERNAME<br>
</code>

<code>sudo systemctl restart smbd</code><br>

<code>sudo ufw allow 'Samba'</code><br>

Och sen hoppar jag in i katalogen via file explorer i windows genom att gå till \\IPADRESS\Shared och får skriva in lite användarnamn och lösenord, klart och betalt verkar det som!

# 2025-11-23
Installerar neofetch för att lättare kunna se min hårdvara. <br>
Jag räknar med att drivrutinerna för grafikkortet inte är optimala med tanke på installationen och att jag var tvungen att köra "nomodeset", så nu försöker jag uppdatera drivrutiner med: <br>
<code>sudo ubuntu-drivers autoinstall</code>
<br>
<code>sudo nvidia-smi</code> visar mig att nuvarande drivrutin är <code>Driver Version: 535.274.02</code> <br>
Det kanske är en nog ny drivrutin men det verkar inte vara den senaste. Men nu kan vi se hur det ser ut i GUI:n. <br>
Jag får No Signal på skärmen nu, så något är tokigt. <br>
En reboot med skärmen igång verkar räcka, för nu fungerar skärmen. Så vi fortsätter. Återkommer hit om problemet dyker upp igen. <br>

## Installera Steam Link
Nu ska vi testa installera Steam Link och se om det fungerar eller inte. <br>
Jag börjar med att installera flatpak med: <br> 
<code>sudo apt install flatpak</code>. <br>
<code>sudo flatpak remote-add flathub https://flathub.org/repo/flathub.flatpakrepo</code><br>
<code>sudo flatpak install flathub com.valvesoftware.SteamLink</code>

Allt verkade fungera fint. Jag slog igång Steam Link på servern och kopplade ihop en xbox-kontroll samt kopplade ihop den med min stationära PC och allt var fint fram till jag startade ett spel, skärmen var bara svart. Kunde inte navigera Steam Big Picture Mode heller. Så nu blir det lite felsökning. <br>

https://help.steampowered.com/en/faqs/view/0689-74B8-92AC-10F2#blackscreen <br>

Första rekommendationen är att uppdatera drivrutiner, så vi går tillbaka dit och tar bort drivrutinerna vi har och installerar den drivrutinen som nvidia själva verkar föreslå. <br>

<code>sudo apt purge nvidia-driver-535</code> <br>
<code>sudo apt-get autoremove</code> <br>
<code>sudo reboot</code> <br>
<code>sudo apt install nvidia-driver-580</code> <br>
<code>sudo reboot</code> <br>
<code>nvidia-smi</code> <br>
NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver. Make sure that the latest NVIDIA driver is installed and running. <br>
Suck. <br>

Det visade sig dock att driver 535 inte var problemet. Så jag har återställt det som det var och det som löste problemet var att "disable hardware encoding" på min stationära PC. <br>
Efter det så verkade allt flyta på fint. Provade spela ett ett par spel och det kändes smidigt. Måste installera Dark Souls eller något där det är noga med precision och testa. <br>

## Lite mer inställningar

Valde att ställa in så att jag inte behöver skriva in login och lösen varje gång jag startar om servern. Men SSH behåller kravet på lösenord. Jag skapade en fil med lite kod i! <br>
<code>sudo nano /etc/lightdm/lightdm.conf.d/50-autologin.conf</code> <br>
Fyllde i följande: <br>
<code>[Seat:*] <br>
autologin-user=YOUR_USERNAME <br>
autologin-user-timeout=0 </code><br>

Och nu funkar det galant!

## XRDP
Nu vill jag få igång Remote Desktop från min windows-maskin, man vet aldrig när det kan behövas. <br>
<code>sudo apt install xrdp</code><br>
<code>sudo adduser xrdp ssl-cert</code> - Om jag förstått det här rätt så kan det uppstå en del problem då man försöker nå TLS certificate files som krävs för att få åtkomst, likt SSH.<br>
<code>nano .xsession</code> - Skapar den här filen i mitt home directory, skriver in "startxfce4" och fortsätter med <br>
<code>chmod +x .xsession</code> - Det här ska tydligen assistera XRDP med att veta vilken desktop environment den ska köra på.<br>
<code>sudo systemctl restart xrdp</code><br>

Försökte connecta via Windows inbyggda Remote Desktop-program men den avslutade min session direkt.<br>
Jag fick ändra några grejer, bl.a. fick jag lägga till <br>
<code>unset DBUS_SESSION_BUS_ADDRESS <br>
unset XDG_RUNTIME_DIR <br>
startxfce4</code> <br>
Efter jag hade commented out följande <br>
<code>#test -x /etc/X11/Xsession && exec /etc/X11/Xsession <br>
#exec /bin/sh /etc/X11/Xsession </code><br>
i filen: <code>/etc/xrdp/startwm.sh</code><br>
Detta behövde jag göra så att en pågående XFCE-session inte skulle krocka med den man skapar i och med remote desktop. Och nu fungerar det! Åtminstone någorlunda.



# 2025-11-22
Har laddat hem ubuntu-24.04.3-live-server-amd64.<br>
https://ubuntu.com/download/server <br><br>
Tar hem programvaran Rufus för USB-installation.<br>
https://rufus.ie/sv/#download<br>

![Rufus screenshot](<Screenshots/Skärmbild 2025-11-22 Rufus.png>)
<br><br>
Först installerar jag en gammal SSD då ingen var installerad sedan tidigare. Efter det så blev det lite stök med att få igång boot från USB men det löste sig till slut.<br>
Där efter blev det dock rätt stökigt, då jag försöker installera från USB så får jag No Signal från skärmen efter en stund, vilket gör mig rätt låst. Måste starta om flera gånger och remounta USB-stickan flera gånger. För att lösa problemet så behövde jag gå in i en meny efter jag får alternativet att installera Ubuntu. Trycker på e för att komma in i en submeny där jag lägger till "nomodeset" för att den ska skippa att ladda video drivers och efter det fungerar det!
<br><br>
Efter detta kör jag: <br>
<code>sudo apt install xubuntu-desktop</code>

Nu bör jag ha tillgång till en GUI. Testar den och allt ser ut att fungera ok, nu ska jag se till att jag kan fjärrkoppla mig till desktop med hjälp av xrdp på ubuntu-servern. Så jag kör: <br>
<code>sudo apt install xrdp</code>




# 2025-11-15 - Hämta hårdvaran och påbörja

Hämtade hem hårdvaran, ett gammalt chassi som jag använt för hundra år sedan. Innanmätet är jag osäker på vad det är för något idag, ska bli intressant att kolla in när jag får igång den gamla härken.

Dammig och jävlig, men charmig!<br>
![Chassi](Screenshots/chassi.jpg)
