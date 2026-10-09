# K2 · Praktikum: kolmekihiline rakendus

Tänase lõpuks töötab kolmes masinas päris rakendus. vm1-s on nginx koormusjaoturina ja PostgreSQL andmebaasina, vm2 ja vm3 jooksutavad rakendust „Labori seis“. `curl http://vm1/` vastab vaheldumisi vm2-st ja vm3-st ning leht töötab edasi ka siis, kui üks rakendus on maas. Kõik see on kirjas neljas rollis ja ühes `site.yml`-is ning teine jooks ei muuda midagi.

<figure style="max-width:690px;margin:.8em auto" class="lx" markdown="0">
<svg viewBox="0 0 690 336" role="img" aria-label="Kolmekihiline pinu: vm1 nginx ja PostgreSQL, vm2 ja vm3 rakendus" xmlns="http://www.w3.org/2000/svg">
<style>.lx svg{width:100%;height:auto;font-family:var(--md-text-font-family,sans-serif)}.lx .box{fill:var(--md-code-bg-color);stroke:var(--md-default-fg-color--lighter);stroke-width:1.2}.lx .hi{fill:var(--md-primary-fg-color);fill-opacity:.16;stroke:var(--md-primary-fg-color);stroke-width:1.5}.lx .ok{fill:#2e7d32;fill-opacity:.14;stroke:#2e7d32;stroke-width:1.3}.lx .bad{fill:#c62828;fill-opacity:.12;stroke:#c62828;stroke-width:1.3}.lx .t{fill:var(--md-default-fg-color);font-size:15px;font-weight:700}.lx .b{fill:var(--md-default-fg-color);font-size:14.5px;font-weight:600}.lx .s{fill:var(--md-default-fg-color--light);font-size:13px}.lx .c{fill:var(--md-default-fg-color);font-size:12.5px;font-family:var(--md-code-font-family,monospace)}.lx .w{fill:var(--md-accent-fg-color);font-size:13.5px;font-weight:700}.lx .a{stroke:var(--md-default-fg-color--light);stroke-width:1.5;fill:none}.lx .d{stroke-dasharray:4 3}.lx .h{fill:var(--md-default-fg-color--light)}</style>
<defs><marker id="lxa" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path class="h" d="M0,0 L10,5 L0,10 z"/></marker></defs>
<rect class="box" x="10" y="20" width="150" height="50" rx="6"/><text class="b" x="85.0" y="42.0" text-anchor="middle">Brauser / curl</text><text class="s" x="85.0" y="58.0" text-anchor="middle">http://vm1/</text>
<line class="a" x1="160" y1="45" x2="232" y2="45" marker-end="url(#lxa)"/><text class="s" x="196.0" y="39.0" text-anchor="middle">:80</text>
<rect class="hi" x="234" y="10" width="200" height="70" rx="6"/><text class="b" x="334.0" y="42.0" text-anchor="middle">vm1 · lb</text><text class="s" x="334.0" y="58.0" text-anchor="middle">nginx koormusjaotur</text>
<line class="a" x1="334" y1="80" x2="180" y2="150" marker-end="url(#lxa)"/><text class="s" x="257.0" y="112" text-anchor="middle">:8080</text>
<line class="a" x1="334" y1="80" x2="488" y2="150" marker-end="url(#lxa)"/><text class="s" x="411.0" y="112" text-anchor="middle">:8081</text>
<rect class="box" x="80" y="152" width="200" height="54" rx="6"/><text class="b" x="180.0" y="176.0" text-anchor="middle">vm2 · app</text><text class="s" x="180.0" y="192.0" text-anchor="middle">labori-app :8080</text>
<rect class="box" x="408" y="152" width="200" height="54" rx="6"/><text class="b" x="508.0" y="176.0" text-anchor="middle">vm3 · app</text><text class="s" x="508.0" y="192.0" text-anchor="middle">labori-app :8081</text>
<rect class="hi" x="244" y="250" width="200" height="54" rx="6"/><text class="b" x="344.0" y="274.0" text-anchor="middle">vm1 · db</text><text class="s" x="344.0" y="290.0" text-anchor="middle">PostgreSQL :5432</text>
<line class="a" x1="180" y1="206" x2="300" y2="248" marker-end="url(#lxa)"/>
<line class="a" x1="508" y1="206" x2="390" y2="248" marker-end="url(#lxa)"/>
<text class="s" x="344" y="326" text-anchor="middle">vm1 on kahes grupis: lb ja db. Grupp stack sisaldab kõiki kolme.</text>
</svg>
</figure>

Osas A ehitad pinu juhendi järgi kiht korraga: inventar, muutujad, rollid, andmebaas, rakendus, koormusjaotur. Osa A lõpus annab leht 502 ja sa leiad põhjuse. Osas B on juhiseid vähem: leiad, miks koormus ei vaheldu, lased malli vea kinni püüda ja võtad ühe rakenduse maha. Oodatav tulemus ja vihjed on kinnistes plokkides. Tee enne ise ja siis võrdle.

??? abstract "Õpiväljundid"

    Praktikumi lõpuks oskad:

    1. kirjeldada mitmekihilist pinu inventari gruppide ja alamgruppidega;
    2. panna muutujad `group_vars`-i ja `host_vars`-i ning ennustada, milline väärtus masinale jõuab;
    3. jagada playbooki rollideks ja kokku panna `site.yml`-i;
    4. kasutada `import_tasks`-i, handlereid, `loop`-i, `when`-i ja `register`-it;
    5. kirjutada Jinja2 malli, mis loeb teiste masinate andmeid `hostvars`-ist, ja kaitsta seda `validate`-iga;
    6. leida 502 põhjus logidest (SELinux, tulemüür) ja parandada see koodis;
    7. tõendada, et koormus jaguneb ja üks rakendus võib maas olla.

## Enne alustamist

Selleks peaksid olema K1 praktikumi osa B teinud: `ssh vm1`, `ssh vm2` ja `ssh vm3` töötavad vm1-s võtmega ja `ansible.posix` kollektsioon on paigaldatud. Kui midagi on puudu, vaata [Töökeskkond](../keskkond.md) ja K1 praktikumi [osa B](../week01/lab.md#b-iseseisev-osa-kolm-serverit).

Ava juhendaja jagatud Classroom 50 link ja klooni uus repo vm1-sse SSH-ga (**Code** → **SSH**):

```bash
cd ~
git clone git@github.com:hkhk-automation/<sinu-repo>.git
cd <sinu-repo>
ls -R
```

??? success "Oodatav tulemus"

    ```
    .:
    README.md  ULESANNE.md  ansible.cfg  files  host_vars  logid  vead

    ./files:
    app.py

    ./host_vars:
    vm3.yml
    ...
    ```

    `ansible.cfg` on sama, mis K1-s. `files/app.py` on rakendus, mille paigaldad. `host_vars/vm3.yml` on kolleegi jäetud fail, sellest tuleb juttu A2-s. Kaust `vead/` on kodutöö jaoks.

Kontrollnimekiri on su repos **Issues** all: issue Lab 02 · Kolmekihiline rakendus. Märgi ruut, kui osa on tehtud. Kinni? Küsi Discordis või ava issue mallist **Vajan abi**.

??? note "Mis repos lõpuks on"

    ```
    <sinu-repo>/
    ├── ansible.cfg
    ├── inventory.ini
    ├── site.yml
    ├── group_vars/
    │   └── all.yml
    ├── host_vars/
    │   └── vm3.yml
    ├── roles/
    │   ├── common/   tasks/ defaults/
    │   ├── db/       tasks/ handlers/ templates/ defaults/
    │   ├── app/      tasks/ handlers/ templates/ files/ defaults/
    │   └── lb/       tasks/ handlers/ templates/ defaults/
    ├── README.md
    └── logid/
        ├── site_teine_jooks.txt
        ├── vaheldumine.txt
        └── uks_maas.txt
    ```

<figure style="max-width:690px;margin:.8em auto" class="lx" markdown="0">
<svg viewBox="0 0 690 190" role="img" aria-label="Repo ülesehitus: inventar, muutujad, site.yml ja neli rolli" xmlns="http://www.w3.org/2000/svg">
<style>.lx svg{width:100%;height:auto;font-family:var(--md-text-font-family,sans-serif)}.lx .box{fill:var(--md-code-bg-color);stroke:var(--md-default-fg-color--lighter);stroke-width:1.2}.lx .hi{fill:var(--md-primary-fg-color);fill-opacity:.16;stroke:var(--md-primary-fg-color);stroke-width:1.5}.lx .ok{fill:#2e7d32;fill-opacity:.14;stroke:#2e7d32;stroke-width:1.3}.lx .bad{fill:#c62828;fill-opacity:.12;stroke:#c62828;stroke-width:1.3}.lx .t{fill:var(--md-default-fg-color);font-size:15px;font-weight:700}.lx .b{fill:var(--md-default-fg-color);font-size:14.5px;font-weight:600}.lx .s{fill:var(--md-default-fg-color--light);font-size:13px}.lx .c{fill:var(--md-default-fg-color);font-size:12.5px;font-family:var(--md-code-font-family,monospace)}.lx .w{fill:var(--md-accent-fg-color);font-size:13.5px;font-weight:700}.lx .a{stroke:var(--md-default-fg-color--light);stroke-width:1.5;fill:none}.lx .d{stroke-dasharray:4 3}.lx .h{fill:var(--md-default-fg-color--light)}</style>
<defs><marker id="lxa" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path class="h" d="M0,0 L10,5 L0,10 z"/></marker></defs>
<rect class="box" x="4" y="18" width="150" height="54" rx="6"/><text class="b" x="79.0" y="42.0" text-anchor="middle">inventory.ini</text><text class="s" x="79.0" y="58.0" text-anchor="middle">kes kuhu gruppi</text>
<rect class="box" x="170" y="18" width="160" height="54" rx="6"/><text class="b" x="250.0" y="42.0" text-anchor="middle">group_vars/</text><text class="s" x="250.0" y="58.0" text-anchor="middle">väärtused gruppidele</text>
<rect class="box" x="346" y="18" width="150" height="54" rx="6"/><text class="b" x="421.0" y="42.0" text-anchor="middle">host_vars/</text><text class="s" x="421.0" y="58.0" text-anchor="middle">erisus ühele masinale</text>
<rect class="hi" x="512" y="18" width="170" height="54" rx="6"/><text class="b" x="597.0" y="42.0" text-anchor="middle">site.yml</text><text class="s" x="597.0" y="58.0" text-anchor="middle">mis roll kuhu</text>
<line class="a" x1="597" y1="72" x2="597" y2="100" marker-end="url(#lxa)"/>
<rect class="box" x="4" y="102" width="160" height="54" rx="6"/><text class="b" x="84.0" y="126.0" text-anchor="middle">roles/common</text><text class="s" x="84.0" y="142.0" text-anchor="middle">firewalld, paketid</text>
<rect class="box" x="176" y="102" width="160" height="54" rx="6"/><text class="b" x="256.0" y="126.0" text-anchor="middle">roles/db</text><text class="s" x="256.0" y="142.0" text-anchor="middle">PostgreSQL</text>
<rect class="box" x="348" y="102" width="160" height="54" rx="6"/><text class="b" x="428.0" y="126.0" text-anchor="middle">roles/app</text><text class="s" x="428.0" y="142.0" text-anchor="middle">rakendus, systemd</text>
<rect class="box" x="520" y="102" width="162" height="54" rx="6"/><text class="b" x="601.0" y="126.0" text-anchor="middle">roles/lb</text><text class="s" x="601.0" y="142.0" text-anchor="middle">nginx, upstream</text>
<text class="s" x="344" y="180" text-anchor="middle">Iga roll: tasks/ handlers/ templates/ files/ defaults/</text>
</svg>
</figure>

!!! tip "Soovitatav töökäik iga sammu juures"

    1. Enne: `ansible-playbook site.yml --syntax-check` ja `--check --diff` (või `--list-tasks`).
    2. Jooks: `ansible-playbook site.yml`, esimesel korral vajadusel `--limit`.
    3. Pärast: üks ad-hoc käsk, mis kontrollib tulemust masinas endas. Mitte „playbook oli roheline“, vaid „teenus vastab“.

## A · Juhendatud osa

### A1 · Kirjelda pinu inventaris

Selle sammu lõpuks on inventaris kolm rollipõhist gruppi ja üks grupp, mis sisaldab kõiki.

K1-s oli üks grupp `veeb`. Nüüd on masinatel erinevad ülesanded, seega grupeerid need ülesande järgi. vm1 on korraga kahes grupis.

Loo `inventory.ini`:

```ini
[lb]
vm1

[db]
vm1

[app]
vm2
vm3

[stack:children] # (1)!
lb
db
app
```

1. `:children` tähendab, et grupi liikmed on teised grupid. `stack` sisaldab kõiki masinaid, mis on `lb`-, `db`- või `app`-grupis.

Kontrolli enne kasutamist:

```bash
ansible-inventory --graph
ansible stack -m ping
ansible app --list-hosts
```

??? success "Oodatav tulemus"

    ```
    @all:
      |--@ungrouped:
      |--@stack:
      |  |--@lb:
      |  |  |--vm1
      |  |--@db:
      |  |  |--vm1
      |  |--@app:
      |  |  |--vm2
      |  |  |--vm3
    ```

    `ping` annab kolm `pong`-i, mitte neli. vm1 on kahes grupis, aga masinana üks. `--list-hosts` näitab `vm2` ja `vm3`.

Grupid on aadressid. Play ütleb `hosts: app` ja Ansible teab, et see tähendab vm2 ja vm3. Kui homme tuleb neljas rakendusserver, lisad ühe rea `[app]` alla ja kõik, mis sihib `app`-i, hõlmab ka seda.

??? question "Mõtle (vabatahtlik)"

    Kumb on parem nimi grupile: `vm2_vm3` või `app`? Mis juhtub esimese nimega, kui rakendus kolib teistesse masinatesse?

??? info "Loe juurde"

    - [loeng §2](lecture.md#2-inventar-grupid-ja-alamgrupid)
    - [Ansible: inventari grupid](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html#grouping-groups-parent-child-group-relationships)

### A2 · Pane muutujad `group_vars`-i

Selle sammu lõpuks on rakenduse ja andmebaasi seaded ühes failis ning sa oskad igalt masinalt küsida, milline väärtus talle jõuab.

Loo `group_vars/all.yml`:

```yaml
app_nimi: "Labori seis"
app_port: 8080

db_nimi: labor
db_kasutaja: labor
db_parool: "Muuda-mind-2026" # (1)!
db_port: 5432
```

1. Parool avatud tekstina Gitis on halb. Täna jätame selle nii, et näeksid, kus muutuja liigub. Lisaülesandes peidad selle Vaulti.

Ansible loeb kausta `group_vars/` ise: fail `all.yml` kehtib kõigile masinatele, `app.yml` oleks ainult `app`-grupile. Sama kehtib `host_vars/` kohta masina nime järgi.

Enne kui edasi lähed, ennusta. Täida tabel vihikus:

| Masin | `app_port` ennustus | Tegelik |
|---|---|---|
| vm1 | | |
| vm2 | | |
| vm3 | | |

Kontrolli:

```bash
ansible stack -m debug -a "var=app_port"
ansible-inventory --host vm3
cat host_vars/vm3.yml
```

??? success "Oodatav tulemus"

    ```
    vm1 | SUCCESS =>
        app_port: 8080
    vm2 | SUCCESS =>
        app_port: 8080
    vm3 | SUCCESS =>
        app_port: 8081
    ```

    `host_vars/vm3.yml`-is on kolleegi kommentaar: vm3 pordil 8080 on testteenus, seepärast kuulab rakendus seal 8081-l.

`host_vars` võidab `group_vars`-i, sest kitsam kehtib laiema üle. See on täiesti õige seadistus, jäta fail alles. B1-s näed, kus see mõne malli juures kätte maksab.

Proovi ka kõige tugevamat taset:

```bash
ansible vm3 -m debug -a "var=app_port" -e "app_port=9999"
```

`-e` võidab kõik. Seda kasutad ühekordseks katseks, mitte püsiseadistuseks.

??? note "Eelistusjärjekord lühidalt (nõrgemast tugevamani)"

    | Kus | Näide tänases repos |
    |---|---|
    | rolli `defaults/main.yml` | `lb_port: 80` |
    | `group_vars/all.yml` | `app_port: 8080` |
    | `group_vars/<grupp>.yml` | |
    | `host_vars/<masin>.yml` | `app_port: 8081` vm3-le |
    | play `vars:` | |
    | rolli `vars/main.yml` | |
    | `-e` käsurealt | `-e app_port=9999` |

    Täielik nimekiri on 22 taset. Igapäevaselt piisab neist seitsmest.

??? info "Loe juurde"

    - [loeng §3](lecture.md#3-group_vars-host_vars-ja-eelistusjarjekord)
    - [Ansible: muutujate eelistusjärjekord](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_variables.html#understanding-variable-precedence)

### A3 · Ehita rollide skelett ja `site.yml`

Selle sammu lõpuks on repos neli rolli ja `site.yml`, mis rakendab esimese rolli kõigile masinatele.

Loo kaustad:

```bash
mkdir -p roles/common/{tasks,defaults}
mkdir -p roles/db/{tasks,handlers,templates,defaults}
mkdir -p roles/app/{tasks,handlers,templates,files,defaults}
mkdir -p roles/lb/{tasks,handlers,templates,defaults}
mv files/app.py roles/app/files/
tree roles
```

Iga kaust on kokkulepe: Ansible otsib task'e failist `tasks/main.yml`, handlereid failist `handlers/main.yml`, malle kaustast `templates/`, kopeeritavaid faile kaustast `files/` ja vaikeväärtusi failist `defaults/main.yml`. Rollis ei pea kirjutama teekonda, `src: app.py` leitakse ise.

Rolli `common` vaikeväärtused, `roles/common/defaults/main.yml`:

```yaml
common_paketid:
  - firewalld
  - curl
  - python3-libselinux
  - policycoreutils-python-utils
```

Task'id, `roles/common/tasks/main.yml`:

```yaml
- name: Põhipaketid on paigaldatud
  ansible.builtin.package:
    name: "{{ common_paketid }}"
    state: present

- name: firewalld käib
  ansible.builtin.service:
    name: firewalld
    state: started
    enabled: true
```

Rollifailis pole `hosts:`-i ega `become:`-i. Roll ütleb, mida teha. Kuhu, ütleb `site.yml`. Loo `site.yml`:

```yaml
- name: Ühine baas kõigile masinatele
  hosts: stack
  become: true
  roles:
    - common
```

Kontrolli ja jooksuta:

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml --list-tasks
ansible-playbook site.yml
```

??? success "Oodatav tulemus"

    ```
    playbook: site.yml

      play #1 (stack): Ühine baas kõigile masinatele	TAGS: []
        tasks:
          common : Põhipaketid on paigaldatud	TAGS: []
          common : firewalld käib	TAGS: []
    ```

    Jooksul on task'i nime ees rolli nimi: `TASK [common : Põhipaketid on paigaldatud]`. Kolmel masinal `failed=0`. `changed` sõltub sellest, mis K1-st juba olemas oli.

Pärast: `ansible stack -m command -a "systemctl is-active firewalld"` vastab kolm korda `active`.

??? question "Mõtle (vabatahtlik)"

    Miks on `common_paketid` failis `defaults/`, mitte `group_vars/all.yml`-is? Kes võiks tahta seda nimekirja muuta ja kuidas ta seda teeks rolli puutumata?

??? info "Loe juurde"

    - [loeng §5](lecture.md#5-rollid)
    - [Ansible: rollid](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_reuse_roles.html)

### A4 · Paigalda andmebaas rolliga

Selle sammu lõpuks käib vm1-s PostgreSQL, selles on andmebaas `labor` ja ainult vm2 ning vm3 pääsevad sellele võrgust ligi.

Andmebaasi roll on pikem, seega jagad selle kolmeks task-failiks. `roles/db/tasks/main.yml` ainult kogub need kokku:

```yaml
- name: PostgreSQL paigaldus
  ansible.builtin.import_tasks: paigaldus.yml

- name: PostgreSQL seadistus
  ansible.builtin.import_tasks: seadistus.yml

- name: Seaded rakenduvad enne kasutaja loomist
  ansible.builtin.meta: flush_handlers # (1)!

- name: Andmebaas ja kasutaja
  ansible.builtin.import_tasks: kasutaja.yml
```

1. Handlerid käivituvad tavaliselt play lõpus. Siin on vaja, et PostgreSQL oleks uute seadetega taaskäivitatud enne, kui kasutaja luuakse, sest parooli räsi sõltub seadest `password_encryption`.

`roles/db/defaults/main.yml`:

```yaml
db_andmekaust: /var/lib/pgsql/data
```

**Samm 1.** `roles/db/tasks/paigaldus.yml`:

```yaml
- name: PostgreSQL server on paigaldatud
  ansible.builtin.package:
    name:
      - postgresql-server
      - postgresql
    state: present

- name: Andmekaust on initsialiseeritud
  ansible.builtin.command: postgresql-setup --initdb
  args:
    creates: "{{ db_andmekaust }}/PG_VERSION" # (1)!

- name: PostgreSQL käib
  ansible.builtin.service:
    name: postgresql
    state: started
    enabled: true
```

1. Sama võte, mis K1 A5-s: kui fail on olemas, on andmekaust juba loodud ja käsku ei käivitata.

**Samm 2.** `roles/db/tasks/seadistus.yml`. Siin on kaks uut asja: `loop` kahe seade jaoks ühe task'iga ja mall, mis loeb teiste masinate IP-sid.

```yaml
- name: postgresql.conf seaded on paigas
  ansible.builtin.lineinfile:
    path: "{{ db_andmekaust }}/postgresql.conf"
    regexp: "^#?{{ item.nimi }}\\s*="
    line: "{{ item.nimi }} = '{{ item.vaartus }}'"
  loop: # (1)!
    - { nimi: listen_addresses, vaartus: "*" }
    - { nimi: password_encryption, vaartus: scram-sha-256 }
  notify: PostgreSQL taaskäivitub

- name: Rakendusserverid pääsevad andmebaasi
  ansible.builtin.template:
    src: pg_hba.conf.j2
    dest: "{{ db_andmekaust }}/pg_hba.conf"
    owner: postgres
    group: postgres
    mode: "0600"
  notify: PostgreSQL laeb seaded uuesti

- name: Andmebaasi port on tulemüüris avatud
  ansible.posix.firewalld:
    port: "{{ db_port }}/tcp"
    permanent: true
    immediate: true
    state: enabled
```

1. Task jookseb iga nimekirja elemendi kohta üks kord, element on muutujas `item`. `regexp` leiab rea ka siis, kui see on välja kommenteeritud (`#listen_addresses = 'localhost'`).

Mall `roles/db/templates/pg_hba.conf.j2`:

```jinja
# {{ ansible_managed }}
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             all                                     peer
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             ::1/128                 scram-sha-256
{% for h in groups['app'] %}
host    {{ db_nimi }}           {{ db_kasutaja }}           {{ hostvars[h].ansible_default_ipv4.address }}/32         scram-sha-256
{% endfor %}
```

Mall jookseb vm1 jaoks, aga kirjutab faili vm2 ja vm3 IP-d. `groups['app']` on nimekiri `['vm2', 'vm3']`, `hostvars['vm2']` on kõik, mida Ansible vm2 kohta teab, ka tema faktid. Faktid on olemas, sest esimene play (`hosts: stack`) kogus need kõigist kolmest.

Handlerid, `roles/db/handlers/main.yml`:

```yaml
- name: PostgreSQL taaskäivitub
  ansible.builtin.service:
    name: postgresql
    state: restarted

- name: PostgreSQL laeb seaded uuesti
  ansible.builtin.service:
    name: postgresql
    state: reloaded
```

`notify` peab olema täpselt sama tekst mis handleri `name`. `listen_addresses` vajab taaskäivitust, `pg_hba.conf` muutusele piisab uuesti laadimisest.

**Samm 3.** `roles/db/tasks/kasutaja.yml`. PostgreSQL-i kasutajate jaoks on olemas kollektsioon `community.postgresql`, aga meie masinates seda pole. Seepärast teed sama asja `command`-iga ja teed selle ise idempotentseks: esmalt küsid, kas kasutaja on olemas, ja lood ta ainult siis, kui pole.

```yaml
- name: Kontrolli, kas andmebaasi kasutaja on olemas
  ansible.builtin.command: psql -tAc "SELECT 1 FROM pg_roles WHERE rolname='{{ db_kasutaja }}'"
  become_user: postgres # (1)!
  register: db_roll # (2)!
  changed_when: false # (3)!
  check_mode: false # (4)!

- name: Andmebaasi kasutaja on olemas
  ansible.builtin.command: psql -c "CREATE ROLE {{ db_kasutaja }} LOGIN PASSWORD '{{ db_parool }}'"
  become_user: postgres
  when: db_roll.stdout != "1" # (5)!
  no_log: true # (6)!

- name: Kontrolli, kas andmebaas on olemas
  ansible.builtin.command: psql -tAc "SELECT 1 FROM pg_database WHERE datname='{{ db_nimi }}'"
  become_user: postgres
  register: db_baas
  changed_when: false
  check_mode: false

- name: Andmebaas on olemas
  ansible.builtin.command: createdb -O {{ db_kasutaja }} {{ db_nimi }}
  become_user: postgres
  when: db_baas.stdout != "1"
```

1. Käsk jookseb kasutajana `postgres`, kellel on andmebaasis kõik õigused. `become: true` tuleb play'st.
2. Käsu tulemus (`stdout`, `rc` jm) salvestatakse muutujasse `db_roll`.
3. Päring ei muuda midagi, seega ei tohi see olla `changed`. Muidu poleks teine jooks kunagi `changed=0`.
4. Päring jookseb ka `--check` all. Ilma selleta jäetaks see vahele ja järgmine task ei teaks, mida otsustada.
5. `psql -tA` vastab `1`, kui rida on olemas, ja tühja reaga, kui ei ole.
6. Parool ei satu väljundisse ega logisse.

Lisa `site.yml`-i teine play:

```yaml
- name: Andmebaas
  hosts: db
  become: true
  roles:
    - db
```

Kontrolli ja jooksuta:

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml
```

??? success "Oodatav tulemus"

    ```
    TASK [db : Andmekaust on initsialiseeritud] *******************
    changed: [vm1]
    ...
    TASK [db : postgresql.conf seaded on paigas] ******************
    changed: [vm1] => (item={'nimi': 'listen_addresses', 'vaartus': '*'})
    changed: [vm1] => (item={'nimi': 'password_encryption', 'vaartus': 'scram-sha-256'})
    ...
    RUNNING HANDLER [db : PostgreSQL taaskäivitub] ****************
    changed: [vm1]
    ...
    TASK [db : Andmebaasi kasutaja on olemas] *********************
    changed: [vm1]
    ```

Pärast: kontrolli masinas, mitte playbooki väljundis.

```bash
ansible db -m wait_for -a "port=5432 timeout=5"
ssh -t vm1 "sudo -u postgres psql -c '\l labor'"
ssh -t vm1 "sudo cat /var/lib/pgsql/data/pg_hba.conf"
```

??? success "Oodatav tulemus"

    `wait_for` vastab `SUCCESS`. `\l labor` näitab andmebaasi, mille omanik on `labor`. `pg_hba.conf`-i lõpus on kaks rida, üks vm2 ja üks vm3 IP-ga.

??? tip "Kui `psql` annab `could not change directory to \"/home/...\": Permission denied`"

    See on hoiatus, mitte viga. Kasutaja `postgres` ei pääse sinu kodukausta, kust käsk käivitati. Käsk ise töötab.

??? question "Mõtle (vabatahtlik)"

    Jooksuta `ansible-playbook site.yml` uuesti. Mitu korda on `Andmebaasi kasutaja on olemas` nüüd `skipped`? Mis juhtuks, kui muudaksid `group_vars/all.yml`-is parooli: kas andmebaasis parool muutuks?

??? info "Loe juurde"

    - [loeng §6–§8](lecture.md#6-task-failid-import-ja-include)
    - [Ansible: handlerid](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html)
    - [Ansible: tingimused](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_conditionals.html)
    - [PostgreSQL: pg_hba.conf](https://www.postgresql.org/docs/13/auth-pg-hba-conf.html)

### A5 · Paigalda rakendus vm2-le ja vm3-le

Selle sammu lõpuks käib vm2-s ja vm3-s systemd teenus `labori-app`, mis kirjutab iga külastuse vm1 andmebaasi.

Vaata enne rakendus läbi: `less roles/app/files/app.py`. See on umbes 100 rida Pythonit. Seaded tulevad keskkonnamuutujatest (`DB_HOST`, `APP_PORT` jt), aadressil `/health` vastab rakendus `ok`, kui andmebaas vastab.

`roles/app/defaults/main.yml`:

```yaml
app_kasutaja: labori
app_kaust: /opt/labori-app
app_versioon: "1.0"
```

Seadete mall `roles/app/templates/labori-app.env.j2`:

```jinja
# {{ ansible_managed }}
APP_NIMI="{{ app_nimi }}"
APP_VERSIOON="{{ app_versioon }}"
APP_PORT={{ app_port }}
DB_HOST={{ hostvars[groups['db'][0]].ansible_default_ipv4.address }}
DB_PORT={{ db_port }}
DB_NIMI={{ db_nimi }}
DB_KASUTAJA={{ db_kasutaja }}
DB_PAROOL={{ db_parool }}
```

`groups['db'][0]` on `db`-grupi esimene masin. Nii ei pea andmebaasi IP-d kuhugi käsitsi kirjutama.

systemd teenuse mall `roles/app/templates/labori-app.service.j2`:

```jinja
# {{ ansible_managed }}
[Unit]
Description={{ app_nimi }}
After=network-online.target
Wants=network-online.target

[Service]
User={{ app_kasutaja }}
EnvironmentFile=/etc/labori-app.env
ExecStart=/usr/bin/python3 {{ app_kaust }}/app.py
Restart=on-failure
RestartSec=3

[Install]
WantedBy=multi-user.target
```

Task'id, `roles/app/tasks/main.yml`:

```yaml
- name: Rakenduse kasutaja on olemas
  ansible.builtin.user:
    name: "{{ app_kasutaja }}"
    system: true
    shell: /sbin/nologin
    create_home: false

- name: Andmebaasi draiver on paigaldatud
  ansible.builtin.package:
    name: python3-psycopg2
    state: present

- name: Rakenduse kaust on olemas
  ansible.builtin.file:
    path: "{{ app_kaust }}"
    state: directory
    mode: "0755"

- name: Rakenduse kood on paigas
  ansible.builtin.copy:
    src: app.py
    dest: "{{ app_kaust }}/app.py"
    mode: "0755"
  notify: Rakendus taaskäivitub

- name: Rakenduse seaded on paigas
  ansible.builtin.template:
    src: labori-app.env.j2
    dest: /etc/labori-app.env
    mode: "0600" # (1)!
  notify: Rakendus taaskäivitub

- name: systemd teenus on kirjeldatud
  ansible.builtin.template:
    src: labori-app.service.j2
    dest: /etc/systemd/system/labori-app.service
    mode: "0644"
  notify: Rakendus taaskäivitub

- name: Rakendus käib
  ansible.builtin.systemd:
    name: labori-app
    state: started
    enabled: true
    daemon_reload: true # (2)!
```

1. Failis on parool, seega loeb seda ainult root. systemd loeb faili root'ina enne, kui rakendus kasutajana `labori` käivitub.
2. systemd loeb uue või muudetud `.service` faili sisse alles pärast `daemon-reload`-i.

Handler, `roles/app/handlers/main.yml`:

```yaml
- name: Rakendus taaskäivitub
  ansible.builtin.systemd:
    name: labori-app
    state: restarted
    daemon_reload: true
```

Kolm task'i teavitavad sama handlerit. Kui muutuvad kõik kolm, taaskäivitub rakendus ikkagi ainult üks kord, play lõpus.

Lisa `site.yml`-i kolmas play (`hosts: app`, roll `app`) samal kujul nagu eelmised. Jooksuta:

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml
```

Pärast: küsi rakenduselt endalt, kas see näeb andmebaasi. Ad-hoc käsus saad kasutada muutujat, see arvutatakse iga masina jaoks eraldi:

```bash
ansible app -m uri -a "url=http://localhost:{{ app_port }}/health return_content=true"
```

??? success "Oodatav tulemus"

    ```
    vm2 | SUCCESS =>
        content: |-
            ok vm2
        status: 200
        url: http://localhost:8080/health
    vm3 | SUCCESS =>
        content: |-
            ok vm3
        status: 200
        url: http://localhost:8081/health
    ```

??? tip "Kui `/health` vastab 503 või `uri` kukub"

    Vaata rakenduse logi: `ansible app -b -m command -a "journalctl -u labori-app -n 15 --no-pager"`.

    - `no pg_hba.conf entry for host`: vm1 `pg_hba.conf`-is pole selle masina IP-d. Kas faktid olid kogutud? Jooksuta `site.yml` tervikuna, ilma `--limit`-ita.
    - `password authentication failed`: parool `group_vars/all.yml`-is ja andmebaasis erinevad. Kasutaja loodi varem teise parooliga.
    - `Connection refused` või `timeout`: vm1 tulemüür (`5432/tcp`) või `listen_addresses`.
    - `Address already in use`: port on teise teenuse all (`ss -tlnp`).

??? question "Mõtle (vabatahtlik)"

    Muuda `group_vars/all.yml`-is `app_nimi` ja jooksuta. Mitu task'i on `changed` ja mitu korda rakendus taaskäivitub? Taasta nimi ja jooksuta uuesti.

??? info "Loe juurde"

    - [loeng §7–§9](lecture.md#7-handlerid)
    - [Ansible: template moodul](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html)
    - [systemd: EnvironmentFile](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html#EnvironmentFile=)

### A6 · Pane koormusjaotur ette

Selle sammu lõpuks saadab vm1 nginx päringud edasi rakendusserveritele ja kogu nginx-i seadistus tuleb mallist.

`roles/lb/defaults/main.yml`:

```yaml
lb_port: 80
```

Mall `roles/lb/templates/nginx.conf.j2` asendab terve `/etc/nginx/nginx.conf`-i. K1 avaleht vm1-s kaob, nii peabki olema.

```jinja
# {{ ansible_managed }}
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log notice;
pid /run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    log_format lb '$remote_addr "$request" $status -> $upstream_addr $upstream_status'; # (1)!
    access_log /var/log/nginx/access.log lb;

    upstream rakendus {
{% for h in groups['app'] %}
        server {{ hostvars[h].ansible_default_ipv4.address }}:{{ app_port }} max_fails=1 fail_timeout=10s;  # {{ h }}
{% endfor %}
    }

    server {
        listen {{ lb_port }} default_server;
        server_name _;

        location / {
            proxy_pass http://rakendus;
            proxy_connect_timeout 2s;
            proxy_next_upstream error timeout http_502 http_503; # (2)!
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

1. Igal logireal on näha, millisele rakendusserverile päring läks ja mis sealt vastati. B1-s on sellest abi.
2. Kui üks rakendusserver ei vasta, proovib nginx sama päringut järgmisega. Kasutaja viga ei näe.

Task'id, `roles/lb/tasks/main.yml`:

```yaml
- name: nginx on paigaldatud
  ansible.builtin.package:
    name: nginx
    state: present

- name: Koormusjaoturi seadistus on paigas
  ansible.builtin.template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    mode: "0644"
    validate: nginx -t -c %s # (1)!
  notify: nginx laeb seaded uuesti

- name: nginx käib
  ansible.builtin.service:
    name: nginx
    state: started
    enabled: true

- name: HTTP on tulemüüris avatud
  ansible.posix.firewalld:
    service: http
    permanent: true
    immediate: true
    state: enabled
```

1. Enne kui fail asendatakse, kontrollib nginx uut versiooni (`%s` on ajutise faili tee). Kui kontroll kukub, jääb vana fail paigale ja nginx töötab edasi. B2-s proovid seda.

Handler, `roles/lb/handlers/main.yml`:

```yaml
- name: nginx laeb seaded uuesti
  ansible.builtin.service:
    name: nginx
    state: reloaded
```

Lisa `site.yml`-i neljas play (`hosts: lb`, roll `lb`). Enne jooksu vaata, mida mall kirjutaks:

```bash
ansible-playbook site.yml --check --diff --limit lb
```

??? tip "Kui `--limit lb` annab `'dict object' has no attribute 'ansible_default_ipv4'`"

    `--limit lb` piirab ka faktide kogumist. vm2 ja vm3 fakte ei kogutud, seega mall ei tea nende IP-d. Jooksuta ilma `--limit`-ita: `ansible-playbook site.yml --check --diff`.

Jooksuta päriselt ja proovi:

```bash
ansible-playbook site.yml
curl -s -i http://localhost/ | head -n 1
```

??? success "Oodatav tulemus"

    ```
    HTTP/1.1 502 Bad Gateway
    ```

    Playbook oli roheline, aga leht ei tööta. Seda parandad järgmises sammus.

??? info "Loe juurde"

    - [loeng §9](lecture.md#9-jinja2-mallid-ja-hostvars)
    - [nginx: upstream](https://nginx.org/en/docs/http/ngx_http_upstream_module.html)

### A7 · Leia, miks leht annab 502

Selle sammu lõpuks vastab `curl http://vm1/` rakenduse lehega ja mõlemad parandused on rollides, mitte käsitsi tehtud.

502 tähendab, et nginx ise töötab, aga ei saanud rakendusserverilt vastust. Põhjus on nginx-i vealogis. Vaata seda vm1-s:

```bash
sudo tail -n 5 /var/log/nginx/error.log
```

??? success "Oodatav tulemus"

    ```
    ... connect() to 10.x.x.12:8080 failed (13: Permission denied) while connecting to upstream ...
    ```

    `Permission denied` ühenduse loomisel, kuigi nginx jookseb ja port on õige.

Kui root-õigustega teenus saab `Permission denied`, on põhjus AlmaLinuxis tavaliselt SELinux. Kontrolli:

```bash
getenforce
sudo ausearch -m avc -ts recent | tail -n 4
getsebool httpd_can_network_connect
```

??? success "Oodatav tulemus"

    ```
    Enforcing
    type=AVC msg=audit(...): avc:  denied  { name_connect } for  pid=... comm="nginx" dest=8080 scontext=system_u:system_r:httpd_t:s0 ...
    httpd_can_network_connect --> off
    ```

SELinux lubab veebiserveril (`httpd_t`) vaikimisi ainult vastu võtta ühendusi, mitte ise teistele masinatele ühenduda. Selleks on lüliti (boolean). Ära lülita SELinuxit välja. Lülita sisse just see üks luba, ja tee seda rollis.

Lisa `roles/lb/tasks/main.yml`-i enne malli task'i:

```yaml
- name: nginx tohib ühenduda rakendusserveritega (SELinux)
  ansible.posix.seboolean:
    name: httpd_can_network_connect
    state: true
    persistent: true # (1)!
```

1. Kehtib ka pärast taaskäivitust. Ilma selleta kaoks luba esimesel reboot'il.

Jooksuta ja proovi uuesti:

```bash
ansible-playbook site.yml
curl -s -i http://localhost/ | head -n 1
sudo tail -n 3 /var/log/nginx/error.log
```

??? success "Oodatav tulemus"

    Ikka `502`, aga vealogis on teine viga:

    ```
    ... connect() to 10.x.x.12:8080 failed (113: No route to host) while connecting to upstream ...
    ```

Nüüd jõuab ühendus võrku, aga teine pool keeldub. `No route to host` on see, mida firewalld vastab suletud pordile. A5-s kontrollisid rakendust `localhost`-ist, mis tulemüürist läbi ei käi. Kontrolli:

```bash
ansible app -b -m command -a "firewall-cmd --list-ports"
```

Lisa `roles/app/tasks/main.yml`-i lõppu:

```yaml
- name: Rakenduse port on tulemüüris avatud
  ansible.posix.firewalld:
    port: "{{ app_port }}/tcp"
    permanent: true
    immediate: true
    state: enabled
```

```bash
ansible-playbook site.yml
for i in 1 2 3 4; do curl -s http://localhost/; echo; done
```

??? success "Oodatav tulemus"

    ```
    Labori seis 1.0
    Vastas: vm2
    Külastused: vm2=1

    Labori seis 1.0
    Vastas: vm2
    Külastused: vm2=2
    ...
    ```

    Leht töötab ja loendur kasvab, nii et andmebaas on päriselt kasutusel. Vastab ainult vm2. Selle põhjuse leiad B1-s.

Kaks kihti, kaks eraldi kontrolli. SELinux otsustab, kas protsess tohib ühendust luua. Tulemüür otsustab, kas pakett pääseb masinasse. Mõlemad annavad kasutajale sama 502. Erinevuse näeb ainult logist: `13: Permission denied` on oma masina SELinux, `113: No route to host` on teise masina tulemüür.

??? question "Mõtle (vabatahtlik)"

    Kolleeg ütleb, et kiirem on `setenforce 0` ja `systemctl stop firewalld`. Mis juhtub järgmisel reboot'il ja mis juhtub järgmisel `site.yml` jooksul?

??? info "Loe juurde"

    - [loeng §10](lecture.md#10-selinux-ja-tulemuur-teenuste-vahel)
    - [Red Hat: SELinux booleans](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/9/html/using_selinux/configuring-selinux-for-applications-and-services-with-non-standard-configurations_using-selinux)
    - [ansible.posix.seboolean](https://docs.ansible.com/ansible/latest/collections/ansible/posix/seboolean_module.html)

### A8 · Tõenda, et teine jooks ei muuda midagi

Selle sammu lõpuks on failis `logid/site_teine_jooks.txt` tõend, et kogu pinu on soovitud olekus.

```bash
ansible-playbook site.yml | tee logid/site_teine_jooks.txt
```

??? success "Oodatav tulemus"

    ```
    PLAY RECAP ****************************************************
    vm1 : ok=19  changed=0  unreachable=0  failed=0  skipped=2
    vm2 : ok=12  changed=0  unreachable=0  failed=0  skipped=0
    vm3 : ok=12  changed=0  unreachable=0  failed=0  skipped=0
    ```

    `skipped=2` vm1-l on kaks `when`-iga task'i: kasutaja ja andmebaas on juba olemas. Täpsed `ok` arvud võivad erineda.

??? tip "Kui mõni task on igal jooksul `changed`"

    Vaata, milline. Tavalised põhjused täna: kontrollpäringul puudub `changed_when: false`, või `lineinfile` `regexp` ei leia oma rida ja lisab igal korral uue (`grep listen_addresses /var/lib/pgsql/data/postgresql.conf`).

Commit'i see seis enne osa B-d:

```bash
git add .
git commit -m "K2: pinu rollidega, teine jooks changed=0"
git push
```

## B · Iseseisev osa

Pinu töötab, aga mitte nii, nagu lubati: koormus ei jagune ja keegi pole kontrollinud, mis juhtub vigase malli või maas oleva rakendusega. Osas B leiad ja parandad. Sammud on antud, lahenduse leiad ise.

Piirangud kogu B-osas:

- Iga parandus on rollis, mallis või muutujates. Käsitsi tehtud muudatus masinas läheb järgmise jooksuga kaotsi.
- `host_vars/vm3.yml` jääb alles: vm3 rakendus kuulab ka lõpus pordil 8081.
- Pärast igat parandust: `--check --diff`, siis jooks, siis ad-hoc kontroll.

### B1 · Pane koormus vahelduma

Selle sammu lõpuks vastavad päringud vaheldumisi vm2-st ja vm3-st ning tõend on failis `logid/vaheldumine.txt`.

Praegu vastab ainult vm2. Leia põhjus ja paranda see koodis. Kontroll:

```bash
for i in $(seq 1 6); do curl -s http://localhost/ | grep Vastas; done | tee logid/vaheldumine.txt
```

??? success "Oodatav tulemus"

    ```
    Vastas: vm2
    Vastas: vm3
    Vastas: vm2
    Vastas: vm3
    Vastas: vm2
    Vastas: vm3
    ```

??? tip "Vihje 1: kust otsida"

    nginx-i juurdepääsulogis on iga päringu kohta kirjas, kuhu see läks: `sudo tail -n 6 /var/log/nginx/access.log`. Vaata `->` järel olevaid aadresse ja porte. Siis vaata, mis on mallist kirjutatud: `grep server /etc/nginx/nginx.conf`.

??? tip "Vihje 2: kelle muutuja"

    Mall jookseb vm1 jaoks. `{{ app_port }}` mallis on vm1 väärtus. Mis on vm1 `app_port`? Mis on vm3 oma? Kuidas küsida mallis vm3 muutujat?

    ??? example "Näide, kui ikka kinni"
        `{{ hostvars[h].app_port }}`. Sama nagu IP-ga real eespool.

??? question "Mõtle (vabatahtlik)"

    Miks ei näinud kasutaja ühtki viga, kuigi pool päringuid läks vale pordi peale? Kas see on hea või halb?

### B2 · Lase vigasel mallil kinni jääda

Selle sammu lõpuks oled näinud, et vigane nginx-i seadistus ei jõua masinasse ja leht töötab vea ajal edasi.

1. Lisa malli `location /` plokki rida `proxy_read_timeout 5s` ilma semikoolonita.
2. Jooksuta `ansible-playbook site.yml`.
3. Samal ajal või kohe pärast: `curl -s http://localhost/ | grep Vastas`.
4. Paranda mall ja jooksuta uuesti.

??? success "Oodatav tulemus"

    ```
    TASK [lb : Koormusjaoturi seadistus on paigas] ****************
    fatal: [vm1]: FAILED! => changed=false
      ...
      msg: failed to validate
      stderr: |-
        nginx: [emerg] directive "proxy_read_timeout" is not terminated by ";" in /home/.../.ansible/tmp/.../source:...
    ```

    `curl` vastab edasi, `/etc/nginx/nginx.conf` on vana. Handler ei käivitunud, sest faili ei muudetud.

Ilma `validate`-ita oleks vigane fail kirjutatud ja handler oleks teinud `reload`-i. `reload` vigase failiga ebaõnnestub ja vana protsess töötab edasi, aga järgmine `restart` või reboot jätaks nginx-i maha.

??? question "Mõtle (vabatahtlik)"

    Milliste teiste failide juures tänases repos oleks `validate` kasulik? Mis käsuga kontrolliksid `sshd_config`-i (K1 kodutöö H2)?

### B3 · Võta üks rakendus maha

Selle sammu lõpuks on tõend, et ühe rakenduse kadumisel töötab leht edasi, ja `site.yml` toob rakenduse tagasi.

```bash
ssh -t vm2 "sudo systemctl stop labori-app"
for i in $(seq 1 6); do curl -s -o /dev/null -w "%{http_code} " http://localhost/; curl -s http://localhost/ | grep Vastas; done | tee logid/uks_maas.txt
```

??? success "Oodatav tulemus"

    Kõik read algavad `200`-ga ja vastab ainult `vm3`. Ühtegi `502`-te pole.

Nüüd ennusta, enne kui jooksutad: mitu `changed`-i ja millises masinas?

```bash
ansible-playbook site.yml --check
ansible-playbook site.yml
for i in $(seq 1 4); do curl -s http://localhost/ | grep Vastas; done
```

??? success "Oodatav tulemus"

    `vm2` näitab `changed=1` (task `Rakendus käib`), teised `changed=0`. Pärast vastavad jälle vm2 ja vm3 vaheldumisi.

Peatatud teenus on drift nagu K1-s. Playbook ei pea teadma, mis katki läks.

??? tip "Vihje: kui `502` siiski tuleb"

    Kontrolli mallis `proxy_next_upstream` rida ja `max_fails`. Ilma nendeta saab kasutaja vea, kuni nginx märkab, et server on maas.

### B4 · Lisa `site.yml`-i lõppu kontroll

Selle sammu lõpuks kukub `site.yml` ise punaseks, kui leht läbi koormusjaoturi ei vasta, isegi kui kõik task'id olid rohelised.

A6 näitas, et roheline playbook ja töötav leht pole sama asi. Lisa `site.yml`-i viimane play `hosts: lb`, mis:

- teeb päringu `http://localhost/` ja salvestab vastuse;
- kordab päringut kuni 5 korda 2-sekundilise vahega, kui vastus pole 200;
- kontrollib, et vastuses on `Vastas:` ja `Külastused:`, ja annab muidu arusaadava veateate;
- ei ole kunagi `changed` ja jookseb ka `--check` all.

??? tip "Vihje: moodulid ja parameetrid"

    `ansible.builtin.uri` (`return_content`), `register`, `until`/`retries`/`delay`, `changed_when`, `check_mode`; kontroll `ansible.builtin.assert` (`that`, `fail_msg`). Faktid pole siin vaja: `gather_facts: false`.

    ??? example "Näide, kui ikka kinni"
        ```yaml
        - name: "Kontroll: leht vastab läbi koormusjaoturi"
          hosts: lb
          gather_facts: false
          tasks:
            - name: Avaleht vastab
              ansible.builtin.uri:
                url: "http://localhost/"
                return_content: true
              register: vastus
              check_mode: false
              changed_when: false
              retries: 5
              delay: 2
              until: vastus.status == 200

            - name: Vastuses on rakenduse masin ja andmebaasi loendur
              ansible.builtin.assert:
                that:
                  - "'Vastas:' in vastus.content"
                  - "'Külastused:' in vastus.content"
                fail_msg: "Leht vastas, aga mitte rakendus: {{ vastus.content | truncate(200) }}"
        ```

Proovi, et kontroll päriselt kukub: peata mõlemas masinas `labori-app` ja jooksuta ainult viimane play (`--start-at-task "Avaleht vastab"`). Siis jooksuta tervet `site.yml`-i ja vaata, et kõik paraneb ja kontroll läbib.

Lõpuks salvesta uuesti teise jooksu tõend:

```bash
ansible-playbook site.yml | tee logid/site_teine_jooks.txt
```

## Lisaülesanne · Peida parool Vaulti

Selle ülesande lõpuks ei ole andmebaasi parool Gitis loetaval kujul, aga `site.yml` töötab nagu enne.

1. Tõsta parool eraldi faili `group_vars/all/vault.yml` muutujana `vault_db_parool` ja krüpti fail: `ansible-vault encrypt group_vars/all/vault.yml`.
2. Tõsta `group_vars/all.yml` sama kausta `group_vars/all/main.yml`-iks ja kirjuta sinna `db_parool: "{{ vault_db_parool }}"`.
3. Jooksuta `ansible-playbook site.yml --ask-vault-pass`. Teine jooks `changed=0`.
4. Vaata `git diff --stat` ja `cat group_vars/all/vault.yml`: failis on ainult `$ANSIBLE_VAULT;1.1;AES256` ja numbrid.

??? tip "Vihje: miks kaks faili"

    Kui `db_parool` on otse krüptitud failis, ei leia keegi `grep db_parool`-iga üles, kust väärtus tuleb. Kaudne nimi (`vault_` ees) hoiab muutujate nimekirja loetavana ja ainult väärtuse saladuses.

??? tip "Vihje: Vaulti parool"

    Vaulti parooli ei pane reposse. Kui ei taha iga kord trükkida, pane see faili väljaspool repot (`~/.vault_pass`, õigused `600`) ja `ansible.cfg`-s `vault_password_file = ~/.vault_pass`.

??? info "Loe juurde"

    - [loeng §11](lecture.md#11-saladused-ja-ansible-vault)
    - [Ansible Vault](https://docs.ansible.com/ansible/latest/vault_guide/index.html)

## Esitamine

### README

Täida repos olev `README.md` mall: asenda nurksulgudes kohad oma tööga ja kustuta ülemine kast. `ULESANNE.md` jääb muutmata. Peegeldusküsimustele vasta lühidalt, üks-kaks lauset piisab. Automaatne kontroll K1 kukub, kui README-s on veel täitmata kohti.

### Commit ja push

```bash
git status
git add .
git commit -m "K2: koormus vaheldub, kontroll site.yml-is"
git push
```

`git status` enne `add`-i näitab, mis faile lisad. Kui tegid lisaülesande, kontrolli, et `group_vars/all/vault.yml` on krüptitud enne, kui see Giti läheb.

### Automaatne kontroll

Ava GitHubis oma repo → **Actions** → viimane **Autograde**:

| Kontroll | Mida vaatab |
|---|---|
| K1 | `inventory.ini`, `site.yml`, `group_vars/all.yml` või `group_vars/all/`, nelja rolli `tasks/main.yml`, `logid/site_teine_jooks.txt` olemas; `README.md` mall täidetud |
| K2 | `site.yml` süntaks |
| K3 | inventaris grupid `lb`, `db`, `app`; `site.yml` kasutab rolle; `db`-rollis `import_tasks`; handlerid `db`-, `app`- ja `lb`-rollis |
| K4 | `lb`-rolli mallis `hostvars` ja `app_port`; `validate`; `seboolean`; `app`-rollis `firewalld` |
| K5 | `site_teine_jooks.txt`: kolm masinat, kõigil `changed=0`; `vaheldumine.txt`: nii vm2 kui vm3; `uks_maas.txt`: ei ühtegi `502`-te |

Kodutöö kontrollid kukuvad seni, kuni kodutöö on tegemata. Loeb punktisumma, mitte värv.

## Kodutöö

Kodune õpe ja kodutöö on eraldi lehel: [K2 · Kodune õpe ja kodutöö](homework.md). Samasse reposse, tähtaeg Classroom 50-s.

## Veaotsing

??? info "Veaotsingu tabel: tüüpilised vead ja lahendused"

    Kontrolli järjekorras: loe veateade algusest lõpuni, korda käsku `-v`-ga, vaata teenuse logi sihtmasinas.

    | Probleem | Põhjus | Lahendus |
    |---|---|---|
    | `the role 'db' was not found` | rolli kaust on vales kohas või vale nimega | `roles/db/tasks/main.yml` repo juurest; `ls roles` |
    | `Could not find or access 'app.py'` | fail pole rolli `files/` kaustas | `mv files/app.py roles/app/files/` |
    | `Could not find or access 'nginx.conf.j2'` | mall pole rolli `templates/` kaustas | vaata `ls roles/lb/templates` |
    | `The requested handler '...' was not found` | `notify` ja handleri `name` erinevad | kopeeri nimi täpselt, suurtähed loevad |
    | `'dict object' has no attribute 'ansible_default_ipv4'` | teise masina fakte pole kogutud (`--limit`) | jooksuta ilma `--limit`-ita |
    | `'app_port' is undefined` | muutuja puudub `group_vars`-ist või failinimi on vale | `ansible-inventory --host vm2` |
    | `group_vars` ei rakendu | kaust pole repo juures või fail on `.yaml`/`.yml` asemel muu | `ls group_vars`, `ansible stack -m debug -a var=db_nimi` |
    | `postgresql-setup: ... Data directory is not empty` | andmekaust on juba olemas | `creates` puudub või vale tee |
    | task `postgresql.conf seaded` igal jooksul `changed` | `regexp` ei leia rida | `regexp: "^#?listen_addresses\\s*="` |
    | `Andmebaasi kasutaja on olemas` igal jooksul `changed` või `failed` (`already exists`) | `when` vaatab vale muutujat | `debug: var=db_roll` |
    | `--check` kukub task'is `Andmebaasi kasutaja on olemas` | kontrollpäringul puudub `check_mode: false` | lisa see |
    | `/health` 503, logis `no pg_hba.conf entry` | masina IP pole `pg_hba.conf`-is | jooksuta `site.yml` tervikuna |
    | `/health` 503, logis `password authentication failed` | parool on andmebaasis teine | `sudo -u postgres psql -c "ALTER ROLE labor PASSWORD '...'"` või tühjenda andmebaas |
    | rakendus ei käivitu, `ModuleNotFoundError: psycopg2` | pakett puudu | `python3-psycopg2` app-rollis |
    | `502`, logis `13: Permission denied` | SELinux | A7, `httpd_can_network_connect` |
    | `502`, logis `113: No route to host` | rakendusserveri tulemüür | A7, `app_port` firewalld-s |
    | `502`, logis `111: Connection refused` | rakendus ei käi või kuulab teisel pordil | `systemctl status labori-app`, `ss -tlnp` |
    | vastab ainult üks rakendusserver | mallis on kõigil sama port | B1, `hostvars` |
    | `failed to validate` | mallis on süntaksiviga | B2, loe `stderr`-i rida |
    | `curl` näitab nginx-i vaikelehte | `nginx.conf` pole asendatud (`dest` vale) | `dest: /etc/nginx/nginx.conf` |
    | `Decryption failed` | vale Vaulti parool | `--ask-vault-pass` |
