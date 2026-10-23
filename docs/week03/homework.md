# K3 · Kodune õpe ja kodutöö

Kodutöö on vabatahtlik boonus: hinne tuleb klassitöö kontrollidest K1–K5. Kodutöö läheb samasse reposse, kuhu klassitöö. Tähtaeg on kirjas Classroom 50-s. Kodutöös viid klassis ehitatud pinu edasi: kaks rakenduse koopiat proxy taga, Podmani konteiner systemd teenusena, vigase versiooni automaatne tagasipööramine ja Ansible'i Docker-moodul. Juhiseid on vähem kui klassis. Kui jääd kinni, küsi Discordis või ava oma repos issue **Vajan abi** ja kirjuta sinna käsk ning veateade.

| Osa | Mida teed | Esitad | Hinne |
|---|---|---|---|
| [I](#i-lugemine-ja-kusimused) | loed, uurid, vastad küsimustele | `markmed.md`, `vastused.md` | boonus |
| [H1](#h1-kaks-rakenduse-koopiat-proxy-taga-boonus) | kaks `app` koopiat, proxy jagab koormust | `logid/kaks_koopiat.txt` | boonus |
| [H2](#h2-podmani-konteiner-systemd-teenuseks-boonus) | Quadlet: konteiner on vm1-s teenus | `quadlet/`, `logid/quadlet.txt` | boonus |
| [H3](#h3-vigane-versioon-poorab-ise-tagasi-boonus) | uuendus, mis vigase versiooni tagasi pöörab | `uuendus.yml`, `logid/tagasi.txt` | boonus |
| [H4](#h4-moodul-command-i-asemel-boonus) | `community.docker.docker_compose_v2` | `requirements.yml`, `logid/moodul_teine_jooks.txt` | boonus |
| [H5](#h5-leia-kolleegi-vead-boonus) | neli istutatud viga kaustas `vead/` | `vead.md` | boonus |
| [III](#iii-eneseanaluus) | eneseanalüüs | `vastused.md` lõpus | – |

Tulemust näed pärast iga push'i: **Actions** → **Autograde**. Kodutöö kontrollide nimi algab sõnaga BOONUS ja need punkte ei anna: iga läbitud boonuskontroll annab [laboritahvlil](../tahvel.md) tähe ja XP-d. Punane rist Actionsis tähendab ainult, et boonus on tegemata.

## I · Lugemine ja küsimused

Enne küsimusi loe läbi:

??? note "Lugemisnimekiri"

    | Mida | Kus |
    |---|---|
    | Loeng: klassis käsitlemata osad §3, §5, §8, §9 | [K3 loeng](lecture.md) |
    | Dockerfile'i head tavad | [docs.docker.com](https://docs.docker.com/build/building/best-practices/) |
    | Compose: käivitusjärjekord | [docs.docker.com](https://docs.docker.com/compose/how-tos/startup-order/) |
    | Podman rootless | [github.com/containers/podman](https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md) |
    | Docker ja pakettfilter | [docs.docker.com](https://docs.docker.com/engine/network/packet-filtering-firewalls/) |

### Uuri → `markmed.md`

1. Ava `podman image history --no-trunc localhost/labori-app:1.0` vm1-s. Kirjuta iga kihi kohta üks rida: mis käsk selle tegi ja kui suur see on. Milline kiht on suurim ja miks?
2. Ava [postgres image'i leht](https://hub.docker.com/_/postgres) ja leia jaotis keskkonnamuutujate kohta. Kirjuta kolm muutujat peale `POSTGRES_PASSWORD`, `POSTGRES_USER` ja `POSTGRES_DB`, ühe lausega, milleks need on. Mida ütleb leht selle kohta, millal neid muutujaid arvestatakse?

### Vasta küsimustele → `vastused.md`

Vasta lühidalt oma sõnadega, üks-kaks lauset igale.

1. A3-s andis `DB_HOST=localhost` vea, kuigi vm1-s kuulas pordil 5432 PostgreSQL. Miks? (lab A3, loeng §5)
2. B4-s parandasid proxy seadistuse ja handler laadis selle uuesti ilma konteinerit taaskäivitamata. Miks ühendasid Compose'is kausta, mitte ühe faili? (lab B2, B4)
3. B6-s vastas leht pordil 80, kuigi firewalld-s polnud porti avatud. Mida see tähendab, kui keegi avaldab andmebaasi pordi? (lab B6, loeng §5)

## II · Harjutused

Kõik harjutused käivad sama pinu peale. Kui klassitöö B-osa on pooleli, tee see enne lõpuni: harjutused eeldavad, et `site.yml` annab `changed=0` ja leht vastab.

!!! tip "Soovitatav töökäik"

    1. `ansible-playbook <fail>.yml --syntax-check`
    2. `ansible-playbook <fail>.yml --check --diff`
    3. teises terminalis vm1-s käib kontroll: `while true; do curl -s -o /dev/null -w "%{http_code} " http://$VM3/; sleep 0.5; done`
    4. päris jooks, siis teine jooks tõendiks

### H1 · Kaks rakenduse koopiat proxy taga · boonus

Selle ülesande lõpuks jookseb vm3-s kaks `app` konteinerit ja proxy jagab päringud nende vahel. K2-s tegid sama kahe VM-iga ja Jinja2 malliga.

- koopiate arv on muutuja `app_koopiad` `group_vars/all.yml`-is ja jõuab `.env`-i kaudu Compose-faili;
- Compose-failis on teenusel `app` koopiate arv (`deploy.replicas`), mitte kaks eraldi teenust;
- proxy leiab mõlemad koopiad nime `app` järgi, ka siis, kui koopiate arv muutub pärast proxy käivitumist;
- teine jooks on `changed=0`.

??? tip "Vihje 1: kuidas Compose koopiaid loob"

    [Compose: deploy.replicas](https://docs.docker.com/reference/compose-file/deploy/#replicas). Muutuja vaikeväärtusega: `${APP_KOOPIAD:-1}`. Kontrolli: `ssh vm3 "cd /opt/labor && docker compose ps app"`.

??? tip "Vihje 2: miks proxy näeb ainult üht"

    Compose'i võrgu DNS vastab nimele `app` kõigi koopiate aadressidega. nginx lahendab `upstream`-is olevad nimed üks kord, käivitumisel. Kui koopia tuli juurde hiljem, ei tea nginx sellest. Variandid: laadi proxy pärast muutust uuesti (millise handleriga?) või lase nginx-il nime ise uuesti lahendada: `resolver 127.0.0.11 valid=10s;` ja `proxy_pass` muutujaga. [nginx: resolver](https://nginx.org/en/docs/http/ngx_http_core_module.html#resolver).

Valmis, kui
{ .silt }

- [ ] `curl` vastab vaheldumisi kahe erineva konteineri ID-ga
- [ ] `app_koopiad: 1` ja tagasi `2` töötab ilma käsitsi sekkumiseta
- [ ] `logid/kaks_koopiat.txt`-is on vähemalt kaks erinevat `Vastas:` rida (Autograde loeb neid)

Esitad
{ .silt }

```bash
for i in $(seq 1 6); do curl -s http://$VM3/ | grep Vastas; done | tee logid/kaks_koopiat.txt
```

### H2 · Podmani konteiner systemd teenuseks · boonus

Selle ülesande lõpuks käivitab vm1-s systemd sinu kasutaja nimel andmebaasi ja rakenduse konteineri, ka pärast vm1 taaskäivitust ja ilma, et sa oleksid sisse logitud.

Podmanil pole deemonit, mis `restart:` reeglit täidaks. Selle teeb systemd ja kirjelduse kirjutad **Quadlet**-failidena:

- kaust `~/.config/containers/systemd/`, failid `labor.network`, `labor-db.container`, `labori-app.container`;
- andmebaas kasutab sama volume'it `labor-andmed` ja sama `~/labor.env` faili;
- rakendus kuulab vm1 pordil 8092, käivitub alles pärast andmebaasi ja taaskäivitub vea korral;
- teenused käivituvad ka siis, kui sa pole sisse logitud.

Enne alustamist võta A-osa konteinerid ja pod maha (`podman pod rm -f labor-pod`), et need ei hoiaks volume'it lukus.

??? tip "Vihje: failid ja käsud"

    [podman-systemd.unit(5)](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html): jaotised `[Container]` (võtmed `Image`, `Network`, `Volume`, `EnvironmentFile`, `Environment`, `PublishPort`), `[Unit]` (`Requires`, `After`), `[Service]` (`Restart`), `[Install]` (`WantedBy=default.target`). `%h` on sinu kodukaust. Võrgu nimi teiste failide jaoks on `labor.network`, teenuse nimi `labor-db.service`.

    ```bash
    systemctl --user daemon-reload
    systemctl --user start labori-app
    systemctl --user status labori-app
    sudo loginctl enable-linger $USER
    ```

    Kui `daemon-reload` järel teenust pole, näitab vea `/usr/libexec/podman/quadlet -dryrun -user`.

Proovi taaskäivitust: `sudo systemctl reboot`, oota, ühendu uuesti ja `curl localhost:8092/` ilma ühegi `podman` käsuta.

Valmis, kui
{ .silt }

- [ ] `systemctl --user status labori-app` näitab `active (running)`
- [ ] pärast vm1 taaskäivitust vastab `curl localhost:8092/` ja loendur jätkab
- [ ] repo kaustas `quadlet/` on sinu kolm faili (Autograde otsib `.container` faile)

Esitad
{ .silt }

```bash
mkdir -p quadlet
cp ~/.config/containers/systemd/* quadlet/
{ systemctl --user status labori-app --no-pager; curl -s localhost:8092/; } | tee logid/quadlet.txt
```

### H3 · Vigane versioon pöörab ise tagasi · boonus

Selle ülesande lõpuks on playbook `uuendus.yml`, mis viib rakenduse uuele versioonile ja pöörab vigase versiooni automaatselt tagasi eelmisele.

Rakendus ei käivitu, kui versiooni nimi lõpeb `-katki`-ga (vaata `app.py` algust). Image'i nimes on versioon (`localhost/labori-app:${APP_VERSIOON}`), seega jääb eelmise versiooni image vm3-sse alles ja tagasipööramine ei vaja uut ehitust.

Uuendus käib nagu K2 kodutöös: uus versioon on `group_vars/all.yml`-is, eelmine tuleb käsurealt.

```bash
ansible-playbook uuendus.yml -e eelmine_versioon=1.1 | tee logid/tagasi.txt
```

Playbook `uuendus.yml`:

- paigaldab `.env`-i uue versiooniga ja käivitab `docker compose up -d --build --wait`;
- kui see kukub, paigaldab `.env`-i eelmise versiooniga, käivitab pinu uuesti ja peatab siis playbooki veaga;
- leht töötab pärast tagasipööramist eelmise versiooniga.

Proovi kaks korda: kõigepealt `app_versioon: "1.1"` (`-e eelmine_versioon=1.0`), mis peab õnnestuma, siis `"1.2-katki"` (`-e eelmine_versioon=1.1`), mis peab tagasi pöörama. Lõpuks pane `group_vars/all.yml`-i tagasi `"1.1"`.

??? tip "Vihje: moodulid ja ülesehitus"

    `block` / `rescue`, rescue lõpus `ansible.builtin.fail`. Tagasipööramisel sama mall teise väärtusega: `ansible.builtin.include_role` (`name: stack`, `tasks_from`) koos `vars:`-iga või otse `template` task, millel on `vars: { app_versioon: "{{ eelmine_versioon }}" }`.

    Pane `docker compose up` käsk `block`-i sisse task'ina, mitte handlerina. Handleri viga ei pruugi `rescue`-sse jõuda, sest handlerid jooksevad oma järjekorras.

??? question "Mõtle (vabatahtlik)"

    Kui kaua oli leht vigase versiooni ajal maas? Kuidas oleks H1 kahe koopiaga võimalik uuendada nii, et leht ei kao kordagi?

Valmis, kui
{ .silt }

- [ ] `logid/tagasi.txt`-is on näha, et `rescue` jooksis ja playbook lõppes veaga
- [ ] pärast seda näitab leht `Labori seis 1.1`
- [ ] `uuendus.yml`-is on `block` ja `rescue` (Autograde otsib neid)

Esitad
{ .silt }

`uuendus.yml`, `logid/tagasi.txt`

### H4 · Moodul `command`-i asemel · boonus

Selle ülesande lõpuks käivitab `stack` roll pinu moodulit `community.docker.docker_compose_v2` kasutades ja sa tead, kas see on sinu versioonidega parem kui klassi lahendus.

- `requirements.yml` loetleb `community.docker` versiooniga, mis töötab `ansible-core 2.14`-ga (vaata kollektsiooni `meta/runtime.yml` või Galaxy lehte, välja *Requires Ansible*);
- `stack` rollis asendab moodul kontroll-task'i, `Kõik teenused käivad` task'i ja `Stack uueneb` handleri;
- teine jooks on `changed=0` ka siis, kui moodul ise ehitab (`build`).

??? tip "Vihje: parameetrid"

    [docker_compose_v2](https://docs.ansible.com/ansible/latest/collections/community/docker/docker_compose_v2_module.html): `project_src`, `state: present`, `build`, `wait`. Moodul loeb Compose'i väljundit, seega sõltub ta Compose'i versioonist. Kui moodul annab vm3 Compose'i versiooniga parsimise vea, on see samuti tulemus: kirjuta `vastused.md`-sse veateade ja mida see sinu arvates tähendab.

Kirjuta `vastused.md`-sse kaks lauset: kumba lahendust (klassi oma või moodul) kasutaksid päris süsteemis ja miks.

Valmis, kui
{ .silt }

- [ ] `requirements.yml`-is on `community.docker` kinnitatud versiooniga
- [ ] `stack` rollis on `docker_compose_v2` (Autograde otsib seda)
- [ ] `logid/moodul_teine_jooks.txt`-is on `changed=0` või `vastused.md`-s on selgitus, miks mitte

Esitad
{ .silt }

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml | tee logid/moodul_teine_jooks.txt
```

### H5 · Leia kolleegi vead · boonus

Selle ülesande lõpuks oled leidnud neli viga, mille kolleeg jättis kausta `vead/`, ja kirjutanud iga kohta, kuidas sa selle leidsid.

Kaustas `vead/` on neli faili. Igaüks on sinu repo mõne faili koopia, milles on üks viga. Iga faili jaoks:

1. vaata faili ja ennusta, mis juhtub;
2. kopeeri see õigesse kohta (failis endas on kommentaar, kuhu), jooksuta `site.yml` ja vaata, mis päriselt juhtub;
3. taasta oma fail: `git checkout -- <fail>` ja jooksuta uuesti, kuni `changed=0`.

!!! warning "Tähelepanu: kolm viga ei anna veateadet"

    Ainult üks viga teeb lehe katki. Ülejäänud kolm jätavad kõik roheliseks ja on ohtlikud alles hiljem: turvaaugu, andmekao või root'ina jooksva protsessina. Kontrolli tulemust masinas (`docker compose ps`, `docker compose exec app id`, `docker volume ls`, `down` ja uuesti üles), mitte ainult playbooki värvi.

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

??? note "Vabatahtlik: parooli vahetus, skaneerimine, väiksem image"

    Parooli vahetus: kirjuta `stack` rolli task'id, mis vahetavad andmebaasi parooli enne `.env`-i (`docker compose exec db psql -U labor -c "ALTER ROLE ..."`), nii et B5 katse lõpeb roheliselt. Mida teed, et parool ei satuks Ansible'i väljundisse?

    Skaneerimine: paigalda vm3-sse [Trivy](https://trivy.dev/latest/getting-started/installation/) ja käivita `trivy image localhost/labori-app:1.1`. Mitu kõrge raskusega haavatavust leidis? Mitu neist on sinu koodis ja mitu alusimage'is?

    Väiksem image: ehita sama rakendus `python:3.13-alpine` alusel ja võrdle `docker images` suurusi. Mis läks lihtsamaks, mis keerulisemaks?
