# Page tables in xv6

**TODO** Kijken of alles aansluit na move uit main assignment

xv6 gebruikt een hoop [C preprocessor](https://en.wikipedia.org/wiki/C_preprocessor) definities om deze page tables correct te initialiseren en bewerken. In de header [`kernel/riscv.h`][riscv] worden vele van deze preprocessor functies gedefinieerd. 
De geïnteresseerden kunnen eens een kijkje nemen in deze header.
We gaan hier verder niet dieper op in.

De code om pagina's te mappen is al illustratiever.

* Bekijk de functie [`mappages`][mappages] in [`kernel/vm.c`][vm]. De code wordt hieronder uitgeklaard.

Deze functie probeert een bereik van virtuele adressen `[va , va + size - 1]` te mappen op fysische adressen `[pa, pa + size - 1]`.

Merk op dat een adres mappen slechts iets betekent indien je weet voor *welk proces* of welke *virtuele address space* je dit zal doen.
Herinner je dat de adressen voor verschillende processen vertaald worden naar andere fysieke adressen.
Elk proces heeft een eigen *virtual address space*. 
Het `satp`-register wijst hiervoor telkens naar de top-level page table van het huidige proces.
`satp` naar een andere page table laten verwijzen zorgt er dan ook voor dat de volledige adresvertaling anders verloopt.

De eerste parameter van [`mappages`][mappages] is een `pagetable_t`. Deze parameter bevat het fysieke adres van de top-level page table van de *virtual address space* waarin je wil mappen.


Merk ook op dat je niet zomaar één adres kan mappen.
Je mapt altijd één of meerdere pagina's.
Indien `va` dus in het midden van een pagina valt, zal deze volledige pagina gemapt moeten worden.
Daarom wordt `va` aan de start van deze functie met `PGROUNDDOWN` afgerond naar het eerste adres van de pagina waarin `va` valt.

Vervolgens zal de `for`-lus elke pagina in het meegegeven bereik proberen mappen.
In woorden doet de `for`-lus het volgende:
1. Voer een *page table walk* uit om de *page table entry* te zoeken die overeenkomt met de pagina van het virtuele adres.
   * De *page table walk* zal dus drie page tables moeten doorwandelen om de entry te vinden die overeenkomt met de pagina van het virtuele adres.
   * Paginatabellen die nog niet bestonden worden automatisch gealloceerd gedurende de walk.
2. Indien de page table entry al bestond (en *valid* is) wil dit zeggen dat de pagina eerder al gemapt was, mogelijks op een ander fysiek adres. Dit is een kritische fout.  De kernel zal *panic* oproepen, het besturingssysteem stopt. Je kan een virtueel adres uiteraard niet op hetzelfde moment mappen op twee verschillende fysieke adressen.

3. `PA2PTE` wordt gebruikt om het fysieke adres (waarop het virtuele adres gemapt moet worden) om te zetten naar een geldige page table entry. Dit resultaat wordt bewaard in de correcte page table. De pagina is nu gemapt.


We doen nu een poging de software page-table walk te begrijpen. 

  * Bekijk de functie [`walk`][walk] in `kernel/vm.c`.

De functie [`walk`][walk] zoekt de page table entry overeenkomstig met de pagina van het gegeven virtuele adres.
De parameter `alloc` bepaalt wat er moet gebeuren indien een bepaalde page table nog niet bestaat.
Indien `alloc` truthy is zal op dat moment een nieuwe paginatabel gealloceerd worden.

De for-loop zal de drie niveau's van page tables aflopen.
`PX` wordt hierbij gebruikt om de index te vinden in de huidige paginatabel.
Herinner je dat de index uit het virtuele adres gehaald wordt ofwel de eerste, tweede of derde groep van 9 bits te bekijken in het virtuele adres.
Vandaar dus dat `PX` als parameters meekrijgt welke groep bits bekeken moet worden in welk virtueel adres.
Zo kan de correcte page table entry bepaald worden.

In de `else` branch van de for-loop kan je zien hoe een paginatabel wordt aangemaakt.
Elke bit van de frame van de page table wordt eerst op 0 gezet met behulp van `memset`.
Je hebt nu een lege page table.

> :bulb: Een lege page table in RISC-V bestaat enkel uit 0-bits.

Vervolgens moet de verwijzing naar deze nieuwe lege page table nog aan de vorige page table worden toegevoegd.
Dat gebeurt door het fysieke adres van de zonet gealloceerde page table om te zetten naar een frame-nummer met `PA2PTE`.

In een page table entry wordt naast het frame-nummer ook bijgehouden of de page table entry effectief geldig is.
Dit gebeurt door een specifieke bit op 1 te zetten. Daardoor wordt de bitwise or-operator `|` gebruikt:

```c
*pte = PA2PTE(pagetable) | PTE_V;
```

> :information_source: Een bitwise OR-operatie kan gebruikt worden om specifieke bits van een binair getal op `1` te zetten. Neem een willekeurig binair getal `b` van 10 bits lang. Stel dat we bit 2 en bit 4 (geteld van rechts naar links) van dit getal op `1` willen zetten. We stellen nu het volgende binaire getal op: `0000001010`. Hierin staat dus een `1` op positie 2 en 4. Overtuig jezelf dat de bitwise OR-operatie `b | 0000001010` ervoor zorgt dat de 2de en 4de bit van `b` op `1` worden gezet, zonder de waarde van de andere bits te wijzigen.

Je kan meteen ook zien dat dezelfde valid-bit gebruikt wordt in de page table walk.
Wanneer de locatie van een specifieke page table entry gevonden is, zal bekeken worden of deze entry effectief valid is.
Dit gebeurt met behulp van de bitwise and-operator `&`: 

```c
if(*pte & PTE_V)
```

> :information_source: Een bitwise AND-operatie kan gebruikt worden om te controleren of een bepaalde bit van een binair getal `1` is. Stel dat we willen controleren dat bit 2  (geteld van rechts naar links) van een willekeurig binair getal `b` op `1` staat. We stellen nu het volgende binaire getal op: `0000000010` (`1` op positie 2). Overtuig jezelf dat indien `b & 0000000010` niet gelijk is aan `0`, `b` een `1` heeft op positie 2.

We hebben nu een zicht op hoe page tables gemapt worden in de code van xv6.
Tijd om te kijken hoe dit alles gebruikt wordt.

[vm]: https://github.com/besturingssystemen/xv6-riscv/blob/d4cecb269f2acc61cc1adc11fec2aa690b9c553b/kernel/vm.c
[mappages]: https://github.com/besturingssystemen/xv6-riscv/blob/d4cecb269f2acc61cc1adc11fec2aa690b9c553b/kernel/vm.c#L138
[walk]: https://github.com/besturingssystemen/xv6-riscv/blob/d4cecb269f2acc61cc1adc11fec2aa690b9c553b/kernel/vm.c#L81
[riscv]: https://github.com/besturingssystemen/xv6-riscv/blob/d9160fb4b98e3ce04d3928c1fbd2ec26b3cc746a/kernel/riscv.h#L323
[proc]: https://github.com/besturingssystemen/xv6-riscv/blob/2821d43cc95b4f9faf79ff94daa5d3a8ea5e7861/kernel/proc.c
[fork]: https://github.com/besturingssystemen/xv6-riscv/blob/2821d43cc95b4f9faf79ff94daa5d3a8ea5e7861/kernel/proc.c#L244
[allocproc]: https://github.com/besturingssystemen/xv6-riscv/blob/2821d43cc95b4f9faf79ff94daa5d3a8ea5e7861/kernel/proc.c#L100
[uvmcopy]: https://github.com/besturingssystemen/xv6-riscv/blob/d4cecb269f2acc61cc1adc11fec2aa690b9c553b/kernel/vm.c#L298
