# Laboratorijska vježba 8: CANopen protokol

Projektni zadatak iz predmeta Industrijske komunikacione mreže

## Priprema za vježbu

<p>
Očekuje se da je student upoznat (kroz prezentacije na predavanjima i konsultovanje dostupne literature) sa osnovnim pravilima komunikacije CANOpen protokola.

Prije početka vježbe, student treba da ažurira stanje lokalnog repozitorijuma izvršavanjem git pull komande u okviru ~/ikm-labs/ direktorijuma. Ako repozitorijum nije ranije preuzet, potrebno ga je klonirati u lokalnom home direktorijumu korišćenjem naredbe git clone https://github.com/knezicm/ikm-labs. Nakon što je repozitorijum ažuriran/kloniran, potrebno je kopirati folder lab8 sa cijelim njegovim sadržajem u home direktorijum trenutnog korisnika.

## Preuzimanje i instalacija can-utils softverskog paketa (ovaj korak radimo u slučaju da nemamo candump)

<p>
  Prvi korak je kloniranje izvornog koda projekta sa repozitorijuma. U tu svrhu, koristimo sljedeću komandu:

    git clone --depth=1 https://github.com/linux-can/can-utils.git
  
Prethodnu komandu treba izvršiti u okviru radnog direktorijuma laboratorijske vježbe (`lab8`).

Sljedeći korak je kroskompajliranje biblioteke. Kroskompajliranje ćemo obaviti na sličan način kao što smo to radili u slučaju *libmodbus* biblioteke, jer se i u ovom projektu koriste alati za automatizovano kompajliranje projekata. Prvo je potrebno napraviti folder (npr. folder `usr` u radnom direktorijumu laboratorijske vježbe) u kojem će se nalaziti prekompajliranja (binarna) verzija biblioteke sa kojom će se kasnije dinamički linkovati izvršni fajl aplikacije.

    mkdir usr
Nakon toga, prelazimo u folder u kojem se nalazi repozitorijum *can-utils* projekta i pokrećemo niz komandi za konfiguraciju *build* sistema i kompajliranje projekta.

    cd can-utils
    ./autogen.sh
    ./configure --prefix=/path/to/usr --host=arm-linux-gnueabihf
    make
    make install
  
Kao rezultat, u okviru `usr` foldera dobijamo binarne fajlove alata koji su sastavni dio *can-utils* projekta.

Kopirati `candump` iz foldera `usr` na ciljnu platformu.
<p/>

## Preuzimanje *CANopenLinux* projekta

Naredni korak je da kloniramo projekat i preuzmemo podmodule.

    git clone https://github.com/CANopenNode/CANopenLinux.git
    cd CANopenLinux
    git submodule init
    git submodule update
    
## Kroskompajliranje za Raspberry Pi platformu

Sljedeći korak je da kroskompajliramo alatku *canopend* za Raspberry Pi platformu pozivanjem komande:

    make CC="arm-linux-gnueabihf-gcc -std=gnu11"
    cd CANopenLinux
    make

Nakon čega prelazimo u direktorijum *cocomm*, otvaramo fajl *cocomm.c* i zakomentarišemo liniju koda `#include getopt_core`, te potom kroskompajliramo alatku *cocomm* za Raspberry Pi platformu.

    cd cocomm
    //gedit cocomm.c
    //make CC="arm-linux-gnueabihf-gcc -std=gnu11"
    make

Kao rezultat dobijamo binarne fajlove alata *canopend* i *cocomm*.

Kopirati *canopend* i *cocomm* na ciljnu platformu.

## CAN interface

Sljedeći korak podrazumijeva aktiviranje CAN interefejsa. Ovo se postiže istim komandama kao kada se radi sa klasičnim mrežnim interfejsima.

    sudo ip link set up can0 type can bitrate 250000  # enable interface
    ip link show dev can0						    	            # print info
    sudo ip link set can0 down      					        # disable interface
    
## Pokretanj CANopenLinux uređaja

Za izvođenje ove vježbe potrebno je otvoriti 4 terminala.

### Prvi terminal

Komandom `ssh` povežemo se na *Rasberry Pi* platformu sa IP adresom 192.168.23.206 i pokrenemo alatku *candump*:

    ./candump can0

### Drugi terminal

Komandom `ssh` povežemo se na *Rasberry Pi* platformu sa IP adresom 192.168.23.206 i pokrenemo `can0` uređaj sa `CANopen NodeID = 4:

    ./canopned can0 -i 4
    
### Treći terminal

Komandom `ssh` povežemo se na *Rasberry Pi* platformu sa IP adresom 192.168.23.212 i pokrenemo kontrolni *CANopen Linux* uređaj na lokalnom soketu koristeći komandu:

    ./canopend can0 -i 1 -c "local-/tmp/CO_command_socket"

### Četvrti terminal

Komandom `ssh` povežemo se na *Rasberry Pi* platformu sa IP adresom 192.168.23.212 i provjerimo da li nam alat *cocomm* radi ispravno pokretanjem komade:

    ./cocomm "help"
    
Ukoliko *cocomm* ispravno radi primićemo `help` string.

### Prvi koraci

*candump* prikazuje sljedeći sadržaj:

    can0  701   [1]  00                       # Boot-up message from canopend
    can0  081   [8]  00 50 01 2F 14 00 00 00  # Emergency from canopend
    can0  704   [1]  00                       # Boot-up message from demoDevice
    can0  084   [8]  00 50 01 2F 74 00 00 00  # Emergency from demoDevice
   
Ništa nije konfigurisano, te nemamo više poruka. // ovdje treba objasniti malo sta znace poruke

### Osnovna SDO komunikacija

Prvo konfigurišemo parametar 0x1017 - *Producer heartbeat time*. Čitamo vrijednost 0x1017 parametra:

    ./cocomm "4 read 0x1017 0 u16"
    
I kao rezultat dobijamo vrijednost 0, što znači da je *heartbeat producer* onemugućen.

*candump* prikazuje sljedeći sadržaj:

    can0  604   [8]  40 17 10 00 00 00 00 00
    can0  584   [8]  4B 17 10 00 00 00 00 00

Prva poruka je zahtjev klijenta, u našem slučaju klijent je *canopend* uređaj preko kojeg *cocomm* uspostavlja komunikaciju (212). Druga poruka je odgovor servera.

Sada postavimo parametar 0x1017 na 1000 milisekundi na oba uređaja:

    cocomm "1 write 0x1017 0 u16 1000"
    cocomm "4 write 0x1017 0 u16 1000"

*candump* prikazuje:

    can0  701   [1]  7F
    can0  604   [8]  2B 17 10 00 E8 03 00 00
    can0  584   [8]  60 17 10 00 00 00 00 00
    can0  704   [1]  7F
    can0  701   [1]  7F

Sada imamo *heartbeat* poruke sa oba uređaja sa intervalom od jedne sekunde. 7F znači da je uređaj u NMT predoperacionom stanju.

### *Network management* - NMT

CANopen NMT messages have highest priority and are sent from NMT master, canopend in our case. They can be sent to specific node or all nodes. They can reset device, communication or set internal state of the remote device to operational, pre-operational(PDO disabled) or stopped(only heartbeat producer and NMT consumer enabled).

    cocomm "4 reset communication"
    cocomm "4 start"
    cocomm "0 reset node"

Observe CAN messages in candump. Second command does not work, because there is critical emergency which sets error register. Third command resets our devices, so go to their terminal windows and restart them.

*Emergency* poruke, *error* registar i NMT predoperaciono stanje su posljedica neinicializovane *non-volatile* memorije. Objekti 0x1010 i 0x1011 koriste se za skladištenje i obnovu podataka, uglavnom iz *CANopen Object Dictionary*.

Obnovimo svu *non-volatile* memoriju na oba uređaja i resetujmo ih:

    cocomm "1 w 0x1011 1 vs load"
    cocomm "4 w 0x1011 1 vs load"
    cocomm "0 reset node"
    # re-run devices in their terminals
    
*candump* je sada bez *emergency* poruka i imamo dvije dodate PDO poruke, jer su uređaji sada u NMT operacionom stanju. *Heratbeat* poruke su nestale:

    can0  701   [1]  00
    can0  704   [1]  00
    can0  184   [2]  00 00
    can0  284   [8]  00 00 00 00 00 00 00 00


