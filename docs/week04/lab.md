# K4 · Praktikum: pipeline K3 pinule

Tänase lõpuks kontrollib GitHub Actions iga push'i järel sinu Ansible'i koodi ja otsib saladusi, ehitab rakenduse image'i ja paneb selle GHCR-i. vm3 ei ehita enam ise: Ansible tarnib vm1-st registris oleva image'i, leht näitab commit'i räsi ja main haru saab muuta ainult rohelise pull request'iga. Pinu on sama mis K3-s, juurde tuleb pipeline ([loengu §1 joonis](lecture.md#k3-tookaigu-norgad-kohad)).

Osa A on juhendatud: workflow, lint, punase pipeline'i lugemine, saladuste otsing ja ehitus. Osa B on iseseisvam: GHCR, tarne vm3-sse Vaulti tokeniga ja kaitstud main haru. Kus on **Ennusta**, mõtle enne käsku läbi, mis juhtub, ja ütle see naabrile. Oodatav tulemus ja vihjed on kinnistes plokkides. Edenemist näed [laboritahvlil](../tahvel.md).

??? abstract "Õpiväljundid"

    Praktikumi lõpuks oskad:

    1. kirjutada GitHub Actionsi workflow'd triggerite, jobide, step'ide ja `needs`-iga;
    2. lisada pipeline'i `yamllint`-i, `ansible-lint`-i ja saladuste otsingu ning parandada nende leitud vead;
    3. leida punase pipeline'i logist esimene viga;
    4. ehitada image pipeline'is ja panna see GHCR-i vähimate vajalike õigustega;
    5. tarnida registris olev image Ansible'iga VPN-i taga olevasse masinasse nii, et teine käivitus on `changed=0`;
    6. töötada kaitstud main haruga: haru, pull request, rohelised kontrollid, liitmine.

## Enne alustamist

Selleks peaksid olema K3 osa B teinud: vm1-st töötab `ssh vm3` võtmega ja vm3-s on Docker. Kui K3 jäi pooleli, pole see takistus: tänane starter sisaldab K3 lahendust ja `site.yml` paigaldab kõik vajaliku.

Ava juhendaja jagatud Classroom 50 link ja klooni uus repositoorium vm1-sse SSH-ga (**Code** → **SSH**):

```bash
cd ~
git clone git@github.com:hkhk-automation/<sinu-repo>.git
cd <sinu-repo>
ls -a
```

??? success "Oodatav tulemus"

    ```
    .  ..  .ansible-lint  .git  .github  .gitignore  .yamllint  README.md  ULESANNE.md
    ansible.cfg  group_vars  inventory.ini  logid  roles  site.yml  stack  vead
    ```

    `roles/`, `stack/`, `site.yml` ja `group_vars/` on K3 lahendus. `.yamllint` ja `.ansible-lint` on lint'i reeglid, millest tuleb juttu A2-s. Kaust `vead/` on kodutöö jaoks.

Kontrollnimekiri on repositooriumi **Issues** all: issue Lab 04 · CI/CD. Märgi ruut, kui osa on tehtud. Kinni? Küsi Discordis või ava issue mallist **Vajan abi**.

??? note "Mis repositooriumis lõpuks on"

    ```
    <sinu-repo>/
    ├── .github/workflows/ci.yml    # pipeline: lint → build → GHCR
    ├── .yamllint, .ansible-lint    # lint'i reeglid
    ├── site.yml, inventory.ini, ansible.cfg
    ├── group_vars/
    │   ├── all.yml                 # + app_image, ghcr_kasutaja
    │   └── docker/vault.yml        # krüptitud GHCR-i token
    ├── roles/docker/, roles/stack/ # stack: GHCR-i sisselogimine, ehitust pole
    ├── stack/compose.yaml          # app: image: ${APP_IMAGE}:${APP_VERSIOON}
    ├── README.md
    └── logid/
        ├── tarne.txt
        └── tarne_kontroll.txt
    ```

!!! tip "Soovitatav töökäik"

    Iga workflow muudatuse järel: commit, `git push`, siis GitHubis **Actions** → viimane käivitus → job → step. Üks tsükkel võtab minuti või kaks. Kui ootad, loe järgmine samm läbi.

## A · Juhendatud osa (GitHub Actions)

### A1 · Kirjuta esimene workflow

Selle sammu lõpuks on repositooriumis workflow, mis käivitub iga push'iga, ja sa tead, mis on runneris olemas.

Workflow'd on kaustas `.github/workflows/`. Seal on juba `abi-teavitus.yml`, mis saadab juhendajale teate, kui avad abi-issue. Loo kõrvale `ci.yml`:

```yaml
name: CI

on:
  push:
    branches: [main]

jobs:
  lint:
    runs-on: ubuntu-24.04
    steps:
      - name: Kus ma olen
        run: |
          head -n 2 /etc/os-release
          whoami
          ls -la
```

`on` ütleb, mis sündmus workflow käivitab, `jobs` all on üks job nimega `lint`, `runs-on` valib runneri ja `steps` on käsud järjekorras ([loeng §2](lecture.md#2-github-actions)). `run: |` järel on mitmerealine shelli skript.

**Ennusta:** mida näitab `ls -la` runneris?

Salvesta ja saada GitHubi:

```bash
git add .github/workflows/ci.yml
git commit -m "K4 A1: esimene workflow"
git push
```

Ava GitHubis repositoorium → **Actions** → **CI** → viimane käivitus → job **lint** → step **Kus ma olen**.

??? success "Oodatav tulemus"

    ```
    PRETTY_NAME="Ubuntu 24.04.3 LTS"
    NAME="Ubuntu"
    runner
    total 8
    drwxr-xr-x 2 runner docker 4096 ... .
    drwxr-xr-x 3 runner docker 4096 ... ..
    ```

    Kaust on tühi. Runner on uus virtuaalmasin, kus sinu repositooriumi veel pole. Step **Set up job** näitab, mis runneri image'it kasutati.

Repositooriumi toob runnerisse action `actions/checkout`. Lisa see esimeseks step'iks, `steps:` alla enne `Kus ma olen` step'i:

```yaml
      - uses: actions/checkout@v7
```

`uses` käivitab valmis action'i, `run` shelli käsu. `@v7` on action'i versioon. Commit, push ja vaata sama step'i uuesti.

??? success "Oodatav tulemus"

    `ls -la` näitab nüüd sinu repositooriumi faile: `ansible.cfg`, `roles`, `site.yml`, `.git` ja teised.

??? question "Mõtle (vabatahtlik)"

    Lisa step'i `python3 --version; ansible --version`. Mis on runneris olemas ja mis mitte? Kust seda ilma proovimata teada saaks?

??? info "Loe juurde"

    - [loeng §2](lecture.md#2-github-actions)
    - [Ubuntu 24.04 runneri tarkvara](https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2404-Readme.md)

### A2 · Lisa lint

Selle sammu lõpuks kontrollib `lint` job iga push'i ja pull request'i juures YAML-i ja Ansible'i koodi.

Pipeline peab käivituma ka pull request'il, et koodi saaks kontrollida enne main harusse jõudmist. `workflow_dispatch` lisab **Actions** lehele nupu **Run workflow**, millega saab workflow'd käsitsi käivitada. Muuda `on:` osa:

```yaml
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
```

Runneris pole `ansible-lint`-i ega `yamllint`-i. Need tulevad `pip`-ist ja versioonid on kinni, et homme oleks sama tulemus. `setup-python` paigaldab Pythoni, mille versiooni tead. Asenda step **Kus ma olen** nende step'idega (checkout jääb esimeseks):

```yaml
      - uses: actions/setup-python@v7
        with:
          python-version: "3.12"
      - name: Paigalda lint'i tööriistad
        run: pip install ansible-lint==26.10.1 yamllint==1.38.0
      - name: yamllint
        run: yamllint .
      - name: ansible-lint
        run: ansible-lint
```

`with:` annab action'ile parameetrid. Mõlemad lint'erid loevad oma reeglid repositooriumi juurest:

```bash
cat .yamllint .ansible-lint
```

`.yamllint` lubab pikemaid ridu kui vaikimisi ja jätab kõrvale kaustad `vead/` ja `logid/`. `.ansible-lint` jätab kõrvale samad kaustad ja `.github/`, sest workflow'd kontrollib `yamllint`.

**Ennusta:** K3 rollid töötasid ja teine käivitus oli `changed=0`. Kas `yamllint` on roheline? Kas `ansible-lint` on?

```bash
git add .github/workflows/ci.yml
git commit -m "K4 A2: yamllint ja ansible-lint"
git push
```

??? success "Oodatav tulemus"

    `yamllint` on roheline. `ansible-lint` on punane ja job `lint` lõpeb punase ristiga. Mida see ütleb, vaatad järgmises sammus.

    Kui juba `yamllint` on punane, on viga sinu `ci.yml`-is: tavaliselt vale taane või tühik rea lõpus. Rea number on väljundis.

??? info "Loe juurde"

    - [loeng §3](lecture.md#3-automaatne-kontroll)
    - [actions/setup-python](https://github.com/actions/setup-python)

### A3 · Loe punast pipeline'i ja paranda

Selle sammu lõpuks on `lint` job roheline ja sa tead, mida iga leitud reegel tahtis.

Ava punane käivitus → job **lint** → step **ansible-lint**. Keri logi lõppu ja siis üles kokkuvõtteni ([loeng §4](lecture.md#4-punase-pipelinei-lugemine)).

??? success "Oodatav tulemus"

    ```
    # Rule Violation Summary

      1 var-naming profile:basic tags:idiom
      1 name profile:basic tags:idiom
      3 no-changed-when profile:basic tags:command-shell,idempotency

    Failed: 5 failure(s), 0 warning(s) in 12 files processed ...
    ```

    Kokkuvõtte kohal on iga viga eraldi: reegli nimi, selgitus, fail ja rida.

??? note "Viis viga, mida `ansible-lint` leidis"

    | Reegel | Fail | Mida tahab |
    |---|---|---|
    | `var-naming[no-role-prefix]` | `roles/stack/defaults/main.yml` | rolli muutujate nimed algavad rolli nimega: `lb_port` → `stack_lb_port` |
    | `no-changed-when` | `roles/stack/handlers/main.yml`, `Stack uueneb` | `command` peab ütlema, millal ta midagi muutis |
    | `name[casing]` | `roles/stack/handlers/main.yml`, `proxy laeb seaded uuesti` | nimi algab suure tähega |
    | `no-changed-when` | sama fail, sama handler | nagu eespool |
    | `no-changed-when` | `roles/stack/tasks/main.yml`, `Kõik teenused käivad` | nagu eespool |

`no-changed-when` tähendab, et `command` on Ansible'i jaoks alati `changed`, kuigi pole teada, kas midagi muutus. K3-s tegid kontroll-task'i idempotentseks `changed_when: false`-iga. Handler aga käivitub ainult siis, kui mõni task muutis midagi. Kui handler käivitub, siis ta ka muudab: `docker compose up` loob konteinerid uuesti. Seega on õige väärtus `true`:

```yaml
- name: Stack uueneb
  ansible.builtin.command: docker compose up -d --build --wait
  args:
    chdir: "{{ stack_kaust }}"
  changed_when: true
```

Ülejäänud neli paranda ise:

- teine handler: nimi suure tähega ja `changed_when` nagu esimesel;
- task `Kõik teenused käivad` käivitub ainult siis, kui `when` ütleb, et teenuseid on puudu. Sama loogika mis handleril;
- `lb_port` → `stack_lb_port` kõigis kohtades, kus seda kasutatakse.

!!! warning "Tähelepanu: handleri nimi on ka `notify` reas"

    Kui muudad handleri nime, muuda ka task'i `notify` rida. `ansible-lint` seda ei kontrolli: vale nimega `notify` annab vea alles siis, kui task midagi muudab ja handlerit teavitab. Lint loeb faile, ta ei käivita neid ([loeng §3](lecture.md#lint-ei-asenda-kaivitust)).

Leia kõik kohad, kus vana nimi veel on:

```bash
grep -rn "lb_port\|proxy laeb" roles site.yml
```

??? example "Näide, kui ikka kinni"

    `roles/stack/defaults/main.yml`:

    ```yaml
    stack_kaust: /opt/labor
    stack_lb_port: 80
    ```

    `roles/stack/templates/env.j2`: `LB_PORT={{ stack_lb_port }}`. `site.yml` kontroll-play's: `url: "http://localhost:{{ stack_lb_port | default(80) }}/"`.

    `roles/stack/handlers/main.yml` teine handler:

    ```yaml
    - name: Proxy laeb seaded uuesti
      ansible.builtin.command: docker compose exec -T proxy nginx -s reload
      args:
        chdir: "{{ stack_kaust }}"
      changed_when: true
    ```

    `roles/stack/tasks/main.yml`: `notify: Proxy laeb seaded uuesti` ja task'i `Kõik teenused käivad` lõppu `changed_when: true`.

Kontrolli, et parandused ei muutnud pinu käitumist. Esimene käivitus võib vm3-s midagi muuta, kui sinu K3 failid erinesid starteri omadest. Teine peab olema `changed=0`:

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml
ansible-playbook site.yml | tail -n 3
```

??? success "Oodatav tulemus"

    ```
    PLAY RECAP ****************************************************
    vm3 : ok=15  changed=0  unreachable=0  failed=0  skipped=1
    ```

    Muutujate nimed muutusid, väärtused mitte, seega `.env` vm3-s jäi samaks.

Saada parandused GitHubi ja vaata, et `lint` job on roheline:

```bash
git add -A
git commit -m "K4 A3: ansible-lint'i leitud vead parandatud"
git push
```

??? tip "Kui `ansible-lint` leiab veel midagi"

    Loe uus kokkuvõte samamoodi. Tavalised põhjused: üks `lb_port` jäi alles (`grep`), `changed_when` on vale taandega (peab olema samal tasemel kui `args`), handleri nimi ja `notify` ei klapi.

??? question "Mõtle (vabatahtlik)"

    Kui lisada `.ansible-lint`-i `skip_list: [no-changed-when]`, on pipeline roheline. Mis sellest kaotame? Millal oleks `# noqa` põhjendatud?

??? info "Loe juurde"

    - [loeng §4](lecture.md#4-punase-pipelinei-lugemine)
    - [ansible-lint: no-changed-when](https://docs.ansible.com/projects/lint/rules/no-changed-when/), [var-naming](https://docs.ansible.com/projects/lint/rules/var-naming/)

### A4 · Otsi saladusi

Selle sammu lõpuks otsib `lint` job saladusi kogu Giti ajaloost.

`gitleaks` on üks programm, mille laadid alla GitHubi releasist kindla versiooniga. Ajaloo läbivaatamiseks peab runneris olema kogu ajalugu. Vaikimisi kloonib `actions/checkout` ainult viimase commit'i. `fetch-depth: 0` toob kõik. Muuda `lint` jobi checkout step'i:

```yaml
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
```

Lisa `lint` jobi lõppu step:

```yaml
      - name: Saladuste otsing (gitleaks)
        run: |
          curl -sSfL https://github.com/gitleaks/gitleaks/releases/download/v8.30.1/gitleaks_8.30.1_linux_x64.tar.gz | tar -xz gitleaks
          ./gitleaks git --redact --verbose .
```

`git` režiimis vaatab `gitleaks` läbi iga commit'i. `--redact` peidab leitud saladuse logis, `--verbose` näitab iga leiu faili ja commit'i.

**Ennusta:** `group_vars/all.yml`-is on `db_parool: "Muuda-mind-2026"` avatud kujul. Kas `gitleaks` leiab selle?

```bash
git add .github/workflows/ci.yml
git commit -m "K4 A4: saladuste otsing"
git push
```

??? success "Oodatav tulemus"

    ```
    ... INF 8 commits scanned.
    ... INF no leaks found
    ```

    `gitleaks` ei leidnud parooli. See otsib teadaolevaid mustreid (tokenid, võtmed) ja juhuslikke stringe võtmesõnade nagu `password` või `secret` juures. `db_parool` ja `Muuda-mind-2026` ei sobi kumbagi. Tööriist ei tähenda, et saladusi pole. K2-s oli selle jaoks Vault ja B2-s kasutad seda uuesti.

??? info "Loe juurde"

    - [loeng §3, saladuste otsing](lecture.md#saladuste-otsing)
    - [gitleaks](https://github.com/gitleaks/gitleaks)

### A5 · Ehita image pärast kontrolli

Selle sammu lõpuks ehitab teine job rakenduse image'i, aga ainult siis, kui `lint` on roheline.

K3-s ehitas image'i vm3. Nüüd teeb seda runner. Image'i nimi tuleb repositooriumi nimest, versioon commit'i räsist ([loeng §5](lecture.md#tag-on-commit)). Lisa `ci.yml`-i lõppu teine job, sama taandega nagu `lint`:

```yaml
  build:
    needs: lint
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
      - name: Image'i nimi ja versioon
        run: |
          echo "IMAGE=ghcr.io/${GITHUB_REPOSITORY,,}" >> "$GITHUB_ENV"
          echo "VERSIOON=${GITHUB_SHA::7}" >> "$GITHUB_ENV"
```

`$GITHUB_ENV` on fail: mis sinna kirjutad, on järgmistes step'ides keskkonnamuutuja. `${GITHUB_REPOSITORY,,}` teeb nime väiketähtedeks, `${GITHUB_SHA::7}` võtab räsi esimesed seitse märki.

Ehituse step kirjuta ise:

- `docker build`, Containerfile on `stack/app/Containerfile` (Docker otsib vaikimisi `Dockerfile`-i, seega `-f`);
- ehituskontekst on `stack/app`;
- tag on `"$IMAGE:$VERSIOON"`;
- silt `--label org.opencontainers.image.source=$GITHUB_SERVER_URL/$GITHUB_REPOSITORY`. See seob image'i repositooriumiga, B1-s on sellest kasu.

??? example "Näide, kui ikka kinni"

    ```yaml
          - name: Ehita image
            run: >
              docker build -f stack/app/Containerfile
              --label org.opencontainers.image.source=$GITHUB_SERVER_URL/$GITHUB_REPOSITORY
              -t "$IMAGE:$VERSIOON" stack/app
    ```

    `run: >` liidab read üheks reaks. Nii jääb pikk käsk loetavaks.

**Ennusta:** mis järjekorras jobid nüüd käivituvad? Mis juhtuks `build`-iga, kui `lint` oleks punane?

```bash
git add .github/workflows/ci.yml
git commit -m "K4 A5: image'i ehitus"
git push
```

??? success "Oodatav tulemus"

    Käivituse lehel on kaks kasti ja nool: `lint` → `build`. `build` alustab alles pärast `lint`-i. Step'is **Ehita image** on lõpus `naming to ghcr.io/hkhk-automation/<sinu-repo>:3f2a9c1 done`.

    Image on olemas ainult runneris ja kaob koos sellega. Järgmises sammus saadad selle registrisse.

??? question "Mõtle (vabatahtlik)"

    Ava `build` jobi step **Set up job** ja leia `GITHUB_TOKEN Permissions`. Mis õigused tokenil on? Kas `packages` on `write` või `read`?

## B · Iseseisev osa (GHCR, tarne, kaitstud main)

Osas A tegi pipeline kontrolli ja ehituse. Osas B jõuab image registrisse ja sealt vm3-sse ning main haru saab kaitse. Sammud on antud, osa lahendusest kirjutad ise.

Piirangud kogu B-osas:

- vm3-s muudab kõike Ansible. Käsitsi teed ainult kontrolle, mida juhend palub.
- Ühtegi tokenit ei kirjuta faili avatud kujul ega käsureale.
- Tarne käib ainult rohelise pipeline'i järel.

### B1 · Saada image GHCR-i

Selle sammu lõpuks on GHCR-is image `ghcr.io/hkhk-automation/<sinu-repo>:<räsi>` ja pipeline paneb sinna uue versiooni iga main haru push'iga.

Lisa `build` jobi lõppu kaks step'i. Mõlemad käivituvad ainult push'i korral, mitte pull request'i korral ([loeng §5](lecture.md#pull-request-ei-pane-registrisse)):

```yaml
      - name: Logi GHCR-i sisse
        if: github.event_name == 'push'
        env:
          GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: echo "$GHCR_TOKEN" | docker login ghcr.io -u "$GITHUB_ACTOR" --password-stdin
```

Teise step'i kirjuta ise: nimi `Saada image GHCR-i`, sama `if`, käsk `docker push` sama tag'iga mis ehitusel.

??? example "Näide, kui ikka kinni"

    ```yaml
          - name: Saada image GHCR-i
            if: github.event_name == 'push'
            run: docker push "$IMAGE:$VERSIOON"
    ```

**Ennusta:** kas `docker push` õnnestub? (Vihje: A5 viimane küsimus.)

```bash
git add .github/workflows/ci.yml
git commit -m "K4 B1: image GHCR-i"
git push
```

??? success "Oodatav tulemus"

    `Logi GHCR-i sisse` on roheline (`Login Succeeded`), `Saada image GHCR-i` on punane:

    ```
    denied: installation not allowed to Create organization package
    Error: Process completed with exit code 1.
    ```

    Sisselogimine õnnestus, sest token on kehtiv. Kirjutada registrisse ei tohi, sest organisatsioonis on tokeni vaikimisi õigus ainult lugeda. Sõnastus võib erineda, aga sõna `denied` on alati.

Anna õigus ainult sellele jobile, kes seda vajab. `lint` ei vaja kirjutamisõigust kuhugi, seega pane faili algusesse `jobs:` kohale kõigile jobidele kehtiv vähim õigus ja `build`-ile oma:

```yaml
permissions:
  contents: read
```

```yaml
  build:
    needs: lint
    runs-on: ubuntu-24.04
    permissions:
      contents: read
      packages: write
```

Jobi `permissions` asendab faili oma. Mida pole loetletud, on keelatud ([loeng §5](lecture.md#oigused-github_token-ja-permissions)). Commit, push ja vaata, et mõlemad jobid on rohelised.

Leia image GitHubist: repositooriumi avalehel paremal on **Packages**. Seal on image'i nimi ja tag, mis on sama mis commit'i lühiräsi:

```bash
git rev-parse HEAD | cut -c1-7
```

??? success "Oodatav tulemus"

    **Packages** all on `its25-automatiseerimine-lab-04-ci-cd-<sinu-kasutaja>` ja selle lehel tag, näiteks `3f2a9c1`, mis on sama mis käsu väljund. Pakett on privaatne nagu repositoorium.

??? tip "Kui `docker push` on ikka `denied`"

    Kontrolli, et `permissions` on `build` jobi all (sama taandega kui `runs-on`) ja et seal on `packages: write`. Vaata **Set up job** → `GITHUB_TOKEN Permissions`: kas `Packages: write` on nimekirjas? Kui on ja push ikka kukub, ütle juhendajale: siis on organisatsiooni seadetes paketi loomine piiratud.

??? info "Loe juurde"

    - [loeng §5, §6](lecture.md#5-ehitus-ja-register)
    - [GHCR-i image'i sidumine repositooriumiga](https://docs.github.com/en/packages/learn-github-packages/connecting-a-repository-to-a-package)

### B2 · Ansible tõmbab image'i GHCR-ist

Selle sammu lõpuks on roll `stack` valmis paigaldama vm3-sse registris oleva image'i ja vm3 saab GHCR-ist privaatset image'it tõmmata.

GitHubi runner ei pääse kooli võrku, seega jääb tarne käivitamine vm1-le. vm3 ühendub ise `ghcr.io`-ga ([loeng §7](lecture.md#7-kuidas-muudatus-serverisse-jouab)).

**Lugemistoken.** Pakett on privaatne. vm3 vajab tokenit, millega saab ainult lugeda. Tee see GitHubis: profiilipilt → **Settings** → **Developer settings** → **Personal access tokens** → **Tokens (classic)** → **Generate new token (classic)**. Nimi `vm3 GHCR`, aegumine 30 päeva, õigus ainult `read:packages`. Kopeeri token, seda näidatakse ainult üks kord.

**Token Vaulti.** Token läheb krüptitud faili grupi `docker` muutujaks, ilma et see satuks shelli ajalukku või avatud kujul kettale. `read -s` loeb tokeni ekraanile näitamata ja `ansible-vault encrypt` krüptib selle otse failiks:

```bash
mkdir -p group_vars/docker
read -rsp "GHCR token: " T; echo
printf 'vault_ghcr_token: %s\n' "$T" | ansible-vault encrypt --output group_vars/docker/vault.yml -
unset T
head -n 1 group_vars/docker/vault.yml
```

??? success "Oodatav tulemus"

    `ansible-vault` küsib kaks korda uut Vaulti parooli. Viimane käsk näitab `$ANSIBLE_VAULT;1.1;AES256`. Vaulti parooli ära unusta: iga `site.yml` käivitus küsib seda (`-J`).

**Muutujad.** Lisa `group_vars/all.yml`-i kolm muutujat:

- `app_image`: sinu image'i nimi ilma tag'ita, `ghcr.io/hkhk-automation/<sinu-repo>` (kopeeri B1 **Packages** lehelt);
- `ghcr_kasutaja`: sinu GitHubi kasutajanimi;
- `ghcr_token: "{{ vault_ghcr_token }}"`.

Kaudne nimi on sama muster mis K2 Vaulti lisaülesandes: `grep ghcr_token` leiab, kust väärtus tuleb, ja ainult väärtus on krüptitud.

**Compose.** vm3 ei ehita enam. Ava `stack/compose.yaml` ja muuda teenust `app`: kustuta `build:` plokk ja pane `image:` reale `${APP_IMAGE}:${APP_VERSIOON}`. Lisa `stack/.env.naide`-sse rida `APP_IMAGE=ghcr.io/hkhk-automation/naide`, et näidisseadetes oleksid kõik muutujad.

Mall `roles/stack/templates/env.j2` saab rea:

```jinja
APP_IMAGE={{ app_image }}
```

**Roll.** Rakenduse koodi pole vm3-s enam vaja. Muuda task'i `Compose-fail ja rakendus on paigas` nii, et see kopeerib ainult `compose.yaml`-i (`loop`-i pole vaja), ja handleris `Stack uueneb` eemalda `--build`.

Sisselogimine käib sama mustriga nagu K3 kontroll-task: kõigepealt kontroll, siis tegevus ainult vajadusel. Lisa task'id `roles/stack/tasks/main.yml`-i pärast nginx-i peatamist. Kontroll:

```yaml
- name: Kontrolli, kas Docker on GHCR-i sisse logitud
  ansible.builtin.command: grep -q ghcr.io /root/.docker/config.json
  register: stack_ghcr_sees
  changed_when: false
  failed_when: false
  check_mode: false
```

`failed_when: false`, sest `grep` annab koodi 1, kui faili või rida pole. Seda tahamegi teada. Play käib `become: true`-ga, seega käivitab Ansible `docker`-it root'ina ja root'i sisselogimine on failis `/root/.docker/config.json`.

Sisselogimise task'i kirjuta ise:

- `docker login ghcr.io -u {{ ghcr_kasutaja }} --password-stdin`, token tuleb `stdin`-iga (mooduli `command` parameeter);
- käivitub ainult siis, kui kontroll ei leidnud rida (`stack_ghcr_sees.rc != 0`);
- `changed_when: true`, sest kui see käivitub, siis ta muudab;
- token ei tohi jõuda Ansible'i väljundisse.

??? example "Näide, kui ikka kinni"

    ```yaml
    - name: Docker on GHCR-i sisse logitud
      ansible.builtin.command:
        cmd: docker login ghcr.io -u {{ ghcr_kasutaja }} --password-stdin
        stdin: "{{ ghcr_token }}"
      when: stack_ghcr_sees.rc != 0
      changed_when: true
      no_log: true
    ```

??? note "Moodulid ja parameetrid, mis on uued"

    | Moodul | Parameeter | Mida teeb |
    |---|---|---|
    | [`command`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/command_module.html) | `stdin` | annab käsule sisendi, nagu `echo ... \|` shellis, aga käsureale see ei jõua |
    | [`command`](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/command_module.html) | `cmd` | käsk, kui on vaja ka teisi parameetreid |
    | (task'i võtmesõna) | `failed_when` | millal task loetakse vigaseks; `false` tähendab mitte kunagi |
    | (task'i võtmesõna) | `no_log` | task'i argumente ja tulemust ei näidata väljundis ega logis |

    `when`, `changed_when`, `failed_when` ja `no_log` on task'i võtmesõnad, mitte mooduli parameetrid. Kontrolli ise: `ansible-doc ansible.builtin.command` ja leia, mida teeb `stdin_add_newline`.

Kontrolli süntaksit ja vaata, kas lint jääb roheliseks:

```bash
ansible-playbook site.yml --syntax-check
grep -rn "build\|labori-app" stack/compose.yaml roles/stack
```

??? success "Oodatav tulemus"

    `playbook: site.yml`. `grep` ei leia `stack/compose.yaml`-ist ega rollist enam `build`-i ega `localhost/labori-app`-i.

??? info "Loe juurde"

    - [loeng §6](lecture.md#6-saladused-pipelineis-ja-serveris)
    - [GHCR: autentimine isikliku tokeniga](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry#authenticating-with-a-personal-access-token-classic)

### B3 · Tarni versioon vm3-sse

Selle sammu lõpuks töötab vm3-s image GHCR-ist, leht näitab commit'i räsi ja teine tarne on `changed=0`.

Tarnida saab ainult image'it, mis on registris olemas. Saada B2 muudatused GitHubi ja oota, kuni pipeline on roheline:

```bash
git add -A
git status
git commit -m "K4 B2: tarne GHCR-ist, token Vaultis"
git push
```

`git status` peab näitama `group_vars/docker/vault.yml`-i, aga mitte ühtegi `.env` faili. Kui `gitleaks` on punane, kirjutasid tokeni kuhugi avatud kujul: vaata [veaotsingut](#veaotsing).

**Ennusta:** mis juhtub, kui käivitad tarne enne, kui `build` job on lõpetanud?

Kui pipeline on roheline, tarni sama commit'i image. Versioon on main haru viimase commit'i lühiräsi, sama, mille pipeline tag'iks pani:

```bash
git pull
V=$(git rev-parse HEAD | cut -c1-7); echo $V
ansible-playbook site.yml -J -e app_versioon=$V
```

`-J` küsib Vaulti parooli. `-e` annab `app_versioon`-ile väärtuse, mis on tähtsam kui `group_vars/all.yml`-is olev `1.0` ([loeng §7](lecture.md#7-kuidas-muudatus-serverisse-jouab)).

??? success "Oodatav tulemus"

    ```
    TASK [stack : Docker on GHCR-i sisse logitud] *******************
    changed: [vm3]
    TASK [stack : Compose-fail on paigas] ***************************
    changed: [vm3]
    TASK [stack : Stacki seaded on paigas] **************************
    changed: [vm3]
    RUNNING HANDLER [stack : Stack uueneb] **************************
    changed: [vm3]
    ...
    vm3 : ok=17  changed=4  unreachable=0  failed=0  skipped=1
    ```

    Task'ide nimed ja `ok` arv võivad sul erineda. Handler tõmbas uue image'i ja lõi `app` konteineri uuesti, `db` ja `proxy` jäid puutumata.

??? tip "Kui handler annab `manifest unknown` või `unauthorized`"

    `manifest unknown`: sellise tag'iga image'it registris pole. Kas pipeline oli selle commit'i peal roheline? Kas `$V` on sama mis **Packages** lehel?

    `unauthorized`: vm3 ei saanud sisse logida või tokenil pole õigust. Kontrolli tokeni õigust (`read:packages`) ja `ghcr_kasutaja` väärtust. Kui vahetad tokenit, kustuta vana sisselogimine (`ssh -t vm3 sudo docker logout ghcr.io`), muidu kontroll-task arvab, et kõik on korras.

Kontrolli vm3-s, kust image tuli, ja mida leht näitab:

```bash
VM3=$(ssh -G vm3 | awk '/^hostname /{print $2}')
ssh vm3 docker ps --filter name=labor-app
curl -s http://$VM3/
```

??? success "Oodatav tulemus"

    ```
    CONTAINER ID   IMAGE                                                         ...   STATUS
    8c41e0d2a7f3   ghcr.io/hkhk-automation/its25-...-<sinu-kasutaja>:3f2a9c1     ...   Up 1 minute (healthy)
    Labori seis 3f2a9c1
    Vastas: 8c41e0d2a7f3
    Külastused: ...
    ```

    Image on registrist ja leht näitab sama räsi. Andmebaas on sama volume'iga mis K3-s, seega loendur jätkab.

Teine tarne sama versiooniga ei tohi midagi muuta. Salvesta see ja kontroll tõenditeks:

```bash
ansible-playbook site.yml -J -e app_versioon=$V | tee logid/tarne.txt
{ ssh vm3 docker ps --filter name=labor-app; curl -s http://$VM3/; } | tee logid/tarne_kontroll.txt
git add logid
git commit -m "K4 B3: tarne GHCR-ist, teine käivitus changed=0"
git push
```

??? success "Oodatav tulemus"

    `logid/tarne.txt` lõpus on `vm3 : ... changed=0 ... failed=0`.

    See push käivitab pipeline'i uuesti ja GHCR-i tuleb uus tag. vm3-s jääb tööle eelmine versioon, kuni tarnid uuesti.

??? question "Mõtle (vabatahtlik)"

    Leht oli tarne ajal mõne sekundi maas. Miks? Mida oleks vaja, et uus versioon tuleks ilma katkestuseta (vihje: K3 kodutöö H1)?

### B4 · Tööta kaitstud main haruga

Selle sammu lõpuks oled näinud, et otse main harusse push'ida ei saa, et punast pull request'i ei saa liita ja et kustutatud saladus jääb ajalukku.

Kaitse lülitab sisse juhendaja, kui klass jõuab selle sammuni ([loeng §8](lecture.md#8-kaitstud-main-haru)). Kaitse nõuab pull request'i ja rohelisi kontrolle `lint` ja `build`.

**Ennusta:** mis juhtub, kui teed main harus tühja commit'i ja push'id?

```bash
git commit --allow-empty -m "Proovi main haru kaitset"
git push
```

??? success "Oodatav tulemus"

    ```
    remote: error: GH013: Repository rule violations found for refs/heads/main.
    remote: - Changes must be made through a pull request.
    ...
     ! [remote rejected] main -> main (push declined due to repository rule violations)
    ```

    Push lükati tagasi. Võta kohalik commit tagasi: `git reset --hard origin/main` (kõik muu on juba GitHubis).

!!! warning "Tähelepanu: kui push õnnestus"

    Siis pole kaitse veel sees. Ütle juhendajale ja ära tee järgmist osa enne, kui push lükatakse tagasi. Ilma kaitseta saab punase PR-i kogemata liita ja võlts-token jõuab main harusse, kust seda enam ei kustuta.

**Saladus pull request'is.** Token jääb vahel kogemata skripti sisse. Tee seda harus ja vaata, mida pipeline teeb. Token on võlts: see on juhuslik string, millel on GitHubi tokeni kuju.

```bash
git switch -c kolleegi-skript
mkdir -p scripts
echo "echo ghp_$(tr -dc 'A-Za-z0-9' </dev/urandom | head -c 36) | docker login ghcr.io -u kolleeg --password-stdin" > scripts/logi-sisse.sh
git add scripts
git commit -m "Kolleegi sisselogimise skript"
git push -u origin kolleegi-skript
```

Push'i väljundis on link pull request'i loomiseks. Ava see, kontrolli, et siht on `main`, ja vajuta **Create pull request**.

??? success "Oodatav tulemus"

    PR-i all on kontrollid. `lint` on punane, `build` on *skipped*. Nupp **Merge pull request** on hall: `Merging is blocked`. `lint` → **Saladuste otsing (gitleaks)** logis:

    ```
    Finding:     echo REDACTED | docker login ghcr.io -u kolleeg --password-stdin
    RuleID:      github-pat
    File:        scripts/logi-sisse.sh
    Commit:      5d1e...
    ... WRN leaks found: 1
    ```

**Ennusta:** kustutad faili uue commit'iga ja push'id. Kas `lint` muutub roheliseks?

```bash
git rm -q scripts/logi-sisse.sh
git commit -m "Token välja"
git push
```

??? success "Oodatav tulemus"

    `lint` on ikka punane ja leid on sama: sama fail, sama commit. `gitleaks` vaatab kogu ajalugu ja token on haru esimeses commit'is alles. Kui token oleks kehtiv, oleks see nüüd lekkinud kõigile, kes repositooriumi näevad. Esimene samm oleks tokeni tühistamine GitHubis.

Main harus ajalugu üle kirjutada ei saa, aga oma harus saab. Viska haru mõlemad commit'id ära:

```bash
git reset --hard origin/main
```

Haru on nüüd sama mis main. Lisa sinna tegelik muudatus: täida `README.md` (vaata [Esitamine](#esitamine)) ja tee commit. Alles siis kirjuta haru GitHubis üle:

```bash
git add README.md
git commit -m "K4: README"
git push --force
```

Järjekord on oluline. Kui push'id haru enne uut commit'i, on haru sama mis main ja GitHub loeb PR-i liidetuks.

??? success "Oodatav tulemus"

    PR-is on üks commit, `lint` ja `build` on rohelised, `build`-is on push'i step'id *skipped* (PR ei pane registrisse). **Merge pull request** on roheline.

Liida PR (**Merge pull request** → **Confirm merge**) ja too main vm1-sse:

```bash
git switch main
git pull
```

Liitmine on push main harusse, seega käivitub pipeline uuesti ja GHCR-i tuleb image selle commit'i räsiga.

??? question "Mõtle (vabatahtlik)"

    Tarni liidetud versioon vm3-sse samamoodi nagu B3-s. Kui palju samme on nüüd muudatusest kuni vm3-ni ja milline neist on veel käsitsi?

??? info "Loe juurde"

    - [loeng §8](lecture.md#8-kaitstud-main-haru)
    - [GitHub: pull request'i liitmine](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/incorporating-changes-from-a-pull-request/merging-a-pull-request)

## Esitamine

### README

Täida `README.md` mall: asenda iga `_vastus_` oma vastusega ja kustuta ülemine kast. `ULESANNE.md` jääb muutmata. README täitmine on boonus: kontroll BOONUS README läbib, kui `_vastus_` kohti enam pole.

### Muudatused main harusse

Autograde hindab main haru. Pärast B4-t läheb iga muudatus main harusse pull request'iga: haru, push, PR, rohelised kontrollid, liitmine. Kui midagi jäi harusse, liida see.

Töö on esitatud, kui kõik muudatused on main harus, Actionsis on viimane **CI** käivitus roheline ja **Autograde** näitab tulemust.

### Automaatne kontroll

Ava GitHubis oma repositoorium → **Actions** → viimane **Autograde**:

??? note "Mida Autograde kontrollib"

    | Kontroll | Mida vaatab |
    |---|---|
    | K1 · 10 p | `ci.yml` on korrektne YAML, selles on kaks tööd (lint ja build) ja image läheb `ghcr.io`-sse |
    | K2 · 10 p | `yamllint` ja `ansible-lint` on rohelised, `gitleaks` ei leia failidest saladusi |
    | K3 · 10 p | `site.yml` süntaks on korras, `vault.yml` on krüptitud, `.env` pole Gitis, Compose võtab rakenduse image'i registrist |
    | K4 · 5 p | kaustas `logid/` on teise tarne väljund, kus vm3 real on `changed=0` ja `failed=0` |
    | K5 · 10 p | kaustas `logid/` on tõend, et vm3 rakendus tuleb `ghcr.io`-st ja leht näitab commit'i räsi |

Hinne tuleb kontrollidest K1–K5 (45 punkti). Kontrollid nimega BOONUS (README ja kodutöö) punkte ei anna ja kukuvad seni, kuni need on tegemata.

## Kodutöö

Kodune õpe ja kodutöö on eraldi lehel: [K4 · Kodune õpe ja kodutöö](homework.md). Samasse repositooriumisse, tähtaeg Classroom 50-s.

## Veaotsing

??? info "Veaotsingu tabel: tüüpilised vead ja lahendused"

    Kontrolli järjekorras: mis job ja step kukkus, esimene veateade logis, kas viga on koodis või workflow's.

    | Probleem | Põhjus | Lahendus |
    |---|---|---|
    | **Actions** all pole ühtegi käivitust | fail pole kaustas `.github/workflows/` või `on:` haru nimi on vale | `ls .github/workflows`, `branches: [main]` |
    | `Invalid workflow file` | YAML-i viga workflow's | rea number on teates; vaata taanet |
    | `ls` runneris on tühi | checkout puudub | `actions/checkout` esimeseks step'iks |
    | `ansible-lint: command not found` | tööriist paigaldamata või vale step'i järjekord | `pip install` enne lint'i |
    | `yamllint`: `trailing spaces`, `wrong indentation` | YAML-i kuju | rida ja veerg on väljundis |
    | `The requested handler '...' was not found` | handleri nimi muutus, `notify` mitte | `grep -rn notify roles` |
    | `gitleaks`: `0 commits scanned` või ainult 1 | `fetch-depth: 0` puudub | A4 |
    | `unable to prepare context: ... Dockerfile: no such file` | `-f stack/app/Containerfile` puudub | A5 |
    | `invalid reference format: repository name must be lowercase` | image'i nimes on suurtäht | `${GITHUB_REPOSITORY,,}` |
    | `denied: installation not allowed to Create organization package` | `packages: write` puudub | B1 |
    | `build` on *skipped* | `lint` kukkus (`needs`) või `if:` | vaata `lint`-i |
    | `Decryption failed` | vale Vaulti parool | `-J`, sama parool mis B2-s |
    | `manifest unknown` | sellise tag'iga image'it pole registris | oota rohelist pipeline'i, võrdle räsi **Packages** lehega |
    | `unauthorized` või `denied` tõmbamisel | token vale või õiguseta, vana sisselogimine | B3 vihje, `sudo docker logout ghcr.io` |
    | `gitleaks` leidis tokeni, mille kirjutasid ise | token `vault.yml`-is avatud kujul | tühista token GitHubis, tee uus, krüpti (B2); vana commit jääb ajalukku, vt allpool |
    | võlts-token jõudis main harusse | kaitse polnud sees | lisa `gitleaks` väljundist rida `Fingerprint:` faili `.gitleaksignore` ja commit |
    | `GH013: Repository rule violations` | main on kaitstud | haru ja PR, B4 |
    | **Merge pull request** on hall | kontrollid pole rohelised | vaata PR-i kontrolle |
