# fork

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
[uvmalloc]: https://github.com/besturingssystemen/xv6-riscv/blob/720a130ceafcc55ec3624b47e8a1368f3f5f00ae/kernel/vm.c#L215
[exec]: https://github.com/besturingssystemen/xv6-riscv/blob/720a130ceafcc55ec3624b47e8a1368f3f5f00ae/kernel/exec.c#L12
[exec load loop]: https://github.com/besturingssystemen/xv6-riscv/blob/720a130ceafcc55ec3624b47e8a1368f3f5f00ae/kernel/exec.c#L41-L59
[PTE_FLAGS_MASK]: https://github.com/besturingssystemen/xv6-riscv/blob/720a130ceafcc55ec3624b47e8a1368f3f5f00ae/kernel/riscv.h#L345
[struct proghdr]: https://github.com/besturingssystemen/xv6-riscv/blob/720a130ceafcc55ec3624b47e8a1368f3f5f00ae/kernel/elf.h#L30
[ELF flags consts]: https://github.com/besturingssystemen/xv6-riscv/blob/720a130ceafcc55ec3624b47e8a1368f3f5f00ae/kernel/elf.h#L44-L47
[elf.h]: https://github.com/besturingssystemen/xv6-riscv/blob/720a130ceafcc55ec3624b47e8a1368f3f5f00ae/kernel/elf.h
