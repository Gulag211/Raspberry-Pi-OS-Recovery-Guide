# Raspberry Pi OS Recovery Guide

Podrobný český návod, jak **stáhnout a zapsat Raspberry Pi OS na novou microSD kartu přímo z jiného funkčního Raspberry Pi**, bez Raspberry Pi Imageru na PC.

Tento postup byl prakticky proveden při obnově systému dne **25. 9. 2026**. Funkční Raspberry Pi běželo ze své původní microSD karty a druhá 16GB microSD byla vložena přes USB čtečku.

> [!CAUTION]
> Příkaz `dd` přepisuje zadané zařízení bez dalšího potvrzení. Nejdůležitější část celého návodu je správně určit, která jednotka je systémová karta a která je nová karta v USB čtečce. **Nikdy nekopíruj `/dev/sda` z tohoto návodu naslepo. Nejdřív vždy proveď kontrolu pomocí `lsblk`.**

## Co potřebuješ

- jedno funkční Raspberry Pi s přístupem k internetu,
- druhou microSD kartu,
- USB čtečku microSD,
- přístup k terminálu nebo SSH,
- dostatek volného místa pro stažený komprimovaný obraz Raspberry Pi OS.

V ověřeném testu byla systémová karta:

```text
/dev/mmcblk0
```

a nová 16GB microSD v USB čtečce:

```text
/dev/sda
```

**U tebe mohou být názvy jiné.**

---

# 1. Vlož novou kartu do USB čtečky

USB čtečku s cílovou microSD připoj k běžícímu Raspberry Pi.

Než uděláš cokoli dalšího, spusť:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL
```

Typický příklad:

```text
NAME         SIZE TYPE FSTYPE MOUNTPOINTS MODEL
mmcblk0     14.8G disk
├─mmcblk0p1  512M part vfat   /boot/firmware
└─mmcblk0p2 14.3G part ext4   /
sda         14.8G disk
├─sda1       ...
└─sda2       ...
```

Zásadní kontrola:

- zařízení obsahující oddíl připojený jako `/` je **běžící systém** a nesmí se přepsat,
- druhé zařízení v USB čtečce je cíl.

V našem ověřeném případě tedy:

```text
ZDROJOVÝ BĚŽÍCÍ SYSTÉM = /dev/mmcblk0
CÍLOVÁ USB microSD      = /dev/sda
```

Pokud si nejsi jistý, **nepokračuj**.

Pro ještě jistější kontrolu můžeš před vložením a po vložení karty spustit `lsblk` znovu a porovnat, které zařízení přibylo.

---

# 2. Zkontroluj volné místo

Při našem testu měl komprimovaný image velikost přibližně **532 MiB**. Adresář `/tmp` měl pouze asi **454 MiB**, protože byl připojen jako tmpfs, takže se tam image nevešel.

Proto jsme použili trvalý adresář v domovském adresáři.

Zkontroluj:

```bash
df -h /home/pi
df -h /tmp
```

Vytvoř pracovní adresář:

```bash
mkdir -p /home/pi/os-images
cd /home/pi/os-images
```

---

# 3. Vyber správný Raspberry Pi OS image

Pro ověřený test jsme použili:

```text
Raspberry Pi OS Lite 32-bit
Debian 13 Trixie
image: 2026-09-15-raspios-trixie-armhf-lite.img.xz
```

Konkrétní tehdy použitá oficiální adresa byla:

```text
https://downloads.raspberrypi.com/raspios_lite_armhf/images/raspios_lite_armhf-2026-09-15/2026-09-15-raspios-trixie-armhf-lite.img.xz
```

Tento soubor je uveden jako **ověřený příklad**, nikoli jako příkaz používat navždy stejnou verzi. Při nové instalaci může být na oficiálním serveru novější obraz.

Oficiální adresář Raspberry Pi OS Lite 32-bit:

https://downloads.raspberrypi.com/raspios_lite_armhf/images/

---

# 4. Stáhni image

Pro přesnou reprodukci našeho testu:

```bash
cd /home/pi/os-images

wget https://downloads.raspberrypi.com/raspios_lite_armhf/images/raspios_lite_armhf-2026-09-15/2026-09-15-raspios-trixie-armhf-lite.img.xz
```

Po dokončení zkontroluj soubor:

```bash
ls -lh /home/pi/os-images/
```

Při našem testu měl stažený soubor přesnou velikost:

```text
557524176 bytes
```

tedy přibližně:

```text
532M
```

Přesnou velikost zobrazíš:

```bash
stat -c '%n  %s bytes' /home/pi/os-images/2026-09-15-raspios-trixie-armhf-lite.img.xz
```

Pro tento konkrétní image očekáváme:

```text
/home/pi/os-images/2026-09-15-raspios-trixie-armhf-lite.img.xz  557524176 bytes
```

U jiné verze obrazu bude velikost samozřejmě jiná.

---

# 5. Ověř, že XZ archiv není poškozený

Tento krok nepřeskakuj.

```bash
xz -t /home/pi/os-images/2026-09-15-raspios-trixie-armhf-lite.img.xz
echo $?
```

Příkaz `xz -t` při úspěchu obvykle nic nevypíše.

Následující:

```bash
echo $?
```

musí vrátit:

```text
0
```

**0 = kontrola archivu proběhla úspěšně.**

Pokud dostaneš jiné číslo nebo `xz` vypíše chybu, image na kartu nezapisuj. Soubor znovu stáhni.

---

# 6. Ještě jednou ověř cílovou kartu

Bezprostředně před `dd` znovu spusť:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL
```

V našem testu platilo:

```text
/dev/mmcblk0 = běžící Raspberry Pi
/dev/sda     = nová microSD v USB čtečce
```

> [!CAUTION]
> Následující příkaz kompletně přepíše obsah cílového zařízení. Pokud zadáš místo USB karty systémový disk, zničíš běžící instalaci.

Pokud se oddíly cílové karty automaticky připojily, zjisti je pomocí:

```bash
lsblk -o NAME,MOUNTPOINTS /dev/sda
```

a případné připojené oddíly před zápisem odpoj, například:

```bash
sudo umount /dev/sda1 2>/dev/null || true
sudo umount /dev/sda2 2>/dev/null || true
```

---

# 7. Zapiš Raspberry Pi OS přímo z komprimovaného image

V našem ověřeném případě byla cílová karta `/dev/sda`, proto jsme použili:

```bash
xzcat /home/pi/os-images/2026-09-15-raspios-trixie-armhf-lite.img.xz | sudo dd of=/dev/sda bs=4M status=progress conv=fsync
```

Co tento příkaz dělá:

- `xzcat` rozbaluje image za běhu,
- data posílá přes pipe přímo do `dd`,
- `of=/dev/sda` určuje **celou cílovou kartu**, ne její oddíl,
- `bs=4M` zapisuje po větších blocích,
- `status=progress` průběžně ukazuje počet zapsaných bajtů,
- `conv=fsync` zajistí dokončení zápisu dat před ukončením `dd`.

Během zápisu uvidíš průběžně rostoucí počet zapsaných bajtů.

Při našem testu se rozbalený obraz zapsal v objemu přibližně:

```text
2.8 GB
```

Na konci `dd` vypíše souhrn počtu načtených/zapsaných bloků, celkový počet bajtů, čas a rychlost.

Přesná rychlost ani čas nejsou kontrolní hodnoty — závisí na kartě a USB čtečce.

Příkaz se musí vrátit zpět na shell prompt **bez chyby**.

---

# 8. Pro jistotu dokonči zápis

Po návratu promptu spusť:

```bash
sync
```

Počkej, až se prompt znovu objeví.

---

# 9. Nech kernel znovu načíst tabulku oddílů

Můžeš použít:

```bash
sudo partprobe /dev/sda
```

Pokud `partprobe` není nainstalovaný, není to důvod celý zápis opakovat. Pomůže také odpojení a opětovné připojení USB čtečky.

Potom:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS /dev/sda
```

Po úspěšném zápisu musí být na kartě vidět nová tabulka oddílů Raspberry Pi OS, typicky například:

```text
sda
├─sda1 ... vfat
└─sda2 ... ext4
```

Velikost root oddílu v samotném image může být před prvním bootem menší než celá karta. To je normální.

---

# 10. První boot nové karty

Bezpečně dokonči práci:

```bash
sync
```

Potom USB čtečku odpoj a novou microSD vlož do Raspberry Pi, které z ní má bootovat.

Při našem testu se root filesystem při prvním bootu automaticky rozšířil a na 16GB kartě měl následně přibližně:

```text
14.3G
```

Po nabootování zkontroluj:

```bash
lsblk
df -h /
cat /etc/os-release
uname -m
```

Pro testovaný 32bit Trixie systém jsme očekávali architekturu:

```text
armv7l
```

a root filesystem na microSD.

---

# 11. Pozor na headless konfiguraci a cloud-init

Při našem testu jsme se pokusili připravit hostname, uživatele, Wi-Fi a SSH offline pomocí cloud-init.

Na obrazu `2026-09-15-raspios-trixie-armhf-lite` se však ukázalo, že tato cesta nebyla v našem konkrétním testu spolehlivá:

- SSH služba nebyla povolena,
- účet `pi` zůstal v `/etc/shadow` zamčený,
- nevznikl očekávaný NetworkManager Wi-Fi profil.

Systém jsme dokázali offline opravit, ale pro obecný návod tuto metodu **nedoporučujeme jako jediný recovery postup**.

Pokud potřebuješ plně headless první boot s přednastavenou Wi-Fi a SSH, je vhodné použít oficiálně podporovanou konfiguraci Raspberry Pi Imageru nebo předem otestovaný provisioning pro konkrétní verzi Raspberry Pi OS.

Tento repozitář se soustředí především na bezpečné stažení, ověření a zápis OS z jednoho Raspberry Pi na druhou kartu.

---

# 12. Nejčastější chyby

### `No space left on device` při stahování do `/tmp`

Zkontroluj:

```bash
df -h /tmp
```

Na našem Raspberry byl `/tmp` tmpfs pouze přibližně 454 MB, zatímco image měl asi 532 MB.

Řešení:

```bash
mkdir -p /home/pi/os-images
```

a image ukládej tam.

### `xz -t` skončí chybou

Soubor může být neúplný nebo poškozený. Nezapisuj ho na SD. Stáhni jej znovu a opakuj test.

### Po zápisu stále vidím staré oddíly

Spusť:

```bash
sync
sudo partprobe /dev/sda
lsblk /dev/sda
```

nebo USB čtečku fyzicky odpoj a znovu připoj.

### Nová karta po prvním bootu využívá celou kapacitu

To je očekávané. Raspberry Pi OS může root filesystem při prvním startu rozšířit.

---

# Rychlý kontrolní postup

Tuto zkrácenou variantu používej pouze tehdy, když už přesně rozumíš tomu, co jednotlivé příkazy dělají:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL

mkdir -p /home/pi/os-images
cd /home/pi/os-images

wget https://downloads.raspberrypi.com/raspios_lite_armhf/images/raspios_lite_armhf-2026-09-15/2026-09-15-raspios-trixie-armhf-lite.img.xz

xz -t 2026-09-15-raspios-trixie-armhf-lite.img.xz
echo $?

lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL

sudo umount /dev/sda1 2>/dev/null || true
sudo umount /dev/sda2 2>/dev/null || true

xzcat 2026-09-15-raspios-trixie-armhf-lite.img.xz | sudo dd of=/dev/sda bs=4M status=progress conv=fsync

sync
sudo partprobe /dev/sda
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS /dev/sda
```

**Znovu: `/dev/sda` je pouze zařízení z našeho ověřeného testu. Vždy nejprve zjisti vlastní cílové zařízení pomocí `lsblk`.**

---

# Ověřený výsledek

Postup byl 25. 9. 2026 prakticky použit pro vytvoření čisté testovací karty Raspberry Pi OS Lite 32-bit Trixie.

Výsledkem byla bootovatelná microSD, na které následně úspěšně proběhla kompletní obnova samostatného projektu: systemd služby, Python/Kivy, KMS displej, 1-Wire, DS18B20, Flask web a historie.

Tím bylo ověřeno, že samotná metoda:

```text
funkční Raspberry Pi
→ stažení .img.xz
→ xz -t
→ xzcat
→ dd na USB microSD
→ první boot
```

funguje bez nutnosti použít druhý počítač.

## Licence

Tento repozitář je zatím veden jako praktická technická dokumentace. Před případným širším použitím lze doplnit samostatnou licenci.
