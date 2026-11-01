# K4 · CI/CD: automaatne kontroll, ehitus ja tarne

Selle loengu jaoks peaksid tundma K3 pinu: vm3-s töötavad Compose'iga proxy, rakendus ja PostgreSQL ning roll `stack` paigaldab need idempotentselt. Täna paned selle pinu ümber pipeline'i: iga push kontrollib koodi, ehitab image'i üks kord ja paneb selle registrisse, kust vm3 selle tõmbab. Klassis räägime peatükkidest 1, 2, 4, 5 ja 7. Ülejäänu loe kodus läbi.

## Õpiväljundid

Pärast seda loengut oskad:

- selgitada, mida tähendavad CI, pidev tarne ja pidev paigaldus ning mis neist on meie kursusel automaatne;
- lugeda ja kirjutada GitHub Actionsi workflow'd: trigger, job, step, runner, `needs`;
- valida, mida kontrollivad `yamllint`, `ansible-lint` ja saladuste otsing ning mida need ei näe;
- leida punasest pipeline'ist esimene viga ja selle põhjus;
- ehitada image pipeline'is, panna see GHCR-i ja anda workflow'le selleks täpselt vajalikud õigused;
- selgitada, kuidas image jõuab VPN-i taga olevasse masinasse ja miks main haru peab olema kaitstud.

## 1. Muudatus, mida keegi ei kontrolli

K3 lõpus paigaldas `site.yml` vm3-sse kogu pinu. Kõik töötas, sest sina käivitasid playbooki, vaatasid väljundit ja kontrollisid lehte. Selles töökäigus on kolm nõrka kohta:

- **kontroll sõltub inimesest.** Kui keegi muudab rolli ja ei käivita `--syntax-check`-i, näeb viga alles see, kes järgmisena playbooki käivitab;
- **image ehitatakse serveris.** vm3 ehitas image'i ise (`docker compose up --build`). Kahes serveris tähendab see kahte ehitust ja kahte võimalikult erinevat image'it: alusimage võis vahepeal uueneda;
- **versioon pole seotud koodiga.** `labori-app:1.0` ei ütle, millisest commit'ist see tehti.

**Pipeline** on sammude ahel, mis käivitub iga muudatusega automaatselt: kontroll, ehitus, tarne. Sammud on koodina repositooriumis, samamoodi nagu playbook.

<figure style="max-width:690px;margin:.8em auto" class="lx" markdown="0">
<svg viewBox="0 0 690 252" role="img" aria-label="Pipeline: push käivitab lint ja build, image läheb GHCR-i; vm1 Ansible tarnib vm3-sse, vm3 tõmbab image GHCR-ist" xmlns="http://www.w3.org/2000/svg">
<style>.lx svg{width:100%;height:auto;font-family:var(--md-text-font-family,sans-serif)}.lx .box{fill:var(--md-code-bg-color);stroke:var(--md-default-fg-color--lighter);stroke-width:1.2}.lx .hi{fill:var(--md-primary-fg-color);fill-opacity:.16;stroke:var(--md-primary-fg-color);stroke-width:1.5}.lx .ok{fill:#2e7d32;fill-opacity:.14;stroke:#2e7d32;stroke-width:1.3}.lx .bad{fill:#c62828;fill-opacity:.12;stroke:#c62828;stroke-width:1.3}.lx .t{fill:var(--md-default-fg-color);font-size:15px;font-weight:700}.lx .b{fill:var(--md-default-fg-color);font-size:14.5px;font-weight:600}.lx .s{fill:var(--md-default-fg-color--light);font-size:13px}.lx .c{fill:var(--md-default-fg-color);font-size:12.5px;font-family:var(--md-code-font-family,monospace)}.lx .w{fill:var(--md-accent-fg-color);font-size:13.5px;font-weight:700}.lx .a{stroke:var(--md-default-fg-color--light);stroke-width:1.5;fill:none}.lx .d{stroke-dasharray:4 3}.lx .h{fill:var(--md-default-fg-color--light)}</style>
<defs><marker id="lxa" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path class="h" d="M0,0 L10,5 L0,10 z"/></marker></defs>
<rect class="hi d" x="148" y="8" width="534" height="100" rx="10"/>
<text class="s" x="160" y="26" text-anchor="start">GitHub</text>
<rect class="box" x="10" y="40" width="112" height="54" rx="6"/><text class="b" x="66.0" y="64.0" text-anchor="middle">git push</text><text class="s" x="66.0" y="80.0" text-anchor="middle">vm1 → GitHub</text>
<line class="a" x1="122" y1="67" x2="160" y2="67" marker-end="url(#lxa)"/>
<rect class="box" x="162" y="36" width="140" height="62" rx="6"/><text class="b" x="232.0" y="64.0" text-anchor="middle">lint</text><text class="s" x="232.0" y="80.0" text-anchor="middle">kood ja saladused</text>
<line class="a" x1="302" y1="67" x2="362" y2="67" marker-end="url(#lxa)"/><text class="s" x="332.0" y="60" text-anchor="middle">needs</text>
<rect class="box" x="364" y="36" width="140" height="62" rx="6"/><text class="b" x="434.0" y="64.0" text-anchor="middle">build</text><text class="s" x="434.0" y="80.0" text-anchor="middle">build + push</text>
<line class="a" x1="504" y1="67" x2="532" y2="67" marker-end="url(#lxa)"/>
<rect class="ok" x="534" y="36" width="138" height="62" rx="6"/><text class="b" x="603.0" y="64.0" text-anchor="middle">GHCR</text><text class="c" x="603.0" y="80.0" text-anchor="middle">:3f2a9c1</text>
<rect class="box d" x="10" y="132" width="672" height="92" rx="10"/>
<text class="s" x="22" y="150" text-anchor="start">kooli võrk, VPN-i taga</text>
<rect class="hi" x="40" y="160" width="190" height="52" rx="6"/><text class="b" x="135.0" y="183.0" text-anchor="middle">vm1</text><text class="s" x="135.0" y="199.0" text-anchor="middle">ansible-playbook site.yml</text>
<line class="a" x1="230" y1="186" x2="352" y2="186" marker-end="url(#lxa)"/><text class="s" x="291.0" y="179" text-anchor="middle">SSH</text>
<rect class="box" x="354" y="160" width="170" height="52" rx="6"/><text class="b" x="439.0" y="183.0" text-anchor="middle">vm3</text><text class="s" x="439.0" y="199.0" text-anchor="middle">docker compose up</text>
<line class="a d" x1="500" y1="160" x2="580" y2="100" marker-end="url(#lxa)"/>
<text class="s" x="560" y="124" text-anchor="start">pull, HTTPS välja</text>
<text class="s" x="345" y="244" text-anchor="middle">GitHubi runner ei näe kooli võrku. vm3 tõmbab image'i registrist ise.</text>
</svg>
</figure>



Kolm mõistet, mida segatakse sageli:

| Mõiste | Mis on automaatne | Meie kursusel |
|---|---|---|
| **CI** (continuous integration, pidev integreerimine) | iga push'i järel kontroll ja ehitus | lint, saladuste otsing, image'i ehitus |
| **pidev tarne** (continuous delivery) | iga roheline muudatus on valmis paigaldamiseks, paigalduse käivitab inimene | image on GHCR-is, `site.yml` tarnib selle vm3-sse |
| **pidev paigaldus** (continuous deployment) | iga roheline muudatus paigaldatakse ise | kodutöö lisa: self-hosted runner |

Täna jõuame pideva tarneni. Pipeline teeb kõik kuni registrini, paigalduse käivitad sina vm1-st. Peatükis 7 on põhjus.

??? question "Kordamisküsimus"

    Kolleeg ütleb: „Meil on CI, sest meil on GitHub Actions.“ Mida peaks workflow tegema, et see väide oleks tõsi?

## 2. GitHub Actions

GitHub Actions käivitab repositooriumi kaustas `.github/workflows/` olevaid YAML-faile. Iga fail on üks **workflow**.

```yaml
name: CI

on:                          # (1)!
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  lint:                      # (2)!
    runs-on: ubuntu-24.04    # (3)!
    steps:
      - uses: actions/checkout@v7      # (4)!
      - name: yamllint
        run: yamllint .                # (5)!

  build:
    needs: lint              # (6)!
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v7
      - run: docker build -f stack/app/Containerfile -t labori-app stack/app
```

1. **Trigger**: mis sündmus workflow käivitab. `push` main harusse ja `pull_request`, mille siht on main.
2. **Job**: sammude rühm, mis töötab ühes masinas. Jobid käivituvad vaikimisi korraga.
3. **Runner**: masin, kus job töötab. GitHubi runner on iga jobi jaoks uus virtuaalmasin, mis pärast kustutatakse.
4. Step, mis kasutab valmis **action**'it. `actions/checkout` kloonib repositooriumi runnerisse.
5. Step, mis käivitab shellis käsu. Kui käsu väljumiskood pole 0, on step punane ja job lõpeb.
6. `needs`: `build` käivitub alles siis, kui `lint` on roheline. Kui `lint` kukub, jääb `build` vahele (*skipped*).

### Runner on iga kord tühi

Runner ei mäleta eelmist käivitust. Repositooriumi seal ei ole enne, kui `actions/checkout` selle kloonib. Tööriistu, mida pildil pole, ei ole enne, kui step need paigaldab. See on hea: iga käivitus algab samast olekust ja „minu masinas töötab“ ei kehti. Paigaldatud tööriistade nimekiri on [runner-image'i lehel](https://github.com/actions/runner-images/blob/main/images/ubuntu/Ubuntu2404-Readme.md). Docker, Git ja Python on olemas, `ansible-lint` mitte.

Samal põhjusel kinnitad tööriistade versioonid: `pip install ansible-lint==26.10.1`. Ilma versioonita paigaldab runner homme uuema `ansible-lint`-i, mis võib leida uusi vigu koodis, mida keegi pole muutnud.

### Muutujad ja kontekst

Workflow'l on kaht liiki muutujaid. `${{ github.sha }}` on **kontekst**: GitHub asendab selle enne, kui step käivitub. `$GITHUB_SHA` on keskkonnamuutuja, mille shell loeb käivitamise ajal. Sisu on sama. Shelli käskudes eelista keskkonnamuutujat, sest konteksti väärtus läheb käsu teksti ja võõra sisendiga (näiteks PR-i pealkiri) saab nii käsku muuta.

| Muutuja | Mida sisaldab |
|---|---|
| `GITHUB_SHA` | commit, mille peal workflow töötab |
| `GITHUB_REPOSITORY` | `org/repo`, näiteks `hkhk-automation/its25-...` |
| `GITHUB_ACTOR` | kasutaja, kes sündmuse tekitas |
| `GITHUB_ENV` | fail; mis sinna kirjutad (`NIMI=väärtus`), on järgmistes step'ides keskkonnamuutuja |

??? question "Kordamisküsimus"

    Workflow's on jobid `lint`, `build` ja `docs`. `build`-il on `needs: lint`, `docs`-il `needs` puudub. `lint` kukub. Mis juhtub kahe ülejäänud jobiga?

??? info "Loe juurde"

    - [GitHub Actions: workflow süntaks](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax)
    - [Sündmused, mis workflow'd käivitavad](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)
    - [Vaikimisi keskkonnamuutujad](https://docs.github.com/en/actions/reference/workflows-and-actions/variables)

## 3. Automaatne kontroll

Esimene job kontrollib koodi enne, kui midagi ehitatakse. Täna on selles kolm tööriista.

| Tööriist | Mida kontrollib | Mida ei näe |
|---|---|---|
| `yamllint` | YAML-i kuju: taane, dubleeritud võtmed, tühikud rea lõpus, rea pikkus | kas playbook on õige |
| `ansible-lint` | Ansible'i head tavad: FQCN moodulite nimed, `command` ilma `changed_when`-ita, rolli muutujate eesliide, nimede kuju; käivitab ka `--syntax-check`-i | kas masinas tuleb soovitud olek |
| `gitleaks` | saladused failides ja **kogu Giti ajaloos**: tokenid, võtmed, suure entroopiaga stringid | saladust, mis ei sarnane ühelegi mustrile |

### Reeglid on failis

Mõlemal lint'eril on konfiguratsioonifail repositooriumi juures: `.yamllint` ja `.ansible-lint`. Seal on meeskonna kokkulepe: rea maksimaalne pikkus, kaustad, mida ei kontrollita, reeglid, mis on välja lülitatud. Kokkulepe on koodis ja kehtib kõigile ühtemoodi.

Kui `ansible-lint` leiab vea, on kaks võimalust: paranda või selgita, miks see reegel siin ei kehti. Teine käib rea lõppu lisatud kommentaariga `# noqa: no-changed-when` või konfiguratsioonifaili `skip_list`-iga. Kumbki ei ole vale, aga põhjendus peab olema kirjas. Sagedasem viga on lülitada reegel välja ainult selleks, et pipeline roheliseks saada.

Täna leiab `ansible-lint` K3 rollist viis viga. Rakendus töötas, seega ei ole need vead selles mõttes, et midagi oleks katki. Need on kohad, kus kood on eksitav: `command` task ilma `changed_when`-ita näitab `changed`, kuigi pole teada, kas midagi muutus.

### Lint ei asenda käivitust

`ansible-lint` loeb faile, ta ei käivita neid. Kui nimetad handleri ümber ja unustad `notify` rea, on lint roheline ja viga tuleb välja alles siis, kui task teavitab handlerit. Lint vähendab vigu, aga tõend on ikka see, mis masinas toimub.

### Saladuste otsing

`gitleaks` tunneb sadu mustreid (GitHubi token algab `ghp_`, AWS-i võti `AKIA`) ja otsib lisaks juhuslikke stringe, mis on võtme moodi. Otsing käib läbi kogu ajaloo, mitte ainult praeguse oleku. Saladus, mis lisati ühes commit'is ja kustutati järgmises, on endiselt ajaloos ja igaüks, kellel on repositooriumi koopia, saab selle kätte.

Seepärast on saladuse lekke järel esimene samm **tühistada saladus** (uus token, uus parool). Ajaloo ümberkirjutamine tuleb alles pärast seda ja ainult siis, kui see on võimalik. Kaitstud harus see üldjuhul pole.

`gitleaks` ei leia kõike. Täna kontrollid ise, kas ta leiab `group_vars/all.yml`-ist andmebaasi parooli.

??? question "Kordamisküsimus"

    Kolleeg commit'is kogemata faili `.env` ja kustutas selle viie minuti pärast järgmise commit'iga. Pipeline on punane, sest `gitleaks` leidis saladuse. Kolleeg tahab lisada faili `.gitleaksignore`, et pipeline roheliseks saada. Mis peab enne seda juhtuma?

??? info "Loe juurde"

    - [yamllint: reeglid](https://yamllint.readthedocs.io/en/stable/rules.html)
    - [ansible-lint: reeglid](https://docs.ansible.com/projects/lint/rules/)
    - [gitleaks](https://github.com/gitleaks/gitleaks)

## 4. Punase pipeline'i lugemine

Punane rist Actionsis ütleb ainult, et midagi kukkus. Viga leiad kolme klõpsuga: **Actions** → workflow käivitus → punane job → punane step. Step'i logi lõpus on väljumiskood (`Process completed with exit code 2`) ja selle kohal käsu väljund.

Loe logi ülevalt alla ja otsi **esimest** viga. Järgmised on sageli esimese tagajärjed. Kui `pip install` kukkus, kukuvad ka kõik järgmised step'id, aga nende veateated ei ütle midagi.

| Kus kukkus | Mida tavaliselt tähendab |
|---|---|
| `actions/checkout` või `setup-python` | võrguviga või vale action'i nimi; proovi **Re-run jobs** |
| `pip install` | vale versiooninumber, tööriista versioon ei toeta Pythoni versiooni |
| lint'i step | koodis on viga, mille lint leidis: failinimi ja rida on väljundis |
| `docker build` | Containerfile'i viga või fail puudub ehituskontekstist |
| `docker push`, sõna `denied` | workflow'l pole õigust registrisse kirjutada (§5) |
| job on hall, *skipped* | eelmine job (`needs`) kukkus või `if:` tingimus ei kehtinud |

Lint'i viga paranda koodis, mitte workflow's. Kui ei saa aru, mida reegel tahab, on reegli nimi (`no-changed-when`) lingiga logis ja dokumentatsioonis on näide nii valest kui ka õigest koodist.

Iga parandus on uus commit ja uus käivitus. Ühe tsükli pikkus on minut või kaks. Kui vigu on palju, paranda need kõik ühe korraga: `ansible-lint` näitab kõik vead korraga, mitte ainult esimese.

??? question "Kordamisküsimus"

    `lint` job on punane. Logis on 40 rida, viimane on `Process completed with exit code 2`, eelviimane on `Failed: 3 failure(s)`. Kust leiad, mis failis ja mis real vead on?

## 5. Ehitus ja register

K3-s ehitas vm3 image'i ise. Nüüd ehitab image'i pipeline ja paneb selle registrisse. Server ainult tõmbab. Põhimõte on **ehita üks kord, paigalda mitu korda**: testis ja tootmises on bitt-bitilt sama image, sest see on sama fail registris.

### GHCR

**GHCR** (GitHub Container Registry, `ghcr.io`) on GitHubi image'ite register. Image'i nimi on `ghcr.io/<omanik>/<nimi>:<tag>`. Kõik nime osad peavad olema väiketähtedega. Repositooriumi nimes võib olla suurtähti (näiteks GitHubi kasutajanimest), seega teeb workflow nime väiketähtedeks: `${GITHUB_REPOSITORY,,}` (Bashi teisendus).

GHCR-is on image privaatne, kui repositoorium on privaatne. Tõmbamiseks on vaja sisse logida (§6).

### Tag on commit

K3 loengus oli juttu, et tag on silt, mitte versioon. Pipeline'is on lihtne panna sildiks see, mis on tõesti unikaalne: **commit'i lühiräsi**, näiteks `3f2a9c1`. Siis on igal image'il oma tag ja selle järgi saab alati öelda, mis koodist see tehti. Tagasipööramine eelmisele versioonile on eelmise räsi paigaldamine.

`latest` või `main` on mugavad, aga need liiguvad iga ehitusega. Kui vm3-s töötab `:main`, ei tea keegi, mis versioon seal tegelikult on.

### Õigused: `GITHUB_TOKEN` ja `permissions`

Iga workflow käivitus saab automaatse tokeni `GITHUB_TOKEN`. See kehtib ainult selle käivituse ajal ja ainult selles repositooriumis. Meie organisatsioonis on tokeni vaikimisi õigus ainult lugeda. Registrisse kirjutamiseks peab job õiguse ise küsima:

```yaml
  build:
    permissions:
      contents: read
      packages: write
```

Kui job ütleb `permissions`, on tal ainult need õigused, mis on loetletud. Teised on `none`. See on **vähima õiguse põhimõte**: `lint` ei vaja kirjutamisõigust kuhugi, seega ei saa ka pahatahtlik action selles jobis midagi muuta.

### Pull request ei pane registrisse

Pull request'i ajal on kood veel üle vaatamata. Ehitus käib, et näha, kas image tuleb kokku, aga `docker push` ainult main harus:

```yaml
      - name: Saada image GHCR-i
        if: github.event_name == 'push'
        run: docker push "$IMAGE:$VERSIOON"
```

??? question "Kordamisküsimus"

    vm3-s töötab `labori-app:main`. Kasutaja teatab veast. Kuidas saad teada, milline commit vm3-s töötab? Mis oleks teisiti, kui tag oleks commit'i räsi?

??? info "Loe juurde"

    - [GHCR: töö konteineriregistriga](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
    - [GITHUB_TOKEN ja permissions](https://docs.github.com/en/actions/tutorials/authenticate-with-github_token)
    - [Paketi avaldamine Actionsist](https://docs.github.com/en/packages/managing-github-packages-using-github-actions-workflows/publishing-and-installing-a-package-with-github-actions)

## 6. Saladused pipeline'is ja serveris

Tänases pipeline'is on kaks saladust kahes kohas:

| Saladus | Kus hoitakse | Kes kasutab | Õigus |
|---|---|---|---|
| `GITHUB_TOKEN` | GitHub loob iga käivituse jaoks | `build` job, `docker push` | kirjutada selle repositooriumi paketti |
| GHCR-i lugemistoken | Ansible Vault, `group_vars/docker/vault.yml` | Ansible vm3-s, `docker login` | ainult lugeda pakette (`read:packages`) |

Kumbki ei ole failis avatud kujul ega workflow logis.

**Actionsi saladused** (**Settings** → **Secrets and variables** → **Actions**) on krüptitud väärtused, mida workflow loeb kujul `${{ secrets.NIMI }}`. GitHub asendab saladuse väärtuse logis tärnidega. Seda ei tasu usaldada: kui saladus on teisendatud (näiteks base64), ei tunne GitHub seda ära. Anna saladus step'ile keskkonnamuutujana ja kasuta seda ainult käsu sisendina:

```yaml
      - name: Logi GHCR-i sisse
        env:
          GHCR_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: echo "$GHCR_TOKEN" | docker login ghcr.io -u "$GITHUB_ACTOR" --password-stdin
```

`--password-stdin` hoiab tokeni käsurealt eemal. Käsurida näeb protsesside nimekirjas (`ps`) iga kasutaja.

Serveri poolel on saladused Ansible Vaultis nagu K2-s. vm3 vajab tokenit, mis lubab ainult lugeda: kui token lekib, saab sellega image'eid tõmmata, aga mitte muuta. Isikliku tokeni (classic) teed GitHubis ise ja annad sellele ainult õiguse `read:packages` ja aegumise kuupäeva. GHCR toetab praegu ainult classic tokeneid. `docker login` salvestab tokeni faili `/root/.docker/config.json` ilma krüptimata, seega on lühike aegumine ja kitsas õigus olulised.

??? question "Kordamisküsimus"

    Miks ei kasuta vm3 tõmbamiseks sama `GITHUB_TOKEN`-it, millega pipeline image'i registrisse pani?

??? info "Loe juurde"

    - [Actionsi saladused](https://docs.github.com/en/actions/how-tos/write-workflows/choose-what-workflows-do/use-secrets)
    - [Turvalisus Actionsis: skriptide süstimine](https://docs.github.com/en/actions/reference/security/secure-use)
    - [Isiklikud tokenid](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens)

## 7. Tarne VPN-i taha

GitHubi runner on GitHubi andmekeskuses. Sinu VM-id on kooli võrgus aadressidel `192.168.x.x` ja neile pääseb ainult VPN-iga. Runner ei saa vm3-ga SSH-ühendust luua ja see on õige: tootmisserverid ei peagi internetist ligipääsetavad olema.

Lahendusi on neli:

| Viis | Kuidas | Plussid | Miinused |
|---|---|---|---|
| runner ühendub serverisse (push) | GitHubi runner teeb SSH-ga `ansible-playbook` | lihtne, kõik ühes workflow's | server peab olema internetist kättesaadav |
| self-hosted runner võrgu sees | runneri programm vm1-s küsib GitHubilt töid, ühendus läheb ainult väljapoole | täisautomaatne, tulemüüris pole vaja auku | runner on sinu masin: hooldus, turvalisus, ligipääs kõigile serveritele |
| server tõmbab ise (pull, GitOps) | serveris või klastris olev agent jälgib registrit või Giti ja paigaldab muudatuse | serverisse ei ühendu keegi | vaja agenti (Argo CD, Flux); K5 Kubernetes'is |
| inimene käivitab tarne | `ansible-playbook` control node'ist pärast rohelist pipeline'i | inimene otsustab, millal; sama `site.yml` mis enne | paigaldus pole automaatne |

Täna kasutad viimast: pipeline paneb image'i registrisse ja sina käivitad vm1-st `site.yml`-i koos versiooniga. Kõik, mis vm3 vajab, on väljaminev HTTPS-ühendus `ghcr.io`-sse. Ansible'i roll `stack` muutub vähe: Compose-failis on `image:` registri aadressiga ja `build:` kaob, sest vm3 enam ei ehita.

```bash
git pull
ansible-playbook site.yml -J -e app_versioon=$(git rev-parse HEAD | cut -c1-7)
```

Versioon on main haru viimase commit'i lühiräsi, sama, mille pipeline image'i tag'iks pani. `-e` annab muutujale väärtuse, mis on kõige kõrgema eelistusega (K2 §3). Leht näitab pärast tarnet sama räsi: `Labori seis 3f2a9c1`.

Self-hosted runneri lisamiseks peab olema repositooriumi administraator. Kodutöö lisas on see kirjeldatud, aga klassis seda ei tee.

??? question "Kordamisküsimus"

    Self-hosted runner vm1-s käivitab iga main haru push'i järel `site.yml`-i. Kes saab nüüd vm3-s käske käivitada? Mis peab selleks olema tõsi main haru kohta?

??? info "Loe juurde"

    - [Self-hosted runnerid](https://docs.github.com/en/actions/concepts/runners/self-hosted-runners)
    - [Argo CD](https://argo-cd.readthedocs.io/)

## 8. Kaitstud main haru

Pipeline on kasulik ainult siis, kui selle tulemus loeb. Kui igaüks võib main harusse otse push'ida, jõuab punane kood sinna enne, kui keegi logi vaatab. **Haru kaitse** (GitHubis *ruleset* või vanem *branch protection*) ütleb, mis tingimustel tohib main haru muuta:

- **pull request on kohustuslik.** Otse push main harusse lükatakse tagasi;
- **kontrollid peavad olema rohelised.** PR-i saab liita alles siis, kui nimetatud jobid (`lint`, `build`) on läbinud;
- **ajalugu ei kirjutata üle.** `git push --force` ja haru kustutamine on keelatud.

Viimane on meie organisatsiooni kõigil repositooriumitel juba sees: Classroom 50 kaitseb nii sinu esitusi. Esimesed kaks lülitab juhendaja sinu praktikumi repositooriumis sisse praktikumi B-osas. Seadeid saab muuta ainult administraator ja sul on repositooriumis kirjutamisõigus.

Kaitstud haruga muutub töökäik: muudatus läheb eraldi harusse, sealt PR main harusse, pipeline käivitub PR-i peal ja alles roheline PR liidetakse. Liitmine on push main harusse ja käivitab pipeline'i uuesti, seekord koos `docker push`-iga.

??? question "Kordamisküsimus"

    PR on punane, sest `gitleaks` leidis tokeni commit'ist, mille järgmine commit eemaldas. Mis on kaks viisi pipeline roheliseks saada ja kumb on õige, kui token oli kehtiv?

??? info "Loe juurde"

    - [Rulesets](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-rulesets/about-rulesets)
    - [Nõutud kontrollid](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/troubleshooting-required-status-checks)

## 9. Kokkuvõte

??? note "Põhimõtted ühes tabelis"

    | Põhimõte | Mida see tähendab |
    |---|---|
    | pipeline on kood | `.github/workflows/ci.yml` on repositooriumis ja läbib sama lint'i |
    | trigger, job, step, runner | `on:` mis käivitab; job ühes masinas; step üks käsk või action; runner on iga kord tühi |
    | `needs` | järjekord ja tingimus: ehitus ainult rohelise kontrolli järel |
    | versioonid kinni | `ansible-lint==26.10.1`, `actions/checkout@v7`; homme sama tulemus |
    | lint ei asenda käivitust | lint loeb faile; tõend on ikka masinas |
    | saladused ajaloos | `gitleaks` vaatab kogu ajalugu; lekke järel esmalt tühista saladus |
    | punane pipeline | Actions → job → step → esimene viga |
    | ehita üks kord | image pipeline'is, server ainult tõmbab |
    | tag = commit | `ghcr.io/...:3f2a9c1`; tagasipööramine on eelmine räsi |
    | vähim õigus | `permissions: packages: write` ainult `build` jobil; vm3 token ainult `read:packages` |
    | tarne VPN-i taha | runner ei näe kooli võrku; tarne käivitub vm1-st, vm3 tõmbab image'i ise |
    | kaitstud main | PR ja rohelised kontrollid enne liitmist |

*Järgmine: [praktikum](lab.md). Juhendatud osas kirjutad workflow, mis kontrollib K3 pinu ja leiab sealt vead. Iseseisvas osas paneb pipeline image'i GHCR-i, Ansible tarnib selle vm3-sse ja main haru saab kaitse. Seejärel [kodune õpe ja kodutöö](homework.md).*
