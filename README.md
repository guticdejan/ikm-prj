# Laboratorijska vježba 8: CANopen protokol

Projektni zadatak iz predmeta Industrijske komunikacione mreže

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

Naredni korak je da kloniramo projekat i preuzmemo podmodule

    git clone https://github.com/CANopenNode/CANopenLinux.git
    cd CANopenLinux
    git submodule init
    git submodule update
    
## Kroskompajliranje za Raspberry Pi platformu

Sljedeći korak je da kroskompajliramo alatku *canopend* za Raspberry Pi platformu pozivanjem komande:

    make CC="arm-linux-gnueabihf-gcc -std=gnu11"

Nakon čega prelazimo u direktorijum *cocomm*, otvaramo fajl *cocomm.c* i zakomentarišemo liniju koda `#include getopt_core`, te potom kroskompajliramo alatku *cocomm* za Raspberry Pi platformu

    cd cocomm
    gedit cocomm.c
    make CC="arm-linux-gnueabihf-gcc -std=gnu11"

Kao rezultat dobijamo binarne fajlove alata *canopend* i *cocomm*.

Kopirati *canopend* i *cocomm* na ciljnu platformu.

## CAN interface

Na ciljnoj platformi pokrenuti komandu:

    sudo ip link set up can0 type can bitrate 250000
    
## Pokretanj CANopenLinux uređaja

Za izvođenje ove vježbe potrebno je otvoriti 4 terminala.

### Prvi terminal

Komandom `ssh` povežemo se na *Rasberry Pi* platformu sa IP adresom 192.168.23.206 i pokrenemo alatku *candump*

    ./candump can0
    

