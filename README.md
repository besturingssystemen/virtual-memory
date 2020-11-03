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
- [Verschillende adresruimten](#verschillende-adresruimten)
  - [Inter-process isolatie](#inter-process-isolatie)
  - [Kernel isolatie](#kernel-isolatie)
- [Levenscyclus proces](#levenscyclus-proces)
  - [exec](#exec)
  - [fork](#fork)
  - [sbrk](#sbrk)
- [Toepassingen van Virtual Memory](#toepassingen-van-virtual-memory)
  - [Trampoline](#trampoline)
  - [Null-pointer exception](#null-pointer-exception)
  - [Guard pages](#guard-pages)
  - [Shared memory](#shared-memory)
  - [Permanente evaluatie: VDSO](#permanente-evaluatie-vdso)
- [TODO list](#todo-list)

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


# Verschillende adresruimten

>**TODO** 
>
>Uitleg over adresruimten:
>* Inter-proces isolatie
>* Proces-kernel isolatie
>* Concept software fault isolatie
>* ...
>Gebruik hiervoor oa.
>* uvmmap
>* kvmmap
>* Figuren over kernel en proces layouts

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

## Trampoline

**TODO**

## Null-pointer exception

**TODO**

## Guard pages

**TODO**

## Shared memory

**TODO**

## Permanente evaluatie: VDSO

**TODO** VDSO

# TODO list

Alle niet-inlined todo's:

**TODO** Identity map op gepaste plaats bespreken


[vm]: https://github.com/besturingssystemen/xv6-riscv/blob/d4cecb269f2acc61cc1adc11fec2aa690b9c553b/kernel/vm.c
[mappages]: https://github.com/besturingssystemen/xv6-riscv/blob/d4cecb269f2acc61cc1adc11fec2aa690b9c553b/kernel/vm.c#L138
[walk]: https://github.com/besturingssystemen/xv6-riscv/blob/d4cecb269f2acc61cc1adc11fec2aa690b9c553b/kernel/vm.c#L81
[riscv]: https://github.com/besturingssystemen/xv6-riscv/blob/d9160fb4b98e3ce04d3928c1fbd2ec26b3cc746a/kernel/riscv.h#L323
[proc]: https://github.com/besturingssystemen/xv6-riscv/blob/2821d43cc95b4f9faf79ff94daa5d3a8ea5e7861/kernel/proc.c
[fork]: https://github.com/besturingssystemen/xv6-riscv/blob/2821d43cc95b4f9faf79ff94daa5d3a8ea5e7861/kernel/proc.c#L244
[allocproc]: https://github.com/besturingssystemen/xv6-riscv/blob/2821d43cc95b4f9faf79ff94daa5d3a8ea5e7861/kernel/proc.c#L100
[uvmcopy]: https://github.com/besturingssystemen/xv6-riscv/blob/d4cecb269f2acc61cc1adc11fec2aa690b9c553b/kernel/vm.c#L298
