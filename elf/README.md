## Exec

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
Meer bepaald, de secties worden samengevoegd in segmenten, ook beschreven in het ELF-bestand.
Een *loader* neemt als invoer een ELF-bestand en laadt de verschillende segmenten in het geheugen.
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