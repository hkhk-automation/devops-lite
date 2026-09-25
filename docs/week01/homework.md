# K1 · Kodune õpe ja kodutöö

**Maht:** ~8,5 tundi kahe kohtumise vahel. **Tähtaeg:** kirjas Classroom 50-s.
**Kuhu:** samasse reposse, kuhu klassitöö. Iga ülesande juures on kirjas, mis fail kuhu läheb ja mida automaatne kontroll vaatab.

Kodutöö on klassitööst raskem. Juhiseid on vähem: parameetrid otsid `ansible-doc`-ist ja dokumentatsioonist ise, nagu tööl. Kui jääd kinni kauemaks kui 30 minutiks, kirjuta Classroom 50 repo Issues alla, mis käsu jooksutasid ja mis veateate said.

Kõik ülesanded käivad grupi `veeb` (kolm VM-i) vastu. Iga playbooki puhul kehtib klassist tuttav töökäik: `--check --diff` → `--limit vm1` → kõik → teine jooks `changed=0`.

| Osa | Sisu | Aeg |
|---|---|---|
| I | Kodune õpe: lugemine ja kordamisküsimused | ~2,3 h |
| II | Harjutused H1–H6 | ~4,5 h |
| III | Oma töö playbook | ~1,3 h |
| IV | Eneseanalüüs (+ vabatahtlik boonus) | ~0,2 h |

---

## I · Kodune õpe (~2,3 h)

### Loe

| Mida | Kus | Aeg |
|---|---|---|
| Loeng: klassis käsitlemata osad §2–§3, §6–§7, §9–§17 | [K1 loeng](lecture.md) | 1,5 h |
| Getting started: sissejuhatus, inventari loomine, esimene playbook | <https://docs.ansible.com/ansible/latest/getting_started/> | 45 min |
| Check mode ja diff | <https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_checkmode.html> | 20 min |
| Faktid | <https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html> | 20 min |

### Tööta dokumentatsiooniga

Ava `ansible-doc` abil need kolm moodulit: `user`, `lineinfile`, `service`. Kirjuta iga kohta faili `markmed.md` kolm parameetrit, mida sa klassis ei kasutanud, ja ühe lausega, milleks need on. Näiteks `user`: `shell`, `groups` + `append`, `state: absent` + `remove`.

### Kordamisküsimused

Vasta failis `vastused.md`, iga vastus 3–6 lauset oma sõnadega. Koopia dokumentatsioonist ei loe.

1. Võta cron-töö, mis teeb igal ööl andmebaasist varukoopia. Nimeta selle viis osa automatiseerimise mudeli järgi. Mis on selle töö tõend, ja kas see on kuskil nähtav? (loeng §3)
2. Miks on `useradd deploy` shelli skriptis ohtlikum kui `ansible.builtin.user`? Mis juhtub kummagagi teisel jooksul, ja kumma viga märkad? (§5)
3. Miks ei pea managed node'is Ansible paigaldatud olema? Mis peab seal olema? (§6)
4. Mis vahe on `inventory_hostname`-il ja `ansible_hostname`-il? Too näide, kus need erinevad. (§13)
5. `PLAY RECAP` näitab ühel masinal `unreachable=1`, teistel `changed=0`. Mida sa selle masina olekust tead? Mida teed järgmiseks? (§14)
6. Kolleeg jätab `--check` vahele, sest "playbook on testitud". Millise olukorra puhul läheb see valesti? Millal näitab `--check` ise valesti? (§16)
7. Mis vahe on privaat- ja avalikul võtmel? Kuhu kumbki käib? Mida teed, kui privaatvõti lekib? (§15)
8. Kirjelda oma töökohast üht triivi juhtumit (või väljamõeldud, aga realistlikku). Mis oli põhjus ja kuidas oleks playbook selle leidnud?

---

## II · Harjutused (~4,5 h)

Iga harjutuse lõpus salvesta teine jooks: `ansible-playbook -i inventory.ini <fail>.yml | tee logid/<fail>_teine_jooks.txt`.

### H1 · `admin.yml`: kasutajad ja ligipääs (~1,5 h)

Masinad `veeb`-grupis peavad olema olekus:

- kasutajad `deploy` ja `monitor` luuakse **ühe task'iga**, mis käib läbi nimekirja (`loop`);
- mõlemal on sinu avalik SSH-võti (`ansible.posix.authorized_key`), nii et `ssh deploy@vm1` töötab;
- `deploy` kuulub sudo-gruppi: Debianis `sudo`, RedHatis `wheel`, vali **fakti järgi**, mitte käsitsi hosti kaupa;
- `chrony` on paigaldatud ja käib;
- `/etc/motd` sisaldab `Hallatud Ansible'iga - <masina nimi>`.

**Valmis, kui:** `ssh deploy@vm1 sudo -n true` õnnestub; teine jooks `changed=0`.
**Fail:** `admin.yml`, `logid/admin_teine_jooks.txt`.

### H2 · `hardening.yml`: SSH turvamine (~1,5 h)

- `/etc/ssh/sshd_config`-is on `PermitRootLogin no` ja `PasswordAuthentication no` (`lineinfile`, uuri `regexp`);
- enne muudatuse rakendamist kontrollitakse konfi süntaksit (`validate: sshd -t -f %s`);
- `sshd` taaskäivitatakse **ainult siis, kui konf muutus** (uuri `notify` ja `handlers`).

⚠️ See ülesanne võib sind masinast välja lukustada. Hoia teine SSH-sessioon lahti, jooksuta esmalt `--check --diff`, siis `--limit vm1`, kontrolli uuest terminalist, et sisse saad, alles siis kõigil. Kui lukustasid end välja, kirjuta README-sse, mis juhtus ja kuidas said tagasi. See on väärtuslikum kui töö, mis kohe õnnestus.

**Valmis, kui:** `ssh root@vm1` keeldub; `ssh vm1` töötab võtmega; teine jooks `changed=0` ja handler ei käivitu.
**Fail:** `hardening.yml`, `logid/hardening_teine_jooks.txt`.

### H3 · `baas.yml`: paketid nimekirjast (~45 min)

Defineeri playbooki `vars` all kaks nimekirja: `paigalda` (vähemalt `htop`, `curl`, `git`, `tree`) ja `eemalda` (vähemalt `telnet`). Üks task paigaldab esimese nimekirja, teine tagab, et teise nimekirja paketid **puuduvad** (`state: absent`).

Proovi: paigalda `telnet` käsitsi ühte masinasse ja jooksuta playbook. Mitu `changed`-i tuleb?

**Valmis, kui:** teine jooks `changed=0`; käsitsi paigaldatud `telnet` eemaldati.
**Fail:** `baas.yml`, `logid/baas_teine_jooks.txt`.

### H4 · `raport.yml`: faktidest raport (~45 min)

Kirjuta playbook, mis loob igas masinas faili `/tmp/raport.txt`, kus on masina nimi, distributsioon ja versioon, IP-aadress, mälu MB-des ja protsessorite arv (kõik faktidest). Seejärel toob faili control node'i kausta `raportid/` (`ansible.builtin.fetch`, uuri `flat: true`), nii et iga masina raport on eraldi failis.

**Valmis, kui:** kaustas `raportid/` on kolm faili, igaüks oma masina andmetega.
**Fail:** `raport.yml`, `raportid/`.

### H5 · `cron.yml`: ajastatud töö (~45 min)

Igas masinas:

- skript `/usr/local/bin/varunda.sh` (`copy`, `mode: "0755"`), mis pakib `/etc` kausta `/var/backups/etc-<kuupäev>.tar.gz`;
- cron-töö, mis käivitab skripti iga päev kell 02:30 (`ansible.builtin.cron`).

Käivita skript korra käsitsi (`ssh vm1 sudo /usr/local/bin/varunda.sh`) ja kontrolli, et arhiiv tekkis. Vaata `crontab -l -u root`: kui jooksutad playbooki kaks korda, kas cron-rida on seal üks või kaks korda? Miks?

**Valmis, kui:** arhiiv tekib; cron-rida on üks; teine jooks `changed=0`.
**Fail:** `cron.yml`, `logid/cron_teine_jooks.txt`.

### H6 · Drift ja koristamine (~45 min)

1. Tekita igasse masinasse erinev drift: ühes kustuta `monitor`-kasutaja, teises muuda `/etc/motd` sisu, kolmandas peata `chrony`.
2. Jooksuta **kõik** oma playbookid `--check` režiimis ja salvesta väljund faili `logid/drift_check.txt`. Kas iga drift tuli välja? Millise playbooki järgi?
3. Paranda drift päris jooksuga.
4. Kirjuta `vastused.md`-sse lõik: kui peaksid seda kontrolli igal ööl automaatselt jooksutama, kuidas see välja näeks ja kes saaks teate?

**Fail:** `logid/drift_check.txt`, lõik `vastused.md`-s.

---

## III · Oma töö (~1,3 h)

Vali oma töökohast või kodulaborist üks korduv käsitsi tegevus: kasutajate lisamine, pakettide uuendamine, konfifaili muutmine, logikausta seadistamine, monitooringuagendi paigaldamine. Kirjuta sellele idempotentne playbook kausta `oma/`.

`oma/README.md`-s:

- mis oli enne käsitsi (sammud);
- mis on nüüd kood;
- tõend, et teine jooks ei muuda midagi;
- mis jäi automatiseerimata ja miks.

Kui töökoha süsteemi kasutada ei saa, tee sama oma VM-ides ja kirjelda, kuidas see töökohal välja näeks. Reposse ei lähe päris hostinimesid, IP-sid ega paroole.

---

## IV · Eneseanalüüs ja vabatahtlik boonus (~10 min)

**Eneseanalüüs** (`vastused.md` lõpus, 5–10 lauset): mis oli kõige raskem, kus ennustus läks mööda, mida teed tööl nüüd teisiti, mis jäi segaseks ja mida tahad järgmisel kohtumisel küsida.

**Vabatahtlik, kui aega jääb:**

**Boonus:** kirjuta `boonus.yml`, mis üritab paigaldada paketti, mida pole olemas, ja püüab vea kinni `block`/`rescue`-ga nii, et playbook kirjutab veast teate ega kuku. Selgita `vastused.md`-s, millal on selline vea püüdmine mõistlik ja millal ohtlik.

**Lint:** jooksuta `ansible-lint *.yml` ja paranda, mis parandada saad. Mida ei parandanud, selgita `vastused.md`-s.

---

## Mis repos lõpuks on

| Fail | Kust |
|---|---|
| `markmed.md`, `vastused.md` | I, H6, IV |
| `admin.yml` + `logid/admin_teine_jooks.txt` | H1 |
| `hardening.yml` + `logid/hardening_teine_jooks.txt` | H2 |
| `baas.yml` + `logid/baas_teine_jooks.txt` | H3 |
| `raport.yml` + `raportid/` (3 faili) | H4 |
| `cron.yml` + `logid/cron_teine_jooks.txt` | H5 |
| `logid/drift_check.txt` | H6 |
| `oma/*.yml` + `oma/README.md` | III |
| `boonus.yml` | IV, vabatahtlik |

Actions vahelehel näed pärast igat push'i, millised kontrollid on rohelised.
