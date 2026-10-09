# K2 · Ansible sügavamalt: kolmekihiline rakendus

Selle loengu jaoks peaksid oskama kirjutada playbooki päris moodulitega, lugema `PLAY RECAP`-i ja jooksutama playbooki kolme masina vastu võtmega (K1 praktikum). Klassis räägime peatükkidest 1, 3, 5, 9 ja 10. Ülejäänu loe kodus läbi, peatükid 12–14 on kodutöö jaoks.

## Õpiväljundid

Pärast seda loengut oskad:

- kirjeldada mitmekihilist rakendust inventari gruppidena ja selgitada, miks grupp on ülesanne, mitte masinate nimekiri;
- panna muutujad õigesse kohta (`group_vars`, `host_vars`, rolli `defaults`) ja ennustada eelistusjärjekorra järgi, milline väärtus masinale jõuab;
- jagada playbooki rollideks ja task-failideks ning selgitada `import_*` ja `include_*` vahet;
- kasutada handlereid, `loop`-i, `when`-i ja `register`-it nii, et teine jooks jääb `changed=0`;
- kirjutada Jinja2 malle, mis loevad teiste masinate andmeid (`groups`, `hostvars`), ja kaitsta neid `validate`-iga;
- leida teenuste vahelise ühenduse viga logist ja eristada SELinuxi ning tulemüüri põhjust;
- selgitada, kuidas teha muudatust masinhaaval (`serial`) ja kuidas viga kinni püüda (`block`/`rescue`).

## 1. Üks server ei ole rakendus

K1-s oli kolm ühesugust veebiserverit. Üks playbook, üks grupp, iga masin sai sama oleku. Päris rakendus näeb harva nii välja.

Väike firma tahab sisemist rakendust, mis näitab, mis laboris parasjagu käib. Arendaja kirjutab selle Pythonis ja ütleb: rakendus vajab PostgreSQL-i ja peab olema kahes eksemplaris, et üks võiks uuenduse või rikke ajal maas olla. Kasutajad tulevad ühe aadressi kaudu. Nii tekib kolm kihti:

- **koormusjaotur** (nginx) võtab päringud vastu ja jagab need rakendusserverite vahel;
- **rakendusserverid** (kaks Pythoni protsessi) teevad töö;
- **andmebaas** (PostgreSQL) hoiab andmeid, mida mõlemad rakendusserverid jagavad.

Iga kiht vajab erinevat seadistust, aga kihid sõltuvad üksteisest. Koormusjaotur peab teadma rakendusserverite aadresse ja porte. Rakendusserver peab teadma andmebaasi aadressi ja parooli. Andmebaas peab teadma, milliselt aadressilt tohib ühenduda. Kui keegi muudab rakendusserveri porti ja unustab koormusjaoturi, ei vasta enam pool päringuid.

Kui see kõik on käsitsi tehtud, on sõltuvused ainult administraatori peas. Tänase lõpuks on need koodis: üks fail ütleb, mis masin millises rollis on, teine ütleb väärtused ja mallid arvutavad sõltuvused välja. Kui rakendusserver tuleb juurde, muudad ühte rida ja koormusjaotur ning andmebaasi luba uuenevad ise.

??? note "Tänane pinu masinate kaupa"

    | Masin | Grupid | Mis seal käib | Port |
    |---|---|---|---|
    | vm1 | `lb`, `db` | nginx (koormusjaotur), PostgreSQL | 80, 5432 |
    | vm2 | `app` | `labori-app` (Python) | 8080 |
    | vm3 | `app` | `labori-app` (Python) | 8081 |

    Päris keskkonnas oleks koormusjaotur ja andmebaas eri masinates. Meil on kolm VM-i, seega kannab vm1 kahte rolli. Koodis on need ikkagi eraldi: kui homme tuleb neljas masin, kolib andmebaas sinna ühe inventari reaga.

??? question "Kordamisküsimus"

    Nimeta tänase pinu kolm sõltuvust kihtide vahel. Milline neist läheb kõige tõenäolisemalt katki, kui keegi teeb muudatuse käsitsi?

## 2. Inventar: grupid ja alamgrupid

K1 inventaris oli grupp `veeb`. Nüüd grupeerid masinad selle järgi, mis ülesanne neil on:

```ini
[lb]
vm1

[db]
vm1

[app]
vm2
vm3

[stack:children]
lb
db
app
```

Üks masin võib olla mitmes grupis. vm1 on nii `lb`- kui ka `db`-grupis ja saab mõlema grupi muutujad ning mõlema play task'id. Masinana on ta ikkagi üks: `ansible stack -m ping` annab kolm vastust, mitte neli.

`[stack:children]` teeb grupi gruppidest. `stack` ei loetle masinaid, vaid ütleb, et tema liikmed on kõik, kes on `lb`-, `db`- või `app`-grupis. Nii saad sihtida kogu pinu korraga (näiteks ühised paketid) ja ei pea masinate nimekirja kahes kohas hoidma.

Lisaks sinu enda gruppidele on alati kaks sisseehitatud gruppi: `all` (kõik masinad) ja `ungrouped` (masinad, mis pole üheski grupis).

### Miks grupp on ülesanne

Grupi nimi peaks ütlema, mida masin teeb, mitte kus see asub või mis on selle nimi. `app` jääb õigeks ka siis, kui rakendus kolib teistesse masinatesse. `vm2_vm3` muutuks valeks esimese kolimisega ja iga play, mis seda kasutab, tuleks üle vaadata.

Suuremas keskkonnas on grupid mitmes mõõtmes korraga: ülesande järgi (`app`, `db`), keskkonna järgi (`test`, `prod`) ja asukoha järgi (`tallinn`, `haapsalu`). Masin on kõigis kolmes ja muutujad tulevad igast grupist. Mustritega saad neid kombineerida:

```bash
ansible 'app:&prod' -m ping      # app-grupis JA prod-grupis
ansible 'stack:!db' -m ping       # stack, aga mitte db
ansible-playbook site.yml --limit 'app'
```

### Inventari kontrollimine

Inventar on kood ja seda saab enne kasutamist kontrollida. Kolm käsku, mida kasutad iga päev:

```bash
ansible-inventory --graph          # grupid puuna
ansible-inventory --host vm3       # kõik muutujad, mis vm3-le jõuavad
ansible app --list-hosts           # kes kuulub mustrisse
```

`--host` on kõige kasulikum, kui muutuja väärtus üllatab. See näitab lõpptulemust pärast kõigi failide kokku liitmist, mitte seda, mis ühes failis kirjas on.

??? question "Kordamisküsimus"

    Kirjuta inventar, kus vm1 on koormusjaotur, vm2 andmebaas ja vm3 rakendusserver, ning grupp `stack` hõlmab kõiki. Mitu rida muutub, kui andmebaas kolib vm1-le?

??? info "Loe juurde"

    - [Ansible: inventari ülesehitus](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html)
    - [Ansible: mustrid](https://docs.ansible.com/ansible/latest/inventory_guide/intro_patterns.html)

## 3. `group_vars`, `host_vars` ja eelistusjärjekord

K1-s olid muutujad play `vars:` plokis. See sobib ühe playbooki jaoks. Kui sama väärtust vajavad mitu playbooki ja mitu rolli (andmebaasi nimi on vaja nii andmebaasile kui ka rakendusele), peab väärtus olema ühes kohas.

Ansible otsib inventari ja playbooki kõrvalt kaht kausta:

```
group_vars/
├── all.yml        # kõigile masinatele
├── app.yml        # app-grupile
└── db.yml         # db-grupile
host_vars/
└── vm3.yml        # ainult vm3-le
```

Faili nimi on grupi või masina nimi. Faili sisse kirjutad ainult muutujad, ilma `vars:` võtmeta:

```yaml
# group_vars/all.yml
app_port: 8080
db_nimi: labor
```

Faili asemel võib olla ka kaust (`group_vars/all/`), siis loetakse kõik selle failid. Seda kasutad, kui tahad saladused eraldi faili panna (§11).

### Kui sama muutuja on mitmes kohas

Kui `app_port` on `group_vars/all.yml`-is 8080 ja `host_vars/vm3.yml`-is 8081, saab vm3 väärtuse 8081. Põhimõte: kitsam võidab laiemat. Ühele masinale kirjutatud väärtus on konkreetsem kui kõigile kirjutatud väärtus.

<figure style="max-width:690px;margin:.8em auto" class="lx" markdown="0">
<svg viewBox="0 0 690 280" role="img" aria-label="Muutujate eelistusjärjekord: alumine kirjutab ülemise üle" xmlns="http://www.w3.org/2000/svg">
<style>.lx svg{width:100%;height:auto;font-family:var(--md-text-font-family,sans-serif)}.lx .box{fill:var(--md-code-bg-color);stroke:var(--md-default-fg-color--lighter);stroke-width:1.2}.lx .hi{fill:var(--md-primary-fg-color);fill-opacity:.16;stroke:var(--md-primary-fg-color);stroke-width:1.5}.lx .ok{fill:#2e7d32;fill-opacity:.14;stroke:#2e7d32;stroke-width:1.3}.lx .bad{fill:#c62828;fill-opacity:.12;stroke:#c62828;stroke-width:1.3}.lx .t{fill:var(--md-default-fg-color);font-size:15px;font-weight:700}.lx .b{fill:var(--md-default-fg-color);font-size:14.5px;font-weight:600}.lx .s{fill:var(--md-default-fg-color--light);font-size:13px}.lx .c{fill:var(--md-default-fg-color);font-size:12.5px;font-family:var(--md-code-font-family,monospace)}.lx .w{fill:var(--md-accent-fg-color);font-size:13.5px;font-weight:700}.lx .a{stroke:var(--md-default-fg-color--light);stroke-width:1.5;fill:none}.lx .d{stroke-dasharray:4 3}.lx .h{fill:var(--md-default-fg-color--light)}</style>
<defs><marker id="lxa" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path class="h" d="M0,0 L10,5 L0,10 z"/></marker></defs>
<rect class="box" x="150" y="8" width="230" height="32" rx="6"/><text class="b" x="265.0" y="29.0" text-anchor="middle">rolli defaults/</text>
<text class="s" x="396" y="29" text-anchor="start">vaikeväärtus, nõrgim</text>
<rect class="box" x="150" y="46" width="230" height="32" rx="6"/><text class="b" x="265.0" y="67.0" text-anchor="middle">group_vars/all.yml</text>
<text class="s" x="396" y="67" text-anchor="start">kõigile masinatele</text>
<rect class="box" x="150" y="84" width="230" height="32" rx="6"/><text class="b" x="265.0" y="105.0" text-anchor="middle">group_vars/&lt;grupp&gt;.yml</text>
<text class="s" x="396" y="105" text-anchor="start">ühele grupile</text>
<rect class="box" x="150" y="122" width="230" height="32" rx="6"/><text class="b" x="265.0" y="143.0" text-anchor="middle">host_vars/&lt;masin&gt;.yml</text>
<text class="s" x="396" y="143" text-anchor="start">ühele masinale</text>
<rect class="box" x="150" y="160" width="230" height="32" rx="6"/><text class="b" x="265.0" y="181.0" text-anchor="middle">play vars:</text>
<text class="s" x="396" y="181" text-anchor="start">ühele play'le</text>
<rect class="box" x="150" y="198" width="230" height="32" rx="6"/><text class="b" x="265.0" y="219.0" text-anchor="middle">rolli vars/</text>
<text class="s" x="396" y="219" text-anchor="start">rolli sisemine, kasutaja ei muuda</text>
<rect class="hi" x="150" y="236" width="230" height="32" rx="6"/><text class="b" x="265.0" y="257.0" text-anchor="middle">-e käsurealt</text>
<text class="s" x="396" y="257" text-anchor="start">ühekordne katse, võidab kõik</text>
<line class="a" x1="120" y1="268" x2="120" y2="12" marker-end="url(#lxa)"/>
<text class="s" x="110" y="150" text-anchor="end">tugevam</text>
</svg>
</figure>

Ansible'i dokumentatsioonis on eelistusjärjekorras 22 taset. Igapäevaselt piisab seitsmest, mis on joonisel: alumine kirjutab ülemise üle. Kaks asja on sageli üllatavad:

- rolli `defaults/` on kõige nõrgem. See on mõeldud vaikeväärtusteks, mida kasutaja võib üle kirjutada. `lb_port: 80` sobib sinna: enamasti 80, aga keegi võib tahta 8080.
- rolli `vars/` on tugevam kui `host_vars`. Kui paned rollis `vars/main.yml`-i `app_port: 8080`, ei saa seda inventarist enam muuta. Seepärast pane rolli `vars/` ainult väärtused, mida kasutaja ei tohi muuta (näiteks paketi nimi).

`-e` (extra vars) võidab alati kõik. See sobib ühekordseks katseks (`-e app_versioon=1.1`), aga mitte püsiseadistuseks: kui väärtus on ainult käsureal, siis Gitis seda pole ja järgmine inimene seda ei tea.

### Kuhu muutuja panna

??? note "Otsustustabel"

    | Küsimus | Koht |
    |---|---|
    | kas väärtus on rolli mõistlik vaikimisi seade, mida keegi võib muuta? | rolli `defaults/main.yml` |
    | kas seda vajavad mitu rolli või kogu pinu? | `group_vars/all.yml` |
    | kas see kehtib ühele grupile (kõik rakendusserverid)? | `group_vars/<grupp>.yml` |
    | kas see on ühe masina erisus? | `host_vars/<masin>.yml` |
    | kas see on saladus? | eraldi krüptitud fail, §11 |
    | kas see on ainult tänase katse jaoks? | `-e` |

Erisus ühele masinale on alati märk, et midagi on teisiti. Pane faili kommentaar, miks see nii on. Tänases praktikumis on vm3-l port 8081, sest pordil 8080 on kolleegi testteenus. Ilma kommentaarita kustutaks järgmine inimene selle faili „ühtluse huvides“.

### Kuidas lõpptulemust näha

```bash
ansible-inventory --host vm3
ansible stack -m debug -a "var=app_port"
ansible-playbook site.yml -e app_port=9999 --check   # mida -e muudaks
```

Kui väärtus üllatab, ära arva. Küsi Ansible'ilt endalt, mis väärtuse masin saab.

??? question "Kordamisküsimus"

    `group_vars/all.yml`-is on `app_port: 8080`, `group_vars/app.yml`-is `app_port: 8000`, `host_vars/vm3.yml`-is `app_port: 8081` ja rolli `app` `defaults/main.yml`-is `app_port: 5000`. Mis väärtuse saavad vm1, vm2 ja vm3?

??? info "Loe juurde"

    - [Ansible: muutujate eelistusjärjekord](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#understanding-variable-precedence)
    - [Ansible: muutujate failid inventaris](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html#organizing-host-and-group-variables)

## 4. Playbook mitme play'ga ja `site.yml`

K1 playbookis oli üks play: `hosts: veeb` ja task'id. Mitmekihilise pinu jaoks on vaja mitu play'd, igaüks oma grupile. Need on samas failis üksteise järel:

```yaml
- name: Ühine baas kõigile masinatele
  hosts: stack
  become: true
  roles:
    - common

- name: Andmebaas
  hosts: db
  become: true
  roles:
    - db

- name: Rakendusserverid
  hosts: app
  become: true
  roles:
    - app

- name: Koormusjaotur
  hosts: lb
  become: true
  roles:
    - lb
```

Play'd jooksevad järjest. Järjekord on oluline: andmebaas peab käima enne, kui rakendus ühendub, ja rakendus peab käima enne, kui koormusjaotur talle liiklust saadab. Iga play sees jooksevad task'id kõigis grupi masinates paralleelselt.

Sellist faili nimetatakse kokkuleppeliselt `site.yml`. See on kogu keskkonna kirjeldus: kui keegi küsib „kuidas meie rakendus üles seatakse?“, on vastus `ansible-playbook site.yml`.

### Faktid teistest masinatest

Esimene play `hosts: stack` kogub faktid kõigist kolmest masinast. Need jäävad mällu kogu jooksu ajaks ja hilisemad play'd saavad neid lugeda, ka teiste masinate omi. Nii teab vm1 koormusjaotur vm2 IP-d, kuigi `lb`-play jookseb ainult vm1-s.

See kehtib ainult ühe jooksu jooksul. Kui jooksutad `--limit lb`, kogutakse faktid ainult vm1-st ja mall, mis vajab vm2 IP-d, kukub veaga `'dict object' has no attribute 'ansible_default_ipv4'`. Lahendusi on kolm: jooksuta tervikuna; lisa algusesse play, mis kogub faktid kõigist (`hosts: all`, ilma task'ideta); või seadista faktide vahemälu (`fact_caching`), mis hoiab neid jooksude vahel alles.

### Suurem projekt: `import_playbook`

Kui `site.yml` kasvab pikaks, jaga see kihtide kaupa failideks:

```yaml
# site.yml
- ansible.builtin.import_playbook: db.yml
- ansible.builtin.import_playbook: app.yml
- ansible.builtin.import_playbook: lb.yml
```

Iga faili saab jooksutada ka eraldi (`ansible-playbook app.yml`), aga `site.yml` jääb ainsaks kohaks, kus kogu keskkond kokku tuleb.

??? question "Kordamisküsimus"

    Miks ei saa `lb`-play'd panna `site.yml`-is esimeseks? Mis läheks valesti esimesel jooksul värskes keskkonnas ja mis teisel jooksul?

??? info "Loe juurde"

    - [Ansible: playbooki ülesehitus](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html)
    - [Ansible: import_playbook](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/import_playbook_module.html)

## 5. Rollid

Kui kõik task'id on ühes failis, on `site.yml` sadu ridu pikk ja sama asja (näiteks nginx-i paigaldust) ei saa teises projektis uuesti kasutada. **Roll** on kaust, kus ühe ülesande task'id, handlerid, mallid, failid ja vaikeväärtused on koos ja kindlates kohtades.

```
roles/
└── app/
    ├── defaults/main.yml     # vaikeväärtused, nõrgim muutujate tase
    ├── vars/main.yml         # rolli sisemised väärtused, tugevad
    ├── tasks/main.yml        # task'id, siit algab
    ├── handlers/main.yml     # handlerid
    ├── templates/            # Jinja2 mallid (.j2)
    ├── files/                # failid, mis kopeeritakse muutmata
    └── meta/main.yml         # sõltuvused teistest rollidest, autor
```

Ükski kaust pole kohustuslik peale `tasks/`-i. Loo ainult need, mida vajad. Rolli skeleti saab teha ka käsuga `ansible-galaxy role init roles/app`, aga see loob kõik kaustad ja README, mida tavaliselt vaja pole.

Rollis kehtivad lühemad teed. `template: src=nginx.conf.j2` otsib faili rolli `templates/` kaustast, `copy: src=app.py` rolli `files/` kaustast. Sa ei pea kirjutama `roles/lb/templates/nginx.conf.j2`.

### Mis rolli ei kuulu

Rollis pole `hosts:`-i ega inventari. Roll ütleb, mida teha, aga mitte kus. Kus, ütleb `site.yml`:

```yaml
- name: Rakendusserverid
  hosts: app
  become: true
  roles:
    - app
```

Rollis pole ka konkreetseid väärtusi, mis sõltuvad keskkonnast. Andmebaasi parool, IP-d ja pordid tulevad `group_vars`-ist või faktidest. Siis saab sama rolli kasutada nii testis kui ka toodangus.

### Kuidas rolle jagada

Hea roll teeb ühte asja. Tänased neli rolli:

??? note "Tänased rollid"

    | Roll | Mida teeb | Vajab teistelt |
    |---|---|---|
    | `common` | firewalld, põhipaketid | – |
    | `db` | PostgreSQL, kasutaja, andmebaas, `pg_hba.conf` | `app`-grupi IP-d |
    | `app` | Pythoni rakendus, systemd teenus, tulemüür | `db`-grupi IP, andmebaasi nimi ja parool |
    | `lb` | nginx, upstream, SELinuxi luba | `app`-grupi IP-d ja pordid |

Rolli saab kasutada ka task'i sees (`ansible.builtin.import_role` või `include_role`), näiteks kui tahad sama rolli rakendada teiste muutujatega. Seda kasutad kodutöös H3.

### Rollid mujalt

Ansible Galaxys on tuhandeid valmis rolle ja kollektsioone (`geerlingguy.nginx`, `geerlingguy.postgresql`). Need säästavad aega, aga vaata enne, mida roll teeb ja kas seda hoitakse elus. Õppimiseks kirjutad täna rollid ise, et näeksid, mis nende sees on.

??? question "Kordamisküsimus"

    Kolleeg paneb andmebaasi parooli rolli `db` `defaults/main.yml`-i, „et roll töötaks kohe“. Mis selles on hea ja mis halb? Kuhu sa selle paneksid?

??? info "Loe juurde"

    - [Ansible: rollid](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)
    - [Ansible Galaxy](https://galaxy.ansible.com/)

## 6. Task-failid: import ja include

Ka üks roll võib minna pikaks. Andmebaasi rollis on kolm osa: paigaldus, seadistus, kasutajad. Need saab panna eraldi failidesse ja `tasks/main.yml` ainult kogub need kokku:

```yaml
- name: PostgreSQL paigaldus
  ansible.builtin.import_tasks: paigaldus.yml

- name: PostgreSQL seadistus
  ansible.builtin.import_tasks: seadistus.yml
```

Ansible'is on kaks moodi faili sisse tõmmata ja vahe on selles, millal see juhtub.

**`import_tasks`** on staatiline. Ansible loeb faili sisse enne, kui playbook käivitub. Kõik task'id on algusest teada: `--list-tasks` näitab neid, `--start-at-task` leiab need üles ja `tags` kehtivad igale task'ile eraldi. Faili nimes ei saa kasutada muutujat, mis selgub jooksu ajal.

**`include_tasks`** on dünaamiline. Fail loetakse sisse alles siis, kui jooks selle reani jõuab. Nii saad faili nime arvutada või kasutada `loop`-i:

```yaml
- name: OS-põhised task'id
  ansible.builtin.include_tasks: "{{ ansible_os_family | lower }}.yml"
```

AlmaLinuxis loeb see faili `redhat.yml`, Debianis `debian.yml`. `import_tasks`-iga seda teha ei saa, sest fakte pole enne jooksu veel olemas.

??? note "Kumba valida"

    | | `import_tasks` | `include_tasks` |
    |---|---|---|
    | millal loetakse | enne jooksu | jooksu ajal |
    | `--list-tasks` näitab task'e | jah | ei, ainult include rida |
    | muutuja failinimes | ainult inventari muutujad | ka faktid ja `register` |
    | `loop` | ei | jah |
    | `when` | kopeeritakse igale task'ile | kehtib include'ile üks kord |

    Vaikimisi kasuta `import_tasks`-i: see on ennustatavam ja vead tulevad välja juba `--syntax-check`-iga. `include_tasks` on siis, kui failinimi või kordus selgub jooksu ajal.

??? question "Kordamisküsimus"

    Sul on rollis failid `almalinux.yml` ja `ubuntu.yml`. Kumma käsuga ja millise muutujaga valid õige faili? Miks teine käsk ei tööta?

??? info "Loe juurde"

    - [Ansible: taaskasutus, import vs include](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse.html)

## 7. Handlerid

Kui nginx-i seadistus muutub, tuleb nginx uuesti laadida. Kui ei muutu, ei tohi seda teha, sest asjatu taaskäivitus katkestab ühendused. **Handler** on task, mis jookseb ainult siis, kui mõni teine task teda teavitab (`notify`) ja ise `changed` oli.

```yaml
# tasks/main.yml
- name: Koormusjaoturi seadistus on paigas
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
  notify: nginx laeb seaded uuesti

# handlers/main.yml
- name: nginx laeb seaded uuesti
  ansible.builtin.service:
    name: nginx
    state: reloaded
```

`notify` väärtus peab olema täpselt handleri `name`. Kirjaviga annab vea `The requested handler ... was not found`.

<figure style="max-width:690px;margin:.8em auto" class="lx" markdown="0">
<svg viewBox="0 0 690 134" role="img" aria-label="Kaks task'i teavitavad sama handlerit, see käivitub play lõpus üks kord" xmlns="http://www.w3.org/2000/svg">
<style>.lx svg{width:100%;height:auto;font-family:var(--md-text-font-family,sans-serif)}.lx .box{fill:var(--md-code-bg-color);stroke:var(--md-default-fg-color--lighter);stroke-width:1.2}.lx .hi{fill:var(--md-primary-fg-color);fill-opacity:.16;stroke:var(--md-primary-fg-color);stroke-width:1.5}.lx .ok{fill:#2e7d32;fill-opacity:.14;stroke:#2e7d32;stroke-width:1.3}.lx .bad{fill:#c62828;fill-opacity:.12;stroke:#c62828;stroke-width:1.3}.lx .t{fill:var(--md-default-fg-color);font-size:15px;font-weight:700}.lx .b{fill:var(--md-default-fg-color);font-size:14.5px;font-weight:600}.lx .s{fill:var(--md-default-fg-color--light);font-size:13px}.lx .c{fill:var(--md-default-fg-color);font-size:12.5px;font-family:var(--md-code-font-family,monospace)}.lx .w{fill:var(--md-accent-fg-color);font-size:13.5px;font-weight:700}.lx .a{stroke:var(--md-default-fg-color--light);stroke-width:1.5;fill:none}.lx .d{stroke-dasharray:4 3}.lx .h{fill:var(--md-default-fg-color--light)}</style>
<defs><marker id="lxa" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path class="h" d="M0,0 L10,5 L0,10 z"/></marker></defs>
<rect class="box" x="10" y="20" width="150" height="46" rx="6"/><text class="b" x="85.0" y="40.0" text-anchor="middle">task: mall</text><text class="s" x="85.0" y="56.0" text-anchor="middle">changed</text>
<line class="a" x1="160" y1="43" x2="196" y2="43" marker-end="url(#lxa)"/>
<rect class="box" x="198" y="20" width="150" height="46" rx="6"/><text class="b" x="273.0" y="40.0" text-anchor="middle">task: kood</text><text class="s" x="273.0" y="56.0" text-anchor="middle">changed</text>
<line class="a" x1="348" y1="43" x2="384" y2="43" marker-end="url(#lxa)"/>
<rect class="box" x="386" y="20" width="150" height="46" rx="6"/><text class="b" x="461.0" y="40.0" text-anchor="middle">task: teenus</text><text class="s" x="461.0" y="56.0" text-anchor="middle">ok</text>
<line class="a" x1="536" y1="43" x2="572" y2="43" marker-end="url(#lxa)"/>
<rect class="ok" x="574" y="20" width="110" height="46" rx="6"/><text class="b" x="629.0" y="40.0" text-anchor="middle">handler</text><text class="s" x="629.0" y="56.0" text-anchor="middle">1 restart</text>
<path class="a d" d="M85 66 V96 H610 V70" marker-end="url(#lxa)"/>
<path class="a d" d="M273 66 V84 H646 V70" marker-end="url(#lxa)"/>
<text class="s" x="180" y="92" text-anchor="middle">notify</text>
<text class="s" x="400" y="80" text-anchor="middle">notify</text>
<text class="s" x="343" y="122" text-anchor="middle">kaks notify'd, üks taaskäivitus play lõpus</text>
</svg>
</figure>

Handlerite kohta on kolm reeglit:

- handler jookseb play lõpus, mitte kohe pärast teavitust. Nii saad mitu muudatust teha ja taaskäivitada ühe korra;
- handler jookseb üks kord, ükskõik mitu task'i teda teavitas;
- kui play kukub enne lõppu, jäävad handlerid jooksmata. Järgmisel jooksul pole fail enam `changed` ja teenus jääb vanade seadetega. Seda saab vältida võtmega `--force-handlers` või `ansible.cfg`-s `force_handlers = True`.

### Kui handler on vaja varem käivitada

Mõnikord peab teenus uute seadetega käima juba play keskel. Andmebaasi rollis peab PostgreSQL olema uue `password_encryption` seadega taaskäivitatud enne, kui kasutaja luuakse. Muidu räsitakse parool vana meetodiga. Selleks on:

```yaml
- name: Seaded rakenduvad enne kasutaja loomist
  ansible.builtin.meta: flush_handlers
```

See käivitab kohe kõik handlerid, mida seni on teavitatud.

### Taaskäivitus või uuesti laadimine

Paljud teenused oskavad seadeid uuesti lugeda ilma protsessi peatamata (`reloaded`). nginx-i `reload` käivitab uued tööprotsessid uute seadetega ja vanad lõpetavad pooleliolevad päringud. Ükski ühendus ei katke. `restarted` peatab teenuse ja käivitab uuesti.

Kasuta `reloaded`, kui teenus seda toetab ja muudatus seda lubab. PostgreSQL-is piisab `pg_hba.conf` muutusele `reload`-ist, aga `listen_addresses` muutus vajab `restart`-i. Seepärast on andmebaasi rollis kaks handlerit.

??? question "Kordamisküsimus"

    Kolm task'i teavitavad handlerit `Rakendus taaskäivitub`. Esimesel jooksul on kõik kolm `changed`, teisel ükski. Mitu korda rakendus kummalgi jooksul taaskäivitub? Mis juhtub, kui teine task kukub?

??? info "Loe juurde"

    - [Ansible: handlerid](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html)

## 8. `loop`, `when` ja `register`

Need kolm märksõna teevad playbookist rohkem kui nimekirja.

### `loop`: üks task, mitu elementi

```yaml
- name: postgresql.conf seaded on paigas
  ansible.builtin.lineinfile:
    path: /var/lib/pgsql/data/postgresql.conf
    regexp: "^#?{{ item.nimi }}\\s*="
    line: "{{ item.nimi }} = '{{ item.vaartus }}'"
  loop:
    - { nimi: listen_addresses, vaartus: "*" }
    - { nimi: password_encryption, vaartus: scram-sha-256 }
```

Task jookseb iga elemendi kohta ja element on muutujas `item`. Element võib olla lihtne väärtus (`- nginx`) või sõnastik (`{ nimi: ..., vaartus: ... }`).

Kui moodul võtab nimekirja ise vastu, ära kasuta `loop`-i. `package: name: [nginx, curl]` paigaldab mõlemad ühe `dnf` kutsega. Sama `loop`-iga teeks kaks eraldi kutset ja oleks aeglasem.

### `register`: salvesta tulemus

Iga task tagastab tulemuse: `changed`, `rc`, `stdout`, `stderr` ja moodulist sõltuvad väljad. `register` salvestab selle muutujasse:

```yaml
- name: Kontrolli, kas andmebaasi kasutaja on olemas
  ansible.builtin.command: psql -tAc "SELECT 1 FROM pg_roles WHERE rolname='labor'"
  become_user: postgres
  register: db_roll
  changed_when: false
  check_mode: false

- name: Näita, mis tagasi tuli
  ansible.builtin.debug:
    var: db_roll
```

Kui ei tea, mis väljad tulemuses on, kasuta `debug`-i. Sisu on iga mooduli puhul erinev.

### `when`: tingimus

```yaml
- name: Andmebaasi kasutaja on olemas
  ansible.builtin.command: psql -c "CREATE ROLE labor LOGIN PASSWORD '...'"
  become_user: postgres
  when: db_roll.stdout != "1"
```

`when` on Jinja2 avaldis ilma `{{ }}`-ta. Kui tingimus ei kehti, on task `skipped`. Tingimuses saad kasutada muutujaid, fakte (`when: ansible_distribution == "AlmaLinux"`) ja varasemate task'ide tulemusi.

### Kuidas `command`-ist teha idempotentne task

Need kolm koos teevad sama, mida moodul teeb sisemiselt: kontrolli olekut, otsusta, muuda ainult vajadusel. Seda on vaja, kui moodulit pole. Tänases praktikumis pole PostgreSQL-i kasutajate moodulit, sest kollektsioon `community.postgresql` pole meie masinates paigaldatud. Muster:

1. kontrolltask loeb olekut, `register`-iga, `changed_when: false` ja `check_mode: false`;
2. muutev task jookseb `when`-iga ainult siis, kui olek pole õige.

`changed_when: false` on vajalik, sest `command` on muidu alati `changed` ja teine jooks poleks kunagi `changed=0`. `check_mode: false` on vajalik, sest `--check` all jäetakse `command` vahele ja `db_roll` jääks tühjaks.

Muster ei ole täiuslik. Kui parool `group_vars`-is muutub, ei näe kontrolltask seda: kasutaja on olemas, `when` ütleb „ei tee midagi“. Moodul `community.postgresql.postgresql_user` võrdleks ka parooli. Kui moodul on olemas, eelista seda.

??? question "Kordamisküsimus"

    Miks on kontrolltask'il vaja nii `changed_when: false` kui ka `check_mode: false`? Mis juhtub, kui üks neist puudub, teisel jooksul ja `--check` jooksul?

??? info "Loe juurde"

    - [Ansible: loop](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_loops.html)
    - [Ansible: tingimused](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_conditionals.html)
    - [Ansible: changed_when ja failed_when](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_error_handling.html#defining-changed)

## 9. Jinja2 mallid ja `hostvars`

K1-s kirjutasid avalehe `copy`-mooduli `content:`-iga. Kui fail on pikem või sõltub masinast, kasutad **malli**: tekstifail laiendiga `.j2`, milles on muutujad ja loogika. Moodul `template` arvutab malli iga masina jaoks ja kopeerib tulemuse:

```yaml
- name: Rakenduse seaded on paigas
  ansible.builtin.template:
    src: labori-app.env.j2
    dest: /etc/labori-app.env
    mode: "0600"
```

Malli sees on kolm märgistust:

- `{{ muutuja }}` paneb väärtuse;
- `{% for ... %}`, `{% if ... %}` on loogika ja ise teksti ei tekita;
- `{# kommentaar #}` jääb lõpptulemusest välja.

Esimene rida `# {{ ansible_managed }}` kirjutab faili hoiatuse, et faili haldab Ansible. Nii teab käsitsi muutja, et tema muudatus kirjutatakse järgmisel jooksul üle.

### Filtrid

Filter muudab väärtust ja käib `|` järel:

```jinja
{{ app_nimi | upper }}                   {# LABORI SEIS #}
{{ db_port | default(5432) }}            {# 5432, kui db_port puudub #}
{{ groups['app'] | length }}             {# 2 #}
{{ groups['app'] | join(', ') }}         {# vm2, vm3 #}
```

`default` on kõige sagedamini kasutatav: mall ei kuku, kui muutujat pole.

### Teiste masinate andmed

Mall jookseb ühe masina jaoks, aga sageli vajab ta teiste masinate andmeid. Selleks on kaks maagilist muutujat:

- `groups` on kõik grupid ja nende liikmed: `groups['app']` on `['vm2', 'vm3']`;
- `hostvars` on kõik, mida Ansible iga masina kohta teab: `hostvars['vm2']` sisaldab vm2 muutujaid ja fakte.

Koormusjaoturi mall käib `app`-grupi masinad läbi ja võtab igaühe IP ja pordi:

```jinja
upstream rakendus {
{% for h in groups['app'] %}
    server {{ hostvars[h].ansible_default_ipv4.address }}:{{ hostvars[h].app_port }};
{% endfor %}
}
```

<figure style="max-width:690px;margin:.8em auto" class="lx" markdown="0">
<svg viewBox="0 0 690 146" role="img" aria-label="Mall loeb teiste masinate muutujaid hostvars-ist" xmlns="http://www.w3.org/2000/svg">
<style>.lx svg{width:100%;height:auto;font-family:var(--md-text-font-family,sans-serif)}.lx .box{fill:var(--md-code-bg-color);stroke:var(--md-default-fg-color--lighter);stroke-width:1.2}.lx .hi{fill:var(--md-primary-fg-color);fill-opacity:.16;stroke:var(--md-primary-fg-color);stroke-width:1.5}.lx .ok{fill:#2e7d32;fill-opacity:.14;stroke:#2e7d32;stroke-width:1.3}.lx .bad{fill:#c62828;fill-opacity:.12;stroke:#c62828;stroke-width:1.3}.lx .t{fill:var(--md-default-fg-color);font-size:15px;font-weight:700}.lx .b{fill:var(--md-default-fg-color);font-size:14.5px;font-weight:600}.lx .s{fill:var(--md-default-fg-color--light);font-size:13px}.lx .c{fill:var(--md-default-fg-color);font-size:12.5px;font-family:var(--md-code-font-family,monospace)}.lx .w{fill:var(--md-accent-fg-color);font-size:13.5px;font-weight:700}.lx .a{stroke:var(--md-default-fg-color--light);stroke-width:1.5;fill:none}.lx .d{stroke-dasharray:4 3}.lx .h{fill:var(--md-default-fg-color--light)}</style>
<defs><marker id="lxa" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path class="h" d="M0,0 L10,5 L0,10 z"/></marker></defs>
<rect class="hi" x="10" y="30" width="190" height="60" rx="6"/><text class="b" x="105.0" y="57.0" text-anchor="middle">mall jookseb vm1-s</text><text class="s" x="105.0" y="73.0" text-anchor="middle">groups['app']</text>
<rect class="box" x="250" y="10" width="190" height="44" rx="6"/><text class="b" x="345.0" y="29.0" text-anchor="middle">hostvars['vm2']</text><text class="s" x="345.0" y="45.0" text-anchor="middle">10.0.0.12, port 8080</text>
<rect class="box" x="250" y="66" width="190" height="44" rx="6"/><text class="b" x="345.0" y="85.0" text-anchor="middle">hostvars['vm3']</text><text class="s" x="345.0" y="101.0" text-anchor="middle">10.0.0.13, port 8081</text>
<line class="a" x1="200" y1="50" x2="248" y2="32" marker-end="url(#lxa)"/>
<line class="a" x1="200" y1="70" x2="248" y2="88" marker-end="url(#lxa)"/>
<line class="a" x1="440" y1="60" x2="488" y2="60" marker-end="url(#lxa)"/>
<rect class="ok" x="490" y="24" width="190" height="72" rx="6"/><text class="b" x="585.0" y="57.0" text-anchor="middle">nginx.conf vm1-s</text><text class="c" x="585.0" y="73.0" text-anchor="middle">…12:8080  …13:8081</text>
<text class="s" x="343" y="134" text-anchor="middle">vm1 muutujad ei loe: iga rida küsib oma masina väärtust</text>
</svg>
</figure>

Kõige sagedasem viga on kirjutada siia `{{ app_port }}` ilma `hostvars`-ita. Mall jookseb vm1 jaoks, seega `app_port` on vm1 väärtus, 8080. Kui vm3 kuulab 8081-l, saadab nginx talle päringud vale pordi peale. Kasutaja seda ei märka, sest nginx proovib järgmist serverit. Praktikumi B1 on just see viga.

Reegel: kui mall räägib teisest masinast, peab iga selle masina väärtus tulema `hostvars[h]`-st.

### Malli kontrollimine enne kirjutamist

Vigane konfiguratsioonifail võib teenuse maha võtta. Mooduli `template` (ja `copy`, `lineinfile`) võti `validate` käivitab enne faili asendamist kontrollkäsu:

```yaml
- name: Koormusjaoturi seadistus on paigas
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    validate: nginx -t -c %s
```

`%s` on ajutine fail. Kui `nginx -t` kukub, jääb vana fail paigale, task on `failed` ja handlerit ei teavitata. Teenus töötab edasi vanade seadetega.

??? note "Kontrollkäsud, mida tasub teada"

    | Fail | `validate` |
    |---|---|
    | `nginx.conf` | `nginx -t -c %s` |
    | `sshd_config` | `sshd -t -f %s` |
    | `/etc/sudoers.d/*` | `visudo -cf %s` |
    | Apache | `httpd -t -f %s` |
    | HAProxy | `haproxy -c -f %s` |

    `validate` kontrollib ainult ühte faili. Kui fail viitab teistele failidele (`include`), peab kontrollkäsk need üles leidma. Seepärast asendab tänane mall terve `nginx.conf`-i, mitte faili `conf.d/` kaustas.

??? question "Kordamisküsimus"

    Mall kirjutab `pg_hba.conf`-i rea iga `app`-grupi masina IP-ga. Mis juhtub failiga, kui lisad inventari `[app]` alla vm1? Mis siis, kui jooksutad `--limit db`?

??? info "Loe juurde"

    - [Ansible: mallid](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_templating.html)
    - [Ansible: maagilised muutujad](https://docs.ansible.com/ansible/latest/reference_appendices/special_variables.html)
    - [Jinja2: mallide disain](https://jinja.palletsprojects.com/en/stable/templates/)

## 10. SELinux ja tulemüür teenuste vahel

K1-s avasid vm2 ja vm3 tulemüüris `http`-i, et leht oleks väljast näha. Täna räägivad masinad omavahel ja iga ühendus läbib kaks kontrolli: saatja SELinuxi ja vastuvõtja tulemüüri.

**SELinux** on AlmaLinuxi turvakiht, mis piirab, mida protsess tohib teha, isegi kui see jookseb root'ina. Iga protsess ja fail on märgistatud (kontekst) ning poliitika ütleb, mis kontekst mida tohib. nginx jookseb kontekstis `httpd_t`. Vaikimisi tohib `httpd_t` ühendusi vastu võtta, aga mitte ise teistele masinatele ühenduda, sest tavaline veebiserver ei peaks seda tegema. Koormusjaotur peab.

Lubade jaoks on poliitikas lülitid (boolean). Koormusjaoturile on vaja üht:

```yaml
- name: nginx tohib ühenduda rakendusserveritega (SELinux)
  ansible.posix.seboolean:
    name: httpd_can_network_connect
    state: true
    persistent: true
```

Ära lülita SELinuxit välja (`setenforce 0`). See eemaldab kaitse kõigilt teenustelt ühe teenuse probleemi tõttu ja kaob niikuinii järgmisel reboot'il, kui keegi pole konfi muutnud.

**firewalld** otsustab, millised pordid on masinas väljastpoolt avatud. Rakendusserver peab avama pordi, millel rakendus kuulab, andmebaas pordi 5432. Avamine käib rollis selle teenuse juures, mis porti vajab. Nii liigub port koos teenusega, kui teenus kolib.

### Kuidas vahet teha

Mõlemad annavad kasutajale sama `502 Bad Gateway`. Erinevus on nginx-i vealogis (`/var/log/nginx/error.log`):

??? note "Ühendusvead upstream'i juures"

    | Logis | Tähendus | Kus otsida |
    |---|---|---|
    | `(13: Permission denied) while connecting to upstream` | oma masina SELinux keelas ühenduse | `ausearch -m avc -ts recent`, `getsebool -a \| grep httpd` |
    | `(113: No route to host)` | teise masina firewalld lükkas tagasi | `firewall-cmd --list-all` sihtmasinas |
    | `(111: Connection refused)` | sihtmasinas ei kuula keegi sel pordil | `systemctl status`, `ss -tlnp` sihtmasinas |
    | `upstream timed out (110: Connection timed out)` | pakett kadus teel või teenus ei vasta | võrk, ülekoormatud teenus |

Viga otsi alati nii: kõigepealt, kas teenus ise töötab (`curl localhost` sihtmasinas). Siis, kas saatja tohib ühenduda (SELinux). Siis, kas vastuvõtja laseb sisse (tulemüür). Iga sammu kontrollid eraldi käsuga, mitte ei arva.

??? question "Kordamisküsimus"

    Rakendusserveri `/health` vastab `localhost`-ist, aga koormusjaoturilt tuleb 502 ja logis on `113: No route to host`. Kumma masina millist seadistust kontrollid ja millise käsuga?

??? info "Loe juurde"

    - [Red Hat: SELinuxi kasutamine](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/index)
    - [ansible.posix.seboolean](https://docs.ansible.com/ansible/latest/collections/ansible/posix/seboolean_module.html)
    - [ansible.posix.firewalld](https://docs.ansible.com/ansible/latest/collections/ansible/posix/firewalld_module.html)

## 11. Saladused ja Ansible Vault

Andmebaasi parool on `group_vars/all.yml`-is avatud tekstina. Kui repo on GitHubis, näeb seda igaüks, kellel on ligipääs, ja see jääb Giti ajalukku ka siis, kui hiljem kustutad.

**Ansible Vault** krüptib failid või üksikud väärtused AES256-ga. Krüptitud fail on Gitis, aga selle sisu saab lugeda ainult Vaulti parooliga:

```bash
ansible-vault create group_vars/all/vault.yml    # uus krüptitud fail
ansible-vault encrypt group_vars/all/vault.yml   # olemasolev fail krüptiks
ansible-vault view group_vars/all/vault.yml      # vaata
ansible-vault edit group_vars/all/vault.yml      # muuda
ansible-playbook site.yml --ask-vault-pass
```

Hea tava on kaks faili samas kaustas:

```yaml
# group_vars/all/main.yml (avatud)
db_parool: "{{ vault_db_parool }}"

# group_vars/all/vault.yml (krüptitud)
vault_db_parool: "tegelik-parool"
```

Nii leiab `grep db_parool` üles, kust väärtus tuleb, ja ainult väärtus on saladus. Vaulti parool ise ei lähe kunagi reposse. Kui ei taha seda iga kord trükkida, pane see faili väljaspool repot ja viita `ansible.cfg`-s: `vault_password_file = ~/.vault_pass`.

Saladus satub logisse ka siis, kui fail on krüptitud: `debug`, veateade või `-v` väljund võivad selle välja printida. Task'il, mis kasutab parooli, on `no_log: true`.

Vault sobib väikesele meeskonnale. Suuremas keskkonnas hoitakse saladusi eraldi süsteemis (HashiCorp Vault, OpenBao, pilve saladuste haldurid) ja Ansible loeb need sealt jooksu ajal. K4-s näed, kuidas GitHub Actions hoiab saladusi.

??? question "Kordamisküsimus"

    Kolleeg krüptis `vault.yml`-i, aga commit'is enne seda sama faili avatud kujul. Kas probleem on lahendatud? Mis on kolm sammu, mida peaksid nüüd tegema?

??? info "Loe juurde"

    - [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html)

## 12. Muudatus mitmes masinas: `serial`, `delegate_to`, `run_once`

Vaikimisi teeb Ansible iga task'i kõigis play masinates korraga. Kui uuendad rakendust, taaskäivituvad vm2 ja vm3 samal ajal ja paariks sekundiks ei vasta kumbki. Kasutaja näeb 502-te.

**`serial`** jagab masinad partiideks. Play jookseb lõpuni esimeses partiis, siis teises:

```yaml
- name: Rakenduse uuendus masinhaaval
  hosts: app
  become: true
  serial: 1
  max_fail_percentage: 0
  tasks:
    - name: Rakendus on uuendatud
      ansible.builtin.import_role:
        name: app

    - name: Uued seaded rakenduvad kohe
      ansible.builtin.meta: flush_handlers

    - name: Rakendus vastab enne järgmise masina juurde minekut
      ansible.builtin.uri:
        url: "http://localhost:{{ app_port }}/health"
      register: tervis
      until: tervis.status == 200
      retries: 10
      delay: 2
```

`serial: 1` tähendab üks masin korraga, `serial: "30%"` kolmandik korraga. `max_fail_percentage: 0` peatab kogu uuenduse, kui üks masin kukub. Nii ei rikku vigane versioon kõiki servereid, vaid ainult esimese. Seni teenindab ülejäänud partii kasutajaid edasi.

**`delegate_to`** jooksutab task'i teises masinas, aga praeguse masina nimel. Näiteks uuenduse logi kirjutatakse koormusjaoturisse:

```yaml
- name: Uuendus on logitud koormusjaoturis
  ansible.builtin.lineinfile:
    path: /var/log/labori-uuendused.log
    line: "{{ inventory_hostname }} versioon {{ app_versioon }}"
    create: true
  delegate_to: "{{ groups['lb'][0] }}"
```

`inventory_hostname` on ikkagi vm2 või vm3, aga fail tekib vm1-s. Päris keskkonnas kasutatakse sama võtet, et võtta server koormusjaoturist välja enne uuendust ja panna tagasi pärast.

**`run_once`** jooksutab task'i ainult ühe korra, mitte iga masina jaoks. Näiteks teade uuenduse algusest või andmebaasi migratsioon, mida tohib teha ainult üks kord. `serial`-iga koos jookseb `run_once` task üks kord iga partii kohta.

??? question "Kordamisküsimus"

    Sul on 10 rakendusserverit ja `serial: 3`. Mitu partiid tuleb? Mis juhtub, kui teise partii esimene masin kukub ja `max_fail_percentage: 0`?

??? info "Loe juurde"

    - [Ansible: strateegiad ja serial](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_strategies.html)
    - [Ansible: delegeerimine](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_delegation.html)

## 13. Vigade käsitlemine: `block` ja `rescue`

Kui task kukub, lõpetab Ansible selle masina jaoks töö. Sageli on see õige. Aga uuenduse puhul tahad, et vigane versioon pööratakse tagasi, mitte et masin jääb poolikusse olekusse.

`block` grupeerib task'id. `rescue` jookseb, kui mõni `block`-i task kukkus. `always` jookseb igal juhul:

```yaml
- name: Uuendus tagasipööramisega
  block:
    - name: Uus versioon on paigas
      ansible.builtin.import_role:
        name: app
    - ansible.builtin.meta: flush_handlers
    - name: Uus versioon vastab
      ansible.builtin.uri:
        url: "http://localhost:{{ app_port }}/health"
      register: tervis
      until: tervis.status == 200
      retries: 5
      delay: 2
  rescue:
    - name: Vana versioon on tagasi
      ansible.builtin.include_role:
        name: app
      vars:
        app_versioon: "{{ eelmine_versioon }}"
    - ansible.builtin.meta: flush_handlers
    - name: Uuendus peatatakse
      ansible.builtin.fail:
        msg: "{{ inventory_hostname }}: versioon {{ app_versioon }} ei käivitunud, pöörasin tagasi"
  always:
    - name: Tulemus on logitud
      ansible.builtin.debug:
        msg: "{{ inventory_hostname }} uuendus lõppes"
```

`rescue` lõpus on `fail` meelega. Kui `rescue` õnnestub, loeb Ansible masina terveks ja `serial`-iga läheks uuendus järgmisele masinale edasi. Tagasipööramine pole edu, seega peatad uuenduse ise.

`block`-il on ka teine kasutus: `when`, `become` või `tags` kogu grupile korraga, ilma iga task'i juurde kirjutamata.

Lihtsamatel juhtudel piisab `ignore_errors: true`-st või `failed_when`-ist (millal task loetakse kukkunuks). Ära kasuta `ignore_errors`-it, et viga peita. See sobib ainult siis, kui viga on päriselt oodatud ja järgmine task kontrollib tulemust.

??? question "Kordamisküsimus"

    Miks on `rescue` lõpus `fail`? Mis juhtuks `serial: 1` uuendusega kolmel masinal, kui see puuduks ja uus versioon oleks vigane?

??? info "Loe juurde"

    - [Ansible: blocks](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_blocks.html)
    - [Ansible: vigade käsitlemine](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_error_handling.html)

## 14. Kollektsioonid ja `requirements.yml`

`ansible-core` sisaldab ainult `ansible.builtin` mooduleid. Kõik muu (`ansible.posix.firewalld`, `community.postgresql.postgresql_user`) on **kollektsioonides**, mis paigaldatakse eraldi. K1-s paigaldasid `ansible.posix` käsitsi. Kui kolleeg kloonib sinu repo, ei tea ta, mida paigaldada.

`requirements.yml` repo juures loetleb kõik, mida playbook vajab, koos versioonidega:

```yaml
collections:
  - name: ansible.posix
    version: "1.5.4"
  - name: community.postgresql
    version: ">=3.0.0,<4.0.0"
```

```bash
ansible-galaxy collection install -r requirements.yml
```

Versioon on oluline. Kollektsioonid uuenevad eraldi Ansible'ist ja uus versioon võib vajada uuemat `ansible-core`-i. K1 alguses nägid, et `ansible.posix` uusim versioon ei tööta `ansible-core 2.14`-ga, seepärast on versioon kinni. Täpne versioon annab korratavuse: sama repo töötab täna ja poole aasta pärast samamoodi.

Sama faili saab kasutada ka rollide jaoks (`roles:` plokk) ja K4-s kasutab CI-konveier seda enne lint'i ja teste.

??? info "Loe juurde"

    - [Ansible: kollektsioonide paigaldus](https://docs.ansible.com/ansible/latest/collections_guide/collections_installing.html)

## 15. Kokkuvõte

??? note "Põhimõtted ühes tabelis"

    | Põhimõte | Mida see tähendab |
    |---|---|
    | grupp on ülesanne | `lb`, `db`, `app`; masin võib olla mitmes grupis; `:children` teeb grupi gruppidest |
    | väärtus on ühes kohas | `group_vars` pinule ja gruppidele, `host_vars` erisustele (koos kommentaariga), rolli `defaults` vaikeväärtustele |
    | kitsam võidab laiemat | `host_vars` > `group_vars` > rolli `defaults`; `-e` võidab kõik; kahtluse korral `ansible-inventory --host` |
    | roll ütleb mida, `site.yml` ütleb kus | rollis pole `hosts`-i ega keskkonna väärtusi; play'de järjekord järgib sõltuvusi |
    | handler taaskäivitab ainult muutusel | play lõpus, üks kord; `flush_handlers`, kui on vaja varem |
    | `command` + `register` + `when` + `changed_when` | idempotentne task, kui moodulit pole; eelista moodulit |
    | mall teise masina kohta loeb `hostvars`-ist | `{{ app_port }}` mallis on selle masina väärtus, kelle jaoks mall jookseb |
    | `validate` enne kui fail asendub | vigane seadistus ei jõua masinasse ja teenus töötab edasi |
    | 502 põhjus on vealogis | `13` SELinux, `113` tulemüür, `111` teenus ei kuula; parandus rollis, mitte `setenforce 0` |
    | saladus Vaulti, mitte Giti | kaks faili, `vault_` eesliide, `no_log` |
    | uuendus masinhaaval | `serial`, `max_fail_percentage`, tervisekontroll, `block`/`rescue` tagasipööramiseks |

??? info "Loe juurde"

    - [pikem materjal: muutujad, Jinja2, handlerid, Vault](https://hkhk-automation.github.io/devops/week04/lecture/)
    - [pikem materjal: rollid](https://hkhk-automation.github.io/devops/week11/lecture_ansible_roles/)

*Järgmine: [praktikum](lab.md). Juhendatud osas ehitad pinu neljast rollist, iseseisvas osas paned koormuse vahelduma ja testid, mis juhtub vigadega. Seejärel [kodune õpe ja kodutöö](homework.md).*
