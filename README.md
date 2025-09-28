#  Rad sa keypad matrix 4x4 modulom na DE1Soc ploci

Ovaj projekat demonstrira upotrebu 4x4 matricnog keypada povezanog na DE1-SoC razvojnu plocu, uz podrsku za rad na Linux operativnom sistemu.  

Za pravilnu funkcionalnost, potrebno je konfigurisati FPGA tako da GPIO kontroler spojen na FPGA može da komunicira sa matricnim keypadom. GPIO linije se koriste za detekciju pritisnutih tastera kroz redove i kolone, a FPGA konfiguracija omogućava njihovo mapiranje i upotrebu na Linux strani.  

FPGA konfiguracija omogucena je fajlom `socfpga.rbf` i  `de1soc_handoff.patch`.

# Konfiguracija

U projektnu je koristen **Crosstool-NG** kao toolchain za kros-kompajliranje, kao alat koji automatizuje izgradnju kompletnog linux sistema je koristen **Buildroot 2024.02**. Da bi koristili ovaj buildroot, prvo je potrebno da preuzmemo repozitorijum ovog build sistema i da se prebacimo na odgovarajuću granu:
 ```bash
   git clone https://gitlab.com/buildroot.org/buildroot.git
   cd buildroot
   git checkout 2024.02
```
U ovom projektu korišten je **U-Boot 2024.01** kao bootloader. Njegova uloga je da učita FPGA konfiguraciju (`socfpga.rbf`), zatim kernel (`zImage`) i device tree (`socfpga.dtb`), te na kraju root filesystem sa SD kartice. Proces je automatizovan fajlom `buildroot/board/terasic/de1soc_cyclone5/boot-env.txt` koji definiše okruženje za pravilno pokretanje. Kao konfiguracija koristi se predefinisani `socfpga_de1_soc_defconfig`.

Da bi se koristio odgovarajući patch fajl u okviru Buildroot-a, potrebno je podesiti putanju u konfiguracionom meniju.  
To se radi unutar opcije **U-Boot → Custom patches directory** gdje treba unijeti: 
`board/terasic/de1soc_cyclone5/patches/u-boot`

Na ovaj način, tokom procesa kompilacije Buildroot automatski primjenjuje sve `.patch` fajlove iz navedenog direktorijuma na izvorni kod U-Boot-a prije nego što započne izgradnju. Ovo omogućava prilagođavanje bootloadera specifično za DE1-SoC ploču i ovaj projekat.

# Linux Kernel
Takodje je potrebno da ukljucimo odgovarajuci driver. To postizemo pozicioniranjem u root folder build-a i poktreanjem komande
 ```bash
make linux-menuconfig
```
Zatim u menu-ju odaberemo

```bash
Device Drivers  --->
   Input device support  --->
      Keyboards  --->
         [*] Keypad support
         <*> GPIO driven matrix keypad
```
Nakon odabira potrebno je sacuvati izmjene i make-ovati kernel.

# Hardware
Harware koji je koristen je:

- **DE1-SoC** (Cyclone V SoC) razvojna ploca:
 
  <img width="300" height="200" alt="image" src="https://github.com/user-attachments/assets/67af7117-571a-4ade-b78a-8c10ffc618cd" />

- **MikroE keypad matrix 4x4**

  <img width="300" height="200" alt="image" src="https://github.com/user-attachments/assets/be1d4c8f-bea6-4774-b3bc-31634ea61ef2" />

## Povezivanje
  ![povezivanje](https://github.com/user-attachments/assets/f45b6479-069e-4faf-8792-aaa4620e549d)
  <img width="534" height="382" alt="image" src="https://github.com/user-attachments/assets/2245ca71-19ff-4e56-9c75-a12995458523" />

  Pinove keypad matrixa koji predstavljaju redove (pinovi P4-P7) povezujemo na aptera_gpio pinove (0-3), a pinove koji predstavljaju kolone (P0-P3) povezujemo na alter_gpio pinove (4-7).

# Device Tree
Da bi module ispravno funkcionisao potrebno je konfigurisati uredjaj i prekida na FPGA GPIO kontroleru. Ova konfiguracija se nalazi u `buildroot/boardterasic/de1soc_cyclone5/socfpga_cyclone5_de1_soc.dts`. Izgled cvorova treba izgledati: 
```
soc {
		gpio_altr: gpio@0xff200000 {
		    compatible = "altr,pio-1.0";
		    reg = <0xff200000 0x10>;
		    interrupts = <0 40 3>;
		    gpio-controller;
		    #gpio-cells = <2>;
		    interrupt-controller;
		    #interrupt-cells = <2>;
		    altr,ngpio = <16>;
		    altr,interrupt-type = <3>; 
		};
			
	};

	matrix-keypad {
	    compatible = "gpio-matrix-keypad";
	    debounce-delay-ms = <20>;
	    col-scan-delay-us = <10>;
	    
	    row-gpios = <
		&gpio_altr 0 1
		&gpio_altr 1 1
		&gpio_altr 2 1
		&gpio_altr 3 1
	    >;
	    
	    col-gpios = <
		&gpio_altr 4 0
		&gpio_altr 5 0
		&gpio_altr 6 0
		&gpio_altr 7 0
	    >;
	    
	    linux,keymap = <
		0x00000002
		0x00010003
		0x00020004
		0x0003001E
		0x01000005
		0x01010006
		0x01020007
		0x01030030
		0x02000008
		0x02010009
		0x0202000A
		0x0203002E
		0x03000037
		0x0301000B
		0x030200E3
		0x03030020
	    >;
	};
```
- gpio_altr - GPIO kontroler koji je povezan na FPGA koji upravlja prekidima. Bitno ga je konfigurisati da bude gpio i interrupt kontroler kao i postaviti da je tip interrupta 3 (IRQ_TYPE_EDGE_BOTH) kao i prekid na 40. Vise o samom kontroleru mozemo vidjeti <a href='https://www.kernel.org/doc/Documentation/devicetree/bindings/gpio/gpio-altera.txt'>ovdje</a>.
- matrix-keypad - Za spajanje 4x4 matricne tastature koristen je linux driver <a href='https://www.kernel.org/doc/Documentation/devicetree/bindings/input/gpio-matrix-keypad.txt'>gpio-matrix-keypad</a>. Redovi su povezane na prva pina koji su definisani kao gpio altera, a kolone da sledeca 4.

# Testiranje
Nakon povezivanja i paljenja ploce, provjeravamo da li je driver prepoznat i testiramo funkcionalnosti modula. Kada ukucamo komandu `gpioinfo` dobijemo sledeci ispis:
<img width="605" height="283" alt="image" src="https://github.com/user-attachments/assets/dfd58a1c-b9bb-4db4-b577-c2b43d770214" />

Sa slike iznad mozemo primjetiti da su redovi i kolone uspjesno identifikovane i da su na dobrim linijama, takodje vidimo da su redovi definisani kao input pinovi, a kolone kao output. Ono sto moze stvarati potencijalni problem jeste sto su i redovi i kolone active-high iako su u dts kolone definisane sa vrijednoscu 0 sto je active-low. Kada smo vidjeli da je module uspjesno registrovan, mozemo preci na testiranje dugmadi. Sa tim da se radi o interrupt i dugmadima, mozemo testirati jednostavno pomocu **evtest** komande.

Kada pokrenemo komandu `evtest /dev/input/event0`, dobijemo sledeci ispis
  <img width="627" height="615" alt="image" src="https://github.com/user-attachments/assets/a6c4f0b9-1649-4b82-871e-5800106dc418" />

  Dio iznad i ispod dijela koji je oznacen, predstavlja ponasanje modula kada nije nijedno dugme pritisnuto. Ono sto mozemo primjetiti jeste da je **value = 2**, to znaci da driver prepoznaje kao da drzimo dugme A. 
  
  Oznaceni dio je prepoznavanje interrupta kada se pritisne dugme sa vrijednosti 1 (prvi red, prva kolona). Iz ispisa mozemo primjetiti da je evtest prepoznao kao da smo pritisnutli dugad **2, 3 i A**. Takodje mozemo vidjeti da prvi ide value = 0 pa value = 1, to znaci da je driver prepoznao da smo prvo otpustili dugmad 2, 3 i A pa onda ponovo pritisnuli.

Na osnovu ovih ispisa smo utvrdili da postoji problem sa identifikacijom dugmadi. Da bih provjerio da li je problem do dugmadi, definisao sam cvor unutar **gpio-keys** cvora: 
```
button_1 {
			label = "button_1";
			gpios = <&gpio_altr 0 1>;
			linux,code = <16>;
		};

```
Tako sto sam jedan pin povezao na gpio altera pin 0, a drugi kraj dugmeta na GND. Nakon pokretanja evtest-a desi se ocekivano ponasanje. Pritisak i pustanje dugmeta se registruje.

# Zakljucak
Iz par testova koje smo sproveli mozemo utvrditi da potencijalni problem moze nastati usled definisanja i redova i kolona kao active_high stanja, zato sto konstatno dobijamo da je dugme pritisnuto, a kad zapravo pritisnemo dumge, driver procita to stanje kao da su otpustena ostala dugmad. Ovdje problem moze nastati kod samog drivera.







