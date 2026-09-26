# K1 · Kodune õpe ja kodutöö

Kodutöö läheb samasse reposse, kuhu klassitöö. Tähtaeg on kirjas Classroom 50-s. Kodutöö on klassitööst raskem: juhiseid on vähem ja parameetrid otsid `ansible-doc`-ist ise. Kui jääd kinni, küsi Discordis või ava oma repos issue **Vajan abi** (kirjuta käsk ja veateade).

| Osa | Mida teed | Esitad | Punkte |
|---|---|---|---|
| [I](#i-lugemine-ja-kusimused) | loed, uurid mooduleid, vastad küsimustele | `markmed.md`, `vastused.md` | 14 koos III-ga |
| [H1](#h1-adminyml-kasutajad-ja-ligipaas-10-p) | kasutajad, võtmed, sudo, chrony, motd | `admin.yml` + logi | 10 |
| [H2](#h2-hardeningyml-ssh-turvamine-10-p) | SSH turvamine | `hardening.yml` + logi | 10 |
| [H3](#h3-baasyml-paketid-nimekirjast-7-p) | paketid nimekirjast | `baas.yml` + logi | 7 |
| [H4](#h4-raportyml-raport-faktidest-7-p) | raport faktidest | `raport.yml`, `raportid/` | 7 |
| [H5–H6](#h5-cronyml-ajastatud-too) | cron ja drift | `cron.yml` + logi, `logid/drift_check.txt` | 7 |
| [III](#iii-oma-too) | oma playbook | `oma/*.yml`, `oma/README.md` | 14 koos I-ga |
| [IV](#iv-eneseanaluus) | eneseanalüüs | `vastused.md` lõpus | – |

Tulemust näed pärast iga push'i: **Actions** → **Autograde**. Loeb punktisumma, mitte värv.

---

## I · Lugemine ja küsimused

Enne küsimusi loe läbi:

| Mida | Kus |
|---|---|
| Loeng: klassis käsitlemata osad §2–§3, §6–§7, §9–§17 | [K1 loeng](lecture.md) |
| Getting started: sissejuhatus, inventar, esimene playbook | [docs.ansible.com](https://docs.ansible.com/ansible/latest/getting_started/) |
| Check mode ja diff | [docs.ansible.com](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_checkmode.html) |
| Faktid | [docs.ansible.com](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html) |

### Uuri mooduleid → `markmed.md`

Ava `ansible-doc`-iga moodulid `user`, `lineinfile` ja `service`. Kirjuta iga kohta kolm parameetrit, mida klassis ei kasutanud, ja ühe lausega, milleks need on. Näiteks `user`: `shell`, `groups` + `append`, `state: absent` + `remove`.

### Vasta küsimustele → `vastused.md`

Iga vastus 3–6 lauset oma sõnadega. Koopia dokumentatsioonist ei loe.

1. Võta cron-töö, mis teeb igal ööl andmebaasist varukoopia. Nimeta selle viis osa automatiseerimise mudeli järgi. Mis on selle töö tõend ja kas see on kuskil nähtav? (§3)
2. Miks on `useradd deploy` shelli skriptis ohtlikum kui `ansible.builtin.user`? Mis juhtub kummagagi teisel jooksul ja kumma viga märkad? (§5)
3. Miks ei pea managed node'is Ansible paigaldatud olema? Mis peab seal olema? (§6)
4. Mis vahe on `inventory_hostname`-il ja `ansible_hostname`-il? Too näide, kus need erinevad. (§13)
5. `PLAY RECAP` näitab ühel masinal `unreachable=1`, teistel `changed=0`. Mida sa selle masina olekust tead? Mida teed järgmiseks? (§14)
6. Kolleeg jätab `--check` vahele, sest "playbook on testitud". Millal läheb see valesti? Millal näitab `--check` ise valesti? (§16)
7. Mis vahe on privaat- ja avalikul võtmel? Kuhu kumbki käib? Mida teed, kui privaatvõti lekib? (§15)
8. Kirjelda üht triivi juhtumit oma VM-idest, koduarvutist või väljamõeldut, aga realistlikku. Mis oli põhjus ja kuidas oleks playbook selle leidnud?

---

## II · Harjutused

Kõik harjutused käivad grupi `veeb` vastu. Iga playbooki puhul sama töökäik nagu klassis:

1. `ansible-playbook <fail>.yml --check --diff`
2. `ansible-playbook <fail>.yml --limit vm1`
3. `ansible-playbook <fail>.yml`
4. teine jooks tõendiks: `ansible-playbook <fail>.yml | tee logid/<fail>_teine_jooks.txt`

### H1 · `admin.yml`: kasutajad ja ligipääs · 10 p

Igas `veeb`-grupi masinas:

- kasutajad `deploy` ja `monitor` luuakse ühe task'iga, mis käib läbi nimekirja (`loop`);
- mõlemal on sinu avalik SSH-võti (`ansible.posix.authorized_key`), nii et `ssh deploy@vm1` töötab;
- `deploy` kuulub gruppi `wheel`;
- `deploy` saab sudo't ilma paroolita: fail `/etc/sudoers.d/deploy` sisuga `deploy ALL=(ALL) NOPASSWD: ALL` (`copy`, `mode: "0440"`, `validate: visudo -cf %s`);
- `chrony` on paigaldatud ja teenus käib (AlmaLinuxis on teenuse nimi `chronyd`);
- `/etc/motd` sisaldab `Hallatud Ansible'iga - <masina nimi>`.

Valmis, kui:

- [ ] `ssh deploy@vm1 sudo -n true` õnnestub
- [ ] teine jooks `changed=0`

Esitad: `admin.yml`, `logid/admin_teine_jooks.txt`

### H2 · `hardening.yml`: SSH turvamine · 10 p

- `/etc/ssh/sshd_config`-is on `PermitRootLogin no` ja `PasswordAuthentication no` (`lineinfile`, uuri `regexp`);
- enne rakendamist kontrollitakse konfi süntaksit (`validate: sshd -t -f %s`);
- `sshd` taaskäivitatakse ainult siis, kui konf muutus (uuri `notify` ja `handlers`).

!!! warning "Võid end masinast välja lukustada"

    1. Hoia teine SSH-sessioon lahti.
    2. Jooksuta esmalt `--check --diff`, siis `--limit vm1`.
    3. Kontrolli uuest terminalist, et sisse saad. Alles siis kõigil.

    Kui lukustasid end välja, kirjuta README-sse, mis juhtus ja kuidas said tagasi. See on väärtuslikum kui töö, mis kohe õnnestus.

Valmis, kui:

- [ ] `ssh root@vm1` keeldub
- [ ] `ssh vm1` töötab võtmega
- [ ] teine jooks `changed=0` ja handler ei käivitu

Esitad: `hardening.yml`, `logid/hardening_teine_jooks.txt`

### H3 · `baas.yml`: paketid nimekirjast · 7 p

- `vars` all kaks nimekirja: `paigalda` (vähemalt `curl`, `git`, `tree`, `wget`, `tar`) ja `eemalda` (vähemalt `telnet`);
- üks task paigaldab esimese nimekirja, teine tagab, et teise nimekirja paketid puuduvad (`state: absent`).

Proovi: paigalda `telnet` käsitsi ühte masinasse ja jooksuta playbook. Mitu `changed`-i tuleb?

Valmis, kui:

- [ ] teine jooks `changed=0`
- [ ] käsitsi paigaldatud `telnet` eemaldati

Esitad: `baas.yml`, `logid/baas_teine_jooks.txt`

### H4 · `raport.yml`: raport faktidest · 7 p

- igas masinas fail `/tmp/raport.txt`: masina nimi, distributsioon ja versioon, IP-aadress, mälu MB-des, protsessorite arv (kõik faktidest);
- fail tuuakse control node'i kausta `raportid/` (`ansible.builtin.fetch`, uuri `flat: true`), iga masina raport eraldi failis `raportid/{{ inventory_hostname }}.txt`.

Valmis, kui:

- [ ] kaustas `raportid/` on kolm faili, igaüks oma masina andmetega

Esitad: `raport.yml`, `raportid/`

### H5 · `cron.yml`: ajastatud töö

Igas masinas:

- kaust `/var/backups` on olemas (AlmaLinuxis vaikimisi puudub);
- pakett `tar` on paigaldatud (AlmaLinuxi pilvepildis vaikimisi puudub);
- skript `/usr/local/bin/varunda.sh` (`copy`, `mode: "0755"`) pakib `/etc` kausta `/var/backups/etc-<kuupäev>.tar.gz`;
- cron-töö käivitab skripti iga päev kell 02:30 (`ansible.builtin.cron`).

Kontrolli vm1-s:

```bash
ssh -t vm1 sudo /usr/local/bin/varunda.sh
ssh -t vm1 sudo ls /var/backups
ssh -t vm1 sudo crontab -l -u root
```

Kui jooksutad playbooki kaks korda, kas cron-rida on üks või kaks korda? Miks?

Valmis, kui:

- [ ] arhiiv tekib
- [ ] cron-rida on üks
- [ ] teine jooks `changed=0`

Esitad: `cron.yml`, `logid/cron_teine_jooks.txt`

### H6 · Drift ja koristamine

1. Tekita igasse masinasse erinev drift: ühes kustuta `monitor`-kasutaja, teises muuda `/etc/motd` sisu, kolmandas peata `chronyd`.
2. Jooksuta kõik oma playbookid `--check` režiimis ja salvesta väljund faili `logid/drift_check.txt`. Kas iga drift tuli välja? Millise playbooki järgi?
3. Paranda drift päris jooksuga.
4. Kirjuta `vastused.md`-sse lõik: kui peaksid seda kontrolli igal ööl automaatselt jooksutama, kuidas see välja näeks ja kes saaks teate?

Esitad: `logid/drift_check.txt`, lõik `vastused.md`-s

---

## III · Oma töö

Vali oma VM-idest või koduarvutist üks korduv käsitsi tegevus: kasutajate lisamine, pakettide uuendamine, konfifaili muutmine, logikausta seadistamine. Kirjuta sellele idempotentne playbook kausta `oma/`.

`oma/README.md`-s:

- mis oli enne käsitsi (sammud);
- mis on nüüd kood;
- tõend, et teine jooks ei muuda midagi;
- mis jäi automatiseerimata ja miks.

Reposse ei lähe paroole ega võtmeid.

---

## IV · Eneseanalüüs

`vastused.md` lõpus, 5–10 lauset: mis oli kõige raskem, kus ennustus läks mööda, mida teeksid nüüd teisiti, mis jäi segaseks ja mida tahad järgmisel kohtumisel küsida.

??? note "Vabatahtlik: boonus ja lint"

    Boonus: kirjuta `boonus.yml`, mis üritab paigaldada paketti, mida pole olemas, ja püüab vea kinni `block`/`rescue`-ga nii, et playbook kirjutab veast teate ega kuku. Selgita `vastused.md`-s, millal on selline vea püüdmine mõistlik ja millal ohtlik.

    Lint: jooksuta `ansible-lint *.yml` ja paranda, mis parandada saad. Mida ei parandanud, selgita `vastused.md`-s.
