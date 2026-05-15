# Game of Active Directory (Light) - I min ESXi-maskin

Mitt mål är att sätta upp en variant av GOAD (Light) för att köra pentesting. Jag kommer potentiellt modifiera vissa aspekter dock. Dels är min setup en ESXi-maskin så jag ska försöka få igång det på den. Sedan skulle jag också vilja implementera lite övervakning så att jag kan se när jag attackerar. Så eventuellt Splunk eller Wazuh eller något på målmaskinerna. Men vi får se var vi landar exakt.

https://github.com/Orange-Cyberdefense/GOAD

## Min setup 

Allt sätts upp i min ESXi-server. Den är gammal och klen med följande hårdvara:

- 4 CPUs x Intel(R) Core(TM) i5-4690K CPU @ 3.50GHz
- 15.94 GB DDR3


Maskin | Operativ | Kommentarer
-|-|-
001 | Ubuntu Server | Min jump host och den som sätter upp allt med Vagrant/Ansible
002 | Windows? | Vem vet? Jag gör bara layout för min README.md just nu
003 | Windows? | Vem vet? Jag gör bara layout för min README.md just nu
004 | Windows? | Vem vet? Jag gör bara layout för min README.md just nu

## 2026-05-15 - Sätter upp min miljö

Inte för avancerat att sätta upp miljön trots allt.

    wget https://releases.hashicorp.com/vagrant/2.4.3/vagrant_2.4.3-1_amd64.deb

En del plugins till Vagrant

    vagrant plugin install vagrant-reload
    vagrant plugin install vagrant-vmware-desktop
    vagrant plugin install vagrant-vmware-esxi
    vagrant plugin install vagrant-env

Tanka hem ovftool

    https://developer.broadcom.com/tools/open-virtualization-format-ovf-tool/latest

Och lite mer smått och gott, men det gick fint till slut i alla fall.