# K1 · Kodune õpe ja kodutöö

Kodutöö läheb samasse reposse, kuhu klassitöö. Tähtaeg on kirjas Classroom 50-s. Kodutöö on klassitööst raskem: juhiseid on vähem ja parameetrid otsid `ansible-doc`-ist ise. Kui jääd kinni, küsi Discordis või ava oma repos issue **Vajan abi**. Kirjuta sinna käsk ja veateade.

| Osa | Mida teed | Esitad | Punkte |
|---|---|---|---|
| [I](#i-lugemine-ja-kusimused) | loed, uurid mooduleid, vastad küsimustele | `markmed.md`, `vastused.md` | 14 |
| [H1](#h1-loo-kasutajad-ja-ligipaas-10-p) | kasutajad, võtmed, sudo, chrony, motd | `admin.yml` + logi | 10 |
| [H2](#h2-turva-ssh-10-p) | SSH turvamine | `hardening.yml` + logi | 10 |
| [H3](#h3-halda-pakette-nimekirjast-7-p) | paketid nimekirjast | `baas.yml` + logi | 7 |
| [H4](#h4-kogu-masinatest-raport-7-p) | raport faktidest | `raport.yml`, `raportid/` | 7 |
| [H5–H6](#h5-ajasta-varundus) | cron ja drift | `cron.yml` + logi, `logid/drift_check.txt` | 7 |
| [III](#iii-eneseanaluus) | eneseanalüüs | `vastused.md` lõpus | – |

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

Ava `ansible-doc`-iga moodulid `user` ja `service`. Kirjuta kummagi kohta kolm parameetrit, mida klassis ei kasutanud. Lisa ühe lausega, milleks need on. Näiteks `user`: `shell`, `groups` + `append`, `state: absent` + `remove`.

### Vasta küsimustele → `vastused.md`

Vasta lühidalt oma sõnadega, üks-kaks lauset igale.

1. Miks on `useradd deploy` shelli skriptis ohtlikum kui `ansible.builtin.user`? Mis juhtub kummagagi teisel jooksul? (A2, loeng §5)
2. `PLAY RECAP` näitab ühel masinal `unreachable=1`, teistel `changed=0`. Mida sa selle masina olekust tead ja mida teed järgmiseks? (loeng §14)
3. Mis vahe on privaat- ja avalikul võtmel? Kuhu kumbki käib ja mida teed, kui privaatvõti lekib? (loeng §15)

---

## II · Harjutused

Kõik harjutused käivad grupi `veeb` vastu.

!!! tip "Soovitatav töökäik iga playbooki puhul"

    1. `ansible-playbook <fail>.yml --check --diff`
    2. `ansible-playbook <fail>.yml --limit vm1`
    3. `ansible-playbook <fail>.yml`
    4. teine jooks tõendiks: `ansible-playbook <fail>.yml | tee logid/<fail>_teine_jooks.txt`

### H1 · Loo kasutajad ja ligipääs · 10 p

Selle ülesande lõpuks on igas masinas kasutaja `deploy`, kes pääseb võtmega sisse ja saab sudo't ilma paroolita.

Playbook `admin.yml` viib iga `veeb`-grupi masina sellisesse olekusse:

- kasutajad `deploy` ja `monitor` on olemas, mõlemad luuakse **ühe** task'iga;
- mõlemal on sinu avalik SSH-võti;
- `deploy` on grupis `wheel` ja saab sudo't ilma paroolita;
- kellaaja teenus `chrony` on paigaldatud ja käib;
- `/etc/motd` sisaldab `Hallatud Ansible'iga - <masina nimi>`.

??? tip "Vihje: moodulid ja parameetrid"

    - üks task mitmele kasutajale: `loop`;
    - võti: `ansible.posix.authorized_key`;
    - paroolita sudo: fail `/etc/sudoers.d/deploy` sisuga `deploy ALL=(ALL) NOPASSWD: ALL` (`copy`, `mode: "0440"`, `validate: visudo -cf %s`);
    - AlmaLinuxis on teenuse nimi `chronyd`, paketi nimi `chrony`.

Valmis, kui
{ .silt }

- [ ] `ssh deploy@vm1 sudo -n true` õnnestub
- [ ] failis on `loop` ja `authorized_key` (Autograde otsib neid)
- [ ] teine jooks `changed=0`

Esitad
{ .silt }

`admin.yml`, `logid/admin_teine_jooks.txt`

### H2 · Turva SSH · 10 p

Selle ülesande lõpuks ei saa masinatesse sisse root'ina ega parooliga, ainult võtmega.

Playbook `hardening.yml`:

- keelab SSH-s root'i sisselogimise ja parooliga sisselogimise;
- kontrollib enne rakendamist, et konf on korrektne;
- taaskäivitab `sshd` ainult siis, kui konf muutus.

!!! warning "Tähelepanu: võid end masinast välja lukustada"

    1. Hoia teine SSH-sessioon lahti.
    2. Jooksuta esmalt `--check --diff`, siis `--limit vm1`.
    3. Kontrolli uuest terminalist, et sisse saad. Alles siis kõigil.

    Kui lukustasid end välja, kirjuta README-sse, mis juhtus ja kuidas said tagasi.

??? tip "Vihje: moodulid ja parameetrid"

    - `/etc/ssh/sshd_config`: `PermitRootLogin no` ja `PasswordAuthentication no` (`lineinfile`, uuri `regexp`);
    - süntaksikontroll: `validate: sshd -t -f %s`;
    - taaskäivitus ainult muutusel: `notify` ja `handlers`.

Valmis, kui
{ .silt }

- [ ] `ssh root@vm1` keeldub
- [ ] failis on `lineinfile`, `validate` ja `handlers` (Autograde otsib neid)
- [ ] `ssh vm1` töötab võtmega
- [ ] teine jooks `changed=0` ja handler ei käivitu

Esitad
{ .silt }

`hardening.yml`, `logid/hardening_teine_jooks.txt`

### H3 · Halda pakette nimekirjast · 7 p

Selle ülesande lõpuks on vajalikud paketid igas masinas olemas ja keelatud paketid puuduvad.

Playbook `baas.yml`:

- paigaldab nimekirja `paigalda`: vähemalt `curl`, `git`, `tree`, `wget`, `tar`;
- tagab, et nimekirja `eemalda` paketid puuduvad: vähemalt `telnet`;
- mõlemad nimekirjad on muutujad, mitte task'i sees.

Kontroll: paigalda `telnet` käsitsi ühte masinasse ja jooksuta playbook. `telnet` peab kaduma ja teistes masinates peab jääma `changed=0`.

??? tip "Vihje: moodulid ja parameetrid"

    `vars` plokk kahe nimekirjaga; `ansible.builtin.package` võtab `name`-ile terve nimekirja; puudumine on `state: absent`.

Valmis, kui
{ .silt }

- [ ] teine jooks `changed=0`
- [ ] käsitsi paigaldatud `telnet` eemaldati
- [ ] failis on `state: absent` (Autograde otsib seda)

Esitad
{ .silt }

`baas.yml`, `logid/baas_teine_jooks.txt`

### H4 · Kogu masinatest raport · 7 p

Selle ülesande lõpuks on vm1-s iga masina kohta raport, mis on tehtud faktide põhjal.

Playbook `raport.yml`:

- loob igas masinas faili `/tmp/raport.txt`: masina nimi, distributsioon ja versioon, IP-aadress, mälu MB-des, protsessorite arv, kõik võetud faktidest;
- toob raportid vm1 kausta `raportid/`, iga masina oma eraldi failis.

??? tip "Vihje: moodulid ja parameetrid"

    Faili toomine: `ansible.builtin.fetch`, uuri `flat: true`. Failinimi inventari nime järgi: `raportid/{{ inventory_hostname }}.txt`, muidu kirjutavad raportid üksteist üle.

Valmis, kui
{ .silt }

- [ ] kaustas `raportid/` on kolm faili, igaüks oma masina andmetega
- [ ] failis on `fetch` (Autograde otsib seda)

Esitad
{ .silt }

`raport.yml`, `raportid/`

### H5 · Ajasta varundus

Selle ülesande lõpuks tehakse igas masinas igal ööl `/etc` varukoopia.

Playbook `cron.yml` tagab igas masinas:

- skripti `/usr/local/bin/varunda.sh`, mis pakib `/etc` faili `/var/backups/etc-<kuupäev>.tar.gz`;
- cron-töö, mis käivitab skripti iga päev kell 02:30.

!!! warning "Tähelepanu: AlmaLinuxis puudub kaks asja"

    Kausta `/var/backups` ja paketti `tar` pole vaikimisi olemas. Ilma nendeta skript ei tee midagi ega anna ka viga. Playbook peab need looma.

??? tip "Vihje: moodulid ja parameetrid"

    Kaust: `ansible.builtin.file` + `state: directory`. Skript: `ansible.builtin.copy` + `mode: "0755"`. Cron: `ansible.builtin.cron` (`name`, `minute`, `hour`, `job`).

Kontrolli vm1-s:

```bash
ssh -t vm1 sudo /usr/local/bin/varunda.sh
ssh -t vm1 sudo ls /var/backups
ssh -t vm1 sudo crontab -l -u root
```

??? success "Oodatav tulemus"

    ```
    etc-2026-10-01.tar.gz
    #Ansible: varundus
    30 2 * * * /usr/local/bin/varunda.sh
    ```

    Näed tänase kuupäevaga arhiivi ja täpselt ühte cron-rida, ka pärast playbooki teist jooksu.

Valmis, kui
{ .silt }

- [ ] arhiiv tekib
- [ ] cron-rida on üks
- [ ] failis on `cron` moodul (Autograde otsib seda)
- [ ] teine jooks `changed=0`

Esitad
{ .silt }

`cron.yml`, `logid/cron_teine_jooks.txt`

### H6 · Leia ja paranda drift

Selle ülesande lõpuks oskad `--check`-iga leida masinad, kus keegi on midagi käsitsi muutnud. Ja oskad selle playbookidega parandada.

1. Tekita igasse masinasse erinev drift: ühes kustuta kasutaja `monitor`, teises muuda `/etc/motd` sisu, kolmandas peata `chronyd`.
2. Jooksuta kõik oma playbookid `--check` režiimis ja salvesta väljund faili `logid/drift_check.txt`.
3. Paranda drift päris jooksuga.
4. Kirjuta `vastused.md`-sse lõik: kas iga drift tuli `--check`-iga välja ja millise playbooki järgi? Kui see kontroll jookseks igal ööl automaatselt, kes peaks teate saama?

Esitad
{ .silt }

`logid/drift_check.txt`, lõik `vastused.md`-s

---

## III · Eneseanalüüs

`vastused.md` lõppu kolm vastust, igaüks üks lause:

1. Mis töötas esimese korraga?
2. Kus jäid kinni ja mis aitas edasi?
3. Mida tahad järgmisel kohtumisel küsida?

??? note "Vabatahtlik: boonus ja lint"

    Boonus: kirjuta `boonus.yml`, mis proovib paigaldada paketti, mida pole olemas. Püüa viga kinni `block`/`rescue`-ga nii, et playbook kirjutab veast teate ega kuku. Selgita `vastused.md`-s, millal on selline vea püüdmine mõistlik ja millal ohtlik.

    Lint: jooksuta `ansible-lint *.yml` ja paranda, mis parandada saad. Mida ei parandanud, selgita `vastused.md`-s.
