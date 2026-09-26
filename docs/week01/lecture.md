# K1 · Ansible alused: idempotentsus ja esimene playbook

Selle loengu jaoks peaksid oskama kasutada Linuxi käsurida, ühenduda SSH-ga, kasutada `sudo`-t, paigaldada pakette `dnf`-iga ja teha Gitis commit'i. Klassis räägime peatükkidest 1, 4, 5 ja 8. Ülejäänu loe kodus läbi.

---

## Õpiväljundid

Pärast seda loengut oskad:

- selgitada, miks käsitsi seadistatud serverid aja jooksul üksteisest erinema hakkavad, ja kirjeldada automatiseerimise üldmudelit;
- eristada käsku ja soovitud oleku kirjeldust ning põhjendada, miks moodul on idempotentne ja `command` mitte;
- kirjeldada, mida Ansible teeb, kui playbook käivitub: control node, SSH, Python, push-mudel, `forks`;
- kirjutada ja lugeda inventari, ad-hoc käske, playbooki ja `PLAY RECAP`-i;
- kasutada fakte ja muutujaid playbookis;
- valida ohutu töökäik muudatusele: `--syntax-check`, `--check --diff`, `--limit`, siis kõik masinad;
- seadistada võtmepõhise SSH-ligipääsu ja lahendada tüüpilised ühendusvead.

---

## 1. Kolm serverit ja üks unustatud samm

Väike Eesti e-pood valmistub jõulukampaaniaks. Seni on üks veebiserver, nüüd on vaja kolme. Administraator seadistab esimese käsitsi: loob teenusekasutaja, paigaldab nginx-i, kopeerib avalehe, lülitab teenuse sisse. Pool tundi ja töötab. Teine server läheb kiiremini, sest käsud on shelli ajaloos. Kolmanda juures heliseb telefon ja `systemctl enable nginx` jääb tegemata.

Kõik kolm töötavad. Koormusjaotur saadab liiklust kõigile kolmele, testid on rohelised, kampaania läheb käima.

Kolm nädalat hiljem tulevad turvauuendused ja serverid taaskäivitatakse. Kaks tulevad üles, kolmas mitte. Seal ei käivitu nginx buutimisel, sest keegi ei lülitanud seda sisse.

Koormusjaoturi tervisekontroll märkab, et üks server ei vasta, ja võtab selle rotatsioonist välja. Kampaania tipptunnil on kolmandik võimsusest puudu. Põhjuse otsimine võtab tunni, sest "kõik serverid on ju samamoodi seadistatud".

Kolleeg teeks sama tööd teisiti. Ta kirjutab ühe faili, mis ütleb: "neis kolmes masinas peab olema kasutaja `saidi`, pakett `nginx`, see avaleht, ja nginx peab käima ning buutimisel käivituma". Siis käivitab ta selle faili kõigi kolme vastu korraga.

Kui homme tuleb juurde neljas server, lisab ta inventari ühe rea. Kui keegi on vahepeal mõnes masinas midagi muutnud, näitab järgmine käivitus täpselt, mis erineb. Ja parandab selle ära.

Sellel kursusel õpid just seda teist teed. Süsteemi olek on kirjas koodis, kood on Gitis ja iga muudatus käib koodi kaudu. Alustame Ansible'iga, sest sellega on kõige lihtsam alustada. Sihtmasinas on vaja ainult SSH-d ja Pythonit, ja need on tavalises Linuxi serveris juba olemas.

### Mis on Ansible

**Ansible** on avatud lähtekoodiga automatiseerimistööriist. Kirjeldad tekstifailis, milline server peab olema, ja Ansible viib selle olekusse palju masinaid korraga. Ansible'i tegi 2012. aastal Michael DeHaan, alates 2015. aastast arendab seda Red Hat.

Ansible ise on kirjutatud Pythonis ja kirjeldused on YAML-failides. Masinatega ühendub ta üle SSH (Windowsi masinatega üle WinRM-i või SSH).

??? note "Ansible'it kasutatakse neljaks asjaks"

    | Kasutus | Näide |
    |---|---|
    | konfiguratsioonihaldus | 50 serveris on samad kasutajad, paketid, SSH-seaded ja NTP |
    | rakenduse paigaldus | uus versioon kopeeritakse serveritesse ja teenus taaskäivitatakse |
    | mitmesammuline muudatus (orkestreerimine) | võta server koormusjaoturist välja, uuenda, kontrolli, pane tagasi, järgmine |
    | ühekordsed toimingud paljudes masinates | kontrolli kõigis serverites kettaruumi või taaskäivita teenus |

Ansible töötab masinatega, mis on juba olemas. Uusi masinaid (virtuaalmasinad, pilveressursid) loob Terraform, see tuleb viiendal kohtumisel. Rakenduse paneb konteinerisse Docker, see tuleb kolmandal. Tavaliselt jaguneb töö nii: Terraform loob masina, Ansible seadistab selle ja rakendus jookseb kas otse masinas või konteineris.

??? note "Ansible'i sõnavara, mida täna kasutame"

    | Mõiste | Tähendus | Kus täna näed |
    |---|---|---|
    | control node | masin, kus Ansible on paigaldatud ja kust käivitad | sinu WSL või Linux |
    | managed node | masin, mida hallatakse | `localhost`, siis `vm1`–`vm3` |
    | inventar | nimekiri hallatavatest masinatest ja gruppidest | `inventory.ini` |
    | moodul | programm, mis haldab üht liiki ressurssi ja kontrollib olekut | `user`, `package`, `copy`, `service` |
    | task | üks moodul koos parameetritega: üks soovitud oleku rida | "nginx on paigaldatud" |
    | play | task'ide jada, mis rakendub kindlatele masinatele | `hosts: veeb` + `tasks:` |
    | playbook | YAML-fail ühe või mitme play'ga | `bootstrap.yml` |
    | fakt | info masina kohta, mille Ansible kogub enne task'e | `ansible_os_family` |
    | ad-hoc käsk | üks moodul üks kord, ilma playbookita | `ansible veeb -m ping` |

Järgmisel kohtumisel lisanduvad **roll** (taaskasutatav task'ide, mallide ja muutujate kogum) ja **Vault** (krüptitud saladused).

??? info "Teised tööriistad: Puppet, Chef, SaltStack"

    Ansible pole ainus selline tööriist. Puppet, Chef ja SaltStack teevad sama tööd, aga tavaliselt on neil vaja igasse masinasse agenti ja lisaks keskserverit. Ansible'iga saad kiiresti alustada: paigaldad selle ühte masinasse ja saad kohe hallata kõiki, kuhu SSH-ga ligi pääsed. Mida see agentless tähendab, vaatame §6-s.

*Allikad: [Ansible: getting started](https://docs.ansible.com/ansible/latest/getting_started/) · raamat: Meijer, Hochstein, Moser, *Ansible: Up and Running*, 3. tr, ptk 1–4*

---

## 2. Konfiguratsiooni triiv

Eelmise loo probleemil on nimi: **konfiguratsiooni triiv** (configuration drift). Masinad pidid olema ühesugused, aga nad erinevad väikestes asjades. Keegi ei märka neid enne, kui midagi katki läheb.

??? note "Triivil on neli tüüpilist põhjust"

    | Põhjus | Näide |
    |---|---|
    | unustatud samm | `systemctl enable` jäi tegemata |
    | teine järjekord | konf kopeeriti enne paketti, pakett kirjutas selle üle |
    | öine käsitsi parandus | `max_connections` tõsteti ühes masinas, teistes mitte |
    | keegi ei pannud kirja | "Andres muutis midagi, aga ta on puhkusel" |

Triiv kasvab ajaga. Esimesel päeval on serverid peaaegu identsed. Pool aastat hiljem on igaühel oma ajalugu: erinevad paketiversioonid, käsitsi lisatud cron-read, ajutised failid, mis jäid alles. Sellist serverit kutsutakse inglise keeles **snowflake server**. Igaüks on isemoodi. Keegi ei tea täpselt, mis seal sees on, ja keegi ei julge seda uuesti paigaldada.

Teine võimalus on vaadata servereid kui asendatavaid. Kui serveri olek on koodis kirjas, saad selle igal ajal uuesti ehitada. Katkist masinat ei pea parandama, selle saab välja vahetada. Päriselt tähendab see, et öine intsident lõpeb käsuga "ehita uus", mitte kolmetunnise veaotsinguga.

Triivi ei hoia ära see, et oled käsitsi eriti hoolikas. Inimesed unustavad, telefon heliseb, on kiire. Aitab ainult see, kui masina olek on kirjas kohas, mis ei unusta. Ja seda kirjeldust rakendatakse ikka ja jälle.

---

## 3. Automatiseerimise üldmudel

Kõik automatiseerimise süsteemid koosnevad samadest osadest. Pole vahet, kas see on cron-skript, Ansible, Terraform, CI-konveier või Kubernetes:

<figure style="max-width:740px;margin:.8em auto" class="lx" markdown="0">
<svg viewBox="0 0 740 156" role="img" aria-label="Automatiseerimise üldmudel" xmlns="http://www.w3.org/2000/svg">
<style>.lx svg{width:100%;height:auto;font-family:var(--md-text-font-family,sans-serif)}.lx .box{fill:var(--md-code-bg-color);stroke:var(--md-default-fg-color--lighter);stroke-width:1.2}.lx .hi{fill:var(--md-primary-fg-color);fill-opacity:.16;stroke:var(--md-primary-fg-color);stroke-width:1.5}.lx .ok{fill:#2e7d32;fill-opacity:.14;stroke:#2e7d32;stroke-width:1.3}.lx .bad{fill:#c62828;fill-opacity:.12;stroke:#c62828;stroke-width:1.3}.lx .t{fill:var(--md-default-fg-color);font-size:14px;font-weight:700}.lx .b{fill:var(--md-default-fg-color);font-size:13.5px;font-weight:600}.lx .s{fill:var(--md-default-fg-color--light);font-size:12px}.lx .c{fill:var(--md-default-fg-color);font-size:12px;font-family:var(--md-code-font-family,monospace)}.lx .a{stroke:var(--md-default-fg-color--light);stroke-width:1.5;fill:none}.lx .h{fill:var(--md-default-fg-color--light)}</style>
<defs><marker id="lxa" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path class="h" d="M0,0 L10,5 L0,10 z"/></marker></defs>
<rect class="box" x="4" y="14" width="130" height="48" rx="6"/><text class="b" x="69.0" y="35.0" text-anchor="middle">Käivitaja</text><text class="s" x="69.0" y="50.0" text-anchor="middle">inimene, cron, push</text>
<line class="a" x1="134" y1="38" x2="152" y2="38" marker-end="url(#lxa)"/>
<rect class="box" x="154" y="14" width="130" height="48" rx="6"/><text class="b" x="219.0" y="35.0" text-anchor="middle">Sisend</text><text class="s" x="219.0" y="50.0" text-anchor="middle">kood, muutujad</text>
<line class="a" x1="284" y1="38" x2="302" y2="38" marker-end="url(#lxa)"/>
<rect class="box" x="304" y="14" width="130" height="48" rx="6"/><text class="b" x="369.0" y="35.0" text-anchor="middle">Soovitud olek</text><text class="s" x="369.0" y="50.0" text-anchor="middle">mis peab olema</text>
<line class="a" x1="434" y1="38" x2="452" y2="38" marker-end="url(#lxa)"/>
<rect class="hi" x="454" y="14" width="130" height="48" rx="6"/><text class="b" x="519.0" y="35.0" text-anchor="middle">Täitmine</text><text class="s" x="519.0" y="50.0" text-anchor="middle">võrdleb ja muudab</text>
<line class="a" x1="584" y1="38" x2="602" y2="38" marker-end="url(#lxa)"/>
<rect class="box" x="604" y="14" width="130" height="48" rx="6"/><text class="b" x="669.0" y="35.0" text-anchor="middle">Tõend</text><text class="s" x="669.0" y="50.0" text-anchor="middle">changed=0, logi</text>
<rect class="box" x="454" y="104" width="130" height="44" rx="6"/><text class="b" x="519.0" y="123.0" text-anchor="middle">Praegune olek</text><text class="s" x="519.0" y="138.0" text-anchor="middle">masin täna</text>
<line class="a" x1="519" y1="104" x2="519" y2="64" marker-end="url(#lxa)"/>
<text class="s" x="250" y="132" text-anchor="middle">Täitmine muudab ainult seda, mis erineb.</text>
</svg>
</figure>

**Käivitaja** paneb protsessi käima: inimene käsurealt, `git push`, cron, monitooringu häire, webhook. **Sisend** on kood, muutujad, masinate nimekiri ja saladused. **Soovitud olek** on kirjeldus sellest, milline süsteem peab olema. **Praegune olek** on see, milline süsteem tegelikult on. **Täitmine** võrdleb kaht olekut ja teeb vahe kinni. **Tõend** on väljund, millest näed, mis juhtus: logi, plaan, testitulemus, Ansible'i `changed`/`ok`.

Selle mudeliga saad aru ka tööriistast, mida sa veel ei tunne. Sama skeem sobib kõigile tööriistadele, mida kursusel kasutame:

??? note "Sama mudel eri tööriistades"

    | Tööriist | Käivitaja | Soovitud olek | Praegune olek | Tõend |
    |---|---|---|---|---|
    | cron + shell | kellaaeg | skripti sisu | failisüsteem | logifail (kui keegi selle kirjutas) |
    | Ansible | `ansible-playbook` | playbook | faktid + moodulite kontroll | `PLAY RECAP` |
    | Docker Compose | `docker compose up` | `compose.yml` | jooksvad konteinerid | `docker compose ps` |
    | GitHub Actions | `git push` | workflow-fail | repo sisu | roheline/punane job |
    | Terraform | `terraform apply` | `.tf` failid | state + päris ressursid | `plan` väljund |
    | Kubernetes | pidevalt | manifest | jooksvad Pod'id | `kubectl get` |

Kõige sagedamini ununeb tõend. Kui cron-skript kirjutab vead `/dev/null`-i, on see küll automatiseeritud, aga keegi ei tea, kas see töötab. Ansible annab tõendi igal jooksul. Kursusel esitad selle tõendi ka oma tööga: `changed=0` failis `logid/teine_jooks.txt` näitab, et sinu kirjeldus on idempotentne.

??? question "Kordamisküsimus"

    Võta cron-töö, mis teeb igal ööl andmebaasist varukoopia. Nimeta selle viis osa ülaltoodud mudeli järgi. Mis on selle töö puhul tõend, ja kas see on kuskil nähtav?

---

## 4. Käsk ja soovitud olek

Käsureal looksid kausta õigete õigustega nii:

```bash
mkdir -p /etc/skel/.ssh
chown root:root /etc/skel/.ssh
chmod 700 /etc/skel/.ssh
```

See on **imperatiivne** viis: ütled, mida teha ja mis järjekorras. Sinu asi on hoolitseda, et kõik kolm sammu tehakse ja et need tehakse õiges masinas.

Ansible'is kirjeldad sama tulemust **deklaratiivselt**, ühe task'ina:

```yaml
- name: .ssh kaust skeletonis
  ansible.builtin.file:
    path: /etc/skel/.ssh
    state: directory
    owner: root
    group: root
    mode: "0700"
```

Siin ei ole ühtegi tegusõna. Task kirjeldab, milline kaust peab olema. `file`-moodul otsustab ise, mida teha. Kui kaust on olemas õigete õigustega, ei tee moodul midagi. Kui õigused on valed, parandab ainult õigused. Kui kausta pole, loob selle.

Suurema näitega on vahe paremini näha. See shelli skript püüab nginx-i paigalduse teha selliseks, et seda saaks ohutult korrata:

```bash
#!/usr/bin/env bash
set -euo pipefail

if ! id saidi >/dev/null 2>&1; then
    useradd -m saidi
fi

if ! rpm -q nginx >/dev/null 2>&1; then
    dnf install -y nginx
fi

UUS="<h1>Hallatud</h1>"
if [ "$(cat /usr/share/nginx/html/index.html 2>/dev/null)" != "$UUS" ]; then
    echo "$UUS" > /usr/share/nginx/html/index.html
fi

systemctl is-enabled nginx >/dev/null 2>&1 || systemctl enable nginx
systemctl is-active nginx >/dev/null 2>&1 || systemctl start nginx
```

Skript töötab, aga ainult RedHati peres (`rpm`, `dnf`). Ta ei ütle, mida muutis ja mida mitte. Iga uue erijuhu jaoks on vaja uut `if`-i. Sama asi playbookina:

```yaml
- name: Veebiserver
  hosts: veeb
  become: true
  tasks:
    - name: Kasutaja saidi on olemas
      ansible.builtin.user:
        name: saidi

    - name: nginx on paigaldatud
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Avaleht on paigas
      ansible.builtin.copy:
        dest: /usr/share/nginx/html/index.html
        content: "<h1>Hallatud</h1>\n"

    - name: nginx käib ja käivitub buutimisel
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

Skripti `if`-id on nüüd moodulite sees. `package` kasutab masina enda paketihaldurit, AlmaLinuxis `dnf`-i. Väljundis näed iga rea kohta, kas see muutis midagi.

Playbooki saab lugeda nagu serveri kirjeldust. Kui uus kolleeg tahab teada, kuidas veebiserverid on seadistatud, avab ta `bootstrap.yml`-i. Kellegi shelli ajalugu ta läbi kaevama ei pea.

---

## 5. Idempotentsus

Deklaratiivsusest tuleb **idempotentsus**: sama tegevus annab sama tulemuse, ükskõik mitu korda sa seda käivitad.

Vaata, mis juhtub, kui käivitad lihtsa shelli skripti kaks korda:

```bash
#!/usr/bin/env bash
useradd -m raporteerija
mkdir /srv/raport
echo "seade=1" >> /srv/raport/conf
```

Esimene jooks:

```
$ sudo bash halb.sh
$ cat /srv/raport/conf
seade=1
```

Teine jooks:

```
$ sudo bash halb.sh
useradd: user 'raporteerija' already exists
mkdir: cannot create directory '/srv/raport': File exists
$ cat /srv/raport/conf
seade=1
seade=1
```

Kaks esimest viga on vähemalt näha. Kolmas rida viga ei anna, vaid lisab faili teise rea `seade=1`. Kui rakendus loeb konfi ja kaks sama võtmega rida ajavad selle segadusse, siis on viga olemas. Aga ükski logi seda ei näita.

Skripti saab idempotentseks teha, kui lisad iga sammu ette kontrolli:

```bash
#!/usr/bin/env bash
set -euo pipefail
id raporteerija >/dev/null 2>&1 || useradd -m raporteerija
mkdir -p /srv/raport
touch /srv/raport/conf
grep -qx 'seade=1' /srv/raport/conf || echo "seade=1" >> /srv/raport/conf
```

Kolmest reast sai viis. Iga uue rea juures pead mõtlema: mis on õige kontroll, mis juhtub, kui faili pole? Ansible'i moodulites on need kontrollid juba olemas:

```yaml
- ansible.builtin.user:
    name: raporteerija
- ansible.builtin.file:
    path: /srv/raport
    state: directory
- ansible.builtin.lineinfile:
    path: /srv/raport/conf
    line: seade=1
    create: true
```

Idempotentsust on vaja, sest automaatikat käivitatakse ikka ja jälle:

- **ajastatult**, et hoida masinaid joonel;
- **pärast katkestust**, kui eelmine jooks kukkus poole peal ja idempotentne kirjeldus teeb ära ainult puuduva osa;
- **arenduse ajal**, kus sama playbooki jooksutad kümneid kordi järjest.

Iga kord peab jooks olema ohutu.

<figure style="max-width:720px;margin:.8em auto" class="lx" markdown="0">
<svg viewBox="0 0 720 104" role="img" aria-label="Skript vs moodul teisel jooksul" xmlns="http://www.w3.org/2000/svg">
<style>.lx svg{width:100%;height:auto;font-family:var(--md-text-font-family,sans-serif)}.lx .box{fill:var(--md-code-bg-color);stroke:var(--md-default-fg-color--lighter);stroke-width:1.2}.lx .hi{fill:var(--md-primary-fg-color);fill-opacity:.16;stroke:var(--md-primary-fg-color);stroke-width:1.5}.lx .ok{fill:#2e7d32;fill-opacity:.14;stroke:#2e7d32;stroke-width:1.3}.lx .bad{fill:#c62828;fill-opacity:.12;stroke:#c62828;stroke-width:1.3}.lx .t{fill:var(--md-default-fg-color);font-size:14px;font-weight:700}.lx .b{fill:var(--md-default-fg-color);font-size:13.5px;font-weight:600}.lx .s{fill:var(--md-default-fg-color--light);font-size:12px}.lx .c{fill:var(--md-default-fg-color);font-size:12px;font-family:var(--md-code-font-family,monospace)}.lx .a{stroke:var(--md-default-fg-color--light);stroke-width:1.5;fill:none}.lx .h{fill:var(--md-default-fg-color--light)}</style>
<defs><marker id="lxa" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path class="h" d="M0,0 L10,5 L0,10 z"/></marker></defs>
<text class="t" x="170" y="14" text-anchor="middle">Skript: tegevused</text>
<text class="t" x="530" y="14" text-anchor="middle">Moodul: soovitud olek</text>
<rect class="box" x="20" y="26" width="140" height="44" rx="6"/><text class="b" x="90.0" y="45.0" text-anchor="middle">1. jooks</text><text class="c" x="90.0" y="60.0" text-anchor="middle">conf: seade=1</text>
<line class="a" x1="160" y1="48" x2="196" y2="48" marker-end="url(#lxa)"/>
<rect class="bad" x="200" y="26" width="140" height="44" rx="6"/><text class="b" x="270.0" y="45.0" text-anchor="middle">2. jooks</text><text class="c" x="270.0" y="60.0" text-anchor="middle">seade=1 kaks korda</text>
<rect class="box" x="380" y="26" width="140" height="44" rx="6"/><text class="b" x="450.0" y="45.0" text-anchor="middle">1. jooks</text><text class="c" x="450.0" y="60.0" text-anchor="middle">changed=1</text>
<line class="a" x1="520" y1="48" x2="556" y2="48" marker-end="url(#lxa)"/>
<rect class="ok" x="560" y="26" width="140" height="44" rx="6"/><text class="b" x="630.0" y="45.0" text-anchor="middle">2. jooks</text><text class="c" x="630.0" y="60.0" text-anchor="middle">changed=0</text>
<text class="s" x="180" y="92" text-anchor="middle">teeb iga kord sama tegevuse uuesti</text>
<text class="s" x="540" y="92" text-anchor="middle">kontrollib enne, muudab ainult vajadusel</text>
</svg>
</figure>

### Kuidas Ansible idempotentsust näitab

??? note "Iga task annab ühe tulemuse"

    | Olek | Tähendus |
    |---|---|
    | `ok` | olek oli juba soovitud, midagi ei muudetud |
    | `changed` | olek erines, moodul muutis seda |
    | `failed` | task ebaõnnestus |
    | `skipped` | task jäeti vahele (nt tingimus ei kehtinud või `--check` all `command`) |
    | `unreachable` | masinani ei saadud ühendust |

Hea playbook annab värskes masinas esimesel jooksul mitu `changed`-i. Kui jooksutad seda kohe uuesti, näed `changed=0`. See `changed=0` on tõend, et kirjeldus on idempotentne.

### Kui task on igal jooksul `changed`

Kui mõni task on igal jooksul `changed`, siis see ei kirjelda olekut, vaid teeb tegevust. Tavaliselt on põhjus selles, et päris mooduli asemel on kasutatud `command`-i või `shell`-i:

```yaml
- name: Halb: igal jooksul changed
  ansible.builtin.command: useradd -m deploy
```

Teisel jooksul annab see task lausa `failed`, sest `useradd` lõpetab veakoodiga. Kui moodulit tõesti pole, saad `command`-i teha idempotentseks kahel viisil. Esimene on `creates`: see ütleb, et käsku pole vaja, kui fail on juba olemas:

```yaml
- name: Genereeri võti ainult siis, kui seda pole
  ansible.builtin.command: ssh-keygen -t ed25519 -N "" -f /etc/app/key
  args:
    creates: /etc/app/key
```

Teine on `changed_when`: see ütleb Ansible'ile, millal väljund tähendab muutust. Sellest räägime teisel kohtumisel.

??? question "Kordamisküsimus"

    Miks on `useradd deploy` shelli skriptis ohtlikum kui `ansible.builtin.user: name=deploy`? Mis juhtub kummagagi teisel jooksul, ja kumma viga sa märkad?

---

## 6. Mida Ansible käivitamisel teeb

Ansible'i maailmas on kaks rolli. **Control node** on masin, kus Ansible on paigaldatud ja kust sa käske käivitad: sinu sülearvuti, WSL või hüppeserver. **Managed node** on masin, mida hallatakse. Managed node'i ei pea Ansible'it paigaldama. Seal peavad olema ainult SSH-server ja Python.

Kui käivitad playbooki, teeb Ansible iga task'i jaoks iga masinaga järgmist:

Sellest skeemist tuleb neli asja, mis on olulised kogu edasise töö jaoks.

<figure style="max-width:760px;margin:.8em auto" class="lx" markdown="0">
<svg viewBox="0 0 760 112" role="img" aria-label="Mida Ansible teeb ühe task'i juures" xmlns="http://www.w3.org/2000/svg">
<style>.lx svg{width:100%;height:auto;font-family:var(--md-text-font-family,sans-serif)}.lx .box{fill:var(--md-code-bg-color);stroke:var(--md-default-fg-color--lighter);stroke-width:1.2}.lx .hi{fill:var(--md-primary-fg-color);fill-opacity:.16;stroke:var(--md-primary-fg-color);stroke-width:1.5}.lx .ok{fill:#2e7d32;fill-opacity:.14;stroke:#2e7d32;stroke-width:1.3}.lx .bad{fill:#c62828;fill-opacity:.12;stroke:#c62828;stroke-width:1.3}.lx .t{fill:var(--md-default-fg-color);font-size:14px;font-weight:700}.lx .b{fill:var(--md-default-fg-color);font-size:13.5px;font-weight:600}.lx .s{fill:var(--md-default-fg-color--light);font-size:12px}.lx .c{fill:var(--md-default-fg-color);font-size:12px;font-family:var(--md-code-font-family,monospace)}.lx .a{stroke:var(--md-default-fg-color--light);stroke-width:1.5;fill:none}.lx .h{fill:var(--md-default-fg-color--light)}</style>
<defs><marker id="lxa" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path class="h" d="M0,0 L10,5 L0,10 z"/></marker></defs>
<rect class="hi" x="10" y="30" width="150" height="56" rx="6"/><text class="b" x="85.0" y="55.0" text-anchor="middle">control node (vm1)</text><text class="s" x="85.0" y="70.0" text-anchor="middle">ansible-playbook</text>
<line class="a" x1="160" y1="46" x2="300" y2="46" marker-end="url(#lxa)"/><text class="s" x="230.0" y="40" text-anchor="middle">1. SSH + moodul</text>
<line class="a" x1="160" y1="70" x2="300" y2="70" marker-end="url(#lxa)"/>
<rect class="box" x="304" y="20" width="170" height="76" rx="6"/><text class="b" x="389.0" y="62.0" text-anchor="middle">sihtmasin (vm2)</text>
<text class="s" x="389" y="64" text-anchor="middle">2. Python käivitab mooduli</text>
<text class="s" x="389" y="80" text-anchor="middle">3. moodul kustutatakse</text>
<line class="a" x1="300" y1="86" x2="162" y2="86" marker-end="url(#lxa)"/>
<text class="s" x="231" y="102" text-anchor="middle">4. tulemus JSON-ina tagasi</text>
<text class="b" x="560" y="40" text-anchor="start">agentless:</text>
<text class="s" x="560" y="56" text-anchor="start">sihtmasinas pole Ansible'it,</text>
<text class="s" x="560" y="72" text-anchor="start">vaja on ainult SSH-d ja Pythonit</text>
</svg>
</figure>

**Agentless.** Sihtmasinasse ei paigaldata ühtegi püsivat teenust. Puppeti ja Chefi puhul jookseb igas masinas agent, mida pead uuendama, jälgima ja turvama. Ansible'i puhul on rünnakupind ainult see, mis serveris nagunii olemas on: SSH.

**Push.** Sina otsustad, millal muudatus tehakse, ja see tehakse kohe. Agendiga pull-mudelis kirjutad muudatuse keskserverisse. Agent tõmbab selle alla järgmisel kontrollil, näiteks 30 minuti pärast. Push sobib hästi, kui tahad muudatust kohe näha ja kontrollida. Pull sobib paremini tuhandetele masinatele, mis peavad ise joonel püsima.

??? note "Ansible (push) vs Puppet ja Chef (pull)"

    | | Ansible (push) | Puppet / Chef (pull) |
    |---|---|---|
    | Sihtmasinas | SSH + Python | agent (teenus) |
    | Muutus toimub | kohe, kui käivitad | agendi järgmisel kontrollil |
    | Keskserver | pole vaja | vaja (Puppet Server, Chef Server) |
    | Sobib | kümned kuni sajad masinad, kontrollitud muudatused | tuhanded masinad, pidev joonel hoidmine |

**Task kõigil, siis järgmine task.** Task'id jooksevad masinates paralleelselt, aga ükshaaval: kõigepealt esimene task kõigis masinates, siis teine task kõigis masinates. Mitu masinat korraga, määrab `forks` (vaikimisi 5). Kui ühes masinas task ebaõnnestub, siis teised jätkavad. Katki läinud masin jääb aga ülejäänud play'st välja.

**Faktid kogutakse alguses.** Enne esimest task'i käivitab Ansible igas masinas `setup`-mooduli. See kogub masina kohta infot ja võtab iga masina juures paar sekundit. Kui fakte pole vaja, lülita kogumine välja (`gather_facts: false`).

??? question "Kordamisküsimus"

    Miks ei pea managed node'is Ansible paigaldatud olema? Mis peab seal siiski olema?

---

## 7. Paigaldamine ja `ansible.cfg`

Materjal on testitud `ansible-core` 2.14-ga (AlmaLinux 9). Oma versiooni näed käsuga `ansible --version`.

Ansible'i paigaldad ainult control node'i. Kuidas, on kirjas [Töökeskkonna](../keskkond.md) sammus 2, koos versiooni kontrolliga.

`ansible-core` on mootor koos väikese hulga sisseehitatud moodulitega (`ansible.builtin`). Paketis `ansible` on lisaks kollektsioonid, näiteks `ansible.posix` ja `community.general`. AlmaLinuxi tavarepos on ainult `ansible-core`. Seepärast paigaldad vajalikud kollektsioonid `ansible-galaxy`-ga.

### `ansible.cfg`

Ansible otsib seadistusfaili selles järjekorras ja kasutab esimest, mille leiab:

1. keskkonnamuutuja `ANSIBLE_CONFIG`;
2. `ansible.cfg` jooksvas kaustas;
3. `~/.ansible.cfg`;
4. `/etc/ansible/ansible.cfg`.

Hoia `ansible.cfg` projekti kaustas. Siis on seaded koodiga koos Gitis:

```ini
[defaults]
inventory = inventory.ini
forks = 10
host_key_checking = True
callback_result_format = yaml

[ssh_connection]
pipelining = True
```

Tänu `inventory`-le ei pea käsule `-i inventory.ini` lisama. `pipelining` teeb vähem SSH-ühendusi ja jooks läheb märgatavalt kiiremaks. `callback_result_format = yaml` teeb väljundi loetavamaks.

!!! warning "Tähelepanu"

    Kui `ansible.cfg` on kaustas, kuhu kõik saavad kirjutada (maailmaloetav kaust), siis Ansible ignoreerib seda turvalisuse pärast ja annab hoiatuse. WSL-is juhtub see siis, kui töötad Windowsi kettal (`/mnt/c/...`). Hoia oma kaust Linuxi failisüsteemis, näiteks `~/`.

*Allikad: [paigaldamine](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) · [ansible.cfg seaded](https://docs.ansible.com/ansible/latest/reference_appendices/config.html)*

---

## 8. Inventar

**Inventar** on nimekiri masinatest, mida haldad. Kõige lihtsam kuju on INI-fail:

```ini
[kohalik]
localhost ansible_connection=local

[veeb]
vm1
vm2
vm3
```

Nurksulgudes on **grupid**. Iga masin võib olla mitmes grupis. Alati on olemas kaks sisseehitatud gruppi: `all` (kõik masinad) ja `ungrouped` (masinad, mis pole üheski grupis).

`ansible_connection=local` ütleb, et selle masinaga ei ühenduta üle SSH. Käsud käivitatakse otse. Nii saad esimest playbooki proovida oma masinas ega pea ühtegi serverit seadistama.

### Ühenduse muutujad

Masina rea järele saab kirjutada muutujaid, mis ütlevad, kuidas masinaga ühenduda:

```ini
[veeb]
vm1 ansible_host=192.168.125.21 ansible_user=kasutaja
vm2 ansible_host=192.168.125.22 ansible_user=kasutaja
vm3 ansible_host=192.168.125.23 ansible_user=kasutaja ansible_port=2222
```

??? note "Ühenduse muutujad"

    | Muutuja | Tähendus |
    |---|---|
    | `ansible_host` | IP või DNS-nimi, kuhu ühenduda |
    | `ansible_user` | SSH kasutajanimi |
    | `ansible_port` | SSH port, vaikimisi 22 |
    | `ansible_connection` | `ssh` (vaikimisi) või `local` |
    | `ansible_python_interpreter` | Pythoni asukoht sihtmasinas, kui automaatne tuvastus ei tööta |

Puhtam on hoida ühenduse andmeid `~/.ssh/config`-is (vt §15). Siis on inventaris ainult nimed ja sama `ssh vm1` töötab nii käsurealt kui ka Ansible'ist.

??? info "YAML-kujul inventar"

    Sama inventar YAML-is:

    ```yaml
    all:
      children:
        kohalik:
          hosts:
            localhost:
              ansible_connection: local
        veeb:
          hosts:
            vm1:
            vm2:
            vm3:
    ```

    INI on lühem ja sobib väikestele inventaridele. YAML sobib, kui gruppe ja muutujaid on palju. Teisel kohtumisel paneme grupid üksteise sisse (`children`) ja lisame grupimuutujad.

### Inventari kontrollimine

Enne esimest jooksu vaata, kuidas Ansible sinu inventarist aru saab:

```bash
ansible-inventory -i inventory.ini --graph
```

```
@all:
  |--@ungrouped:
  |--@kohalik:
  |  |--localhost
  |--@veeb:
  |  |--vm1
  |  |--vm2
  |  |--vm3
```

`--list` näitab sama JSON-ina koos kõigi muutujatega.

### Sihtimine ja mustrid

??? note "Masinaid saab valida grupi, nime või mustri järgi"

    | Muster | Valib |
    |---|---|
    | `all` | kõik masinad |
    | `veeb` | grupi `veeb` |
    | `vm1` | ühe masina |
    | `vm1:vm2` | mõlemad |
    | `veeb:!vm3` | grupi `veeb` ilma `vm3`-ta |
    | `veeb:&test` | masinad, mis on nii `veeb`- kui `test`-grupis |
    | `vm*` | kõik, mille nimi algab `vm` |

Playbookis kirjutad mustri ritta `hosts:`. Käsureal saad `--limit`-iga valikut veel kitsamaks teha, playbooki `hosts:` rea peale:

```bash
ansible-playbook bootstrap.yml --limit vm1
ansible-playbook bootstrap.yml --limit 'veeb:!vm3'
```

Päriselt jagatakse inventar tavaliselt keskkondade (`test`, `prod`) ja rollide (`veeb`, `andmebaas`, `koormusjaotur`) järgi. Teisel kohtumisel lisame gruppidele muutujad. Siis seadistab sama playbook test- ja toodangukeskkonna erinevalt.

*Allikad: [inventar](https://docs.ansible.com/ansible/latest/inventory_guide/intro_inventory.html) · [mustrid](https://docs.ansible.com/ansible/latest/inventory_guide/intro_patterns.html)*

---

## 9. Ad-hoc käsud

Ad-hoc käsk käivitab ühe mooduli ühe korra ilma playbookita. Süntaks:

```
ansible <muster> -i <inventar> -m <moodul> -a "<argumendid>" [-b]
```

`-b` (`--become`) käivitab mooduli `sudo` kaudu.

??? note "Ad-hoc käskude näited"

    | Eesmärk | Käsk |
    |---|---|
    | kas masinad vastavad | `ansible veeb -m ping` |
    | faktid | `ansible vm1 -m setup -a "filter=ansible_distribution*"` |
    | kettaruum | `ansible veeb -m command -a "df -h /"` |
    | paketi paigaldamine | `ansible veeb -b -m package -a "name=htop state=present"` |
    | teenuse taaskäivitus | `ansible veeb -b -m service -a "name=nginx state=restarted"` |
    | faili kopeerimine | `ansible veeb -b -m copy -a "src=motd dest=/etc/motd"` |
    | kasutaja eemaldamine | `ansible veeb -b -m user -a "name=vana state=absent"` |

`ping`-moodul ei saada ICMP-paketti. See ühendub SSH-ga, käivitab sihtmasinas Pythoni ja vastab `pong`, kui kõik töötab. Nii kontrollib `ping` korraga kolme asja: ühendust, autentimist ja seda, kas Python on olemas.

```
vm1 | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

Ad-hoc käsud sobivad küsimustele ("mis versioon kõigis masinates on?") ja ühekordseteks töödeks ("taaskäivita teenus kohe"). Kõik, mis peab olema korratav või mida tahad Gitis hoida, käib playbooki.

*Allikad: [ad-hoc käsud](https://docs.ansible.com/ansible/latest/command_guide/intro_adhoc.html)*

---

## 10. YAML lühidalt

Playbookid on YAML-failid. Algajal läheb kõige sagedamini midagi viltu just YAML-i süntaksiga. Seepärast õpi põhireeglid selgeks.

**Taane loeb.** Struktuuri määravad tühikud, mitte sulud. Kasuta alati tühikuid, mitte tabulaatorit. Kursusel kasutame taanet 2 tühikut.

**Loend** algab kriipsuga:

```yaml
paketid:
  - htop
  - curl
  - git
```

**Sõnastik** on võti-väärtus paarid:

```yaml
kasutaja:
  nimi: deploy
  shell: /bin/bash
```

**Loend sõnastikest** on playbookis kõige levinum kuju. Iga task on üks loendi element:

```yaml
tasks:
  - name: Esimene task
    ansible.builtin.ping:

  - name: Teine task
    ansible.builtin.debug:
      msg: tere
```

**Jutumärgid** pane siis, kui väärtus algab `{`-ga (Jinja2 muutuja), kui selles on `:` ja selle järel tühik, või kui see peab jääma stringiks:

```yaml
mode: "0644"                    # ilma jutumärkideta loetakse kaheksandarvuks
sisu: "{{ inventory_hostname }}" # algab {-ga
pealkiri: "Viga: fail puudub"    # koolon + tühik
```

**Tõeväärtused** kirjuta kujul `true` ja `false`. YAML lubab ka `yes`, `no`, `on`, `off`, aga `ansible-lint` hoiatab nende eest.

??? note "Tüüpilised veateated"

    | Veateade | Põhjus |
    |---|---|
    | `mapping values are not allowed in this context` | koolon + tühik väärtuses ilma jutumärkideta |
    | `found character '\t' that cannot start any token` | tabulaator taandes |
    | `did not find expected '-' indicator` | loendi elemendi taane on vale |
    | `We were unable to read either as JSON nor YAML` | fail pole korrektne YAML, sageli taane |
    | `this task has extra params` | mooduli parameeter on vale taandega |

Enne esimest jooksu kontrolli süntaksit:

```bash
ansible-playbook bootstrap.yml --syntax-check
```

*Allikad: [YAML süntaks](https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html)*

---

## 11. Moodulid ja toores käsk

**Moodul** on väike programm, mis oskab hallata üht liiki ressurssi. Enne muutmist vaatab moodul, milline on praegune olek. Lõpus tagastab ta struktureeritud tulemuse.

??? note "Moodulid, mida esimestel kohtumistel kasutad"

    | Moodul | Mida haldab | Olulised parameetrid |
    |---|---|---|
    | `ansible.builtin.user` | kasutaja | `name`, `groups`, `append`, `shell`, `state` |
    | `ansible.builtin.group` | grupp | `name`, `state` |
    | `ansible.builtin.package` | pakett, OS-ist sõltumatult | `name`, `state` (`present`, `absent`, `latest`) |
    | `ansible.builtin.dnf` | pakett `dnf`-iga | `name`, `state`, `update_cache` |
    | `ansible.builtin.copy` | fail sisuga või kopeeritud failist | `dest`, `src` või `content`, `mode`, `owner` |
    | `ansible.builtin.file` | kaust, õigused, link, kustutamine | `path`, `state`, `mode`, `owner` |
    | `ansible.builtin.lineinfile` | üks rida failis | `path`, `line`, `regexp`, `create` |
    | `ansible.builtin.service` | teenus | `name`, `state`, `enabled` |
    | `ansible.builtin.cron` | cron-töö | `name`, `minute`, `hour`, `job` |
    | `ansible.builtin.fetch` | fail sihtmasinast control node'i | `src`, `dest`, `flat` |
    | `ansible.posix.authorized_key` | SSH avalik võti kasutajale | `user`, `key` |
    | `ansible.builtin.debug` | väljund jooksu ajal | `msg`, `var` |

`command` ja `shell` lihtsalt käivitavad käsu. Nad ei tea, mida käsk teeb, seega ei oska nad ka öelda, kas midagi muutus. Seepärast märgivad nad end vaikimisi alati `changed`-ks.

Mis vahe on `command`-il ja `shell`-il? `command` käivitab programmi otse, ilma shellita. Seal ei tööta torud (`|`), ümbersuunamised (`>`) ega muutujad (`$HOME`). `shell` käivitab käsu läbi `/bin/sh` ja siis töötab see kõik. `command` on ohutum, sest shelli erimärgid ei saa seal midagi ootamatut teha.

??? note "Käsk vs moodul"

    | Ülesanne | Käsk | Moodul |
    |---|---|---|
    | kasutaja olemas | `useradd deploy` | `ansible.builtin.user` |
    | pakett paigaldatud | `dnf install nginx` | `ansible.builtin.package` |
    | fail sisuga | `echo … > fail` | `ansible.builtin.copy` |
    | rida konfis | `echo … >> conf` | `ansible.builtin.lineinfile` |
    | teenus käib | `systemctl start nginx` | `ansible.builtin.service` |
    | õigused | `chmod 600 fail` | `ansible.builtin.file` |

Reegel on lihtne: kui moodul on olemas, kasuta moodulit. `command`/`shell` jäta käskudele, millele moodulit pole, või käskudele, mis ainult loevad.

### FQCN ja `ansible-doc`

Mooduli nimed kirjutame täiskujul (FQCN, fully qualified collection name): `ansible.builtin.copy`, mitte lihtsalt `copy`. Lühike kuju töötab ka. Täiskujust näed aga kohe, millisest kollektsioonist moodul pärit on, ja `ansible-lint` nõuab seda.

Mooduli parameetrid leiad käsurealt:

```bash
ansible-doc ansible.builtin.service      # täielik kirjeldus koos näidetega
ansible-doc -s ansible.builtin.copy      # lühikokkuvõte, sobib kopeerimiseks
ansible-doc -l | grep -i cron            # otsi mooduleid nime järgi
```

`ansible-doc` väljundi lõpus on alati jaotis `EXAMPLES`. Sealt saad näite, mis töötab. Kõiki parameetreid ei mäleta peast keegi, ja päris töös kasutad `ansible-doc`-i iga päev.

*Allikad: [ansible.builtin moodulid](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/)*

---

## 12. `become`: administraatori õigused

Paljud moodulid vajavad root-õigusi. Näiteks siis, kui paigaldad pakette, haldad teenuseid või muudad faile `/etc` all. Ansible ühendub tavakasutajana ja võtab `sudo` kaudu õigused juurde, kui sa nii ütled:

```yaml
- name: Veebiserver
  hosts: veeb
  become: true
```

Kui `become: true` on play tasemel, kehtib see kõigile task'idele. Võid selle panna ka ühele task'ile, kui ülejäänud töö root-õigusi ei vaja.

Kui sihtmasinas nõuab `sudo` parooli, lisa käsule `-K` (`--ask-become-pass`):

```bash
ansible-playbook bootstrap.yml -K
```

Ansible küsib parooli üks kord ja kasutab seda kõigis masinates. Kui masinatel on erinevad paroolid, siis see ei tööta. Laboris on kasutajal tavaliselt paroolita sudo:

```
kasutaja ALL=(ALL) NOPASSWD: ALL
```

Toodangus tehakse tavaliselt eraldi automaatikakonto. Sellel on paroolita sudo ainult nende käskude jaoks, mida on vaja. Vähima õiguse põhimõttest räägime teisel kohtumisel Vaulti juures.

`become_user`-iga saad task'i käivitada mõne teise kasutajana kui root, näiteks andmebaasi kasutajana:

```yaml
- name: Andmebaasi varukoopia
  ansible.builtin.command: pg_dumpall -f /tmp/dump.sql
  become: true
  become_user: postgres
```

*Allikad: [become](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_privilege_escalation.html)*

---

## 13. Faktid ja muutujad

Enne esimest task'i kogub Ansible igast masinast **fakte** (facts): OS, distributsioon, IP-aadressid, mälu, ketaste info ja palju muud.

```bash
ansible vm1 -m setup
ansible vm1 -m setup -a "filter=ansible_os_family"
ansible vm1 -m setup -a "filter=ansible_default_ipv4"
```

??? note "Sagedamini vajalikud faktid"

    | Fakt | Näide väärtusest |
    |---|---|
    | `ansible_os_family` | `RedHat` |
    | `ansible_distribution` | `AlmaLinux` |
    | `ansible_distribution_version` | `9.8` |
    | `ansible_hostname` | `hkhk-vm-17` |
    | `ansible_default_ipv4.address` | `192.168.125.21` |
    | `ansible_memtotal_mb` | `3915` |
    | `ansible_processor_vcpus` | `2` |

Playbookis saad fakte kasutada nagu muutujaid. Lisaks on olemas **maagilised muutujad**. Need annab Ansible alati, ka siis, kui fakte ei koguta. Kõige olulisem neist on `inventory_hostname`: masina nimi inventaris.

`inventory_hostname` (`vm1`) ja `ansible_hostname` (`hkhk-vm-17`) võivad erineda. Inventari nime paned sina ise, masina hostname'i mitte. Seepärast kasutame kursusel sildiks `inventory_hostname`-i.

### Jinja2

Muutujaid kirjutad **Jinja2** süntaksiga, kahekordsete loogeliste sulgude vahele:

```yaml
- name: Avaleht näitab masina nime
  ansible.builtin.copy:
    dest: /usr/share/nginx/html/index.html
    content: "<h1>{{ inventory_hostname }}</h1>\n"
```

Jinja2 lubab lihtsaid tingimusi ja filtreid:

```yaml
vars:
  os_nimi: "{{ ansible_distribution }} {{ ansible_distribution_version }}"
  keskkond: "{{ env | default('test') }}"
  pealkiri: "{{ inventory_hostname | upper }}"
```

`default` annab väärtuse, kui muutujat pole määratud. `upper` teeb teksti suurtähtedeks. Filtreid on sadu. Teisel kohtumisel kasutame neid mallides.

Tingimusi (`if … else …`) läheb vaja siis, kui masinad on erinevad. Näiteks siis, kui masinate hulgas on eri distributsioone. Meie kolm VM-i on kõik AlmaLinux. Seega kasutame fakte peamiselt selleks, et infot näidata: avalehel (A8) ja raportis (H4).

### `debug` ja muutujate vaatamine

Kui pole kindel, mis väärtus muutujal on, näita seda:

```yaml
- name: Näita, mis juurkaust valiti
  ansible.builtin.debug:
    var: os_nimi
```

```
ok: [vm1] =>
    os_nimi: AlmaLinux 9.8
```

*Allikad: [faktid ja muutujad](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_vars_facts.html)*

---

## 14. Playbooki anatoomia ja käivitamine

Playbook on YAML-fail, milles on üks või mitu **play**'d. Play seob masinad ja task'id:

```yaml
- name: Bootstrap veebiserver        # play nimi
  hosts: veeb                        # millistele masinatele
  become: true                       # sudo kõigile task'idele
  gather_facts: true                 # vaikimisi true
  vars:                              # muutujad
    lehe_pealkiri: "Hallatud Ansible'iga"
  tasks:                             # mida teha, järjekorras
    - name: Kasutaja saidi on olemas
      ansible.builtin.user:
        name: saidi

    - name: nginx on paigaldatud
      ansible.builtin.package:
        name: nginx
        state: present

    - name: Avaleht näitab masina nime
      ansible.builtin.copy:
        dest: /usr/share/nginx/html/index.html
        content: "<h1>{{ inventory_hostname }}</h1>\n"

    - name: nginx käib ja käivitub buutimisel
      ansible.builtin.service:
        name: nginx
        state: started
        enabled: true
```

Iga task'i `name` on see, mida näed väljundis. Kirjuta nimi soovitud olekuna ("nginx on paigaldatud"), mitte tegevusena ("paigalda nginx"). Nii saad väljundit lugeda nagu kontrollnimekirja.

Ühes failis võib olla mitu play'd, näiteks üks andmebaasidele ja teine veebiserveritele. Need jooksevad järjest.

### Käivitamise võtmed

??? note "Käivitamise võtmed"

    | Käsk | Mida teeb |
    |---|---|
    | `ansible-playbook p.yml --syntax-check` | kontrollib ainult YAML-i ja struktuuri |
    | `ansible-playbook p.yml --list-hosts` | näitab, milliseid masinaid play puudutaks |
    | `ansible-playbook p.yml --list-tasks` | näitab task'ide nimekirja |
    | `ansible-playbook p.yml --check --diff` | kuiv jooks, näitab muudatusi |
    | `ansible-playbook p.yml --limit vm1` | ainult ühele masinale |
    | `ansible-playbook p.yml --start-at-task "nginx on paigaldatud"` | alusta kindlast task'ist |
    | `ansible-playbook p.yml -v` / `-vvv` | rohkem väljundit; `-vvv` näitab SSH-ühendust |

Kui otsid viga, alusta alati `-v`-st. `-vvv` näitab, milliste parameetritega SSH-ühendus luuakse. Sellest leiad enamiku ühendusvigade põhjuse.

### Väljundi lugemine

Jooks näeb välja nii:

```
PLAY [Bootstrap veebiserver] **********************************

TASK [Gathering Facts] ****************************************
ok: [vm1]
ok: [vm2]

TASK [Kasutaja saidi on olemas] *******************************
ok: [vm1]
changed: [vm2]

TASK [nginx on paigaldatud] ***********************************
ok: [vm1]
ok: [vm2]
```

Iga task'i all on iga masina tulemus. Jooksu lõpus on kokkuvõte:

```
PLAY RECAP ****************************************************
vm1  : ok=5  changed=0  unreachable=0  failed=0  skipped=0
vm2  : ok=5  changed=1  unreachable=0  failed=0  skipped=0
vm3  : ok=0  changed=0  unreachable=1  failed=0  skipped=0
```

Siit loed välja kolm asja. `vm1` on soovitud olekus. `vm2`-s oli üks erinevus ja see parandati. `vm3`-ga ei saadud ühendust, nii et selle olekust ei tea sa midagi. Viimast rida on kõige lihtsam märkamata jätta, sest ka seal on `changed=0`.

*Allikad: [playbookid](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_intro.html)*

---

## 15. SSH-võtmed ja ligipääs

Ansible ühendub masinatega tavalise OpenSSH-kliendiga. Me ei taha, et iga masina juures küsitaks parooli. Seepärast kasutame võtmepõhist autentimist.

### Võtmepaar

Võtmepaar koosneb kahest failist. **Privaatvõti** (`~/.ssh/id_ed25519`) jääb control node'i ega lahku sealt kunagi. **Avalik võti** (`~/.ssh/id_ed25519.pub`) läheb igasse masinasse, kuhu tahad siseneda, faili `~/.ssh/authorized_keys`.

Ühendumisel tõestab klient, et tal on avalikule võtmele vastav privaatvõti. Parooli ei küsita.

```bash
ssh-keygen -t ed25519 -C "maria@kursus"
```

`ed25519` on tänapäeval tavaline valik: võti on lühike, kiire ja turvaline. RSA võtmeid näed vanemates süsteemides. Kui kasutad RSA-d, siis vähemalt 3072 bitti.

Võtmele saab panna parooli (passphrase). See kaitseb võtit, kui keegi faili varastab. Aga siis küsitakse parooli iga kord, kui võtit kasutad. Selle vastu aitab `ssh-agent`: see hoiab lahtikrüptitud võtit mälus, kuni sessioon lõpeb:

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
```

<figure style="max-width:490px;margin:.8em auto" class="lx" markdown="0">
<svg viewBox="0 0 490 160" role="img" aria-label="SSH-võtmepaar: kuhu kumb võti läheb" xmlns="http://www.w3.org/2000/svg">
<style>.lx svg{width:100%;height:auto;font-family:var(--md-text-font-family,sans-serif)}.lx .box{fill:var(--md-code-bg-color);stroke:var(--md-default-fg-color--lighter);stroke-width:1.2}.lx .hi{fill:var(--md-primary-fg-color);fill-opacity:.16;stroke:var(--md-primary-fg-color);stroke-width:1.5}.lx .ok{fill:#2e7d32;fill-opacity:.14;stroke:#2e7d32;stroke-width:1.3}.lx .bad{fill:#c62828;fill-opacity:.12;stroke:#c62828;stroke-width:1.3}.lx .t{fill:var(--md-default-fg-color);font-size:14px;font-weight:700}.lx .b{fill:var(--md-default-fg-color);font-size:13.5px;font-weight:600}.lx .s{fill:var(--md-default-fg-color--light);font-size:12px}.lx .c{fill:var(--md-default-fg-color);font-size:12px;font-family:var(--md-code-font-family,monospace)}.lx .a{stroke:var(--md-default-fg-color--light);stroke-width:1.5;fill:none}.lx .h{fill:var(--md-default-fg-color--light)}</style>
<defs><marker id="lxa" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path class="h" d="M0,0 L10,5 L0,10 z"/></marker></defs>
<rect class="hi" x="10" y="40" width="170" height="70" rx="6"/><text class="b" x="95.0" y="79.0" text-anchor="middle">vm1</text>
<rect class="box" x="24" y="72" width="66" height="28" rx="4"/><text class="c" x="57" y="90" text-anchor="middle">id_ed25519</text>
<rect class="ok" x="98" y="72" width="72" height="28" rx="4"/><text class="c" x="134" y="90" text-anchor="middle">.pub</text>
<text class="s" x="57" y="124" text-anchor="middle">privaatvõti: jääb siia</text>
<line class="a" x1="172" y1="86" x2="318" y2="25" marker-end="url(#lxa)"/>
<rect class="box" x="322" y="8" width="150" height="32" rx="6"/><text class="b" x="397.0" y="21.0" text-anchor="middle">GitHub</text><text class="c" x="397.0" y="36.0" text-anchor="middle">SSH and GPG keys</text>
<line class="a" x1="172" y1="86" x2="318" y2="63" marker-end="url(#lxa)"/>
<rect class="box" x="322" y="46" width="150" height="32" rx="6"/><text class="b" x="397.0" y="59.0" text-anchor="middle">vm2</text><text class="c" x="397.0" y="74.0" text-anchor="middle">authorized_keys</text>
<line class="a" x1="172" y1="86" x2="318" y2="101" marker-end="url(#lxa)"/>
<rect class="box" x="322" y="84" width="150" height="32" rx="6"/><text class="b" x="397.0" y="97.0" text-anchor="middle">vm3</text><text class="c" x="397.0" y="112.0" text-anchor="middle">authorized_keys</text>
<line class="a" x1="172" y1="86" x2="318" y2="139" marker-end="url(#lxa)"/>
<rect class="box" x="322" y="122" width="150" height="32" rx="6"/><text class="b" x="397.0" y="135.0" text-anchor="middle">vm1 ise</text><text class="c" x="397.0" y="150.0" text-anchor="middle">authorized_keys</text>
<text class="s" x="245" y="26" text-anchor="middle">avalik võti</text>
</svg>
</figure>

### Avaliku võtme kopeerimine

```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub kasutaja@192.168.125.21
ssh kasutaja@192.168.125.21 hostname
```

`ssh-copy-id` küsib esimesel korral parooli, sest võtit veel pole. Pärast seda enam mitte.

Kui `ssh-copy-id` pole olemas (näiteks Windowsi PowerShellis), teeb sama asja see käsk:

```bash
cat ~/.ssh/id_ed25519.pub | ssh kasutaja@192.168.125.21 \
  "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

SSH-server on õigustega range. Kui teised saavad `~/.ssh`-i või `authorized_keys`-i kirjutada, ignoreerib server võtit ja küsib parooli.

### `~/.ssh/config`

Failis `~/.ssh/config` saad anda masinatele lühikesed nimed ning määrata kasutaja ja võtme:

```
Host vm1
    HostName 192.168.125.21
    User kasutaja
    IdentityFile ~/.ssh/id_ed25519

Host vm2
    HostName 192.168.125.22
    User kasutaja
    IdentityFile ~/.ssh/id_ed25519

Host vm*
    ServerAliveInterval 30
```

Nüüd töötab `ssh vm1`. Ansible kasutab sama SSH-klienti, seega töötab ka inventaris lihtsalt `vm1`. Viimane plokk kehtib kõigile masinatele, mille nimi algab `vm`-ga.

### `known_hosts`

Esimesel ühendumisel küsib SSH, kas usaldad masina võtit (host key). Siis salvestab ta selle faili `~/.ssh/known_hosts`. Kui masin hiljem uuesti paigaldatakse, siis võti muutub ja SSH keeldub ühendumast:

```
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
@    WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!     @
@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@
```

Laboris ehitatakse VM-e tihti uuesti. Siis eemalda vana kirje:

```bash
ssh-keygen -R vm1
ssh-keygen -R 192.168.125.21
```

Toodangus on see hoiatus põhjus peatuda ja uurida. See võib tähendada, et keegi on end ühenduse vahele pannud.

Ka Ansible küsib host key kinnitust. Kui masinaid on palju, jääb jooks iga uue masina juures seisma. Et seda ei juhtuks, ühendu esimest korda käsitsi (`ssh vm1 hostname`) või kogu võtmed eelnevalt kokku:

```bash
ssh-keyscan vm1 vm2 vm3 >> ~/.ssh/known_hosts
```

!!! warning "Tähelepanu"

    Kui paned `ansible.cfg`-sse `host_key_checking = False`, lülitub kontroll välja. Laboris on see mugav, toodangus mitte.

!!! warning "Tähelepanu"

    Privaatvõti ei lähe Giti, jagatud kausta ega vestlusesse. Kui see lekib, loo uus võtmepaar ja eemalda vana avalik võti kõigist `authorized_keys` failidest. Esimese kodutöö harjutuses H1 teed seda Ansible'iga.

### Tüüpilised SSH-vead

SSH-vead ja nende lahendused on [praktikumi veaotsingus](lab.md#veaotsing).

*Allikad: [ssh_config](https://man.openbsd.org/ssh_config) · [ssh-keygen](https://man.openbsd.org/ssh-keygen)*

---

## 16. Ohutu muudatus

Automaatika teeb muudatuse kõigis masinates sekunditega. Sama kiiresti levib ka viga. Käsitsi tehtud viga rikub ühe serveri, sama viga playbookis rikub kõik. Seepärast tee iga muudatus samade sammudega:

<figure style="max-width:740px;margin:.8em auto" class="lx" markdown="0">
<svg viewBox="0 0 740 96" role="img" aria-label="Ohutu muudatuse järjekord" xmlns="http://www.w3.org/2000/svg">
<style>.lx svg{width:100%;height:auto;font-family:var(--md-text-font-family,sans-serif)}.lx .box{fill:var(--md-code-bg-color);stroke:var(--md-default-fg-color--lighter);stroke-width:1.2}.lx .hi{fill:var(--md-primary-fg-color);fill-opacity:.16;stroke:var(--md-primary-fg-color);stroke-width:1.5}.lx .ok{fill:#2e7d32;fill-opacity:.14;stroke:#2e7d32;stroke-width:1.3}.lx .bad{fill:#c62828;fill-opacity:.12;stroke:#c62828;stroke-width:1.3}.lx .t{fill:var(--md-default-fg-color);font-size:14px;font-weight:700}.lx .b{fill:var(--md-default-fg-color);font-size:13.5px;font-weight:600}.lx .s{fill:var(--md-default-fg-color--light);font-size:12px}.lx .c{fill:var(--md-default-fg-color);font-size:12px;font-family:var(--md-code-font-family,monospace)}.lx .a{stroke:var(--md-default-fg-color--light);stroke-width:1.5;fill:none}.lx .h{fill:var(--md-default-fg-color--light)}</style>
<defs><marker id="lxa" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path class="h" d="M0,0 L10,5 L0,10 z"/></marker></defs>
<rect class="box" x="4" y="18" width="128" height="46" rx="6"/><text class="b" x="68.0" y="38.0" text-anchor="middle">--syntax-check</text><text class="s" x="68.0" y="53.0" text-anchor="middle">kas YAML on õige</text>
<line class="a" x1="132" y1="41" x2="150" y2="41" marker-end="url(#lxa)"/>
<rect class="box" x="152" y="18" width="128" height="46" rx="6"/><text class="b" x="216.0" y="38.0" text-anchor="middle">--check --diff</text><text class="s" x="216.0" y="53.0" text-anchor="middle">mis muutuks</text>
<line class="a" x1="280" y1="41" x2="298" y2="41" marker-end="url(#lxa)"/>
<rect class="box" x="300" y="18" width="128" height="46" rx="6"/><text class="b" x="364.0" y="38.0" text-anchor="middle">--limit vm1</text><text class="s" x="364.0" y="53.0" text-anchor="middle">üks masin</text>
<line class="a" x1="428" y1="41" x2="446" y2="41" marker-end="url(#lxa)"/>
<rect class="box" x="448" y="18" width="128" height="46" rx="6"/><text class="b" x="512.0" y="38.0" text-anchor="middle">kõik</text><text class="s" x="512.0" y="53.0" text-anchor="middle">ülejäänud</text>
<line class="a" x1="576" y1="41" x2="594" y2="41" marker-end="url(#lxa)"/>
<rect class="ok" x="596" y="18" width="128" height="46" rx="6"/><text class="b" x="660.0" y="38.0" text-anchor="middle">teine jooks</text><text class="s" x="660.0" y="53.0" text-anchor="middle">changed=0</text>
<text class="s" x="370" y="86" text-anchor="middle">Kui mõni samm annab üllatuse, peatu ja uuri, enne kui lähed edasi.</text>
</svg>
</figure>

### Kuiv jooks: `--check --diff`

`--check` käivitab playbooki kuivalt: moodulid ütlevad, mida nad teeksid, aga ei muuda midagi. `--diff` näitab failide vana ja uue sisu vahet:

```
TASK [Avaleht näitab masina nime] *****************************
--- before: /usr/share/nginx/html/index.html
+++ after: /usr/share/nginx/html/index.html
@@ -1 +1 @@
-<h1>Tere käsitsi</h1>
+<h1>vm1</h1>
changed: [vm1]
```

Dry run ei näe kõike. `command`- ja `shell`-task'id jäetakse vahele, sest Ansible ei tea, mida need teeksid.

Mõni task sõltub sellest, mida eelmine task päriselt tegi. Näiteks teenust saab käivitada alles siis, kui pakett on paigaldatud. Sellisel juhul võib dry run anda vea, mida päris jooksul ei tuleks. Dry run annab hea ülevaate, aga ei garanteeri midagi.

### Mõjuala: `--limit`

**Blast radius** (mõjuala) näitab, kui suure osa süsteemist üks viga katki teeb. `--limit` hoiab mõjuala ühe masina piires, kuni oled kindel, et muudatus töötab:

```bash
ansible-playbook bootstrap.yml --limit vm1
curl -s http://vm1
ansible-playbook bootstrap.yml
```

Suuremates keskkondades kasutatakse `serial`-it. See teeb muudatuse partiide kaupa (näiteks 2 masinat korraga) ja jääb seisma, kui mõni partii ebaõnnestub. Sellest räägime teisel kohtumisel.

### Drift ja kontroll

**Drift** tekib, kui keegi muudab masinat käsitsi: peatab teenuse, parandab konfi, kustutab faili. Järgmine playbooki jooks näitab iga triivinud asja `changed`-ina ja viib selle tagasi soovitud olekusse.

Nii on playbook ka kontrollimise tööriist. Kui jooksutad seda igal ööl `--check` režiimis, saad nimekirja masinatest, mis on soovitud olekust eemale triivinud:

```bash
ansible-playbook bootstrap.yml --check > drift-$(date +%F).log
grep -q 'changed=[1-9]' drift-$(date +%F).log && echo "Drift leitud"
```

Päris keskkonnas käivitab selle CI-konveier või AWX/Ansible Automation Platform ja teade läheb meeskonna kanalisse. Neljandal kohtumisel ehitame konveieri, mis teeb sarnast kontrolli iga push'i peale.

### Ohutuse kontrollnimekiri

Iga muudatuse juures:

- ennusta: kirjuta üles, mitu `changed`-i ootad ja kus;
- vaata enne: `--check --diff`;
- piira mõjuala: `--limit` ühe masinaga;
- tõenda: teine jooks `changed=0` kõigil, `unreachable=0`;
- pane kirja: muudatus käib Giti, commit-sõnum ütleb miks.

??? question "Kordamisküsimus"

    Kolleeg ütleb, et `--check` on aeglane ja ta jätab selle vahele, sest "playbook on ju testitud". Millise olukorra puhul läheb see valesti? Ja millal näitab `--check` ise valesti?

*Allikad: [check mode ja diff](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_checkmode.html)*

---

## 17. Tüüpilised vead esimesel päeval

Tüüpilised vead ja nende lahendused on koos [praktikumi veaotsingus](lab.md#veaotsing).

Vigu otsi alati samas järjekorras. Loe veateade algusest lõpuni läbi. Korda käsku `-v`-ga. Proovi sama asja käsitsi sihtmasinas. Enamasti on viga kirjas juba veateate esimeses reas.

*Allikad: [ansible-lint](https://ansible.readthedocs.io/projects/lint/)*

---

## 18. Kokkuvõte

??? note "Põhimõtted ühes tabelis"

    | Põhimõte | Mida see tähendab |
    |---|---|
    | triiv tekib alati, kui masinaid seadistatakse käsitsi | kaitse on kirjeldus koodis, mida käivitatakse korduvalt |
    | task kirjeldab olekut, moodul otsustab tegevuse | `command`/`shell` ainult siis, kui moodulit pole |
    | `changed=0` teisel jooksul on idempotentsuse tõend | task, mis on igal jooksul `changed`, teeb tegevust ega kirjelda olekut |
    | Ansible on agentless ja push-põhine | control node ühendub SSH-ga, kopeerib mooduli, käivitab selle ja saab JSON-i tagasi |
    | inventar ütleb kus, playbook mis, faktid milline masin | faktid on playbookis muutujad, nt `ansible_distribution` |
    | võtmepõhine SSH on eeldus | privaatvõti jääb control node'i, avalik võti läheb `authorized_keys`-i, lühinimed `~/.ssh/config`-ist |
    | ohutu muudatus | `--syntax-check` → `--check --diff` → `--limit` → kõik → teine jooks; `unreachable` tähendab, et masina olekut sa ei tea |

*Allikad: [pikem Ansible'i materjal](https://hkhk-automation.github.io/devops/week03/lecture/)*

---

*Järgmine: [praktikum](lab.md). Juhendatud osas teed kõik ühel masinal, iseseisvas osas kolmel. Seejärel [kodune õpe ja kodutöö](homework.md).*
