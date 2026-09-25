# K1 · Praktikum: esimene playbook

## Eesmärk

Tänase lõpuks on sul playbook, mis viib kolm serverit samasse olekusse: kasutaja olemas, nginx paigaldatud ja käimas, avaleht näitab serveri nime. Teine jooks ei muuda midagi (`changed=0`), ja see on tõend, et kirjeldus on idempotentne.

Praktikumil on kaks osa:

- **A · Juhendatud:** kõik ühel masinal (`localhost`), samm-sammult.
- **B · Iseseisev:** sama oskus kolmel VM-il. Antud on eesmärk ja piirangud, lahenduse leiad ise.

Iga samm on kujul **Tegevus → Oodatav tulemus → Miks → Tõend**.

---

## 0 · Valmisolek

```bash
git --version && ansible --version | head -1
systemctl is-system-running
```

- **Oodatav tulemus:** mõlemad versioonid on näha; `systemctl` vastab `running` või `degraded`.
- Ansible puudub: `sudo apt update && sudo apt install -y ansible`. `systemctl` ei vasta (WSL): ütle juhendajale.

Ava Classroom 50 link ja nõustu ülesandega. Klooni oma repo:

```bash
git clone https://github.com/hkhk-automation/<sinu-repo>.git
cd <sinu-repo>
```

Kõik failid lähevad repo juurkausta.

---

## A · Juhendatud osa

### A1 · Käsitsi seadistus

**Tegevus:** seadista `localhost` käsitsi veebiserveriks. Kirjuta kõrvale kontrolltabel: käsk | tulemus, mis pidi tekkima.

```bash
sudo useradd -m saidi
sudo apt install -y nginx
echo "<h1>Tere käsitsi</h1>" | sudo tee /var/www/html/index.html
sudo systemctl enable --now nginx
```

**Oodatav tulemus:** `curl -s localhost` vastab `<h1>Tere käsitsi</h1>`.
**Miks:** enne automatiseerimist pead teadma, mida masin peab tegema. Kontrolltabeli read muutuvad hiljem playbooki task'ideks.
**Tõend:** kontrolltabel (4 rida) vihikus.

### A2 · Halb skript ja parem skript

**Tegevus:** loo `halb.sh` ja jooksuta seda kaks korda.

```bash
#!/usr/bin/env bash
useradd -m raporteerija
mkdir /srv/raport
echo "seade=1" >> /srv/raport/conf
```

```bash
sudo bash halb.sh; sudo bash halb.sh; cat /srv/raport/conf
```

**Oodatav tulemus:** teisel jooksul vead "already exists" ja "File exists"; failis `conf` on `seade=1` kaks korda.
**Miks:** skript andis vigadest teada, aga duplikaadist mitte. Vaikne duplikaat on tootmises kõige ohtlikum.

**Tegevus:** kirjuta `parem.sh`, mis ei anna teisel jooksul vigu ega duplikaati (vihje: `id`, `mkdir -p`, `grep -qx`).
**Oodatav tulemus:** kaks jooksu järjest ilma vigadeta, `conf`-is üks rida.
**Miks:** loe kokku, mitu rida kontrolli pidid lisama. Seda tööd teeb Ansible'i moodul sinu eest.
**Tõend:** `halb.sh` ja `parem.sh` repos.

### A3 · Inventar ja ad-hoc käsud

**Tegevus:** loo `inventory.ini`:

```ini
[kohalik]
localhost ansible_connection=local
```

```bash
ansible -i inventory.ini kohalik -m ping
ansible -i inventory.ini kohalik -m setup -a "filter=ansible_os_family"
ansible -i inventory.ini kohalik -m command -a "uptime"
```

**Oodatav tulemus:** `pong`; `ansible_os_family: Debian`; `uptime` väljund tekstina.
**Miks:** moodul (`ping`, `setup`) tagastab struktureeritud info, mida playbook saab kasutada. `command` tagastab ainult teksti. `ansible_os_family` läheb vaja osas B.
**Tõend:** `inventory.ini` repos.

### A4 · Esimene playbook

**Tegevus:** loo `bootstrap.yml`. Alusta ühest task'ist ja jooksuta iga lisanduse järel.

```yaml
- name: Bootstrap veebiserver
  hosts: kohalik
  become: true
  tasks:
    - name: Kasutaja saidi on olemas
      ansible.builtin.user:
        name: saidi
```

Lisa ise (`ansible-doc <moodul>`):

1. `package`: `nginx`, `state: present`
2. `copy`: `/var/www/html/index.html`, `content: "<h1>Hallatud Ansible'iga</h1>\n"`
3. `service`: `nginx`, `state: started`, `enabled: true`

Enne esimest jooksu **kirjuta üles ennustus**: mitu `changed`-i tuleb ja millistel task'idel? A1 käsitsitöö on masinas juba olemas.

```bash
ansible-playbook -i inventory.ini bootstrap.yml
curl -s localhost
```

**Oodatav tulemus:** `changed=1` (ainult avaleht), `curl` näitab uut lehte.
**Miks:** moodul võrdleb soovitud olekut praegusega ja muudab ainult erinevuse.
**Tõend:** ennustus vihikus, `bootstrap.yml` repos.

### A5 · Teine jooks

**Tegevus:**

```bash
ansible-playbook -i inventory.ini bootstrap.yml | tee logid/teine_jooks.txt
```

**Oodatav tulemus:** `changed=0`.
**Miks:** see on idempotentsuse tõend: masin on juba soovitud olekus.

**Tegevus:** lisa ajutiselt task `ansible.builtin.command: echo tere` ja jooksuta kaks korda. Eemalda see siis.
**Oodatav tulemus:** `command` on igal jooksul `changed`.
**Miks:** toores käsk ei tea olekut, seega pole ta idempotentne.
**Tõend:** `logid/teine_jooks.txt` (ilma `command`-task'ita).

### A6 · Dry run

**Tegevus:** muuda playbookis avalehe teksti ja jooksuta:

```bash
ansible-playbook -i inventory.ini bootstrap.yml --check --diff
curl -s localhost
```

**Oodatav tulemus:** diff näitab vana ja uut sisu, `changed=1`, aga `curl` näitab ikka vana lehte.
**Miks:** enne muutust tootmises vaatad, mida see teeks. Päris jooks (ilma `--check`) muudab lehe.

### A7 · Drift

**Tegevus:** tekita kolm kõrvalekallet. **Ennusta**, mitu `changed`-i tuleb, siis jooksuta playbook.

```bash
sudo rm /var/www/html/index.html
sudo systemctl stop nginx
sudo userdel saidi
ansible-playbook -i inventory.ini bootstrap.yml
```

**Oodatav tulemus:** 3 `changed`-i; `nginx` pakett jääb `ok`; `curl` vastab uuesti.
**Miks:** playbook parandab ainult selle, mis triivis, ilma et ütleksid, mis katki on.
**Tõend:** vihikus: mis triivis, mis taastati, kas ennustus klappis.

---

## ☕ Paus

Pärast pausi jätka osaga B. Kui A on pooleli, lõpeta enne A5, sest B ehitab `bootstrap.yml` peale.

---

## B · Iseseisev osa: kolm serverit

### Eesmärk

Juhendaja annab sulle kolme VM-i aadressid, kasutajanime ja esialgse parooli. Vii kõik kolm samasse olekusse sama `bootstrap.yml`-iga. Iga server näitab avalehel oma nime.

### Piirangud

- Ansible ühendub SSH-võtmega. Parool ei tohi olla üheski failis.
- Üks playbook kõigile kolmele. OS-ist sõltuvad väärtused (veebi juurkaust) valitakse **fakti järgi**, mitte käsitsi hosti kaupa.
- Enne kõiki masinaid proovi ühel (`--limit`).
- `command`/`shell` pole lubatud seal, kus on olemas moodul.

### Valmis, kui

- [ ] `ssh <vm> hostname` töötab iga masina kohta ilma paroolita.
- [ ] `ansible -i inventory.ini veeb -m ping` annab kolm `pong`-i.
- [ ] `curl http://<vm-ip>` näitab iga masina puhul selle nime.
- [ ] Teine jooks kõigil kolmel: `changed=0`, salvestatud faili `logid/kolm_masinat.txt`.
- [ ] Drift ühes masinas (nt peatatud nginx) parandub ühe jooksuga ja teisi ei puudutata.

### Vihjed

Ava ainult siis, kui jääd kinni.

??? tip "SSH-võti"
    `ssh-keygen -t ed25519`, siis `ssh-copy-id <kasutaja>@<ip>` iga masina kohta. Privaatvõti jääb sinu masinasse, sihtmasinasse läheb ainult `.pub`. `~/.ssh/config`-is saad anda masinatele lühinimed (`Host vm1`, `HostName`, `User`).

??? tip "Inventar"
    Lisa `inventory.ini`-sse grupp `[veeb]` kolme masinaga ja muuda playbookis `hosts:`.

??? tip "Veebi juurkaust erineb"
    Debiani peres `/var/www/html`, RedHati peres `/usr/share/nginx/html`. Kontrolli `ansible veeb -m setup -a "filter=ansible_os_family"` ja kasuta playbookis muutujat, mille väärtus sõltub `ansible_os_family`-st (Jinja2 `if … else`).

??? tip "Masina nimi lehel"
    `copy` `content:` võib sisaldada muutujat: `{{ inventory_hostname }}`.

??? tip "Missing sudo password"
    Sihtmasinas pole paroolita sudo. Lisa käsule `-K`.

---

## Dokumenteerimine

**Tegevus:** kirjuta `README.md`:

- masinate soovitud olek (mis peab igas masinas olema);
- käivituskäsk;
- mis A7-s triivis ja mis taastati;
- **peegeldus**, 2–3 lauset iga küsimuse kohta: (1) Mitu rida pidid muutma, et üks masin asenduks kolmega? Mitu oleks 50 puhul? (2) Mis ennustus läks mööda ja miks? (3) Mis sinu töös praegu triivib?

```bash
git add . && git commit -m "K1: bootstrap, idempotentne, 3 masinat"
git push
```

**Oodatav tulemus:** Actions vahelehel on kontrollid 1–5 rohelised.
**Tõend:** repos on `halb.sh`, `parem.sh`, `inventory.ini`, `bootstrap.yml`, `logid/teine_jooks.txt`, `logid/kolm_masinat.txt`, `README.md`.

💡 `git push` küsib parooli: GitHub kontoparooli ei võta. Kasuta Personal Access Tokenit või lisa SSH-võti GitHubi ja vaheta remote: `git remote set-url origin git@github.com:hkhk-automation/<sinu-repo>.git`.

---

## Kodutöö

Samasse reposse. Tähtaeg on Classroom 50-s. Kodutöö on klassitööst raskem: juhiseid on vähem, parameetrid otsid ise `ansible-doc`-ist ja dokumentatsioonist.

**1. `admin.yml`** grupile `veeb`:

- kasutajad `deploy` ja `monitor` luuakse ühe task'iga, mis käib läbi nimekirja (`loop`);
- mõlemale lisatakse sinu avalik SSH-võti (`ansible.posix.authorized_key`), nii et saad nendena sisse logida;
- `deploy` kuulub sudo-gruppi (Debianis `sudo`, RedHatis `wheel`, vali fakti järgi);
- `chrony` on paigaldatud ja käib;
- `/etc/motd` sisaldab "Hallatud Ansible'iga - <masina nimi>".

Teine jooks `changed=0`, salvesta `logid/admin_teine_jooks.txt`.

**2. `hardening.yml`**: SSH turvamine grupile `veeb`.

- `/etc/ssh/sshd_config`-is: `PermitRootLogin no` ja `PasswordAuthentication no` (`lineinfile`);
- enne muudatuse rakendamist kontrollitakse konfi süntaksit (`validate: sshd -t -f %s`);
- `sshd` taaskäivitatakse ainult siis, kui konf muutus (uuri `notify` ja `handlers`).

**Ohutus:** see võib sind masinast välja lukustada. Hoia teine SSH-sessioon lahti, proovi esmalt `--check --diff` ja `--limit vm1`, alles siis kõigil. Kui lukustasid end välja, kirjuta README-sse, mis juhtus ja kuidas said tagasi.

Teine jooks `changed=0`, salvesta `logid/hardening_teine_jooks.txt`.

**3. Oma töö.** Vali oma tööst üks korduv käsitsi tegevus ja kirjuta sellele idempotentne playbook kausta `oma/`. `oma/README.md`: mis oli enne käsitsi, mis on nüüd kood, kuidas tõestasid, et teine jooks ei muuda midagi.

**4. Teooria.** Loe loengu §2 ja §6. Vasta §2 ja §4 kordamisküsimustele failis `vastused.md`.

**Boonus:** kirjuta `boonus.yml`, mis üritab paigaldada paketti, mida pole olemas, ja püüab vea kinni `block`/`rescue`-ga nii, et playbook kirjutab veast teate ega kuku. Selgita `vastused.md`-s, millal see on mõistlik ja millal ohtlik.

---

## Veaotsing

| Probleem | Kontroll |
|---|---|
| `ping` localhostile ei vasta | failis `ansible_connection=local`; käsus `-i inventory.ini` |
| VM: `UNREACHABLE` | kas `ssh <vm> hostname` töötab? kas nimi on sama mis `~/.ssh/config`-is? |
| `Permission denied (publickey)` | võti pole sihtmasinas: korda `ssh-copy-id` |
| `Missing sudo password` | lisa `-K` |
| `apt` "Could not get lock" | oota, korda |
| `Permission denied` playbookis | `become: true` puudu |
| task on igal jooksul `changed` | `command`/`shell` mooduli asemel |
| `curl` näitab vaikelehte | leht läks vale kausta; kontrolli juurkausta muutujat |

---

## Allikad

| Allikas | URL |
|---|---|
| Ansible: getting started | <https://docs.ansible.com/ansible/latest/getting_started/> |
| Ansible builtin moodulid | <https://docs.ansible.com/ansible/latest/collections/ansible/builtin/> |
| Ansible faktid | <https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html> |
