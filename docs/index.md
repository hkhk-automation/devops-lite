# DevOps Lite

IT-infrastruktuuri automatiseerimine: käsitsi tööst korratava, versioonihallatud ja kontrollitud muutuseni. Viie kohtumisega ehitad kihthaaval terve pinu, kus iga kiht on kood Gitis.

<figure style="max-width:760px;margin:.8em auto" class="dl-stack" markdown="0">
<svg viewBox="0 0 760 330" role="img" aria-labelledby="dl-stack-title" xmlns="http://www.w3.org/2000/svg">
<title id="dl-stack-title">Kursuse pinu: Git ja CI/CD tarnivad muutuse neljakihilisse taristusse</title>
<style>
.dl-stack svg{width:100%;height:auto;font-family:var(--md-text-font-family,sans-serif)}
.dl-stack .box{fill:var(--md-code-bg-color);stroke:var(--md-default-fg-color--lighter);stroke-width:1.5}
.dl-stack .layer{fill:var(--md-primary-fg-color);stroke:var(--md-primary-fg-color);stroke-width:1.5}
.dl-stack .l1{fill-opacity:.10}.dl-stack .l2{fill-opacity:.18}.dl-stack .l3{fill-opacity:.26}.dl-stack .l4{fill-opacity:.34}
.dl-stack .t{fill:var(--md-default-fg-color);font-size:15px;font-weight:600}
.dl-stack .s{fill:var(--md-default-fg-color--light);font-size:12.5px}
.dl-stack .k{fill:var(--md-default-fg-color);font-size:13px;font-weight:700}
.dl-stack .arrow{stroke:var(--md-default-fg-color--light);stroke-width:2;fill:none}
.dl-stack .head{fill:var(--md-default-fg-color--light)}
</style>
<defs><marker id="dl-ah" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="7" markerHeight="7" orient="auto"><path class="head" d="M0,0 L10,5 L0,10 z"/></marker></defs>
<rect class="box" x="20" y="20" width="180" height="62" rx="8"/>
<text class="t" x="110" y="47" text-anchor="middle">Sinu Git-repo</text>
<text class="s" x="110" y="67" text-anchor="middle">playbookid, Dockerfile, HCL</text>
<path class="arrow" d="M110 82 V130" marker-end="url(#dl-ah)"/>
<rect class="box" x="20" y="134" width="180" height="120" rx="8"/>
<text class="k" x="110" y="160" text-anchor="middle">K4 · CI/CD</text>
<text class="t" x="110" y="182" text-anchor="middle">GitHub Actions</text>
<text class="s" x="110" y="204" text-anchor="middle">lint → test → build</text>
<text class="s" x="110" y="222" text-anchor="middle">image → GHCR</text>
<text class="s" x="110" y="240" text-anchor="middle">roheline = võib tarnida</text>
<path class="arrow" d="M200 194 H232" marker-end="url(#dl-ah)"/>
<text class="s" x="216" y="186" text-anchor="middle">tarne</text>
<rect class="layer l4" x="240" y="20" width="500" height="62" rx="8"/>
<text class="t" x="260" y="47">Mitu masinat, ise taastuv</text>
<text class="s" x="260" y="67">Pod, Deployment, Service, skaleerimine</text>
<text class="k" x="725" y="56" text-anchor="end">K5 · Kubernetes (k3s)</text>
<rect class="layer l3" x="240" y="90" width="500" height="62" rx="8"/>
<text class="t" x="260" y="117">Rakendus konteinerites</text>
<text class="s" x="260" y="137">image, volume, võrk, rootless</text>
<text class="k" x="725" y="126" text-anchor="end">K3 · Podman, Docker, Compose</text>
<rect class="layer l2" x="240" y="160" width="500" height="62" rx="8"/>
<text class="t" x="260" y="187">OS, kasutajad, teenused</text>
<text class="s" x="260" y="207">soovitud olek, changed=0, rollid, Vault</text>
<text class="k" x="725" y="196" text-anchor="end">K1–K2 · Ansible</text>
<rect class="layer l1" x="240" y="230" width="500" height="62" rx="8"/>
<text class="t" x="260" y="257">Virtuaalmasinad</text>
<text class="s" x="260" y="277">plan → apply, state, drift</text>
<text class="k" x="725" y="266" text-anchor="end">K5 · Terraform (OpenTofu)</text>
<text class="s" x="490" y="318" text-anchor="middle">Kooli Proxmoxi klaster: vm1 (control node), vm2, vm3</text>
</svg>
</figure>

| | |
|---|---|
| **Maht** | 5 kohtumist × 4 akadeemilist tundi, üle nädala (loeng kuni 30 min, ülejäänu praktikum) + 58 akadeemilist tundi iseseisvat tööd |
| **Eeldused** | Linuxi käsurida, SSH, tekstiredaktor, Giti alused, GitHubi konto |
| **Keskkond** | kooli Proxmoxis kolm AlmaLinux 9 VM-i; ühendus VS Code Remote-SSH või PowerShelli `ssh` kaudu, kodust VPN-iga |
| **Abi** | kursuse Discord; oma repos issue **Vajan abi** |

---

## Kohtumised

| | Teema | Klassis | Kodus |
|---|---|---|---|
| **[K1](week01/lab.md)** | Ansible alused: idempotentsus ja esimene playbook | halb skript vs soovitud olek; inventar, ad-hoc, faktid; playbook (`user`, `package`, `copy`, `service`); `changed=0`; `--check --diff`; drift; SSH-võtmed; üks playbook kolmele masinale | `admin.yml`, SSH turvamine, paketid, raport, cron, drift; oma playbook |
| **K2** | Ansible sügavamalt: kolmekihiline rakendus | Pinu: vm1 nginx koormusjaotur + PostgreSQL, vm2 ja vm3 rakendus. Grupid `lb`/`app`/`db`; `group_vars` ja `host_vars`, eelistusjärjekord; playbooki ülesehitus: mitu play'd, task-failid (`import_tasks`, `include_tasks`), handlerid, Jinja2 mall (`hostvars`), `loop`, `when`, `register`; rollid ja `site.yml`. Iga sammu juures: kontroll enne käivitamist ja ad-hoc käsk pärast. Vead, mille leiad ja parandad: 502 (SELinux, tulemüür), vale port `host_vars`-ist, vigane mall (`validate`), üks rakendus maas. Lisaülesanne: andmebaasi parool Vaulti. Tõend: `curl` vastab vaheldumisi vm2 ja vm3, üks rakendus maas ja leht töötab edasi | kolmas rakendusserver ainult inventari kaudu; uuendus masinhaaval (`serial`), `delegate_to`/`run_once`, `block`/`rescue`, `requirements.yml`; istutatud vigade otsing |
| **K3** | Konteinerid: Podman, Docker ja Compose | lühike Dockeri kordus (image, konteiner, Dockerfile, volume, võrk); **Podman**: AlmaLinuxi vaikimisi tööriist, sama CLI, ilma deemonita ja ilma root'ita, pod'id; Docker vs Podman: millal kumb; Compose (`docker compose` / `podman compose`): `depends_on`, healthcheck, `.env`; Ansible paigaldab stacki | kolme teenusega stack (rakendus + andmebaas + reverse proxy) Ansible rolliga VM-ile, teine jooks `changed=0` |
| **K4** | CI/CD: automaatne kontroll, ehitus ja tarne | GitHub Actions (trigger, job, step, runner); `ansible-lint`, `yamllint`, saladuste otsing; punase pipeline'i lugemine; image GHCR-i; secrets, `needs`; branch protection | pipeline K3 stackile: lint → ehitus → GHCR; tarne VM-ile Ansible'iga |
| **K5** | Terraform (OpenTofu) ja Kubernetes | `init` → `plan` → `apply` → `destroy`; state ja drift; muutujad, outputs; k3s, `kubectl`; Pod, Deployment, Service; skaleerimine, rolling update; ConfigMap, Secret | lõputöö |

K5-s kasutame OpenTofut (`tofu`): Terraformi avatud lähtekoodiga haru, sama keel ja samad käsud (`terraform plan` = `tofu plan`).

## Lõputöö

Üks probleem oma VM-idest või kodulaborist, lahendatud vähemalt kolme kursuse kihiga (nt Terraform → Ansible roll → Compose või Kubernetes, pipeline kontrollib). Saladused krüptitud, README-s käivitusjuhis ja tõend, et teine jooks ei muuda midagi.

---

## Läbivad tööviisid

| | |
|---|---|
| **Idempotentsus** | sama samm annab teisel käivitusel sama tulemuse: `mkdir -p` → Ansible `changed=0` → OpenTofu `No changes` |
| **Ennusta, siis kontrolli** | enne olulist käsku kirjuta üles, mida ootad. Kui tulemus erineb, oled leidnud koha, kus õppida |
| **Viga on samm** | igas praktikumis läheb midagi valesti, kas teadlikult (drift, halb skript) või nii nagu päris serveris (tulemüür, SELinux, 502). Leiad põhjuse ja parandad selle koodiga, mitte käsitsi |

## Esitamine

| | |
|---|---|
| **Kuhu** | iga praktikum on Classroom 50 ülesanne; link tuleb kohtumisel ja sellest tekib sulle oma repo |
| **Tähtaeg** | kirjas Classroom 50-s; lubatud on üks hilinenud esitus, pärast seda on hinne MA |
| **Läbimine** | vähemalt 80% praktikumidest, muidu kirjalik eksam |
| **Reegel** | reposse ei lähe paroole, võtmeid ega tokeneid |
