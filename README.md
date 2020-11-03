In deze oefenzitting leren jullie over virtual memory.

- [Voorbereiding](#voorbereiding)
- [GitHub classroom](#github-classroom)
- [Introductie](#introductie)
- [Pagina mappen](#pagina-mappen)
  - [Page tables in xv6 en RISC-V](#page-tables-in-xv6-en-risc-v)
  - [Structuur page table](#structuur-page-table)
    - [Bitvoorstellling PTE](#bitvoorstellling-pte)
    - [Access control](#access-control)
    - [Page faults](#page-faults)
- [Address spaces in xv6](#address-spaces-in-xv6)
  - [Kernel address space](#kernel-address-space)
    - [Identity mapping](#identity-mapping)
    - [Kernel stacks](#kernel-stacks)
    - [Opbouw kernel address space in xv6](#opbouw-kernel-address-space-in-xv6)
  - [Process address space](#process-address-space)
  - [Null pointer exceptions](#null-pointer-exceptions)
  - [Trampoline](#trampoline)
  - [Security](#security)
  - [Inter-process isolatie](#inter-process-isolatie)
  - [Kernel isolatie](#kernel-isolatie)
- [Levenscyclus proces](#levenscyclus-proces)
  - [exec](#exec)
  - [fork](#fork)
  - [sbrk](#sbrk)
- [Pagetables inspecteren](#pagetables-inspecteren)
- [Toepassingen van Virtual Memory](#toepassingen-van-virtual-memory)
  - [Shared memory](#shared-memory)
  - [Permanente evaluatie: VDSO](#permanente-evaluatie-vdso)

# Voorbereiding

Ter voorbereiding van deze oefenzitting word je verwacht:
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

In deze sessie gaan we dieper in op het concept virtual memory.
We kijken hoe paging werkt en hoe dit gebruikt wordt om virtual memory te implementeren.
Vervolgens bekijken we enkele wijdverspreide toepassingen van virtual memory.

# Pagina mappen

Voor we duiken in de code van xv6 willen we eerst verzekeren dat je begrijpt hoe een besturingssysteem ervoor kan zorgen dat een specifiek virtueel adres van een proces gemapt wordt op een fysiek adres in het werkgeheugen.

Herinnner je dat een virtueel adres in een RISC-V processor die het Sv39 schema volgt, 39 bits lang is. xv6 is ontworpen voor dit RISC-V schema.

Onderstaande voorstelling vinden we terug in de [RISC-V privileged specification](https://riscv.org/technical/specifications/):

![Sv39-scheme](img/sv39-virtual-address.png)

* Neem een stuk papier en geef antwoord op de volgende vragen.

> :bulb: Op het einde van deze sectie kunnen jullie de antwoorden op deze vragen terugvinden. Dit is echter een zelftest om te kijken of je de concepten goed snapt. Door meteen naar de antwoorden te kijken zal je voor jezelf niet kunnen ontdekken welke delen nog onduidelijk zouden zijn. Indien je begrip van de concepten onvoldoende is zal je waarschijnlijk in de problemen komen bij de permanente evaluatie.


* Wat is het bereik van virtuele adressen in Sv39 (`[minimale adres, maximale adres]`)?

In xv6 worden slechts 38 bits van de 39 bits effectief gebruikt. Het maximale virtuele adres in xv6 wordt `MAXVA` genoemd.

> :bulb: `MAXVA` is eigenlijk het maximale adres + 1. `MAXVA` zelf is dus geen geldig adres.

* Wat is de waarde van `MAXVA`?
* Hoeveel gigabyte werkgeheugen kan dus maximaal geadresseerd worden door een xv6-proces?

> :bulb: Interessant weetje: 32-bit machines waren voor een lange periode de meest voorkomende consumentenmachines. 32-bit machines hebben registers van 32-bit lang. Er werd dus ook meestal gekozen om virtuele adressen 32-bit lang te maken. Je kan nu dus uitrekenen wat het maximaal geheugen is dat processen op dit soort machines konden adresseren (~4gb). Dit bleek voor sommige processen te weinig te zijn, een belangijke reden om van 32-bit naar 64-bit te schakelen.

Stel dat xv6 een nieuw proces inlaadt in het geheugen. Op dat moment moet xv6 de pagina's van dit proces in het fysieke geheugen plaatsen, en vervolgens een virtueel-naar-fysieke mapping opstellen. Deze mapping gebeurt niet willekeurig.
Onderstaande figuur, uit hoofdstuk 2 van het xv6-boek, toont de virtual memory layout van een process.

![xv6-virtual-layout](img/xv6-virtual-layout.png)

Een pagina die in de virtuele adresruimte van ieder xv6-proces geladen wordt, is de *trampoline*. 
In deze sessie gaan we in detail bekijken waarom deze pagina daar gemapt staat. 
Op dit moment in de sessie zijn we echter nog niet geinteresseerd in *wat* de trampolinepagina doet, wel in *hoe* de trampolinepagina gemapt wordt.

Neem aan dat:
1. De trampolinepagina in het geheugen gemapt staat in de frame met nummer 1234 (`PPN` = 1234).
2. Het virtuele adres `MAXVA-PGSIZE` (0x3ffffff000) moet verwijzen naar de eerste byte van deze trampolinepagina.
   
Je moet nu, als besturingssysteem, ervoor zorgen dat wanneer het nieuwe proces `MAXVA-PGZISE` probeert te dereferencen, de RISC-V hardware dit kan vertalen naar de eerste byte van frame 1234.

**TODO** Beter adres dat niet enkel uit index 511 bestaat?

* Welke stappen moet xv6 zetten om ervoor te zorgen dat deze adresvertaling correct uitgevoerd wordt?  Neem aan dat de top-level page table reeds bestaat en dat dit de enige page table is die al gealloceerd is voor dit proces.
  * Hoeveel page tables moet het besturingssysteem aanmaken?
  * Welke waarden moeten in deze tables ingevuld worden?

> :bulb: De vraag kan ook zo gesteld worden: hoe kan een besturingssysteem de trampolinepagina mappen op frame 1234?

> :warning: Indien je na 30 minuten in de oefenziting nog niet klaar bent met deze sectie, roep dan zeker een assistent om je verder te helpen.

## Page tables in xv6 en RISC-V

In xv6 worden pagina's gemapt met behulp van de functie `mappages`.
Dit is de definitie van deze functie:

```c
// Create PTEs for virtual addresses starting at va that refer to
// physical addresses starting at pa. va and size might not
// be page-aligned. Returns 0 on success, -1 if walk() couldn't
// allocate a needed page-table page.
int
mappages(pagetable_t pagetable, uint64 va, uint64 size, uint64 pa, int perm);
```

> :information_source: De implementatie van mappages vergt veel uitleg en leert ons weinig nieuwe informatie. Voor de geïnteresseerden geven we [hier](page-table-code/README.md) deze uitleg. Volg echter eerst de rest van de oefenzitting.



Stel dat je de trampolinepagina zou moeten mappen met behulp van de functie [`mappages`][mappages].

  * Welke waarden zou je toekennen aan de parameters `va` en `size`?

## Structuur page table

Page tables volgen een zeer specifieke structuur waarin elke bit een eigen betekenis heeft, zodat de RISC-V Memory Management Unit (MMU) deze efficiënt in hardware kan doorlopen.

### Bitvoorstellling PTE

Een page table kan je in het geheugen telkens terugvinden in het begin van een frame.
Elke page table bestaat uitsluitend uit 512 page table entries (PTE).
Een page table entry is niets anders dan een lange bitstring (32 bits) waarbij elke bit (of groep van bits) een eigen betekenis heeft.

Beantwoordt de volgende vragen:

* We weten dat een page table 512 entries heeft en we weten dat elke entry 32 bit groot is. Hoe groot is een volledige page table?
* We weten dat we een offset van 12 bit gebruiken om een byte in een frame of pagina te addresseren. Hoe groot zijn pagina's of frames in Sv32?
* We weten dat een page table geplaatst wordt aan de start van een frame. Past een page table in één enkele frame?



![sv39 page table entry](img/sv39-pte.png)

Bovenstaande figuur geeft de bit-layout weer van zo'n page table entry.

> :information_source: Zoals je ziet worden bits genummerd van rechts naar links.  [Hier](https://stackoverflow.com/questions/23551187/why-bits-are-numbered-from-right-to-left) kan je een goede verklaring vinden. Dit is een conventie die zo goed als altijd toegepast wordt. De meest rechtse bit wordt ook vaak de LSB (least significant bit) genoemd, aangezien deze bit het minste gewicht heeft. De meest linkse bit wordt vaak de MSB (most significant bit) genoemd, deze heeft het meeste gewicht. 

Laten we de verschillende bits van een page table entry verder bespreken:

* `PPN`: Bits 10 - 31 zijn de 21 bits die naar een frame verwijzen. Ze bevatten dus het frame nummer (*physical page number*, PPN)
* `U`: Bit 4 is de user bit. Een pagina kan enkel in user-mode gebruikt worden indien de `U`-bit actief is.
* `X`: Bit 3 is de executable bit. Indien een pagina code bevat kan deze enkel uitgevoerd worden indien de `X`-bit van deze pagina actief is.
* `W`: Bit 2 is de writeable bit. Data kan enkel naar een pagina geschreven worden indien `W` actief is.
* `R`: Bit 1 is de readable bit. Data kan enkel van een pagina gelezen worden indien `R` actief is.
* `V`: Bit 0 is de valid bit. Enkel indien deze bit actief is, wordt de gehele page table entry als geldig beschouwd. Alle voorgaande bits hebben enkel een betekenis indien `V` actief is. Om een pagina te *unmappen* is het dus voldoende om `V` op 0 te zetten.

> :information_source: Enkele minder relevante bits voor de geïnteresseerden: 
> * `RSW`: Bits 8 - 9 mogen door een besturingssysteem gebruikt worden op een manier naar keuze. Ze worden door de MMU genegeerd. xv6 gebruikt deze bits niet.
> * `D`, `A`: Bit 7 is de dirty bit, bit 6 de accessed bit. De dirty bit houdt bij of een pagina gelezen, geschreven of opgehaald werd sinds de laatste keer dat de accesed bit op 0 werd gezet. Deze bits worden gebruikt om het cachen en swappen van pagina's efficiënter te laten verlopen. Voor meer infomatie verwijzen we jullie naar Silberschatz (8.4.1 Basic Page Replacement of zoek in je termen-index naar de dirty bit).
> * `G`: Bit 5 is de *global mapping* bit. Page tables worden per proces gealloceerd. Het is echter mogelijk om bepaalde page tables te delen tussen *alle* processen. Indien je de `G` bit in dat geval actief maakt kan de processor betere performantie leveren. Deze page tables zou je bijvoorbeeld permanent in een cache geladen kunnen houden.

### Access control

Bij de bespreking van de bitvoorstelling van een page table entry is duidelijk geworden dat paging niet uitsluitend gebruikt wordt om adressen te vertalen.
Het vertalingsproces wordt ook gebruikt om aan access control te doen.

Zo kan de kernel bepaalde pagina's ontoegankelijk maken voor een user-space proces, door de `U`-bit op 0 (inactief) te zetten.
Pagina's kunnen read-only gemaakt worden door `R` te activeren en `W` en `X` te deactiveren.

Onderstaande figuur geeft enkele mogelijke combinaties van `R`, `W` en `X` en hun betekenis:

![rwx encoding](img/rwx-encoding.png)


### Page faults

Wanneer we een virtueel adres of pagina aanspreken en ons niet aan de access control regels geëncodeerd in de page table entry houden, treedt een *page fault exception* op.

Een *exception* in hardware zorgt ervoor dat de huidige programma-executie onderbroken wordt.
De processor switcht naar van modus en springt naar een vast adres in de code van de kernel.
De code op dit adres noemen we de *trap handler*.

> :information_source: Merk op dat ook de `ecall`-instructie uit vorige oefenzitting ervoor zorgde dat er naar de *trap handler* gesprongen werd. Deze trap handler moet dus bepalen wat de reden is van de trap.

De meesten van jullie zullen onbewust al een exception hebben veroorzaakt, door een bug in jullie code.
xv6 zal in dat geval het falende proces meteen beëindigen.
Vervolgens print xv6, via de trap handler, de reden van de exception.
Hier enkele mogelijkheden:

> :information_source: De onderstaande tabel is enkel geldig indien een exception optreedt die in supervisor mode kan worden afgehandeld. Sommige exceptions worden afgehandeld in machine mode, daarvoor kan je een andere tabel vinden in de [RISC-V specificaties](https://riscv.org/technical/specifications/).

![Supervisor exception codes](img/supervisor-exception-codes.png)


Denk nu terug aan de eerste oefenzitting. 
Je schreef een hello world programma met een simpele return uit main.
`crt0` was nog niet toegevoegd, dus de `jr ra` instructie sprong naar een waarde in `ra` die nergens geïnitialiseerd was.
Stel dat je springt naar een willekeurig adres in de virtuele adresruimte van je proces.

  * Welke excepties uit bovenstaande tabel kunnen optreden? Merk op dat page faults niet de enige vorm van exceptions zijn.

**TODO** Oefening die fault veroorzaakt hier?

Page faults kunnen ten slotte dus ook optreden op het moment dat de MMU een virtuele adres probeert te vertalen maar een bepaalde page table entry niet gevonden wordt.
De pagina is op dat moment dus niet gemapt.

Bepaalde schema's zoals *demand paging* zullen pagina's pas mappen nadat de pagina is aangesproken.
Ze vangen de *page fault* op, mappen de betrokken pagina (indien mogelijk) en hervatten de executie van het proces.
xv6 heeft geen demand paging.


# Address spaces in xv6

Genoeg over page tables.
Laten we eens op een hoger niveau kijken hoe dit alles door xv6 gebruikt wordt om de kernel en processen eigen adresruimten toe te kennen.

## Kernel address space

In hoofdstuk 3 van het xv6 boek kwamen we de volgende figuur tegen:

![kernel-address-space](img/xv6-kernel-address-space.png)

Op het moment dat code in de kernel uitvoert, wijst het `satp`-register naar de top-level page table van de kernel.
Hierdoor worden adressen vertaald zoals afgebeeld op bovenstaande afbeelding.

Zo zie je dat de code (`text`-sectie) van de kernel ingeladen is op adres `0x80000000`.
De pagina's met code zijn gemapt als read/execute.
Je kan de code van de kernel dus niet overschrijven (tenzij je eerst de page tables aanpast).
Net boven de code wordt de `data`-sectie (globale variabelen) van de kernel gemapt.

### Identity mapping

Deze mapping is speciaal en volgt een identity mapping.
Elk fysisch adres wordt gemapt op hetzelfde overeenkomstige virtueel adres.
Hiermee bedoelen we: virtueel adres `0x80000000` wordt gemapt op fysisch adres `0x80000000`.
Virtueel adres `0x80000001` wordt gemapt of fysisch adres `0x80000001`, enzovoort.

Er is een belangrijke reden om deze mapping op deze manier uit te voeren.
De code om paginatabellen te bewerken zit in de text section van de kernel.
Deze code moet continu schrijven naar fysieke adressen.
Door de identity map werk je in feite rechtstreeks met fysieke adressen, waardoor page table code veel eenvoudiger geschreven kan worden.
De code die page tables bewerkt in xv6 zou niet werken zonder deze identity map.

### Kernel stacks

De kernel reserveert voor ieder proces een eigen *kernel stack*
Deze stack wordt gebruikt als *call stack* op het moment dat een proces switcht naar supervisor-mode, bijvoorbeeld als gevolg van een `ecall` of een *exception*.

Tussen elke kernel stack vind je een *guard page*.
Dit is meteen een interessante toepassing van virtual memory.
De grootte van een per-process kernel stack wordt door xv6 gelimiteerd tot 1 pagina.
Indien een stack groter wordt dan het gealloceerde geheugen spreken we over een *stack overflow*.
Zonder bescherming tegen een *stack overflow* zou dit ervoor kunnen zorgen dat de stack kritische kerneldata overschrijft.

Om te detecteren wanneer een xv6 kernel stack vol is, wordt de pagina boven de stack niet gemapt.
Wanneer je probeert te schrijven naar een unmapped pagina krijg je een page fault.
Op die manier kan een stack overflow automatisch gedetecteerd en vermeden worden.

### Opbouw kernel address space in xv6

De functie [`kvmmake`][kvmmake] roept `mappages` op (via `kvmmap`) om de address space van de kernel op te bouwen.

* Bekijk de functie [`kvmmake`][kvmmake]. Deze code zou ondertussen begrijpbaar moeten zijn.

## Process address space

Op het moment dat een proces in user-mode uitvoert zorgt de kernel ervoor dat het `satp`-register wijst naar de top-level page table van het huidige proces.
Zo zorgen we ervoor dat wanneer een user-space programma naar een virtueel adres aanspreekt, dit vertaald wordt naar een eerder gekozen fysieke locatie in het geheugen.
Elk proces heeft zo een eigen virtuele adresruimte.

Ook voor processen kiest xv6 een vaste layout om deze adresruimte op te delen:

![xv6 process address space](img/xv6-process-address-space.png)

Ieder proces krijgt een eigen pagina gealloceerd om de *call stack* van het programma te bewaren.
Onder deze stackpagina wordt, net zoals in de kernel, een guard pagina geplaatst die niet gemapt wordt in het geheugen, om stack overflows te detecteren.

De code van een proces wordt in xv6 gemapt op virtueel adres 0, dus op de eerste pagina in de virtuele adresruimte.
Deze code kan één of meerdere pagina's groot zijn.
Boven de code (op hogere addressen) wordt de global data van het proces gemapt.

> :information_source: We hebben xv6 reeds aangepast zodat de text-sectie en de data-secties elk op eigen pagina's gemapt worden. Dit wordt niet gereflecteerd in de figuur. Door deze aanpassing is het voor ons mogelijk om bepaalde secties (bvb .rodata) read-only te mappen. Dat zou niet mogelijk zijn indien code en data pagina's zouden delen.

**TODO** Hier ergens oefening om .rodata read only te mappen invoegen

## Null pointer exceptions

* Compileer en voer in je Linuxdistributie (niet in xv6) het volgende simpele programma uit:

```c
#include <stdio.h> //#include "user/user.h" in xv6! 
int main(){
  int *p = 0; //p is a pointer to the address 0
  printf("The value at address 0 is %d", *p);
  return 0;
}
```

* Wat is de output van je programma? *(hint: een welbekende foutmelding)*
* Compileer nu hetzelfde programma in xv6 en voer dit uit. Wat is daar je output?

De fout die we krijgen in onze eigen Linuxdistributie krijgen we niet in xv6.
Tijd om eens na te denken over wat bovenstaande code net doet.
In feite niet veel meer dan het adres 0 uitlezen.

In de meeste Linux-distributies wordt het adres 0 gebruikt om aan te geven dat een pointer nergens naar verwijst, of niet geïnitialiseerd is.
Denk bijvoorbeeld aan een simpele linked list.
Indien het *next* veld van een lijstelement 0 is, hebben we het laatste element van de lijst bereikt.
Kortom, het is niet de bedoeling dat er nuttige informatie op adres 0 te vinden is.

Het lezen van informatie uit adres 0 wijst dus bijna zeker op een fout in je code.
Om ervoor te zorgen dat deze fouten gedetecteerd kunnen worden, zullen Linux-distributies ervoor zorgen dat het adres 0 ontoegankelijk is.

* Hoe zou je in xv6 ervoor kunnen zorgen dat het adres 0 ontoegankelijk is? Kan je op een of andere manier een *fault* veroorzaken bij het lezen van adres 0?

xv6 doet dit echter niet. In xv6 staat er code op adres 0. Adres 0 is gewoon toegankelijk. Dat is in feite een zeer slechte beslissing, want zo is het veel lastiger fouten te vinden in C-code (in plaats van een exception krijg je willekeurge data, waarschijnlijk de bytecode van een functie, bij het uitlezen van adres 0).

* De error message die je in Linux kreeg was waarschijnlijk niet *page fault*, maar het leek er wel op. Zou je dit verschil kunnen verklaren?

## Trampoline

Het is je misschien opgevallen dat een pagina genaamd *trampoline* gemapt is in de adresruimte van de kernel en in de adresruimte van ieder proces.
Misschien herinner je je zelfs dat we verwezen hebben naar de trampolinepagina in de vorige oefenzitting.
Tijd om uit te leggen wat deze pagina daar doet.

Eerder hebben we vermeld dat, wanneer er zich een exception voordoet, of wanneer we een `ecall`-uitvoeren, de RISC-V hardware van uitvoermodus verandert en springt naar de trap handler.
Neem nu bijvoorbeeld de `ecall`.
Uitgevoerd vanuit user-mode zorgt dit ervoor dat de processor switcht naar supervisor-mode en vervolgens springt naar het adres van de trap handler.

> :information_source: Ook interrupts zorgen voor een sprong naar de trap handler. Dit wordt in een latere sessie behandeld.

De processor wil nu dus code van de kernel uitvoeren, om te system call af te kunnen handelen.
De code van de kernel is echter gemapt in een eigen adresruimte.
Op het moment dat de `ecall` wordt uitgevoerd, wijst het
`satp`-register echter nog steeds naar de page table van het user proces.
De processor springt dus naar de trap handler, de trap handler moet bijgevolg gemapt zijn in het user proces.

Om die reden mappen we de trampolinepagina in iedere adresruimte op hetzelfde adres.
Zo kan de RISC-V hardware simpelweg springen naar een vast adres en weten we zeker welke pagina daar gemapt zal staan.

* Bekijk de code van `kernel/trampoline.S` **TODO** link

De trampoline zal alle registers bewaren in het trapframe bij het wisselen van user-space naar kernel-space.
Daarnaast zal de trampoline ook `satp` wisselen zodat deze wijst naar de top-level page table van de kernel.

Je vraagt je misschien af waarom de trampoline ook in de kernel address space gemapt moet staan.
Meteen nadat het `satp`-register aangepast wordt, wisselt meteen de volledige adresvertaling van de processor.
De programmateller laadt instructies uit het geheugen op basis van adressen.
Stel dat je `satp` zou wijzigen zonder op dezelfde plaats in de andere adresruimte dezelfde code te mappen, zou plots de code uitgevoerd worden die in de andere adresruimte op dezelfde plaats gemapt staat (en dus niet meer de trampolinecode).

* De trampolinepagina staat gemapt met `R` (read) en `X` (execute) permissies. Stel dat de trampolinepagina ook `W` (write) permissies zou hebben. Kan je bedenken hoe dit voor problemen zou kunnen zorgen?

## Security

>**TODO** 
>
>Uitleg over adresruimten:
>* Inter-proces isolatie
>* Proces-kernel isolatie
>* Concept software fault isolatie
>* ...

Een belangrijk gevolg van virtual memory dat we tot nog toe niet besproken hebben is het feit dat virtual memory grotendeels zorgt voor *inter-process isolatie*.

## Inter-process isolatie

**TODO** Deze sectie is nog heel temporary, beter uitschrijven en bondiger maken

Om veiligheidsredenen is het belangrijk dat verschillende processen op een machine niet zomaar in elkaars geheugenruimte kunnen.
Stel je voor dat je een password manager gebruikt en even later een online spel opstart.
Het zou rampzalig zijn indien de makers van dat online spel hun code zo zouden kunnen schrijven dat deze het geheugen van de password manager op de achtergrond zouden kunnen uitlezen en doorsturen naar hun eigen servers.

Je zou kunnen zeggen: oké, maar ik vertrouw de makers van het online spel.
Zelfs dan kan het zijn dat het spel *exploitable* is.
Jullie weten ondertussen allemaal hoe eenvoudig het is om een foutief C-programma te schrijven.
Elke programmeur maakt fouten en vele programma's zijn gigantisch groot.
Eén enkele kleine fout kan voldoende zijn om een proces volledig over te nemen.
Indien één van je medespelers jouw spel kan exploiten, zou deze vervolgens nog steeds aan de inhoud van je password manager geraken.

Het is belangrijk dat ieder proces op een machine geïsoleerd is van elkaar. 
Communicatie tussen processen kan (denk bijvoorbeeld aan pipes), maar dit moet expliciet de bedoeling zijn. 
Processen mogen elkaars geheugen niet uitlezen, en zeker niet bewerken.

## Kernel isolatie

**TODO**

# Levenscyclus proces

In de [sessie over os interfaces](https://github.com/besturingssystemen/os-interfaces) hebben jullie in de permanente evaluatie ontdekt dat wanneer een proces geforked wordt, je plots twee processen hebt met *dezelfde* adressen maar toch mogelijks andere waarden op deze adressen.

* Verklaar dit aan de hand van je kennis over virtual memory

In de [sessie over system calls](https://github.com/besturingssystemen/system-calls#levenscyclus-proces) hebben we jullie vervolgens verteld hoe een proces aangemaakt wordt met `fork` en een taak toegewezen krijgt met `exec`.

Ondertussen hebben we voldoende informatie om te duiken in de implementatie van deze functies.

## exec

> **TODO**  Korte uitleg geven over werking exec, verder verwijzen naar het boek en eventueel naar [exec/README.md](exec/README.md)

**TODO: Oefening** Page table dumps bij exec, fork, sbrk?

**TODO: Oefening** R/W/X .rodata (etc) fixen bij exec met tweede pass

## fork

* Bekijk de implementatie van [`fork`][fork] in [`kernel/proc.c`][proc]

> :information_source: In de komende delen zullen we soms code tegenkomen waarin gebruik wordt gemaakt van locks. Hierop zullen we dieper ingaan in de sessie over synchronisatie.

De eerste functie die `fork` oproept is [`allocproc`][allocproc].
Hier wordt het nieuwe proces toegevoegd in de process table, een structuur die gebruikt wordt om alle actieve processen te bewaren. 
Het proces krijgt een nieuwe `pid` toegewezen.
Daarnaast worden de nodige datastructuren aangemaakt die het besturingssysteem bewaart per proces:
* De trapframe
* Een lege top-level page table
  * We maken dus een nieuwe, lege virtuele adresruimte
* Een context. Hier gaan we dieper op in gedurende de sessie over scheduling

Op dit moment heeft ons nieuwe proces dus nog een lege virtuele adresruimte.
We weten echter dat fork een kopie maakt van de oude adresruimte.
Alle adressen die gemapt waren in het parent proces moeten ook gemapt worden in het child proces.
Elk van die adressen moet vervolgens ook dezelfde waarde toegekend krijgen.

* Denk even na welke stappen je conceptueel zou moeten nemen om de virtuele adresruimte te kopiëren zoals hierboven beschreven

We kunnen nu kijken of je assumpties correct waren.
De functie `uvmcopy` voert deze kopie uit.

* Bekijk de functie [`uvmcopy`][uvmcopy] in `kernel/vm.c`

De functie krijgt drie parameters: de paginatabel van de parent (`old`), de paginatabel van de child (`new`), en de grootte van het geheugen van het parent-proces (`sz`).

> In xv6 wordt ervoor gezorgd dat alle gebruikte virtuele adressen in het bereik [0, `sz`-1] vallen. `sz` kan worden vergroot gedurende de levensduur van een proces met behulp van de system call `sbrk`.

Vervolgens zal `uvmcopy` itereren over alle gemapte pagina's in dat bereik.
Per pagina worden de volgende stappen uitgevoerd:
1. Eerst wordt door middel van een software page walk de correcte page table entry gevonden
   * Uit deze entry wordt (met `PTE2PA`) het fysieke adres van de frame gehaald, waar de pagina gemapt staat
   * Daarnaast worden ook de flags van de pagina opgehaald met `PTE_FLAGS`
2. Vervolgens wordt een lege frame gevonden in het geheugen met behulp van `kalloc()`
3. Met `memmove` wordt de volledige inhoud van de pagina van de parent gekopieerd naar de nieuw gealloceerde pagina
4. Met `mappages` wordt deze nieuwe pagina gemapt op de zonet aangemaakte frame. Hierbij wordt er dus voor gezorgd dat het virtuele adres van de pagina naar de nieuwe frame mapt in het child-proces.

Hopelijk was je zelf ook tot gelijkaardige stappen gekomen in de voorgaande denkoefening. Laten we terugspringen naar fork.
Nu we de virtuele adresruimte hebben gekopieerd, resteert ons voornamelijk nog wat boekhouding. 

We kopiëren het veld `sz` in de `struct proc` van parent naar child en we zetten laten het `parent`-veld van de child verwijzen naar het parent-proces.
We kopiëren de bewaarde registers in het `trapframe`.
Door `a0` te overschrijven in de child zorgen we ervoor dat `fork` 0 zal returnen in het geforkte proces.

We zorgen ervoor dat het child-proces eigen verwijzingen heeft naar elke open file van het parent proces, zodat wanneer het parent proces een file zou sluiten, xv6 nog steeds weet dat dit bestand open moet blijven (want de child kan dit bestand nog nodig hebben).

Ten slotte wordt de naam van het proces gekopieerd, de `pid` van de child wordt correct ingesteld en de `state` wordt op `RUNNABLE` gezet (hierover meer in de scheduling oefenzitting).

Fork returned het `pid` van de parent in het parent-proces, vandaar ten slotte de return statement.

## sbrk

**TODO**

# Pagetables inspecteren

Om het gemakkelijk te maken de pagetables van processen te bekijken, hebben we een syscall toegevoegd: [`vmprintmappings`][sys_vmprintmappings].
Deze syscall zal alle geldige mappings van het oproepende proces afprinten in het volgende formaat:

<pre>
{va} -> {pa}, mode={U|S}, perms={r|-}{w|-}{x|-}
</pre>

Hier is `va` het virtuele adres van een page, `pa` het fysieke adres vet het overeenkomende frame.
`mode` is `U` voor een user page of `S` voor een supervisor (kernel) page.
`perms` toont de permissie flags voor de pagina: readable (`r`), writable (`w`), en executable (`x`).
Elk veldje bevat een `-` als de permissie niet gezet is.

De implementatie van `vmprintmappings` vind je in [`vm.c`][sys_vmprintmappings impl].
Alhoewel je zeker niet elk detail hoeft te begrijpen, is het nuttig om de implementatie eens te bekijken.

- Roep `vmprintmappings` in je hello world programma en vergelijk het resultaat met figuur 3.4 in het xv6 boek.
  Probeer elke mapping te begrijpen en kijk zeker naar de `mode` en `perms` velden.
- Maak een programma dat `vmprintmappings` oproept voor en na een oproep naar `sbrk(1)`.
  Verklaar het verschil in de outputs.
- Maak een programma dat `fork` gebruikt om een child proces te maken en vervolgens `vmprintmappings` oproept in parent en child.
  Verklaar de output.
  (Hint: gebruik `wait` in de parent om te wachten tot het child klaar is met uitvoeren om te voorkomen dat de outputs van `vmprintmappings` door elkaar geprint worden.)
- Bekijk nu het effect van `exec` op de mappings.
  Roep `vmprintmappings` voor de oproep naar `exec` en ook in het programma dat je met `exec` uitvoert.

# Toepassingen van Virtual Memory

Het concept virtual memory zou ondertussen heel duidelijk moeten zijn.
We weten wat het is, hoe xv6 dit implementeert en hoe de RISC-V processor dit gebruikt.

Virtual memory heeft vele handige toepassingen. 
Enkele van deze toepassingen hebben we reeds theoretisch bekeken:

* Programmacode kan geschreven worden in de aanname dat elk mogelijk adres (in de regio [0, `MAXVA`-1]) tot het proces behoort.
* Pagina's van processen kunnen naar de harde schijf geswapt worden wanneer het werkgeheugen vol is. Consulteer je handboek voor meer informatie.

> :information_source: Swapping als concept was vroeger belangrijker dan nu. Het is beter om te voorkomen dat je werkgeheugen vol komt te zitten door op tijd [meer RAM te downloaden](https://downloadmoreram.com).

* Externe fragmentatie wordt vermeden. 
  * Je hebt wel nog interne fragmentatie, wanneer je een pagina reserveert en niet de volledige pagina moet gebruiken. Dit is op moderne machines echter niet echt een probleem.

* **TODO** Interprocess isolatie (al besproken)

## Shared memory

**TODO**

## Permanente evaluatie: VDSO

**TODO** VDSO


[vm]: https://github.com/besturingssystemen/xv6-riscv/blob/d4cecb269f2acc61cc1adc11fec2aa690b9c553b/kernel/vm.c
[mappages]: https://github.com/besturingssystemen/xv6-riscv/blob/d4cecb269f2acc61cc1adc11fec2aa690b9c553b/kernel/vm.c#L138
[walk]: https://github.com/besturingssystemen/xv6-riscv/blob/d4cecb269f2acc61cc1adc11fec2aa690b9c553b/kernel/vm.c#L81
[riscv]: https://github.com/besturingssystemen/xv6-riscv/blob/d9160fb4b98e3ce04d3928c1fbd2ec26b3cc746a/kernel/riscv.h#L323
[proc]: https://github.com/besturingssystemen/xv6-riscv/blob/2821d43cc95b4f9faf79ff94daa5d3a8ea5e7861/kernel/proc.c
[fork]: https://github.com/besturingssystemen/xv6-riscv/blob/2821d43cc95b4f9faf79ff94daa5d3a8ea5e7861/kernel/proc.c#L244
[allocproc]: https://github.com/besturingssystemen/xv6-riscv/blob/2821d43cc95b4f9faf79ff94daa5d3a8ea5e7861/kernel/proc.c#L100
[uvmcopy]: https://github.com/besturingssystemen/xv6-riscv/blob/d4cecb269f2acc61cc1adc11fec2aa690b9c553b/kernel/vm.c#L298
[sys_vmprintmappings]: https://github.com/besturingssystemen/xv6-riscv/blob/b26f9c647c1b8d27b7a7b3b374422c87591a8e1a/kernel/sysproc.c#L99
[sys_vmprintmappings impl]: https://github.com/besturingssystemen/xv6-riscv/blob/b26f9c647c1b8d27b7a7b3b374422c87591a8e1a/kernel/vm.c#L433-L460
[kvmmake]: https://github.com/besturingssystemen/xv6-riscv/blob/d4cecb269f2acc61cc1adc11fec2aa690b9c553b/kernel/vm.c#L20
