# K1 · Labor: esimene playbook

**Klassis:** Osad 1–10. **Kodus:** Kodutöö.
**Töövahend:** control node = sinu masin (WSL2, VM või Linux) Ansible'i ja Gitiga. Sihtmasinad: Osad 1–7 `localhost`, Osad 8–10 klastri VM-id, mille aadressid annab juhendaja.

Iga osa algab lühikese selgitusega, mida ja miks teed. Loe, tee, vaata tulemust.

---

## 🎯 Õpiväljundid

1. Selgitad, miks käsitsi seadistus triivib.
2. Näitad, miks toores käsk pole idempotentne.
3. Kirjutad playbooki, mis viib masina soovitud olekusse.
4. Tõestad idempotentsust `changed`/`ok` väljundist.
5. Tuvastad ja parandad drift'i.
6. Seadistad võtmepõhise SSH-ligipääsu ja rakendad sama playbooki mitmele masinale.
7. Kasutad fakte, et sama playbook töötaks eri distributsioonidel.

---

## Eeltöö

Kontrolli tööriistu:

```bash
git --version && ansible --version | head -1
systemctl is-system-running
```

Kui Ansible puudub: `sudo apt update && sudo apt install -y ansible`. WSL-is peab `systemctl` vastama (`running` või `degraded`); kui ei vasta, ütle juhendajale.

Ava Classroom 50 link, mille juhendaja jagas, ja nõustu ülesandega. Sulle tekib oma repo. Klooni see:

```bash
git clone https://github.com/hkhk-automation/<sinu-repo>.git
cd <sinu-repo>
```

Kõik failid lähevad selle repo juurkausta.

---

## Osa 1 · Käsitsi töö

Enne kui automatiseerid, tee asi üks kord käsitsi. Nii tead täpselt, mida hiljem masinale usaldad. Seadista `localhost` veebiserveriks ja täida kõrvale kontrolltabel kahe veeruga: **käsk** ja **tulemus, mis pidi tekkima**. Playbook, mille hiljem kirjutad, teeb täpselt need read.

```bash
sudo useradd -m saidi
sudo apt install -y nginx
echo "<h1>Tere käsitsi</h1>" | sudo tee /var/www/html/index.html
sudo systemctl start nginx
sudo systemctl enable nginx
```

**Kontroll:** `curl -s localhost` vastab `<h1>Tere käsitsi</h1>`. Kontrolltabelis on 5 rida. Kui `curl` ei vasta, ütleb `systemctl status nginx`, kas teenus seisab või on paigaldamata.

💭 Kui peaksid sama tegema kümnele masinale, mitmendal ununeks esimene samm?

---

## Osa 2 · Halb automaatika

Skript võib olla kiire ja ikka vale, kui seda ei saa ohutult korrata. Kirjutad tahtlikult halva skripti ja jooksutad seda kaks korda. Pane tähele kolme asja: `useradd` eeldab, et kasutajat pole; `mkdir` eeldab, et kausta pole; `echo >>` lisab rea, kontrollimata, kas see on juba olemas.

Loo `halb.sh`:

```bash
#!/usr/bin/env bash
useradd -m raporteerija
mkdir /srv/raport
echo "seade=1" >> /srv/raport/conf
```

Jooksuta kaks korda:

```bash
sudo bash halb.sh; sudo bash halb.sh
cat /srv/raport/conf
```

**Kontroll:** teine jooks annab `useradd`-ilt "already exists" ja `mkdir`-ilt "File exists". Failis `conf` on rida `seade=1` kaks korda. Skript andis vigadest teada, aga duplikaadist mitte, ja just see on ohtlik.

**Paranda skript** nii, et teine jooks ei annaks vigu ega duplikaati. Vihje: `id raporteerija`, `mkdir -p`, `grep -qx`. Salvesta see `parem.sh`-na ja jooksuta kaks korda. Kui palju ridu tuli juurde? Seda tööd teeb Ansible'i moodul sinu eest.

💭 Milline kolmest käitumisest teeks tootmises kõige suuremat kahju?

---

## Osa 3 · Inventar ja ad-hoc käsud

Iga Ansible-töö algab **inventarist**: nimekirjast masinatest, mida haldad. Praegu on seal üks masin, sinu enda oma. Ad-hoc käsk jookseb ühe korra ja näitab olulist vahet: **moodul** (`ping`, `setup`) mõistab olekut ja tagastab struktureeritud infot, **toores käsk** (`command`) ainult käivitab midagi.

Loo `inventory.ini`:

```ini
[kohalik]
localhost ansible_connection=local
```

```bash
ansible -i inventory.ini kohalik -m ping
ansible -i inventory.ini kohalik -m setup -a "filter=ansible_distribution*"
ansible -i inventory.ini kohalik -m setup -a "filter=ansible_os_family"
ansible -i inventory.ini kohalik -m command -a "uptime"
```

**Kontroll:** `ping` vastab `pong`. `setup` tagastab fakte, näiteks `ansible_distribution: Ubuntu` ja `ansible_os_family: Debian`. `command` tagastab ainult teksti. `ansible_os_family` läheb vaja Osa 10-s.

💭 Kui tahad playbookis öelda "kui masin on Debian-pere, tee X", kumb annab selleks info?

---

## Osa 4 · Esimene playbook

Nüüd paned Osa 1 käsitsitöö kirja **soovitud olekuna**: kasutaja on olemas, pakk paigaldatud, avaleht paigas, teenus käib. Ehita **üks task korraga** ja jooksuta iga lisanduse järel, siis tead alati, milline task vea tekitas.

Loo `bootstrap.yml`:

```yaml
- name: Bootstrap veebiserver
  hosts: kohalik
  become: true
  tasks:
    - name: Kasutaja saidi on olemas
      ansible.builtin.user:
        name: saidi
```

`become: true` annab sudo-õigused. Lisa ise ülejäänud task'id, parameetrid leiad `ansible-doc <moodul>` abil:

1. pakett `nginx` paigaldatud (`package`, `state: present`);
2. fail `/var/www/html/index.html` sisuga `<h1>Hallatud Ansible'iga</h1>` (`copy`, `content:`);
3. teenus `nginx` käib ja käivitub buutimisel (`service`, `state: started`, `enabled: true`).

**Enne esimest jooksu ennusta:** Osa 1 käsitsitöö on masinas juba olemas. Mitu `changed`-i tuleb ja millistel task'idel? Kirjuta vastus üles.

```bash
ansible-playbook -i inventory.ini bootstrap.yml
curl -s localhost
```

**Kontroll:** 4 task'i, iga task on päris moodul. `curl` näitab uut lehte. Võrdle `PLAY RECAP`-i oma ennustusega. Muutuma pidi ainult avaleht, sest ainult selle sisu erines käsitsi tehtust.

💡 `Permission denied` tähendab, et `become: true` on puudu. `apt` "Could not get lock" tähendab, et taustal käib teine apt: oota ja korda.

---

## Osa 5 · Teine jooks

Korralik deklaratiivne kirjeldus ei tee teisel jooksul midagi, sest masin on juba soovitud olekus. Kui mõni task näitab igal jooksul `changed`, siis ta teeb tegevust, mitte ei kirjelda olekut. Tavaliselt on põhjus `command`/`shell` seal, kus oleks pidanud olema päris moodul.

Jooksuta kohe uuesti ja salvesta väljund:

```bash
ansible-playbook -i inventory.ini bootstrap.yml | tee logid/teine_jooks.txt
```

**Kontroll:** `PLAY RECAP` näitab `changed=0`.

**Katse:** lisa playbooki task `ansible.builtin.command: echo tere` ja jooksuta kaks korda. Mida näitab `PLAY RECAP` teisel korral? Eemalda task ja jooksuta uuesti, kuni on jälle `changed=0`.

💭 Osa 2 skript andis teisel jooksul vea ja duplikaadi. Miks `bootstrap.yml` seda ei tee, kuigi teeb sama tööd?

---

## Osa 6 · Dry run

Enne muutust tasub vaadata, mida Ansible teeks, ilma et ta midagi muudaks. Muuda `bootstrap.yml`-is avalehe tekst ja jooksuta:

```bash
ansible-playbook -i inventory.ini bootstrap.yml --check --diff
curl -s localhost
```

**Kontroll:** väljundis on `---`/`+++` diff vana ja uue sisu vahel, `changed=1`, aga `curl` näitab ikka vana lehte. Alles päris jooks (ilma `--check`) muudab lehe.

---

## Osa 7 · Drift

Automaatika päris väärtus on kõrvalekalde parandamine. Tekita kolm kõrvalekallet:

```bash
sudo rm /var/www/html/index.html
sudo systemctl stop nginx
sudo userdel saidi
```

**Ennusta enne jooksu:** mitu `changed`-i tuleb ja millistel task'idel? Siis jooksuta `bootstrap.yml`.

**Kontroll:** täpselt 3 `changed`-i, `nginx` pakett jäi `ok`. `curl -s localhost` vastab uuesti.

💭 Kust Ansible teadis, mida taastada, kui sa talle ei öelnud, mis katki oli?

---

## Osa 8 · SSH-võtmed sihtmasinatele

Siiani oli sihtmasin sinu enda arvuti. Päris töös haldad masinaid üle SSH ja ilma paroolita, et Ansible saaks neid automaatselt kasutada. **Privaatvõti** jääb sinu masinasse, sihtmasinasse läheb ainult **avalik võti**.

Juhendaja annab sulle sihtmasinate aadressid, kasutajanime ja esialgse parooli.

```bash
ssh-keygen -t ed25519 -C "<eesnimi>@kursus"
ssh-copy-id -i ~/.ssh/id_ed25519.pub <kasutaja>@<vm1>
ssh <kasutaja>@<vm1> hostname
```

Korda `ssh-copy-id` iga sihtmasina kohta. Lisa mugavuseks `~/.ssh/config`:

```
Host vm1
    HostName <vm1-ip>
    User <kasutaja>
    IdentityFile ~/.ssh/id_ed25519
```

**Kontroll:** `ssh vm1 hostname` vastab ilma parooli küsimata, iga masina kohta.

💡 `Permission denied (publickey)`: võti pole sihtmasinas. Kontrolli `ssh-copy-id` väljundit ja `ssh -v vm1`.

---

## Osa 9 · Inventar mitme masinaga

Lisa `inventory.ini`-sse uus grupp:

```ini
[kohalik]
localhost ansible_connection=local

[veeb]
vm1
vm2
vm3
```

**Ennusta enne:** kui palju `pong`-e tuleb järgmisest kolmest käsust?

```bash
ansible -i inventory.ini veeb -m ping
ansible -i inventory.ini all -m ping
ansible -i inventory.ini veeb -m ping --limit vm2
```

Seejärel vaata, mis OS igal masinal on:

```bash
ansible -i inventory.ini veeb -m setup -a "filter=ansible_os_family"
```

**Kontroll:** vastused klapivad ennustusega. Tead iga sihtmasina OS-i perekonda.

---

## Osa 10 · Sama playbook kolmele masinale

Muuda `bootstrap.yml`-is rida `hosts: kohalik` kujule `hosts: veeb`.

Kaks probleemi, mis nüüd välja tulevad:

- **Veebi juurkaust erineb:** Debiani peres on see `/var/www/html`, RedHati peres (AlmaLinux, Rocky) `/usr/share/nginx/html`.
- **Leht peaks ütlema, mis masin see on**, et näeksid, kust vastus tuli.

Lisa playbooki algusesse muutuja, mis valib kausta fakti järgi:

```yaml
  vars:
    veebi_juur: "{{ '/var/www/html' if ansible_os_family == 'Debian' else '/usr/share/nginx/html' }}"
```

Muuda `copy`-task'i:

```yaml
        dest: "{{ veebi_juur }}/index.html"
        content: "<h1>{{ inventory_hostname }} - hallatud Ansible'iga</h1>\n"
```

Käivita esmalt **blast radius'e** kontrolliga ühel masinal, siis kõigil:

```bash
ansible-playbook -i inventory.ini bootstrap.yml --limit vm1
ansible-playbook -i inventory.ini bootstrap.yml
ansible-playbook -i inventory.ini bootstrap.yml | tee logid/kolm_masinat.txt
```

**Kontroll:** `curl -s http://<vm1-ip>` näitab `vm1`, `vm2` näitab `vm2` jne. `logid/kolm_masinat.txt` `PLAY RECAP`-is on kolm rida, kõigil `changed=0`.

Kui aega jääb: tekita ühes masinas drift (`ssh vm2 "sudo systemctl stop nginx"`) ja jooksuta playbook. Mitu `changed`-i tuleb ja millisel masinal?

💭 Mitu rida pidid muutma, et üks masin asenduks kolmega? Mitu oleks 50 masina puhul?

---

## Osa 11 · Git

Playbook on kood, seega käib see versioonihaldusesse. README ütleb järgmisele lugejale, ka sulle endale kolme kuu pärast, milline on masinate soovitud olek ja kuidas seda rakendada.

Kirjuta `README.md`: masinate soovitud olek, käivituskäsk ja lühidalt, mis Osa 7-s triivis ja mis taastati.

```bash
git add . && git commit -m "K1: bootstrap playbook, idempotentne, 3 masinat"
git push
```

**Kontroll:** GitHubis on repos `halb.sh`, `parem.sh`, `inventory.ini`, `bootstrap.yml`, `logid/teine_jooks.txt`, `logid/kolm_masinat.txt`, `README.md`. Actions vahelehel on klassi kontrollid rohelised.

💡 `git push` küsib parooli: GitHub ei võta enam kontoparooli. Loo token (GitHub → Settings → Developer settings → Personal access tokens) või lisa oma SSH-võti GitHubi ja vaheta remote: `git remote set-url origin git@github.com:hkhk-automation/<sinu-repo>.git`.

---

## Kodutöö (~8 h)

Samasse reposse.

**1. `admin.yml`.** Playbook grupile `veeb`, mis viib masinad olekusse:

- kasutaja `deploy` on olemas ja kuulub sudo-gruppi (Debianis `sudo`, RedHatis `wheel`: vali fakti järgi);
- `chrony` on paigaldatud ja käib;
- `/etc/motd` sisaldab teksti "Hallatud Ansible'iga - <masina nimi>".

`command`/`shell` pole lubatud seal, kus on olemas päris moodul. Teine jooks peab olema `changed=0`. Salvesta see väljund faili `logid/admin_teine_jooks.txt`.

**2. Oma töö.** Vali oma tööst üks korduv käsitsi tegevus: kasutajate loomine, paketid, konfifail, logide kaust vms. Kirjuta sellele idempotentne playbook kausta `oma/`. Kirjuta `oma/README.md`-sse, mis oli enne käsitsi ja mis on nüüd kood.

**3. Teooria.** Loe teooria §2, §5 ja §6 ning vasta kolmele kordamisküsimusele (§2, §4 ja Osa 7 "Mõtle") failis `vastused.md`.

---

## ✅ Lõpukontroll

- [ ] Kontrolltabel täidetud; `halb.sh` duplikaat nähtud; `parem.sh` töötab kaks korda järjest.
- [ ] `bootstrap.yml`: ennustus kirjas, teine jooks `changed=0`, `command`-katse tehtud ja eemaldatud.
- [ ] `--check --diff` näitas muutust ilma seda tegemata.
- [ ] Drift: 3 `changed`-i, ennustus klappis.
- [ ] SSH ilma paroolita kõigisse sihtmasinatesse.
- [ ] Kolm masinat: iga leht näitab oma nime, `logid/kolm_masinat.txt` `changed=0`.
- [ ] Kodutöö: `admin.yml` teine jooks `changed=0`; `oma/` playbook ja README; `vastused.md`.

---

## Veaotsing

| Probleem | Kontroll |
|---|---|
| `ping` localhostile ei vasta | failis `ansible_connection=local`; käsus `-i inventory.ini` |
| `ping` VM-ile: `UNREACHABLE` | `ssh vm1 hostname` töötab? Nimi sama nagu `~/.ssh/config`-is? |
| `Permission denied (publickey)` | võti pole sihtmasinas: korda `ssh-copy-id` |
| `Missing sudo password` | sihtmasinas pole paroolita sudo; lisa käsule `-K` |
| `apt` "Could not get lock" | oota, korda |
| `Permission denied` playbookis | `become: true` puudu |
| task on igal jooksul `changed` | vale moodul (`command`); kasuta `package`/`service`/`user`/`copy` |
| `curl` näitab vaikelehte | `copy` kirjutas vale kausta; kontrolli `veebi_juur` väärtust |
| `systemctl` ei tööta WSL-is | systemd pole WSL-is sisse lülitatud; ütle juhendajale |

---

## 📚 Allikad

| Allikas | URL |
|---|---|
| Ansible: inventory ja esimene playbook | <https://docs.ansible.com/ansible/latest/getting_started/> |
| Ansible builtin moodulid | <https://docs.ansible.com/ansible/latest/collections/ansible/builtin/> |
| Ansible faktid | <https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html> |
| Pro Git (eesti k) | <https://git-scm.com/book/et/v2> |
