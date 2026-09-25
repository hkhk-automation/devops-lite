# DevOps Lite

IT-infrastruktuuri automatiseerimine täiskasvanud õppijale: käsitsi tööst korratava, versioonihallatud ja kontrollitud muutuseni.

Tempo on kiire: klassis tehakse põhiosa, kodus lõpetatakse ja süvendatakse.

**Maht:** 5 kohtumist × 4 akadeemilist tundi, kohtumised üle nädala (loeng kuni 30 min, ülejäänu praktikum) + 58 h iseseisvat tööd kohtumiste vahel.
**Eeldused:** Linuxi käsurida, SSH, tekstiredaktor, Git baas, GitHubi konto. Bash ja Git on eeldused, mitte teemad.

---

## K1 · Ansible alused: idempotentsus ja esimene playbook

- Miks käsitsi seadistus triivib; käsk vs soovitud olek; imperatiivne vs deklaratiivne
- Halb skript vs idempotentne skript
- Inventar, ad-hoc käsud, moodulid vs `command`/`shell`, faktid (`setup`)
- Esimene playbook: `user`, `package`, `copy`, `service`; `become`
- Idempotentsus ja `changed=0` kui tõend
- Dry run: `--check --diff`
- Drift ja selle parandamine
- SSH-võtmed, `ssh-copy-id`, `~/.ssh/config`
- Sama playbook mitmele masinale, fakti järgi valitud väärtused, blast radius (`--limit`)

**Kodus:** `admin.yml` kolmele masinale; playbook ühele oma töö korduvale tegevusele; teooria küsimused.

## K2 · Ansible sügavamalt: serveripark, mallid, saladused, rollid

- Inventari grupid ja pesastatud grupid, `ansible-inventory --graph`
- `group_vars` ja `host_vars`; muutujate eelistusjärjekord; üks kood, kaks keskkonda (test/prod)
- Jinja2 mallid (`template`): muutujad, tingimused, tsüklid
- Handlerid: taaskäivitus ainult muutuse korral
- `loop`, `when`, `register`
- Ansible Vault: krüptitud muutujad, `--ask-vault-pass`, parool väljaspool Giti
- Rollid: `ansible-galaxy init`, `tasks`, `handlers`, `templates`, `defaults`; `site.yml`
- Tagid ja `--tags`

**Kodus:** teine roll (kasutajad või `chrony`), mõlemad rollid kolmel masinal test/prod muutujatega; drift hunt (istutatud vead).

## K3 · Konteinerid: Docker ja Compose

- Image vs konteiner; `run`, `ps`, `logs`, `exec`, `inspect`
- Dockerfile: `FROM`, `COPY`, `RUN`, `EXPOSE`, `CMD`; kihid ja cache; `.dockerignore`
- Pordid, keskkonnamuutujad, image'i versioonid (tag'id)
- Volume ja bind mount: andmete püsivus
- Võrgud: konteinerid nime järgi
- Compose: mitu teenust, `depends_on`, healthcheck, `.env`, volume'id
- Konteiner vs Ansible'iga seadistatud VM: millal kumb
- Ansible paigaldab Dockeri ja käivitab Compose-stacki sihtserveris

**Kodus:** kolme teenusega stack (rakendus + andmebaas + reverse proxy), mille Ansible roll paigaldab VM-ile; teine jooks `changed=0`.

## K4 · CI/CD: automaatne kontroll, ehitus ja tarne

- Pipeline'i mõte: iga muutus kontrollitakse enne, kui see jõuab serverisse
- GitHub Actions: workflow, trigger (`push`, `pull_request`), job, step, runner
- Kontrollid: `ansible-lint`, `yamllint`, `--syntax-check`, saladuste otsing
- Punane pipeline: logi lugemine, vea leidmine, parandus; revert kui rollback
- Image'i ehitamine pipeline'is ja push registrisse (GHCR), `GITHUB_TOKEN`
- Secrets GitHubis; `needs`: ehitus ainult siis, kui testid läbisid
- Branch protection: `main`-i ei jõua ilma rohelise kontrollita

**Kodus:** pipeline K3 stackile: lint → ehitus → push GHCR-i; tarne VM-ile käib Ansible'iga uue image'i tag'iga.

## K5 · Terraform ja Kubernetes

Kasutame OpenTofut (`tofu`), mis on Terraformi avatud lähtekoodiga haru: sama HCL-keel, samad provider'id, samad käsud (`terraform plan` = `tofu plan`). Mida siin õpid, töötab tööl ka Terraformiga.

**Terraform (esimene pool)**

- Deklaratiivne infra: provider, resource
- Elutsükkel: `init` → `plan` → `apply` → `destroy`; plaani lugemine (`+`, `~`, `-/+`)
- State: mis see on, miks see Giti ei käi, `state list`
- Drift: käsitsi muudatus ja selle tuvastamine plaanis
- Muutujad, outputs, viited ressursside vahel

**Kubernetes (teine pool)**

- Miks orkestreerimine: mis juhtub, kui Compose-stack peab jooksma mitmel masinal
- Klaster (k3s), `kubectl`; Pod, Deployment, Service
- Soovitud olek Kubernetes'is: kustutatud Pod tuleb ise tagasi
- Skaleerimine (`replicas`) ja rolling update uue image'i tag'iga
- ConfigMap ja Secret
- K3 Compose-stack Kubernetes'i manifestideks

**Kodus:** Terraformi moodulid ja Terraform loob → Ansible seadistab (outputs → inventar); K4 image Kubernetes'i Deployment'iks; lõputöö plaan ja algus.

## Lõputöö (~18 h, iseseisev)

Probleem sinu töökohast või kodulaborist. Vähemalt kolm kursuse kihti koos (nt Terraform → Ansible roll → Compose-stack või Kubernetes, pipeline kontrollib). Saladused krüptitud, README-s käivitusjuhis, tõend, et teine jooks ei muuda midagi.

---

## Läbivad tööviisid

**Idempotentsus.** Sama samm annab sama tulemuse ka teisel käivitusel: `mkdir -p` → Ansible `changed=0` → OpenTofu `No changes`.

**Ennusta, siis kontrolli.** Enne olulist käsku kirjuta üles, mida ootad. Kui tulemus erineb, oled midagi valesti mõistnud, ja just see on kõige kasulikum koht õppimiseks.

**Viga on samm.** Igas praktikumis on koht, kus midagi teadlikult ei tööta, et näha, kuidas tööriist selle lahendab.

## Keskkond

- **Control node:** sinu masin (WSL2, oma VM või Linux), kus on Ansible, Docker, OpenTofu, `kubectl` ja Git.
- **Sihtmasinad:** alguses `localhost`, seejärel klastri VM-id, mille aadressid annab juhendaja.

## Esitamine

Iga praktikumi jaoks on Classroom 50 ülesanne. Link tuleb kohtumisel, sellest tekib sulle oma repo.

- Tähtaeg on kirjas Classroom 50-s. Lubatud on üks hilinenud esitus, pärast seda on hinne MA.
- Kursuse läbimiseks on vaja vähemalt 80% praktikumidest. Muidu tuleb kirjalik eksam.
- Reposse ei lähe paroole, võtmeid ega tokeneid.
