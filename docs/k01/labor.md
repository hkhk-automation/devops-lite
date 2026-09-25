# K1 · Labor: esimene playbook

**Klassis:** Osad 1–7 (~125 min). **Kodus:** Kodutöö.
**Töövahend:** üks Ubuntu/Debian masin sudo-õigustega (WSL2, VM või Linux). Control node ja sihtmasin on sama masin: `localhost`.

Iga osa algab lühikese selgitusega, mida ja miks teed. Loe, tee, vaata tulemust.

---

## 🎯 Õpiväljundid

1. Selgitad, miks käsitsi seadistus triivib.
2. Näitad, miks toores käsk pole idempotentne.
3. Kirjutad playbooki, mis viib masina soovitud olekusse.
4. Tõestad idempotentsust `changed`/`ok` väljundist.
5. Tuvastad ja parandad drift'i.
6. Paned soovitud oleku Giti.

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
git clone git@github.com:hkhk-automation/<sinu-repo>.git
cd <sinu-repo>
```

Kõik tänased failid lähevad selle repo juurkausta.

---

## Osa 1 · Käsitsi töö

Enne kui automatiseerid, tee asi üks kord käsitsi. Nii tead täpselt, mida hiljem masinale usaldad. Seadista `localhost` veebiserveriks ja täida kõrvale kontrolltabel kahe veeruga: **käsk** ja **tulemus, mis pidi tekkima**. Playbook, mille hiljem kirjutad, teeb täpselt need read.

```bash
sudo useradd -m saidi
sudo mkdir -p /srv/site
echo "<h1>Tere</h1>" | sudo tee /srv/site/index.html
sudo apt install -y nginx
sudo systemctl start nginx
```

**Kontroll:** `curl -s localhost` vastab HTML-iga. Kontrolltabelis on 5 rida. Kui `curl` ei vasta, ütleb `systemctl status nginx`, kas teenus seisab või on paigaldamata.

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

💭 Milline neist kolmest käitumisest teeks tootmises kõige suuremat kahju?

---

## Osa 3 · Inventar ja ad-hoc käsud

Iga Ansible-töö algab **inventarist**: nimekirjast masinatest, mida haldad. Täna on seal üks masin, sinu enda oma. Ad-hoc käsk jookseb ühe korra ja näitab olulist vahet: **moodul** (`ping`, `setup`) mõistab olekut ja tagastab struktureeritud infot, **toores käsk** (`command`) ainult käivitab midagi.

Loo `inventory.ini`:

```ini
[kohalik]
localhost ansible_connection=local
```

```bash
ansible -i inventory.ini kohalik -m ping
ansible -i inventory.ini kohalik -m setup -a "filter=ansible_distribution*"
ansible -i inventory.ini kohalik -m command -a "uptime"
```

**Kontroll:** `ping` vastab `pong`. `setup` tagastab fakte, näiteks `ansible_distribution: Ubuntu`, mida saab playbookis otsuste tegemiseks kasutada. `command` tagastab ainult teksti.

💭 Kui tahad playbookis öelda "kui masin on Ubuntu, tee X", kumb annab selleks info?

---

## Osa 4 · Esimene playbook

Nüüd paned Osa 1 käsitsitöö kirja **soovitud olekuna**: kasutaja on olemas, pakk paigaldatud, kaust ja fail paigas, teenus käib. Ehita **üks task korraga** ja jooksuta iga lisanduse järel, siis tead alati, milline task vea tekitas.

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

`become: true` annab sudo-õigused. Lisa ise ülejäänud neli task'i, parameetrid leiad `ansible-doc <moodul>` abil:

1. pakett `nginx` paigaldatud (`apt`, `state: present`);
2. kaust `/srv/site` (`file`, `state: directory`);
3. fail `/srv/site/index.html` sinu sisuga (`copy`, `content:`);
4. teenus `nginx` käib ja käivitub buutimisel (`service`, `state: started`, `enabled: true`).

**Enne esimest jooksu ennusta:** Osa 1 käsitsitöö on masinas juba olemas. Mitu `changed`-i tuleb ja millistel task'idel? Kirjuta vastus üles.

```bash
ansible-playbook -i inventory.ini bootstrap.yml
curl -s localhost | head -1
```

**Kontroll:** 5 task'i, iga task on päris moodul. `curl` näitab sinu lehte. Võrdle `PLAY RECAP`-i oma ennustusega. Tõenäoliselt muutus ainult avaleht, sest selle sisu erineb Osa 1 omast.

💡 `Permission denied` tähendab, et `become: true` on puudu. `apt` "Could not get lock" tähendab, et taustal käib teine apt: oota ja korda.

---

## Osa 5 · Teine jooks

Korralik deklaratiivne kirjeldus ei tee teisel jooksul midagi, sest masin on juba soovitud olekus. Kui mõni task näitab igal jooksul `changed`, siis ta teeb tegevust, mitte ei kirjelda olekut. Tavaliselt on põhjus `command`/`shell` seal, kus oleks pidanud olema päris moodul.

Jooksuta kohe uuesti ja salvesta väljund:

```bash
mkdir -p logid
ansible-playbook -i inventory.ini bootstrap.yml | tee logid/teine_jooks.txt
```

**Kontroll:** `PLAY RECAP` näitab `changed=0`.

💭 Osa 2 skript andis teisel jooksul vea ja duplikaadi. Miks `bootstrap.yml` seda ei tee, kuigi teeb sama tööd?

---

## Osa 6 · Drift

Automaatika päris väärtus on kõrvalekalde parandamine. Tekita kaks kõrvalekallet:

```bash
sudo rm /srv/site/index.html
sudo systemctl stop nginx
```

**Ennusta enne jooksu:** mitu `changed`-i tuleb? Siis jooksuta `bootstrap.yml`.

**Kontroll:** täpselt 2 `changed`-i (avaleht ja teenus), ülejäänud `ok`. `curl -s localhost` vastab uuesti.

💭 Kust Ansible teadis, mida taastada, kui sa talle ei öelnud, mis katki oli?

---

## Osa 7 · Git

Playbook on kood, seega käib see versioonihaldusesse. README ütleb järgmisele lugejale, ka sulle endale kolme kuu pärast, milline on masina soovitud olek ja kuidas seda rakendada.

Kirjuta `README.md`, kus on kaks asja: masina soovitud olek (mis peab masinas olema) ja käivituskäsk. Lisa sinna ka lühidalt, mis Osa 6-s triivis ja mis taastati.

```bash
printf "*.retry\n__pycache__/\n" > .gitignore
git add . && git commit -m "K1: bootstrap playbook, idempotentne"
git push
```

**Kontroll:** GitHubis on repos `inventory.ini`, `bootstrap.yml`, `halb.sh`, `logid/teine_jooks.txt`, `README.md`.

💡 Kui `git push` küsib parooli, on remote HTTPS-il. Vaheta SSH-le: `git remote set-url origin git@github.com:hkhk-automation/<sinu-repo>.git`.

---

## Kodutöö (~8 h)

Samasse reposse.

**1. `admin.yml`.** Sama inventariga playbook, mis viib masina olekusse:

- kasutaja `deploy` on olemas;
- `chrony` on paigaldatud ja käib;
- `/etc/motd` sisaldab teksti "Hallatud Ansibleiga".

`command`/`shell` pole lubatud seal, kus on olemas päris moodul. Teine jooks peab olema `changed=0`. Salvesta see väljund faili `logid/admin_teine_jooks.txt`.

**2. Oma töö.** Vali oma tööst üks korduv käsitsi tegevus: kasutajate loomine, paketid, konfifail, logide kaust vms. Kirjuta sellele idempotentne playbook `localhost`-ile kausta `oma/`. Kirjuta `oma/README.md`-sse, mis oli enne käsitsi ja mis on nüüd kood.

**3. Teooria.** Loe teooria §2, §5 ja §6 ning vasta kolmele kordamisküsimusele (§2, §4 ja Osa 6 "Mõtle") failis `vastused.md`.

---

## ✅ Lõpukontroll

- [ ] Kontrolltabel täidetud; `curl localhost` vastas käsitsi seadistuse järel.
- [ ] `halb.sh`: nägid teisel jooksul vigu ja `conf`-failis duplikaati.
- [ ] `bootstrap.yml`: 5 task'i, ennustus kirjas, teine jooks `changed=0`.
- [ ] Drift tekitatud ja taastatud, README-s kirjas.
- [ ] Kodutöö: `admin.yml` teine jooks `changed=0`; `oma/` playbook ja README; `vastused.md`.

---

## Veaotsing

| Probleem | Kontroll |
|---|---|
| `Permission denied (publickey)` Gitis | SSH-võti pole GitHubis: `cat ~/.ssh/id_ed25519.pub` |
| `ping` ei vasta | failis `ansible_connection=local`; käsus `-i inventory.ini` |
| `apt` "Could not get lock" | oota, korda |
| `Permission denied` playbookis | `become: true` puudu |
| task on igal jooksul `changed` | vale moodul (`command`); kasuta `apt`/`service`/`user`/`file`/`copy` |
| teenus "started", aga ei vasta | `journalctl -u nginx -n 20` |
| `systemctl` ei tööta WSL-is | systemd pole WSL-is sisse lülitatud; ütle juhendajale |

---

## 📚 Allikad

| Allikas | URL |
|---|---|
| Ansible: inventory ja esimene playbook | <https://docs.ansible.com/ansible/latest/getting_started/> |
| Ansible builtin moodulid | <https://docs.ansible.com/ansible/latest/collections/ansible/builtin/> |
| Pro Git (eesti k) | <https://git-scm.com/book/et/v2> |
