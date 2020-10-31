# Virtual Memory

- [Virtual Memory](#virtual-memory)
  - [Voorbereiding](#voorbereiding)
- [GitHub classroom](#github-classroom)
- [Introductie](#introductie)
- [Pagina mappen](#pagina-mappen)

## Voorbereiding

Ter voorbereiding van deze oefenzitting wordt je verwacht:
  * De oefenzitting system calls te hebben voltooid.
  * Hoofdstuk 3 van het [xv6 boek](https://github.com/besturingssystemen/xv6-riscv) te hebben gelezen.
  * Begrip te hebben van de theorie rond *virtual memory*, *paging* en *page tables*
    * Weten hoe een *virtual address* vertaald kan worden naar een *physical address* via page tables


Onderstaande video kan je bekijken om meer vertrouwd te geraken met het concept paging:

[![Bekijk de video](https://img.youtube.com/vi/JgTXJ-ZV5Zw/hqdefault.jpg)](https://www.youtube.com/watch?v=JgTXJ-ZV5Zw)

Daarnaast kan je:
* Hoofdstuk 8 in Siblerschatz raadplegen
* De online les over Hoofdstuk 8, deel 3 bekijken op Toledo

# GitHub classroom

**TODO**


# Introductie

**TODO**

# Pagina mappen

Voor we duiken in de code van xv6 willen we eerst verzekeren dat je begrijpt op welke manier een besturingssysteem ervoor kan zorgen dat een specifiek virtueel adres van een proces gemapt wordt op een fysiek adres in het werkgeheugen.

Herinnner je dat een virtueel adres in een RISC-V processor die het Sv39 schema volgt, 39 bits lang is. xv6 is ontworpen voor dit RISC-V schema.

Onderstaande voorstelling vinden we terug in de [RISC-V privileged specification](https://riscv.org/technical/specifications/):

![Sv39-scheme](img/sv39-addresses.png)

* Neem een stuk papier en geef antwoord op de volgende vragen.

> :bulb: Op het einde van deze sectie kunnen jullie de antwoorden op deze vragen terugvinden. Dit is echter een zelftest om te kijken of je de concepten goed snapt. Door meteen naar de antwoorden te kijken zal je voor jezelf niet kunnen ontdekken welke delen nog onduidelijk zouden zijn. Indien je begrip van de concepten onvoldoende is zal je waarschijnlijk in de problemen komen bij de permanente evaluatie.


* Wat is het bereik van virtuele adressen in Sv39 (`[minimale adres, maximale adres]`)?

In xv6 worden slechts 38 bits van de 39 bits effectief gebruikt. Het maximale virtuele adres in xv6 wordt `MAXVA` genoemd.

* Wat is de waarde van `MAXVA`?
* Hoeveel gigabyte werkgeheugen kan dus maximaal geaddresseerd worden door een xv6-proces?

> :bulb: Interessant weetje: 32-bit machines waren voor een lange periode de meest voorkomende consumentenmachines. 32-bit machines hebben registers van 32-bit lang. Er werd dus ook meestal gekozen om virtuele adressen 32-bit lang te maken. Je kan nu dus uitrekenen wat het maximaal geheugen is dat processen op dit soort machines konden adresseren (~4gb). Dit bleek voor sommige processen te weinig te zijn, een belangijke reden om van 32-bit naar 64-bit te schakelen.

Stel dat xv6 een nieuw proces inlaadt in het geheugen. Op dat moment moet xv6 de pagina's van dit proces in het fysieke geheugen plaatsen, en vervolgens een virtueel-naar-fysieke mapping opstellen. Deze mapping gebeurt niet willekeurig.
Onderstaande figuur, uit hoofdstuk 2 van het xv6-boek, toont de virtual memory layout van een process.

![xv6-virtual-layout](img/xv6-virtual-layout.png)

Een pagina die in de virtuele adresruimte van ieder xv6-proces geladen wordt, is de *trampoline*. In deze sessie gaan we in detail bekijken waarom deze pagina daar gemapt staat.

Op dit moment zijn we echter nog niet geinteresseerd in *wat* de trampolinepagina doet, wel in *hoe* de trampolinepagina gemapt wordt.

Neem aan dat:
1. De trampolinepagina in het geheugen gemapt staat in de frame met nummer 1234 (`PPN` = 1234).
2. Het virtuele adres `MAXVA` moet verwijzen naar de laatste byte van deze trampolinepagina (dit is het geval!) 
   
Je moet nu, als besturingssysteem, ervoor zorgen dat wanneer het nieuwe proces `MAXVA` probeert te dereferencen, de RISC-V hardware dit kan vertalen naar de laatste byte van frame 1234.

* Welke stappen moet het besturingssysteem zetten om ervoor te zorgen dat deze adresvertaling correct kan uitgevoerd worden?  Neem aan dat de top-level page table reeds bestaat en dat dit de enige page table is die al gealloceerd is voor dit proces.
  * Hoeveel page tables moet het besturingssysteem aanmaken?
  * Welke waarden moeten in deze tables ingevuld worden?

> :bulb: De vraag kan ook zo gesteld worden: hoe kan een besturingssysteem de trampolinepagina mappen op frame 1234?


<!--
## Exec

**TODO** ELF-comments van Job in Slack verwerken

Om `exec` te begrijpen is het belangrijk om eerst te snappen wat we bedoelen wanneer we spreken over een *executable*.
Een executable file of uitvoerbaar bestand bevat de volledige informatie, onder andere de machinecode en data, die nodig is om een bepaald programma uit te voeren.

Executables worden meestal gegenereerd door een *linker*. Met behulp van *compilers* worden broncode-bestanden omgezet in *object files*. De linker neemt als input verschillende object-files, verbindt (linkt) deze met elkaar en genereert vervolgens een executable als output.

In UNIX volgen `executables` het [`ELF`](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)-formaat.
Onderstaande afbeelding (bron: [Wikipedia](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format#/media/File:ELF_Executable_and_Linkable_Format_diagram_by_Ange_Albertini.png)) illustreert de opbouw van een ELF-bestand:

![Opbouw ELF-bestand](https://upload.wikimedia.org/wikipedia/commons/e/e4/ELF_Executable_and_Linkable_Format_diagram_by_Ange_Albertini.png)

In `ELF`-bestanden worden programma's opgedeeld in verschillende secties. Enkele veelvoorkende secties:

* `.text` bevat de machinecode van het programma
* `.data` bevat de initiële data die bij opstart van het proces geïnitialiseerd moet worden in het geheugen. Geïnitialiseerde global variables horen in de data-sectie
* `.bss` bevat globale data die niet geïnitialiseerd moet worden. Gedeclareerde maar niet-geïnitialiseerde globals horen in de bss-sectie
* `.rodata` bevat read-only data. Hierin worden vaak gedefinieerde strings geplaatst.
* ...

Elk van deze secties moeten in het geheugen geladen worden om een proces op te starten.
Een *loader* neemt als invoer een ELF-bestand en laadt de verschillende secties in het geheugen.
Een loader zet dus als het ware een programma, beschreven in een ELF-bestand, om in een proces dat kan uitvoeren binnen een besturingssysteem.

Naast secties bevat een ELF-bestand ook een *symbol table*. De symbol table bevat informatie over de inhoud van een ELF-bestand, bijvoorbeeld welke functies gedefinieerd zijn in het ELF-bestand en op welke (relatieve) locatie je deze kan terugvinden.

> :bulb: Een proces is niet hetzelfde als een programma. Beide woorden hebben een verschillende betekenis. Een programma beschouwen is de abstracte voorstelling van een taak die door een machine uitgevoerd kan worden. Een ELF-bestand of executable beschrijft een programma. Een proces bevat een instantie van een programma en kan geïnitialiseerd worden met behulp van een executable. Processen zijn de structuren die het mogelijk maken voor een besturingssysteem om programma's uit te voeren.

### ELF in Linux

Om ELF-bestanden te leren kennen zullen we deze eerst bekijken in onze eigen Linux-omgeving (dus niet in xv6):

* Schrijf een C-programma `hello-world.c` binnen je Linux-omgeving

```c
#include <stdio.h>

int main(){
    printf("Hello, world!\n");
    return 0;
}
```

* Compileer en link het programma met `gcc`

```shell
gcc hello-world.c -o hello-world
```
* Bekijk alle secties in het gegenereerde ELF-bestand met behulp van `readelf`

```shell
readelf -S hello-world
```

* Bekijk alle symbolen in het generereerde ELF-bestand met behulp van `readelf`

```shell
readelf -s hello-world
```

* Bekijk de ELF-header met `readelf`. 
```shell
readelf -h hello-world | less
```

* Gebruik het programma `objdump` om de uitvoerbare ELF-secties met machinecode in hello-world te *disassemblen* (omzetten van machinetaal naar leesbare assembly). Met <kbd>↑</kbd> en <kbd>↓</kbd> kan je scrollen in de `less`-omgeving. Met <kbd>q</kbd> kan je de `less`-omgeving afsluiten.

  
```shell
objdump -d hello-world | less
```

### ELF in xv6

We bekijken nu de ELF-files van xv6. ELF-files van xv6 zijn niet rechtstreeks uit te voeren binnen je Linux-omgeving. De ELF-files zijn namelijk gegenereerd voor de RISC-V ISA en bevatten dus enkel RISC-V instructies. 

De programma's `objdump` en `readelf` zijn in Linux gecompileerd voor de specifieke ISA van je processor. Deze programma's kunnen dus niet rechtstreeks gebruikt worden om de ELF-files van xv6 te bekijken.

In de eerste oefenzitting hebben we een RISC-V compiler geïnstalleerd. In dezelfde package zaten gelukkig ook RISC-V varianten van `objdump` en `readelf`.

* Analyseer de `_helloworld` ELF-file die we in vorige oefenzitting gemaakt hebben met behulp van `riscv64-linux-gnu-readelf`

```shell
riscv64-linux-gnu-readelf -a user/_helloworld | less
```

Merk op dat het aantal secties in de RISC-V ELF verschilt van het aantal secties in de x86-ELF. De compiler en linker beslissen welke secties worden toegevoegd aan een ELF-bestand.

* Disassemble de RISC-V machinecode in `_helloworld` met `riscv64-linux-gnu-objdump` 

```shell
riscv64-linux-gnu-objdump -d user/_helloworld | less
```


### Application Binary Interface (ABI)

> **TODO** Kort ABI's introduceren?

### De `exec` system call

Nadat een proces met behulp van `fork` gekopieerd is, willen we in vele gevallen dat dit proces een andere taak kan uitvoeren.
Hiervoor gebruiken we de system call `exec`.

`exec` neemt als invoer een ELF-bestand en initialiseert de ELF-secties in het werkgeheugen. Exec is dus de loader van UNIX-omgevingen.

Exec is ook verantwoordelijk voor een correcte initialisatie van de registers. Nadat het geheugen en de registers correct geïnitialiseerd worden, wordt het programma gestart door te springen naar het *entry point*. Het entry point wordt gespecifieerd in de header-sectie van een ELF-bestand.


* Zoek het entry point dat gebruikt wordt in xv6-programma's door gebruik te maken van `readelf`

De linker is verantwoordelijk om het entry point van een programma correct te bewaren in een ELF-bestand. De GNU linker [ld](https://ftp.gnu.org/old-gnu/Manuals/ld-2.9.1/html_mono/ld.html#SEC24) die standaard door `gcc` gebruikt wordt, maakt gebruik van de volgende prioriteitenlijst om te bepalen wat er in het entry point bewaard wordt:

    1. the `-e' entry command-line option;
    2. the ENTRY(symbol) command in a linker control script;
    3. the value of the symbol start, if present;
    4. the address of the first byte of the .text section, if present;
    5. The address 0. 

In `xv6` wordt in de `Makefile` de `-e` flag opgegeven met als waarde `main`. De uitvoering van een xv6 executable zal dus starten bij het symbool `main`.

-->