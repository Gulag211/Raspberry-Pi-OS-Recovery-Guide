# Raspberry Pi OS Recovery Guide

Podrobný český návod, jak **stáhnout a zapsat Raspberry Pi OS na novou microSD kartu přímo z jiného funkčního Raspberry Pi**, bez Raspberry Pi Imageru na PC.

Tento postup byl prakticky proveden při obnově systému dne **25. 9. 2026**. Funkční Raspberry Pi běželo ze své původní microSD karty a druhá 16GB microSD byla vložena přes USB čtečku.

Návod je záměrně psaný **pro úplného začátečníka**. Příkazy jsou uváděny celé a po důležitých krocích následuje kontrola s očekávaným výsledkem. Není potřeba předem znát Linux.

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

# 12. Aktualizace systému a základní nástroje

Po prvním úspěšném bootu a připojení k internetu nejdříve aktualizuj systém:

```bash
sudo apt update
sudo apt full-upgrade -y
sudo apt autoremove -y
```

`apt update` obnoví seznam dostupných balíčků. `full-upgrade` nainstaluje aktualizace včetně změn závislostí a `autoremove` odstraní již nepotřebné automaticky instalované balíčky.

Ověř, zda ještě něco čeká na aktualizaci:

```bash
apt list --upgradable
```

Pokud je systém aktuální, pod hlavičkou `Listing... Done` nebude seznam balíčků čekajících na upgrade.

Potom nainstaluj základní nástroje používané v tomto návodu:

```bash
sudo apt install -y git wget xz-utils ca-certificates
```

Ověř je:

```bash
git --version
wget --version | head -n 1
xz --version | head -n 1
```

U Git musíš dostat například:

```text
git version 2.x.x
```

Konkrétní číslo verze se může lišit. Důležité je, že příkaz neskončí `command not found`.

Po větší systémové aktualizaci je vhodné Raspberry restartovat:

```bash
sudo reboot
```

---

# 13. Vlastní hostname, SSH a Wi-Fi SSID

Pokud připravuješ kartu **bez Raspberry Pi Imageru** a po prvním bootu potřebuješ nastavit vlastní název Raspberry Pi, SSH a Wi-Fi, níže jsou přesné cesty k souborům používaným na testovaném Raspberry Pi OS Trixie.

> [!IMPORTANT]
> Při našem testu se ukázalo, že ručně připravený cloud-init nebyl pro první boot spolehlivý. Následující část proto popisuje především **kde je nastavení po nabootování systému** a jak jej nastavit přímo v běžícím Raspberry Pi.

## Hostname

Aktuální hostname zobrazíš:

```bash
hostname
hostnamectl
```

Hlavní soubor s názvem zařízení je:

```text
/etc/hostname
```

Název se používá také v:

```text
/etc/hosts
```

Nejjednodušší změna hostname je:

```bash
sudo hostnamectl set-hostname MOJE-RASPBERRY
```

Potom zkontroluj:

```bash
cat /etc/hostname
hostnamectl
```

Pokud má `/etc/hosts` řádek se starým hostname, uprav jej:

```bash
sudo nano /etc/hosts
```

Typicky například:

```text
127.0.1.1       MOJE-RASPBERRY
```

Po restartu:

```bash
sudo reboot
```

ověř:

```bash
hostname
```

Výstup musí být nový název zařízení.

## SSH

Systemd služba SSH je:

```text
ssh.service
```

SSH zapneš a současně spustíš:

```bash
sudo systemctl enable --now ssh
```

Kontrola:

```bash
systemctl is-enabled ssh
systemctl is-active ssh
```

Správný výsledek je:

```text
enabled
active
```

Hlavní konfigurace OpenSSH serveru je:

```text
/etc/ssh/sshd_config
```

Doplňkové konfigurační soubory jsou v:

```text
/etc/ssh/sshd_config.d/
```

Při našem recovery testu jsme pro explicitní povolení přihlášení heslem použili například:

```text
/etc/ssh/sshd_config.d/99-kotel-test.conf
```

s obsahem:

```text
PasswordAuthentication yes
```

Po změně SSH konfigurace nejprve ověř syntaxi:

```bash
sudo sshd -t
echo $?
```

Správný výsledek je:

```text
0
```

A potom:

```bash
sudo systemctl restart ssh
```

Stav:

```bash
systemctl status ssh --no-pager
```

musí obsahovat:

```text
Active: active (running)
```

Skutečně použitou hodnotu pro přihlášení heslem ověříš:

```bash
sudo sshd -T | grep -i '^passwordauthentication'
```

Pokud je přihlášení heslem povolené, očekávej:

```text
passwordauthentication yes
```

## Wi-Fi – SSID a heslo

Na testovaném Raspberry Pi OS Trixie spravuje síť **NetworkManager**.

Uložené profily jsou v:

```text
/etc/NetworkManager/system-connections/
```

Nejdříve zobraz dostupné Wi-Fi sítě:

```bash
nmcli device wifi list
```

Nejjednodušší připojení k vlastnímu SSID:

```bash
sudo nmcli device wifi connect "MOJE_SSID" password "MOJE_WIFI_HESLO"
```

Potom ověř:

```bash
nmcli connection show
nmcli device status
ip addr show wlan0
```

Aktivní Wi-Fi profil zjistíš:

```bash
nmcli -t -f NAME,DEVICE connection show --active
```

### Kontrola SSID a skutečně uloženého Wi-Fi hesla

Protože je tento návod určen i začátečníkům, můžeš si při diagnostice nechat zobrazit **skutečně uložené heslo**, abys přesně viděl, zda jsi jej zadal správně.

Nejdříve zjisti jméno profilu:

```bash
nmcli connection show
```

Potom zobraz SSID a heslo:

```bash
sudo nmcli --show-secrets -g 802-11-wireless.ssid,802-11-wireless-security.psk connection show "JMENO_PROFILU"
```

Například:

```bash
sudo nmcli --show-secrets -g 802-11-wireless.ssid,802-11-wireless-security.psk connection show "Internet-Doma"
```

Očekávaný tvar výstupu:

```text
MOJE_SSID
MOJE_WIFI_HESLO
```

První řádek je SSID, druhý skutečně uložené heslo. Celý profil včetně tajných údajů lze zobrazit:

```bash
sudo nmcli --show-secrets connection show "JMENO_PROFILU"
```

> [!WARNING]
> Tento výstup může obsahovat tvoje skutečné Wi-Fi heslo. Pro vlastní kontrolu je to užitečné, ale **nevkládej takový výstup ani screenshot veřejně na GitHub, fórum nebo do issue bez odstranění hesla**.

NetworkManager po úspěšném vytvoření profilu uloží konfiguraci do:

```text
/etc/NetworkManager/system-connections/
```

Soubory v tomto adresáři obsahují citlivé údaje a mají být chráněné oprávněním `600` a vlastníkem `root:root`.

Seznam lze bezpečně zobrazit:

```bash
sudo ls -l /etc/NetworkManager/system-connections/
```

### Změna SSID u existujícího profilu

Nejdříve zjisti jméno profilu:

```bash
nmcli connection show
```

Potom lze změnit SSID a heslo přes NetworkManager:

```bash
sudo nmcli connection modify "JMENO_PROFILU" 802-11-wireless.ssid "NOVE_SSID"
sudo nmcli connection modify "JMENO_PROFILU" wifi-sec.psk "NOVE_WIFI_HESLO"
sudo nmcli connection up "JMENO_PROFILU"
```

Pokud pracuješ vzdáleně přes právě měněnou Wi-Fi, poslední příkaz může okamžitě přerušit SSH spojení. Potom se musíš připojit na novou IP adresu v nové síti.

### Kontrola po restartu

Po nastavení hostname, SSH a Wi-Fi je vhodné provést:

```bash
sudo reboot
```

Po naběhnutí ověř:

```bash
hostname
systemctl is-enabled ssh
systemctl is-active ssh
nmcli -t -f NAME,DEVICE connection show --active
ip -4 addr show wlan0
```

Očekáváme:

- správný vlastní hostname,
- SSH `enabled`,
- SSH `active`,
- aktivní Wi-Fi profil na `wlan0`,
- IPv4 adresu přidělenou Wi-Fi síti.

---

# 14. Připojení k Raspberry Pi z telefonu přes Termius

Pro práci bez počítače lze použít telefon a SSH klient **Termius**. Termius je dostupný pro Android i iPhone/iPad a umožňuje otevřít terminál Raspberry Pi přímo v telefonu.

Oficiální odkazy:

- Android – Termius: https://termius.com/download/android
- Android – Google Play: https://play.google.com/store/apps/details?id=com.server.auditor.ssh.client
- iPhone/iPad – Termius: https://termius.com/download/ios
- iPhone/iPad – App Store: https://apps.apple.com/app/termius-modern-ssh-client/id549039908

## Nejdříve připrav Raspberry Pi

Na Raspberry musí být SSH zapnuté:

```bash
sudo systemctl enable --now ssh
```

Ověř:

```bash
systemctl is-enabled ssh
systemctl is-active ssh
```

Správný výstup:

```text
enabled
active
```

Zjisti IP adresu Raspberry Pi:

```bash
hostname -I
```

Příklad:

```text
192.168.1.123
```

Pokud se zobrazí více adres, Wi-Fi adresu lze zobrazit přes:

```bash
ip -4 addr show wlan0
```

Hledej řádek začínající `inet`, například:

```text
inet 192.168.1.123/24 ...
```

Pro Termius použiješ pouze IP adresu před lomítkem:

```text
192.168.1.123
```

Telefon a Raspberry Pi musí být pro tento jednoduchý lokální postup ve stejné síti – například na stejné domácí Wi-Fi nebo hotspotu.

## Nastavení v Termius

Po instalaci Termius vytvoř nový **Host** a vyplň:

```text
Label:    libovolný název, například Raspberry
Address:  IP adresa Raspberry, například 192.168.1.123
Port:     22
Username: pi
Password: heslo uživatele pi
```

Pokud jsi při instalaci použil jiného uživatele než `pi`, zadej samozřejmě jeho jméno.

Host ulož a otevři.

Při úplně prvním připojení se může zobrazit dotaz na důvěryhodnost SSH host key/fingerprintu. Pokud jde skutečně o tvoje Raspberry na právě zjištěné IP adrese, potvrď připojení.

Potom Termius požádá o přihlášení, pokud jsi heslo neuložil už v nastavení hostu.

## Jak poznám, že jsem opravdu připojený

Po úspěšném přihlášení se zobrazí linuxový terminál. Prompt může vypadat například:

```text
pi@raspberrypi:~ $
```

Ověř identitu Raspberry:

```bash
whoami
hostname
hostname -I
```

Příklad správného výsledku:

```text
pi
raspberrypi
192.168.1.123
```

Od této chvíle můžeš příkazy z tohoto README kopírovat do Termius stejně, jako kdybys seděl u Raspberry s klávesnicí a monitorem.

## Když Termius hlásí, že se nemůže připojit

Na Raspberry nejdříve zkontroluj:

```bash
systemctl is-active ssh
hostname -I
```

SSH musí vrátit:

```text
active
```

Dále zkontroluj v Termius:

```text
Address  = aktuální IP Raspberry
Port     = 22
Username = správný uživatel
Password = správné heslo
```

Pokud telefon používá mobilní data a Raspberry je na domácí Wi-Fi, obyčejná lokální IP typu `192.168.x.x` nebo `10.x.x.x` obvykle nestačí. Pro tento základní návod měj telefon i Raspberry ve stejné lokální síti.

> [!NOTE]
> Tato část je zatím základní ověřený postup. Později ji můžeme rozšířit o přesný postup podle reálného nastavení v mobilní aplikaci Termius, screenshoty a případně připojení přes hotspot nebo Tailscale.

---

# 15. Nejčastější chyby

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
