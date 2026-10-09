# K2 · Kodune õpe ja kodutöö

Kodutöö läheb samasse reposse, kuhu klassitöö. Tähtaeg on kirjas Classroom 50-s. Kodutöös kasutad klassis ehitatud pinu edasi: lisad rakendusserveri, uuendad rakendust nii, et kasutaja seda ei märka, ja pöörad vigase versiooni automaatselt tagasi. Juhiseid on vähem kui klassis. Kui jääd kinni, küsi Discordis või ava oma repos issue **Vajan abi** ja kirjuta sinna käsk ning veateade.

| Osa | Mida teed | Esitad | Punkte |
|---|---|---|---|
| [I](#i-lugemine-ja-kusimused) | loed, uurid, vastad küsimustele | `markmed.md`, `vastused.md` | 14 |
| [H1](#h1-lisa-kolmas-rakendusserver-ainult-inventariga-8-p) | kolmas rakendusserver ühe reaga | `logid/kolm_rakendust.txt` | 8 |
| [H2](#h2-uuenda-rakendust-masinhaaval-12-p) | uuendus `serial`-iga, `delegate_to`, `run_once` | `uuendus.yml` + 2 logi | 12 |
| [H3](#h3-poora-vigane-versioon-tagasi-10-p) | tagasipööramine `block`/`rescue`-ga | `uuendus.yml`, `logid/tagasi.txt` | 10 |
| [H4](#h4-pane-soltuvused-kirja-5-p) | `requirements.yml`, README käivitusjuhis | `requirements.yml` | 5 |
| [H5](#h5-leia-kolleegi-vead-6-p) | neli istutatud viga kaustas `vead/` | `vead.md` | 6 |
| [III](#iii-eneseanaluus) | eneseanalüüs | `vastused.md` lõpus | – |

Tulemust näed pärast iga push'i: **Actions** → **Autograde**. Loeb punktisumma, mitte värv.

## I · Lugemine ja küsimused

Enne küsimusi loe läbi:

??? note "Lugemisnimekiri"

    | Mida | Kus |
    |---|---|
    | Loeng: klassis käsitlemata osad §2, §4, §6–§8, §11–§14 | [K2 loeng](lecture.md) |
    | Muutujate eelistusjärjekord | [docs.ansible.com](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#understanding-variable-precedence) |
    | Rollid | [docs.ansible.com](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html) |
    | Strateegiad ja `serial` | [docs.ansible.com](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_strategies.html) |
    | Blocks | [docs.ansible.com](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_blocks.html) |

### Uuri → `markmed.md`

1. Ava eelistusjärjekorra leht ja kirjuta üles kõik tasemed, mida sinu repos kasutatakse, nõrgemast tugevamani. Iga taseme juurde üks näide sinu repost.
2. Ava `ansible-doc ansible.builtin.template` ja kirjuta kolm parameetrit, mida klassis ei kasutanud, ühe lausega, milleks need on. Näiteks `backup`, `trim_blocks`, `lstrip_blocks`, `force`.

### Vasta küsimustele → `vastused.md`

Vasta lühidalt oma sõnadega, üks-kaks lauset igale.

1. B1-s läks pool päringuid vale pordi peale, aga kasutaja ei näinud viga. Miks? Mis oleks juhtunud ilma `proxy_next_upstream`-ita? (lab B1, loeng §9)
2. `--limit lb` jooks kukub veaga `'dict object' has no attribute 'ansible_default_ipv4'`. Mis on põhjus ja kaks viisi seda vältida? (loeng §4)
3. Kolleeg ütleb: „SELinux segab, lülitame välja.“ Mida vastad ja mida teed selle asemel? (lab A7, loeng §10)

## II · Harjutused

Kõik harjutused käivad sama pinu peale. Kui klassitöö B-osa on pooleli, tee see enne lõpuni: harjutused eeldavad, et `site.yml` annab `changed=0` ja koormus vaheldub.

!!! tip "Soovitatav töökäik iga playbooki puhul"

    1. `ansible-playbook <fail>.yml --syntax-check`
    2. `ansible-playbook <fail>.yml --check --diff`
    3. teises terminalis vm1-s käib kontroll: `while true; do curl -s -o /dev/null -w "%{http_code} " http://localhost/; sleep 0.5; done`
    4. päris jooks, siis teine jooks tõendiks

### H1 · Lisa kolmas rakendusserver ainult inventariga · 8 p

Selle ülesande lõpuks jagab koormusjaotur päringuid kolme rakendusserveri vahel ja selleks muutsid ühte faili.

vm1-l on vaba võimsust. Pane rakendus ka sinna:

- muuda ainult `inventory.ini`-d: vm1 tuleb ka `app`-gruppi;
- ükski roll, mall ega muutujafail ei muutu;
- `site.yml` jooks lisab vm1-sse rakenduse, uuendab nginx-i upstream'i ja andmebaasi `pg_hba.conf`-i.

Enne jooksu ennusta ja kirjuta `vastused.md`-sse: mis task'id on vm1-s `changed` ja miks just need?

??? tip "Vihje: mida kontrollida"

    `ansible-inventory --graph`, `grep server /etc/nginx/nginx.conf`, `sudo cat /var/lib/pgsql/data/pg_hba.conf`. vm1 rakendus ühendub andmebaasiga vm1 enda IP kaudu, mitte `localhost`-i kaudu. Kas `pg_hba.conf` lubab seda?

Valmis, kui
{ .silt }

- [ ] `curl` vastab vaheldumisi vm1, vm2 ja vm3
- [ ] `git diff --stat` näitab pärast klassitööd muudetuna ainult `inventory.ini`-d
- [ ] `logid/kolm_rakendust.txt`-is on kõik kolm nime (Autograde otsib neid)

Esitad
{ .silt }

```bash
for i in $(seq 1 9); do curl -s http://localhost/ | grep Vastas; done | tee logid/kolm_rakendust.txt
```

### H2 · Uuenda rakendust masinhaaval · 12 p

Selle ülesande lõpuks on playbook `uuendus.yml`, mis viib rakenduse uuele versioonile ühe masina kaupa ja kasutaja ei saa uuenduse ajal ühtegi viga.

Versioon on muutuja `app_versioon` (rolli `app` `defaults`-is `"1.0"`), see on näha lehe esimesel real. Uuendus käib nagu päris meeskonnas: kirjutad uue versiooni `group_vars/all.yml`-i (`app_versioon: "1.1"`), commit'id ja jooksutad `ansible-playbook uuendus.yml`. Nii on Gitis alati kirjas, mis versioon peab masinates olema, ja `site.yml` ei vii versiooni tagasi.

Playbook `uuendus.yml`:

- käib `app`-grupi läbi üks masin korraga ja peatub kohe, kui mõni masin kukub;
- kasutab sama rolli `app`, mitte ei kopeeri selle task'e;
- taaskäivitab rakenduse enne järgmise masina juurde minekut ja ootab, kuni `/health` vastab 200;
- kirjutab iga masina kohta rea vm1 faili `/var/log/labori-uuendused.log`: masin ja versioon. Sama versiooni teisel jooksul uut rida ei tule;
- trükib uuenduse alguses ühe korra (mitte iga masina kohta) teate, mis versioonile minnakse.

??? tip "Vihje: moodulid ja parameetrid"

    `serial`, `max_fail_percentage`; `ansible.builtin.import_role`; `meta: flush_handlers`; `ansible.builtin.uri` + `until`/`retries`; logirida `lineinfile` + `delegate_to` + `create`; teade `debug` + `run_once`. Loeng §12. Kuupäeva logireale ei pane: siis oleks rida igal jooksul uus ja teine jooks poleks kunagi `changed=0`.

Kontrolli teises terminalis, et uuenduse ajal ei tule ühtegi `502`-te.

Valmis, kui
{ .silt }

- [ ] leht näitab `Labori seis 1.1` kõigil masinatel ja `site.yml` teine jooks on samuti `changed=0`
- [ ] uuenduse ajal ei tulnud ühtegi `502`-te
- [ ] `/var/log/labori-uuendused.log`-is on rida iga masina kohta
- [ ] failis on `serial`, `delegate_to` ja `run_once` (Autograde otsib neid)
- [ ] sama versiooniga teine jooks `changed=0`

Esitad
{ .silt }

`uuendus.yml`, `logid/uuendus.txt` (playbooki väljund), `logid/uuendus_curl.txt` (teise terminali väljund uuenduse ajal)

### H3 · Pööra vigane versioon tagasi · 10 p

Selle ülesande lõpuks pöörab `uuendus.yml` vigase versiooni ise tagasi ja jätab ülejäänud masinad puutumata.

Rakendus ei käivitu, kui versiooni nimi lõpeb `-katki`-ga (vaata `app.py` algust). Nii saad vigast väljalaset turvaliselt harjutada.

Muuda `uuendus.yml`-i nii, et:

- kui uus versioon ei vasta `/health`-ile, paigaldatakse samasse masinasse tagasi eelmine versioon;
- vigane versioon on `group_vars/all.yml`-is (`app_versioon: "1.2-katki"`), eelmine versioon on muutujas `eelmine_versioon`, mille annad käsurealt;
- pärast tagasipööramist peatub uuendus ega lähe järgmise masina juurde;
- leht töötab kogu aja.

```bash
ansible-playbook uuendus.yml -e eelmine_versioon=1.1 | tee logid/tagasi.txt
```

Pärast pane `group_vars/all.yml`-i tagasi `"1.1"`. Muidu proovib järgmine `site.yml` jooks vigast versiooni uuesti.

??? tip "Vihje: moodulid ja parameetrid"

    `block` / `rescue`; tagasipööramisel `include_role` koos `vars:`-iga; lõpus `ansible.builtin.fail`. Miks `fail`, loe loengu §13 lõpust.

??? tip "Vihje: miks mitte `-e app_versioon=...`"

    `-e` võidab kõik, ka `include_role`-i `vars:`-i. Kui annaksid uue versiooni `-e`-ga, paigaldaks tagasipööramine uuesti vigase versiooni. Seepärast on uus versioon `group_vars`-is ja ainult eelmine käsureal. Tagasipööramisel kasuta `include_role`-i, sest `import_role` seob muutujad juba enne jooksu.

Valmis, kui
{ .silt }

- [ ] `logid/tagasi.txt`-is on `rescue` task'id jooksnud esimeses masinas ja teist masinat pole puudutatud
- [ ] pärast jooksu näitab leht kõigil masinatel `1.1`
- [ ] failis on `block` ja `rescue` (Autograde otsib neid)

Esitad
{ .silt }

`uuendus.yml`, `logid/tagasi.txt`

### H4 · Pane sõltuvused kirja · 5 p

Selle ülesande lõpuks saab kolleeg sinu repo kloonida ja ühe käsuga paigaldada kõik, mida playbookid vajavad.

- `requirements.yml` repo juures loetleb kõik kollektsioonid, mida sinu playbookid kasutavad, kinnitatud versiooniga;
- README-s on jaotis „Käivitamine“: kloonimine, sõltuvuste paigaldus, `site.yml`, `uuendus.yml` näitega.

Kontrolli, et fail päriselt töötab: `ansible-galaxy collection install -r requirements.yml --force`.

??? tip "Vihje: mis kollektsioone kasutad"

    `grep -rhoE '[a-z]+\.[a-z]+\.[a-z_]+:' roles *.yml | sort -u`. Kõik, mis ei alga `ansible.builtin`-iga, on kollektsioonist.

Valmis, kui
{ .silt }

- [ ] `requirements.yml`-is on `ansible.posix` versiooniga (Autograde otsib seda)
- [ ] README jaotis „Käivitamine“ on olemas

Esitad
{ .silt }

`requirements.yml`, `README.md`

### H5 · Leia kolleegi vead · 6 p

Selle ülesande lõpuks oled leidnud neli viga, mille kolleeg jättis kausta `vead/`, ja kirjutanud iga kohta, kuidas sa selle leidsid.

Kaustas `vead/` on neli faili. Igaüks on sinu repo mõne faili koopia, milles on üks viga. Iga faili jaoks:

1. vaata faili ja ennusta, mis juhtub;
2. kopeeri see õigesse kohta (failis endas on kommentaar, kuhu), jooksuta `site.yml` ja vaata, mis päriselt juhtub;
3. taasta oma fail: `git checkout -- <fail>` ja jooksuta uuesti, kuni `changed=0`.

!!! warning "Tähelepanu: üks viga ei anna veateadet"

    Vähemalt üks viga ei kuku, vaid jätab midagi vaikselt tegemata. Vaata `PLAY RECAP`-i ja kontrolli tulemust masinas, mitte ainult värvi.

Kirjuta `vead.md`-sse iga vea kohta pealkiri `## <faili nimi>` ja selle alla kolm rida: mida nägid (sümptom), mis oli põhjus, mis käsuga selle leidsid.

Valmis, kui
{ .silt }

- [ ] `vead.md`-is on neli `##` pealkirja (Autograde loeb neid)
- [ ] su enda failid on taastatud, `site.yml` teine jooks `changed=0`

Esitad
{ .silt }

`vead.md`

## III · Eneseanalüüs

`vastused.md` lõppu kolm vastust, igaüks üks lause:

1. Mis töötas esimese korraga?
2. Kus jäid kinni ja mis aitas edasi?
3. Mida tahad järgmisel kohtumisel küsida?

??? note "Vabatahtlik: kollektsioon ja lint"

    Kollektsioon: paigalda `community.postgresql` (vaata Galaxys, milline versioon toetab `ansible-core 2.14`-t) ja asenda `db`-rolli `kasutaja.yml` moodulitega `postgresql_user` ja `postgresql_db`. Kas nüüd muutub ka parool, kui muudad seda `group_vars`-is?

    Lint: `pip install --user ansible-lint` ja `ansible-lint site.yml`. Paranda, mis saad. Mida ei parandanud, selgita `vastused.md`-s.
