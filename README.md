# Laboratorijska vježba 8: CANopen protokol

Projektni zadatak iz predmeta Industrijske komunikacione mreže

## Priprema za vježbu

<p>
Očekuje se da je student upoznat (kroz prezentacije na predavanjima i konsultovanje dostupne literature) sa osnovnim pravilima komunikacije CANOpen protokola.

Prije početka vježbe, student treba da ažurira stanje lokalnog repozitorijuma izvršavanjem git pull komande u okviru ~/ikm-labs/ direktorijuma. Ako repozitorijum nije ranije preuzet, potrebno ga je klonirati u lokalnom home direktorijumu korišćenjem naredbe git clone https://github.com/knezicm/ikm-labs. Nakon što je repozitorijum ažuriran/kloniran, potrebno je kopirati folder lab8 sa cijelim njegovim sadržajem u home direktorijum trenutnog korisnika.


## Preuzimanje *CANopenLinux* projekta

Naredni korak je da kloniramo projekat sa git repozitorijuma i preuzmemo podmodule.

    git clone https://github.com/CANopenNode/CANopenLinux.git
    cd CANopenLinux
    git submodule init
    git submodule update
    
## Kroskompajliranje za Raspberry Pi platformu

Sljedeći korak je da kroskompajliramo alatku *canopend* za Raspberry Pi platformu pozivanjem komande:
    
    cd CANopenLinux
    make CC="arm-linux-gnueabihf-gcc -std=gnu11"

Nakon čega prelazimo u direktorijum *cocomm* te potom kroskompajliramo alatku *cocomm* za Raspberry Pi platformu.

    cd cocomm
    make CC="arm-linux-gnueabihf-gcc -std=gnu11"

Kao rezultat dobijamo binarne fajlove alata *canopend* i *cocomm*.

Kopirati *canopend* i *cocomm* na ciljnu platformu.
   
  
## Preuzimanje i instalacija can-utils softverskog paketa (ovaj korak radimo u slučaju da nemamo candump)

<p>
  Potrebno je kloniranje izvornog koda projekta sa repozitorijuma. U tu svrhu, koristimo sljedeću komandu:

    git clone --depth=1 https://github.com/linux-can/can-utils.git
  
Prethodnu komandu treba izvršiti u okviru radnog direktorijuma laboratorijske vježbe (`lab8`).

Sljedeći korak je kroskompajliranje biblioteke. Kroskompajliranje ćemo obaviti na sličan način kao što smo to radili u slučaju *libmodbus* biblioteke, jer se i u ovom projektu koriste alati za automatizovano kompajliranje projekata. Prvo je potrebno napraviti folder (npr. folder `usr` u radnom direktorijumu laboratorijske vježbe) u kojem će se nalaziti prekompajliranja (binarna) verzija biblioteke sa kojom će se kasnije dinamički linkovati izvršni fajl aplikacije.

    mkdir usr
Nakon toga, prelazimo u folder u kojem se nalazi repozitorijum *can-utils* projekta i pokrećemo niz komandi za konfiguraciju *build* sistema i kompajliranje projekta.

    cd can-utils
    ./autogen.sh
    ./configure --prefix=/path/to/usr --host=arm-linux-gnueabihf
    make
    sudo make install
  
Kao rezultat, u okviru `usr` foldera dobijamo binarne fajlove alata koji su sastavni dio *can-utils* projekta.

Kopirati `candump` iz foldera `usr` na ciljnu platformu.
<p/>  

## CAN interface

Sljedeći korak podrazumijeva aktiviranje CAN interefejsa. Ovo se postiže istim komandama kao kada se radi sa klasičnim mrežnim interfejsima.

    sudo ip link set up can0 type can bitrate 250000  # enable interface
    ip link show dev can0			          # print info
    sudo ip link set can0 down                        # disable interface

### Start devices
This is an example, devices may also be started different way.

#### candump

U prvom terminalu potrebno je da pokrenemo alatku candump na prvoj ciljnoj platformi, u zavisnosti od folderu(u našem slučaju to je lab8) na ciljnoj platformi u koji smo kopirali candump alatku: 

cd lab8
candump --help 
candump -td -a can0

Više informacija za ovu alatke možemo vidjeti komandom candump --help

#### canopend

U drugom termina pokrećemo alatku canopend na drugoj ciljnoj platformi u sačuvanom folderu na ciljnoj platformi:

cd lab8 
canopend --help 
canopend can0 -i 4


Na terminalu gdje je pokrenuta alatka *candump* trebali bi dobiti iduću poruku:

    can0  701   [1]  00                       # Boot-up message from canopend
    can0  081   [8]  00 50 01 2F 14 00 00 00  # Emergency from canopend
    can0  704   [1]  00                       # Boot-up message from demoDevice
    can0  084   [8]  00 50 01 2F 74 00 00 00  # Emergency from demoDevice

Boot-up poruka nam sadrži 11-bitni CAN ID koji predstavlja Hearbeat poruku: 0x700 + CANopen Node-ID.
Poruka 0x080 je SYNC poruka, poruka 0x080 + Node ID je EMCY(Emergency message).

Ništa nije konfigurisano, te nemamo više poruka. 

### Emergency messages
Vidimo da nam uredjaj šalje Emergency message nako izvršenog boot-up. Sadržaj ove poruke je:

bajtovi 0..1: u našem slučaju 0x5000 (Device Hardware) (nam označava da je CANopen little endian format-struktura(popravi)).
bajt 2: u našem slučaju 0x01 (generički eror).
bajt 3: u našem slučaju 0x2F (CO_EM_NON_VOLATILE_MEMORY) nam predstavlja grešku pristupa stalnoj memoriji(non-volatile memory).

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

Sada imamo *heartbeat* poruke sa oba uređaja sa intervalom od jedne sekunde. 7F znači da je uređaj u predoperacionom stanju.

### *Network management* - NMT

CANopen NMT poruke imaju največi priorite i u našem slučaju poslane se od strane canopen mastera. Ove poruke mogu biti poslane odredjenom čvoru ili svim čvorovima.
One mogu resetovati uredjaj, komunikaciju ili setovati stanje u kojem će se dati uredjaj nalaziti, pa tako imamo operaciono, pre-operaciono(u kojima je razmjena procesnih podataka onemogućena) ili stop stanje. 

    cocomm "4 reset communication"
    cocomm "4 start"
    cocomm "0 reset node"

CAN poruke posmatramo na terminalu gdje je pokrenut candump. Druga komanda(cocomm "4 start") neće raditi jer postoji EMCY poruka koja setuje error registar, da bi setovali uredjaje u operaciono stanje potrebno je riješiti ove greške. Treća komanda resetuje uredjaje, nakon toga potrebno ih je u terminalu opet pokrenuti.

*Emergency* poruke, *error* registar i NMT predoperaciono stanje su posljedica neinicializovane *non-volatile* memorije. Objekti 0x1010 i 0x1011 koriste se za skladištenje i obnovu podataka, uglavnom iz *CANopen Object Dictionary*.

Obnovimo svu *non-volatile* memoriju na oba uređaja i resetujmo ih:

    cocomm "1 w 0x1011 1 vs load"
    cocomm "4 w 0x1011 1 vs load"
    cocomm "0 reset node"
    # re-run devices in their terminals
    
*candump* je sada bez *emergency* poruka i imamo dvije dodate PDO poruke, jer su uređaji sada u operacionom stanju. *Heratbeat* poruke su nestale:

    can0  701   [1]  00
    can0  704   [1]  00
    can0  184   [2]  00 00
    can0  284   [8]  00 00 00 00 00 00 00 00

Object Dictionary
-----------------
Object Dictionary is central part of CANopen device. It contains well structured communication, manufacturer specific or standardized device profile parameters, accessible by different types of communication. See picture below or complete OD documentation in [demoDevice.md](demoDevice.md). Note also project file `demoDevice.xdd`, which can be opened and edited with EDSEditor.exe (Linux or Windows application). EDSEditor can export other files, including OD.h and OD.c source files for this example. `demoDevice.xdd` and `demoDevice.eds` are standard CANopen device description files and can be opened also with other CANopen tools.

![EDSEditor](demoDevice.png)

