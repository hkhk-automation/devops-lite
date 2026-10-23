# K3 · Praktikum: rakendus konteinerites

Tänase lõpuks jookseb K2 rakendus „Labori seis“ konteinerites kahel moel. vm1-s ehitad sellest image'i ja käivitad selle Podmaniga koos PostgreSQL-iga, kõik tavakasutajana. vm3-s paigaldab Ansible Dockeri ja kogu pinu (proxy, rakendus, andmebaas) ühe Compose-failiga, teine jooks ei muuda midagi ja andmed jäävad alles ka siis, kui kõik konteinerid kustutad. Rakenduse kood on sama mis K2-s. Muutub ainult pakend.

<figure style="max-width:690px;margin:.8em auto" class="lx" markdown="0">
<svg viewBox="0 0 690 300" role="img" aria-label="vm1: Podman, rakendus ja andmebaas konteinerites ning pod. vm3: Docker Compose, proxy, app ja db. Ansible jookseb vm1-st vm3-le" xmlns="http://www.w3.org/2000/svg">
<style>.lx svg{width:100%;height:auto;font-family:var(--md-text-font-family,sans-serif)}.lx .box{fill:var(--md-code-bg-color);stroke:var(--md-default-fg-color--lighter);stroke-width:1.2}.lx .hi{fill:var(--md-primary-fg-color);fill-opacity:.16;stroke:var(--md-primary-fg-color);stroke-width:1.5}.lx .ok{fill:#2e7d32;fill-opacity:.14;stroke:#2e7d32;stroke-width:1.3}.lx .t{fill:var(--md-default-fg-color);font-size:15px;font-weight:700}.lx .b{fill:var(--md-default-fg-color);font-size:14.5px;font-weight:600}.lx .s{fill:var(--md-default-fg-color--light);font-size:13px}.lx .a{stroke:var(--md-default-fg-color--light);stroke-width:1.5;fill:none}.lx .d{stroke-dasharray:4 3}.lx .h{fill:var(--md-default-fg-color--light)}</style>
<defs><marker id="lxp" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path class="h" d="M0,0 L10,5 L0,10 z"/></marker></defs>
<rect class="hi" x="10" y="10" width="320" height="250" rx="10"/><text class="t" x="24" y="34">vm1 · osa A · Podman (rootless)</text>
<rect class="box" x="24" y="48" width="292" height="40" rx="6"/><text class="b" x="170" y="73" text-anchor="middle">podman build → localhost/labori-app:1.0</text>
<rect class="box" x="24" y="98" width="140" height="70" rx="6"/><text class="b" x="94" y="120" text-anchor="middle">võrk labor</text><text class="s" x="94" y="138" text-anchor="middle">app :8090</text><text class="s" x="94" y="154" text-anchor="middle">db (nimi järgi)</text>
<rect class="box" x="176" y="98" width="140" height="70" rx="6"/><text class="b" x="246" y="120" text-anchor="middle">pod labor-pod</text><text class="s" x="246" y="138" text-anchor="middle">app + db :8091</text><text class="s" x="246" y="154" text-anchor="middle">ühine localhost</text>
<rect class="ok" x="24" y="178" width="292" height="30" rx="6"/><text class="s" x="170" y="198" text-anchor="middle">volume labor-andmed (andmed jäävad alles)</text>
<rect class="box" x="24" y="216" width="292" height="34" rx="6"/><text class="s" x="170" y="238" text-anchor="middle">repo, Ansible, git push</text>
<line class="a" x1="316" y1="233" x2="378" y2="140" marker-end="url(#lxp)"/><text class="s" x="350" y="200" text-anchor="middle">ansible</text>
<rect class="hi" x="360" y="10" width="320" height="250" rx="10"/><text class="t" x="374" y="34">vm3 · osa B · Docker + Compose</text>
<rect class="box" x="374" y="48" width="292" height="34" rx="6"/><text class="s" x="520" y="70" text-anchor="middle">roll docker: repo, paketid, teenus, grupp</text>
<rect class="hi d" x="374" y="92" width="292" height="158" rx="8"/><text class="s" x="386" y="110">/opt/labor · projekt labor</text>
<rect class="box" x="386" y="120" width="80" height="54" rx="6"/><text class="b" x="426" y="143" text-anchor="middle">proxy</text><text class="s" x="426" y="160" text-anchor="middle">:80</text>
<line class="a" x1="466" y1="147" x2="480" y2="147" marker-end="url(#lxp)"/>
<rect class="box" x="482" y="120" width="80" height="54" rx="6"/><text class="b" x="522" y="143" text-anchor="middle">app</text><text class="s" x="522" y="160" text-anchor="middle">build</text>
<line class="a" x1="562" y1="147" x2="576" y2="147" marker-end="url(#lxp)"/>
<rect class="box" x="578" y="120" width="80" height="54" rx="6"/><text class="b" x="618" y="143" text-anchor="middle">db</text><text class="s" x="618" y="160" text-anchor="middle">postgres</text>
<rect class="ok" x="578" y="184" width="80" height="26" rx="6"/><text class="s" x="618" y="202" text-anchor="middle">andmed</text>
<text class="s" x="520" y="236" text-anchor="middle">roll stack: failid, .env, up --wait</text>
<text class="s" x="345" y="288" text-anchor="middle">Sama Containerfile ja sama image töötavad mõlemas.</text>
</svg>
</figure>

Osa A on juhendatud: Podman vm1-s, image, võrk, volume ja pod. Osa B on iseseisvam: Ansible paigaldab vm3-sse Dockeri ja Compose'i pinu, sa leiad, miks proxy annab 502, ja proovid, mis juhtub parooli, peatatud konteineri ja kustutatud konteineritega. Oodatav tulemus ja vihjed on kinnistes plokkides. Tee enne ise ja siis võrdle.

??? abstract "Õpiväljundid"

    Praktikumi lõpuks oskad:

    1. käivitada konteinereid Podmaniga tavakasutajana ja selgitada, mida rootless hostis tähendab;
    2. kirjutada Containerfile'i, ehitada image'i ja näidata, millised kihid tulevad vahemälust;
    3. ühendada konteinerid võrgu kaudu nime järgi ja hoida andmeid volume'is;
    4. panna konteinerid pod'i ja kirjutada pod'ist Kubernetes'i YAML;
    5. paigaldada Ansible'i rolliga Docker ja Compose'i pinu nii, et teine jooks on `changed=0`;
    6. leida proxy 502 põhjus konteineri logist ja panna parandus käima ilma konteinerit kustutamata;
    7. tõendada, et andmed jäävad alles ja käsitsi peatatud teenus tuleb Ansible'iga tagasi.

## Enne alustamist

Selleks peaksid olema K2 praktikumi osa A teinud: `ansible stack -m ping` töötab vm1-s võtmega ja vm3 on sinu inventaris. K2 pinu võib vm-ides käima jääda. vm1-s kasutad porte 8088–8091, need on vabad. vm3 pordil 80 on veel K1 nginx: sellest tuleb juttu B3-s.

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
    README.md  ULESANNE.md  ansible.cfg  logid  stack  vead

    ./stack:
    app  nginx

    ./stack/app:
    app.py

    ./stack/nginx:
    default.conf
    ...
    ```

    `stack/app/app.py` on sama rakendus mis K2-s. `stack/nginx/default.conf` on kolleegi jäetud proxy seadistus, sellest tuleb juttu B4-s. `stack/.env.naide` on näidisseaded (`ls -a stack`). Kaust `vead/` on kodutöö jaoks.

Kontrollnimekiri on su repos **Issues** all: issue Lab 03 · Rakendus konteinerites. Märgi ruut, kui osa on tehtud. Kinni? Küsi Discordis või ava issue mallist **Vajan abi**.

??? note "Mis repos lõpuks on"

    ```
    <sinu-repo>/
    ├── ansible.cfg
    ├── inventory.ini
    ├── site.yml
    ├── group_vars/
    │   └── all.yml
    ├── roles/
    │   ├── docker/   tasks/ defaults/
    │   └── stack/    tasks/ handlers/ templates/ defaults/
    ├── stack/                    # see läheb vm3-sse /opt/labor alla
    │   ├── compose.yaml
    │   ├── .env.naide
    │   ├── app/
    │   │   ├── Containerfile
    │   │   ├── .dockerignore
    │   │   └── app.py
    │   └── nginx/
    │       └── default.conf
    ├── README.md
    └── logid/
        ├── podman.txt
        ├── labor-pod.yaml
        ├── site_teine_jooks.txt
        ├── stack.txt
        └── andmed_alles.txt
    ```

    Kaust `stack/` on see, mis läheb serverisse: rakendus ja selle kooslus. `roles/` ütleb, kuidas see sinna läheb. Sama jaotus mis loengu §1-s: rakendus image'is ja Compose'is, server Ansible'is.

!!! tip "Soovitatav töökäik"

    Iga konteineri käsu järel vaata kolme asja: `podman ps -a` (kas jookseb või lõpetas), `podman logs <nimi>` (mida rakendus ütles), `curl` (kas vastab). Ansible'i sammudes sama mis K2-s: `--syntax-check`, `--check --diff`, jooks, siis ad-hoc kontroll.

!!! warning "Kui näed `toomanyrequests`"

    Docker Hub piirab anonüümseid tõmbeid IP kohta ja klass on ühe IP taga. Asenda image'i nimes `docker.io/library/` aadressiga `mirror.gcr.io/library/` (näiteks `mirror.gcr.io/library/postgres:17-alpine`). Sisu on sama.

## A · Juhendatud osa (vm1, Podman)

### A1 · Podman ja esimesed konteinerid

Selle sammu lõpuks jooksevad vm1-s konteinerid sinu kasutajana ja sa oskad näidata, kes on konteineri `root` hostis.

Kõik osa A käsud käivad vm1-s tavakasutajana. `sudo`-t on vaja ainult paigalduseks.

```bash
rpm -q podman || sudo dnf install -y podman
podman version
podman info --format '{{.Host.Security.Rootless}}'
```

??? success "Oodatav tulemus"

    ```
    Client:       Podman Engine
    Version:      5.8.2
    ...
    true
    ```

    Üks plokk, `Client`. Serverit pole, sest deemonit pole. `true` tähendab, et töötad rootless.

Esimene konteiner. Ennusta enne: mis distributsiooni ja mis tuuma versiooni näed konteineris?

```bash
podman run --rm docker.io/library/alpine:3.22 sh -c 'grep PRETTY /etc/os-release; uname -r'
uname -r
```

??? success "Oodatav tulemus"

    ```
    PRETTY_NAME="Alpine Linux v3.22"
    5.14.0-...el9_...x86_64
    5.14.0-...el9_...x86_64
    ```

    Kasutajaruum on Alpine, tuum on vm1 AlmaLinuxi oma. `--rm` kustutab konteineri, kui protsess lõpetab.

Nüüd teenus, mis jääb käima. `-d` käivitab taustal, `-p 8088:80` suunab vm1 pordi 8088 konteineri pordile 80:

```bash
podman run -d --name proovi -p 8088:80 docker.io/library/nginx:1.30-alpine
curl -s localhost:8088 | grep title
podman ps
```

Vaata, kes nginx-i protsessid hostis on. `podman top` näitab mõlemat vaadet: kasutaja konteineris (`USER`) ja hostis (`HUSER`):

```bash
podman top proovi user huser pid args
grep "^$USER:" /etc/subuid
```

??? success "Oodatav tulemus"

    ```
    USER   HUSER    PID   COMMAND
    root   1000     1     nginx: master process nginx -g daemon off;
    nginx  165636   ...   nginx: worker process
    ...
    <sinu-kasutaja>:165536:65536
    ```

    Konteineri `root` on hostis sinu UID (siin 1000). Konteineri kasutaja `nginx` (UID 101) on hostis `165536 + 101 - 1 = 165636`, sinu vahemikust failis `/etc/subuid`. Kumbki pole hostis root.

Proovi sama pordiga 80:

```bash
podman run -d --name proovi80 -p 80:80 docker.io/library/nginx:1.30-alpine
podman rm -f proovi proovi80
```

??? success "Oodatav tulemus"

    ```
    Error: rootlessport cannot expose privileged port 80, you can add 'net.ipv4.ip_unprivileged_port_start=80' to /etc/sysctl.conf (currently 1024), or choose a larger port number (>= 1024): ...
    ```

    Tavakasutaja ei saa siduda porte alla 1024. Ära muuda sysctl-i, kasuta kõrgemat porti. vm1 port 80 on niikuinii K2 nginx-i all.

??? question "Mõtle (vabatahtlik)"

    Proovi `podman pull nginx` ilma registrita ja loe väljundi esimest rida. Kust Podman teadis, et `nginx` tähendab `docker.io/library/nginx`? Mis juhtuks nimega, mida selles failis pole?

??? info "Loe juurde"

    - [loeng §2, §6](lecture.md#6-docker-ja-podman)
    - [Podman: rootless](https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md)

### A2 · Pane rakendus image'isse

Selle sammu lõpuks on vm1-s image `localhost/labori-app:1.0` ja sa tead, millised selle kihid ehitatakse koodi muutudes uuesti.

K2-s paigaldas roll `app` Pythoni draiveri, lõi kasutaja, kopeeris koodi ja kirjutas systemd teenuse. Containerfile teeb sama, aga image'i sisse. Mine rakenduse kausta ja loo `Containerfile` neljas osas.

**Alus.** Ametlik Pythoni image, väike variant (`slim`), täisnimega:

```dockerfile
FROM docker.io/library/python:3.13-slim
```

**Teegid ja kasutaja.** Draiver tuleb `pip`-ist kindla versiooniga. Kasutaja on sama mis K2-s. Mõlemad on ühes `RUN`-is, et need oleksid ühes kihis:

```dockerfile
RUN pip install --no-cache-dir psycopg2-binary==2.9.13 \
 && useradd --system --no-create-home labori
```

**Kood.** `WORKDIR` loob kausta ja teeb selle töökaustaks. `COPY` tuleb pärast teeke:

```dockerfile
WORKDIR /app
COPY app.py .
```

**Käivitus.** Kirjuta ise: rakendus jookseb kasutajana `labori`, vaikimisi port on 8080 (keskkonnamuutuja `APP_PORT`), port on dokumenteeritud ja käivituskäsk on exec-kujul `python3 app.py`.

??? example "Näide, kui ikka kinni"
    ```dockerfile
    USER labori
    ENV APP_PORT=8080
    EXPOSE 8080
    CMD ["python3", "app.py"]
    ```

Lisa `.dockerignore`, et Pythoni vahemälu ei satuks image'isse:

```bash
cd ~/<sinu-repo>/stack/app
printf '__pycache__/\n*.pyc\n' > .dockerignore
```

Ehita. Punkt lõpus on **ehituskontekst**: kaust, mille failid `COPY` näeb.

```bash
podman build -t localhost/labori-app:1.0 .
podman images
podman image history localhost/labori-app:1.0 | head -n 9
```

??? success "Oodatav tulemus"

    ```
    STEP 1/8: FROM docker.io/library/python:3.13-slim
    ...
    STEP 2/8: RUN pip install --no-cache-dir psycopg2-binary==2.9.13 && useradd ...
    Successfully installed psycopg2-binary-2.9.13
    ...
    STEP 8/8: CMD ["python3", "app.py"]
    COMMIT localhost/labori-app:1.0
    Successfully tagged localhost/labori-app:1.0
    ```

    `podman images` näitab kaht image'it: `docker.io/library/python` ja `localhost/labori-app`. `history` näitab iga käsu kihti ja suurust. Suurim on `pip install`, `COPY app.py` on mõni kilobait.

Käivita image ilma seadeteta ja vaata, mis juhtub:

```bash
podman run -d --name app-uksi -p 8090:8080 localhost/labori-app:1.0
curl -s -i localhost:8090/health
podman logs app-uksi | tail -n 4
```

??? success "Oodatav tulemus"

    `curl: (52) Empty reply from server`. Logis:

    ```
    Labori seis 1.0 kuulab pordil 8080
    ...
    KeyError: 'DB_HOST'
    ```

    Rakendus käivitus, aga esimene päring kukkus, sest keskkonnamuutujat `DB_HOST` pole. K2-s tuli see failist `/etc/labori-app.env`. Konteinerile annad seaded käivitamisel, mitte image'isse. Seda teed A3-s.

Nüüd vahemälu. Ennusta enne: kui muudad `app.py`-s ühte rida ja ehitad uuesti, millised sammud tehakse uuesti ja millised võetakse vahemälust?

```bash
echo "# proovi vahemälu" >> app.py
time podman build -t localhost/labori-app:1.0 .
git checkout app.py
```

??? success "Oodatav tulemus"

    ```
    STEP 2/8: RUN pip install ...
    --> Using cache 5b1c...
    ...
    STEP 4/8: COPY app.py .
    --> 9a3e...
    ```

    Sammud 1–3 tulevad vahemälust, alates `COPY`-st ehitatakse uuesti. Ehitus võtab mõne sekundi. `git checkout` taastab faili.

??? question "Mõtle (vabatahtlik)"

    Tõsta `COPY app.py .` ajutiselt enne `RUN pip install`-i, muuda `app.py`-d ja ehita uuesti. Mitu sekundit nüüd? Taasta järjekord ja fail.

??? info "Loe juurde"

    - [loeng §4](lecture.md#4-containerfile)
    - [podman-build](https://docs.podman.io/en/latest/markdown/podman-build.1.html)

### A3 · Andmebaas, võrk ja volume

Selle sammu lõpuks vastab vm1 port 8090 rakenduse lehega, andmebaas on teises konteineris ja külastuste arv jääb alles ka siis, kui andmebaasi konteiner kustutada.

Tee kõigepealt koht, kus andmed ja ühendused elavad:

```bash
podman rm -f app-uksi
podman network create labor
podman volume create labor-andmed
```

Seaded lähevad faili, mitte käsureale ja mitte reposse. Fail on sinu kodukaustas ja sellele on õigus ainult sinul:

```bash
cat > ~/labor.env <<'EOF'
POSTGRES_DB=labor
POSTGRES_USER=labor
POSTGRES_PASSWORD=Muuda-mind-2026
DB_NIMI=labor
DB_KASUTAJA=labor
DB_PAROOL=Muuda-mind-2026
EOF
chmod 600 ~/labor.env
```

`POSTGRES_*` loeb andmebaasi image, `DB_*` loeb rakendus. Mõlemale sama fail.

Käivita andmebaas. Konteineri nimi `db` on ka tema nimi võrgus `labor`:

```bash
podman run -d --name db --network labor \
  -v labor-andmed:/var/lib/postgresql/data \
  --env-file ~/labor.env \
  docker.io/library/postgres:17-alpine
podman logs db 2>&1 | tail -n 2
```

??? success "Oodatav tulemus"

    ```
    ... LOG:  database system is ready to accept connections
    ```

    Esimesel käivitusel loob image andmekausta, andmebaasi `labor` ja kasutaja `labor`. Kui viimast rida veel pole, oota paar sekundit.

Nüüd rakendus. K2 harjumusega oleks andmebaasi aadress `localhost`. Ennusta enne, kas see töötab. vm1-s kuulab ju pordil 5432 isegi K2 PostgreSQL.

```bash
podman run -d --name app --network labor -p 8090:8080 \
  --env-file ~/labor.env -e DB_HOST=localhost \
  localhost/labori-app:1.0
curl -s localhost:8090/health
```

??? success "Oodatav tulemus"

    ```
    andmebaas ei vasta: connection to server at "localhost" (::1), port 5432 failed: Connection refused
    ...
    ```

    Konteineri `localhost` on konteiner ise. Selles konteineris ei kuula pordil 5432 keegi. vm1 K2 PostgreSQL on hosti võrgus, `db` konteiner on võrgus `labor`. Kumbki pole rakenduse `localhost`.

Paranda: andmebaas on võrgus nimega `db`.

```bash
podman rm -f app
podman run -d --name app --network labor -p 8090:8080 \
  --env-file ~/labor.env -e DB_HOST=db \
  localhost/labori-app:1.0
curl -s localhost:8090/health
for i in 1 2 3; do curl -s localhost:8090/; echo; done
```

??? success "Oodatav tulemus"

    ```
    ok 3f2a9c1b7d4e
    Labori seis 1.0
    Vastas: 3f2a9c1b7d4e
    Külastused: 3f2a9c1b7d4e=1
    ...
    Külastused: 3f2a9c1b7d4e=3
    ```

    `Vastas:` on nüüd konteineri ID, sest see on konteineri hostname. Loendur kasvab: rakendus kirjutab teise konteineri andmebaasi.

Andmebaasi konteiner on ajutine, andmed ei ole. Kustuta `db` ja ennusta, mis juhtub lehel ja mis juhtub loenduriga pärast uut konteinerit:

```bash
podman rm -f db
curl -s localhost:8090/ | tail -n 1
podman run -d --name db --network labor \
  -v labor-andmed:/var/lib/postgresql/data \
  --env-file ~/labor.env \
  docker.io/library/postgres:17-alpine
sleep 5
curl -s localhost:8090/
```

??? success "Oodatav tulemus"

    Andmebaasita: `Andmebaas: VIGA could not translate host name "db" ...`. Pärast uut konteinerit jätkab loendur sealt, kus pooleli jäi (`=4`), sest andmed on volume'is, mitte konteineris.

Kus volume päriselt on? Rootless režiimis sinu kodukaustas ja failide omanikud on sinu UID-vahemikust:

```bash
podman volume inspect labor-andmed --format '{{.Mountpoint}}'
podman unshare ls -ln "$(podman volume inspect labor-andmed --format '{{.Mountpoint}}')" | head -n 4
```

??? success "Oodatav tulemus"

    ```
    /home/<sinu-kasutaja>/.local/share/containers/storage/volumes/labor-andmed/_data
    total ...
    drwx------. 6 70 70 ... base
    ...
    ```

    `podman unshare` näitab faile nii, nagu konteiner neid näeb: omanik on UID 70, Alpine'i `postgres` kasutaja. Ilma `unshare`-ita näeksid hostis numbrit sinu `/etc/subuid` vahemikust.

Salvesta tõend:

```bash
cd ~/<sinu-repo>
{ podman ps; for i in 1 2 3; do curl -s localhost:8090/; done; } | tee logid/podman.txt
```

??? question "Mõtle (vabatahtlik)"

    Mis juhtub loenduriga, kui teeksid `podman rm -f db` ja `podman volume rm labor-andmed`? Mis Compose'i käsk teeb sama kogu pinuga korraga?

??? info "Loe juurde"

    - [loeng §5](lecture.md#5-andmed-ja-vork)
    - [postgres image: keskkonnamuutujad](https://hub.docker.com/_/postgres)

### A4 · Pane rakendus ja andmebaas pod'i

Selle sammu lõpuks jookseb sama pinu Podmani pod'is ja sul on sellest Kubernetes'i YAML, mida K5-s uuesti näed.

Pod'is on konteineritel üks võrk. Rakendus leiab andmebaasi aadressilt `127.0.0.1`, nagu oleksid nad samas masinas. Port avaldatakse pod'ile.

```bash
podman rm -f app db
podman pod create --name labor-pod -p 8091:8080
```

Käivita pod'is andmebaas. Sama käsk mis A3-s, aga `--network labor` asemel `--pod labor-pod`. Konteineri nimeks pane `labor-db`. Volume on sama, seega andmed on samad.

??? example "Näide, kui ikka kinni"
    ```bash
    podman run -d --pod labor-pod --name labor-db \
      -v labor-andmed:/var/lib/postgresql/data \
      --env-file ~/labor.env \
      docker.io/library/postgres:17-alpine
    ```

Rakendus pod'is. Mis on nüüd `DB_HOST`? Kirjuta käsk ise, konteineri nimi `labor-app`, porti `-p` ei anna, see on pod'il.

??? example "Näide, kui ikka kinni"
    ```bash
    podman run -d --pod labor-pod --name labor-app \
      --env-file ~/labor.env -e DB_HOST=127.0.0.1 \
      localhost/labori-app:1.0
    ```

```bash
podman pod ps
podman ps --pod
curl -s localhost:8091/
```

??? success "Oodatav tulemus"

    `podman ps --pod` näitab kolme konteinerit: sinu kaks ja `…-infra`, mis hoiab pod'i võrku ja porti üleval. Leht vastab ja loendur jätkab A3 numbrist. `Vastas:` on nüüd `labor-pod`, sest pod'i konteineritel on ka ühine hostname.

Kirjuta pod'ist Kubernetes'i kirjeldus ja vaata see läbi:

```bash
podman kube generate labor-pod -f logid/labor-pod.yaml
grep -nE 'kind:|image:|containerPort|hostPort|claimName|DB_HOST' logid/labor-pod.yaml
```

??? success "Oodatav tulemus"

    `kind: Pod`, kaks konteinerit oma image'itega, `hostPort: 8091`, volume `labor-andmed` kui `persistentVolumeClaim` ja `DB_HOST` väärtusega `127.0.0.1`. Faili alguses on kommentaar Podmani versiooniga.

Pane tähele: failis on ka parool avatud kujul, sest `--env-file` muutujad lähevad konteineri kirjeldusse. Meie labori parool on niikuinii avalik. Päris süsteemis eemaldaksid selle enne commit'i. K5-s on selle jaoks Kubernetes'i `Secret`.

Jäta pod käima. Laboritahvel kontrollib, et see vastab.

??? question "Mõtle (vabatahtlik)"

    Miks vastas A3-s rakendus konteineri ID-ga, aga pod'is pod'i nimega? Mis veel peale võrgu ja hostname'i on pod'i konteineritel ühine (`podman pod inspect labor-pod`)?

??? info "Loe juurde"

    - [loeng §6, Pod](lecture.md#pod)
    - [podman-kube-generate](https://docs.podman.io/en/latest/markdown/podman-kube-generate.1.html)

### A5 · Commit

Selle sammu lõpuks on Containerfile ja tõendid GitHubis.

```bash
cd ~/<sinu-repo>
git status
git add stack/app logid
git commit -m "K3 A: image Podmaniga, võrk, volume, pod"
git push
```

`git status` ei tohi näidata faili `~/labor.env`: see on kodukaustas, mitte repos.

## B · Iseseisev osa (vm3, Docker, Compose, Ansible)

vm1-s tegid kõike käsitsi, et näha, mis toimub. Serveris ei tee seda keegi käsitsi. Osas B paigaldab Ansible vm3-sse Dockeri ja sama rakenduse koos andmebaasi ja proxyga. Sammud on antud, osa lahendusest kirjutad ise.

Piirangud kogu B-osas:

- vm3-s muudab kõike Ansible. Käsitsi (`ssh vm3 ...`) teed ainult kontrolle ja katseid, mida juhend palub.
- Compose-faili ega `.env`-i ei kirjuta vm3-s käsitsi. Need tulevad repost.
- Pärast igat parandust: `--check --diff`, siis jooks, siis kontroll.

### B1 · Paigalda Docker rolliga

Selle sammu lõpuks käib vm3-s Docker ja sinu kasutaja saab käivitada `docker` käske ilma `sudo`-ta.

Loo repo juurde `inventory.ini`. vm3 on grupis `docker`:

```ini
[docker]
vm3
```

Rolli kaustad:

```bash
mkdir -p roles/docker/{tasks,defaults}
mkdir -p roles/stack/{tasks,handlers,templates,defaults}
```

Vaikeväärtused `roles/docker/defaults/main.yml`. Kasutaja tuleb control node'i keskkonnast: meil on kõigis masinates sama kasutajanimi.

```yaml
docker_paketid:
  - docker-ce
  - docker-ce-cli
  - containerd.io
  - docker-buildx-plugin
  - docker-compose-plugin
docker_kasutajad:
  - "{{ lookup('ansible.builtin.env', 'USER') }}"
```

Task'id `roles/docker/tasks/main.yml` järgivad [Dockeri paigaldusjuhendit RHEL-i perekonnale](https://docs.docker.com/engine/install/rhel/). Esimesed kaks:

```yaml
- name: Dockeriga vastuolus paketid on eemaldatud
  ansible.builtin.package:
    name:
      - podman
      - buildah
      - runc
    state: absent

- name: Dockeri allkirjavõti on usaldatud
  ansible.builtin.rpm_key:
    key: https://download.docker.com/linux/rhel/gpg
    state: present
```

Podman ja Docker samas masinas lähevad paketitasandil tülli (`runc` ja `containerd.io`). vm3-s on Docker, vm1-s Podman. Võti peab olema usaldatud enne, kui `dnf` sellest repost midagi paigaldab, muidu kukub paigaldus allkirja kontrollil.

Ülejäänud neli kirjuta ise:

- repo fail `https://download.docker.com/linux/rhel/docker-ce.repo` kausta `/etc/yum.repos.d/` (moodul `get_url`, õigused `0644`);
- paketid `docker_paketid` paigaldatud;
- teenus `docker` käib ja käivitub buutimisel;
- iga `docker_kasutajad` liige on lisaks oma gruppidele ka grupis `docker` (moodul `user`, `groups`, `append: true`, `loop`).

??? example "Näide, kui ikka kinni"
    ```yaml
    - name: Dockeri repo on lisatud
      ansible.builtin.get_url:
        url: https://download.docker.com/linux/rhel/docker-ce.repo
        dest: /etc/yum.repos.d/docker-ce.repo
        mode: "0644"

    - name: Docker on paigaldatud
      ansible.builtin.package:
        name: "{{ docker_paketid }}"
        state: present

    - name: Docker käib
      ansible.builtin.service:
        name: docker
        state: started
        enabled: true

    - name: Kasutaja on docker-grupis
      ansible.builtin.user:
        name: "{{ item }}"
        groups: docker
        append: true
      loop: "{{ docker_kasutajad }}"
    ```

`append: true` on oluline. Ilma selleta eemaldab `user` kasutaja kõigist teistest gruppidest, ka `wheel`-ist, ja `sudo` lakkab töötamast.

Loo `site.yml`:

```yaml
- name: Docker vm3-s
  hosts: docker
  become: true
  roles:
    - docker
```

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml
ssh vm3 docker version --format '{{.Server.Version}}'
ssh vm3 docker compose version
```

??? success "Oodatav tulemus"

    ```
    29.x.x
    Docker Compose version v5.x.x
    ```

    `ssh vm3 docker ...` töötab ilma `sudo`-ta, sest uus SSH-sessioon näeb uut gruppi.

??? tip "Kui `ssh vm3 docker version` annab `permission denied ... docker.sock`"

    Grupp kehtib uutes sessioonides. Kui sul on SSH-ühenduste taaskasutus (`ControlMaster` failis `~/.ssh/config`), sulge vana ühendus: `ssh -O exit vm3`.

??? question "Mõtle (vabatahtlik)"

    Sinu kasutaja on nüüd vm3-s `docker`-grupis. Kuidas saaks ta ilma `sudo`-ta lugeda faili `/etc/shadow`? (Vihje: loengu §6.) Mida see tähendab selle kohta, kellele seda gruppi anda?

### B2 · Kirjelda pinu Compose-failina

Selle sammu lõpuks on repos `stack/compose.yaml`, mis kirjeldab andmebaasi, rakendust ja proxy't koos tervisekontrollide ja käivitusjärjekorraga.

Vaata enne näidisseadeid: `cat stack/.env.naide`. Need on muutujad, mida Compose-fail kasutab. Päris `.env` tekib vm3-sse Ansible'i mallist B3-s.

Loo `stack/compose.yaml`. Alusta projekti nime ja andmebaasiga. Kõik on A3-st tuttav, uus on tervisekontroll:

```yaml
name: labor

services:
  db:
    image: docker.io/library/postgres:17-alpine
    environment:
      POSTGRES_DB: ${DB_NIMI}
      POSTGRES_USER: ${DB_KASUTAJA}
      POSTGRES_PASSWORD: ${DB_PAROOL}
    volumes:
      - andmed:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"] # (1)!
      interval: 5s
      timeout: 3s
      retries: 10
    restart: unless-stopped
```

1. `pg_isready` jookseb konteineri sees iga 5 sekundi järel. `$$` tähendab, et muutuja asendab konteineri shell, mitte Compose.

Rakendus. Compose ehitab image'i A2 Containerfile'ist ja paneb sellele versiooniga nime:

```yaml
  app:
    build:
      context: ./app
      dockerfile: Containerfile
    image: localhost/labori-app:${APP_VERSIOON}
    environment:
      APP_NIMI: ${APP_NIMI}
      APP_VERSIOON: ${APP_VERSIOON}
      DB_HOST: db
      DB_NIMI: ${DB_NIMI}
      DB_KASUTAJA: ${DB_KASUTAJA}
      DB_PAROOL: ${DB_PAROOL}
    depends_on:
      db:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "python3", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8080/health', timeout=2)"]
      interval: 5s
      timeout: 3s
      retries: 10
    restart: unless-stopped
```

`python:3.13-slim`-is pole `curl`-i, seega kontrollib rakendus ennast Pythoniga. `urlopen` viskab erindi, kui vastus pole 200, ja käsk lõpeb veakoodiga.

Proxy kirjuta ise:

- image `docker.io/library/nginx:1.30-alpine`;
- hosti port `${LB_PORT}` suunatud konteineri pordile 80;
- repo kaust `./nginx` ühendatud konteineri kausta `/etc/nginx/conf.d` ainult lugemiseks (`:ro`);
- käivitub alles siis, kui `app` on terve;
- taaskäivitub nagu teised.

Lõppu nimega volume `andmed`.

??? example "Näide, kui ikka kinni"
    ```yaml
      proxy:
        image: docker.io/library/nginx:1.30-alpine
        ports:
          - "${LB_PORT}:80"
        volumes:
          - ./nginx:/etc/nginx/conf.d:ro
        depends_on:
          app:
            condition: service_healthy
        restart: unless-stopped

    volumes:
      andmed:
    ```

Miks kaust, mitte üks fail? Kui ühendad ühe faili ja Ansible selle hiljem asendab, näeb konteiner edasi vana faili. Fail on uus, aga konteiner hoiab kinni vanast. Kausta puhul näeb konteiner kausta uut sisu. B4-s on sellest kasu.

Pane tähele, mida failis **pole**: `db` ja `app` ei avalda ühtegi porti. Väljast on nähtav ainult proxy.

??? question "Mõtle (vabatahtlik)"

    Kolleeg lisab `db` alla `ports: ["5432:5432"]`, „et saaks DBeaveriga vaadata“. vm3 tulemüüris pole 5432 lahti. Kas andmebaas on võrgust kättesaadav? Kontrolli B6-s.

??? info "Loe juurde"

    - [loeng §7](lecture.md#7-compose-mitu-teenust-uhes-failis)
    - [Compose: services](https://docs.docker.com/reference/compose-file/services/)

### B3 · Paigalda pinu rolliga `stack`

Selle sammu lõpuks on vm3-s kaust `/opt/labor` Compose-faili, rakenduse ja `.env`-iga ning kõik kolm teenust käivad.

Muutujad, mida `.env` vajab, on `group_vars/all.yml`-is. Sama fail mis K2-s:

```yaml
app_nimi: "Labori seis"
app_versioon: "1.0"

db_nimi: labor
db_kasutaja: labor
db_parool: "Muuda-mind-2026"
```

Rolli vaikeväärtused `roles/stack/defaults/main.yml`:

```yaml
stack_kaust: /opt/labor
lb_port: 80
```

Mall `roles/stack/templates/env.j2`. Muutujate nimed vasakul on need, mida Compose-fail ootab:

```jinja
# {{ ansible_managed }}
APP_NIMI="{{ app_nimi }}"
APP_VERSIOON={{ app_versioon }}
LB_PORT={{ lb_port }}
DB_NIMI={{ db_nimi }}
DB_KASUTAJA={{ db_kasutaja }}
DB_PAROOL={{ db_parool }}
```

Task'id `roles/stack/tasks/main.yml`. Esimene loob kausta. Kirjuta ise: `{{ stack_kaust }}`, õigused `0755`.

Teine kopeerib Compose-faili ja rakenduse kausta. Kaust `stack/` on repo juures, mitte rolli sees, seega tee algab `playbook_dir`-ist:

```yaml
- name: Compose-fail ja rakendus on paigas
  ansible.builtin.copy:
    src: "{{ playbook_dir }}/stack/{{ item }}"
    dest: "{{ stack_kaust }}/"
    mode: preserve
  loop:
    - compose.yaml
    - app
  notify: Stack uueneb
```

Kolmas paigaldab proxy seaded. Sama moodul, `src` on kaust `nginx`, teavitab handlerit `proxy laeb seaded uuesti`. Kirjuta ise.

Neljas paigaldab `.env`-i mallist. Failis on parool: õigused `0600`, `no_log: true`, teavitab handlerit `Stack uueneb`. Kirjuta ise.

??? example "Näide, kui ikka kinni"
    ```yaml
    - name: Stacki kaust on olemas
      ansible.builtin.file:
        path: "{{ stack_kaust }}"
        state: directory
        mode: "0755"

    - name: Proxy seaded on paigas
      ansible.builtin.copy:
        src: "{{ playbook_dir }}/stack/nginx"
        dest: "{{ stack_kaust }}/"
        mode: preserve
      notify: proxy laeb seaded uuesti

    - name: Stacki seaded on paigas
      ansible.builtin.template:
        src: env.j2
        dest: "{{ stack_kaust }}/.env"
        mode: "0600"
      no_log: true
      notify: Stack uueneb
    ```

Viimased kolm task'i teevad käivituse idempotentseks. Kõigepealt käivitatakse ootel handlerid, siis kontrollitakse, mitu teenust käib, ja ainult siis, kui mõni puudub, käivitatakse pinu:

```yaml
- name: Muudatused rakenduvad kohe
  ansible.builtin.meta: flush_handlers

- name: Kontrolli, mis teenused käivad
  ansible.builtin.command: docker compose ps --services --status running
  args:
    chdir: "{{ stack_kaust }}"
  register: stack_kaivad
  changed_when: false # (1)!
  check_mode: false

- name: Kõik teenused käivad
  ansible.builtin.command: docker compose up -d --wait --force-recreate # (2)!
  args:
    chdir: "{{ stack_kaust }}"
  when: stack_kaivad.stdout_lines | length < 3
```

1. Sama muster mis K2 andmebaasi kasutajaga: kontroll ei muuda midagi ja jookseb ka `--check` all.
2. `--wait` ootab, kuni teenused on käivitunud ja terved. Kui mõni jääb haigeks, kukub task ja playbook on punane. `--force-recreate` loob konteinerid uuesti: kui konteiner on katki jäänud (näiteks võrguta, B3), siis lihtne `up -d` seda ei paranda. Andmed on volume'is ja jäävad alles.

Handlerid `roles/stack/handlers/main.yml`. Handlerid jooksevad selles järjekorras, nagu nad failis on, mitte teavitamise järjekorras. Seepärast on pinu käivitus esimene ja proxy seadete laadimine teine: esimesel jooksul peab proxy enne olemas olema.

```yaml
- name: Stack uueneb
  ansible.builtin.command: docker compose up -d --build --wait
  args:
    chdir: "{{ stack_kaust }}"
```

Teine handler laeb nginx-i seaded konteineris uuesti, ilma konteinerit taaskäivitamata. Kirjuta ise: `docker compose exec -T proxy nginx -s reload` kaustas `{{ stack_kaust }}`.

??? example "Näide, kui ikka kinni"
    ```yaml
    - name: proxy laeb seaded uuesti
      ansible.builtin.command: docker compose exec -T proxy nginx -s reload
      args:
        chdir: "{{ stack_kaust }}"
    ```

Lisa roll `site.yml`-i `docker`-rolli järele ja jooksuta. Esimene jooks tõmbab image'id ja ehitab rakenduse, see võtab mõne minuti. Ennusta enne: mis juhtub pordiga 80?

```bash
ansible-playbook site.yml --syntax-check
ansible-playbook site.yml
```

??? success "Oodatav tulemus"

    ```
    RUNNING HANDLER [stack : Stack uueneb] ************************
    fatal: [vm3]: FAILED! => changed=true
      ...
      stderr: |-
        ...
        Error response from daemon: ... failed to bind host port ... 0.0.0.0:80 ...: address already in use
    ```

    `db` ja `app` käivitusid, `proxy` mitte.

Kes pordil 80 kuulab?

```bash
ssh -t vm3 "sudo ss -tlnp | grep ':80 '"
```

??? success "Oodatav tulemus"

    ```
    LISTEN 0  511  0.0.0.0:80  0.0.0.0:*  users:(("nginx",pid=872,fd=6),...)
    ```

    See on K1 nginx, mis paigaldati kõigisse kolme masinasse. vm3-s teeb selle töö nüüd proxy-konteiner, seega hosti nginx peab seisma, ka pärast taaskäivitust.

Lisa `roles/stack/tasks/main.yml`-i kausta task'i järele task, mis peatab teenuse `nginx` ja keelab selle käivitumise buutimisel (moodul `service`, `state`, `enabled`). Kirjuta ise ja jooksuta uuesti.

??? example "Näide, kui ikka kinni"
    ```yaml
    - name: K1 nginx ei hoia porti 80 kinni (seda teeb nüüd proxy-konteiner)
      ansible.builtin.service:
        name: nginx
        state: stopped
        enabled: false
    ```

    Handlerit uuesti ei teavitata, sest failid ei muutunud. Pinu tõstab üles kontroll-task `Kõik teenused käivad`: see näeb kahte töötavat teenust kolme asemel ja loob konteinerid uuesti. See on oluline: proxy konteiner, mille pordi sidumine ebaõnnestus, jääb võrguta (`docker inspect labor-proxy-1` näitab `Networks` all `{}`) ja `up -d` ilma `--force-recreate`-ta ei paranda seda.

```bash
ansible-playbook site.yml
ssh vm3 "cd /opt/labor && docker compose ps"
```

??? success "Oodatav tulemus"

    ```
    RUNNING HANDLER [stack : Stack uueneb] ************************
    changed: [vm3]
    RUNNING HANDLER [stack : proxy laeb seaded uuesti] ************
    changed: [vm3]
    ...
    NAME            IMAGE                                  SERVICE   STATUS                    PORTS
    labor-app-1     localhost/labori-app:1.0               app       Up 30 seconds (healthy)   8080/tcp
    labor-db-1      docker.io/library/postgres:17-alpine   db        Up 36 seconds (healthy)   5432/tcp
    labor-proxy-1   docker.io/library/nginx:1.30-alpine    proxy     Up 24 seconds             0.0.0.0:80->80/tcp
    ```

    `db` ja `app` on `healthy`. Avaldatud on ainult proxy port.

Proovi lehte vm1-st. vm3 aadressi võtad oma SSH seadistusest:

```bash
VM3=$(ssh -G vm3 | awk '/^hostname /{print $2}')
curl -s -i http://$VM3/ | head -n 1
```

??? success "Oodatav tulemus"

    ```
    HTTP/1.1 502 Bad Gateway
    ```

    Kõik kolm teenust käivad ja kaks on terved, aga leht ei tööta. Seda parandad järgmises sammus.

??? info "Loe juurde"

    - [loeng §8](lecture.md#8-ansible-ja-konteinerid)
    - [docker compose up](https://docs.docker.com/reference/cli/docker/compose/up/)

### B4 · Leia, miks proxy annab 502

Selle sammu lõpuks vastab `http://vm3/` rakenduse lehega ja parandus on repos, mitte käsitsi vm3-s.

502 tähendab sama mis K2-s: proxy töötab, aga taga ei vastanud keegi. Põhjus on proxy logis. Konteineri logi on `docker compose logs`:

```bash
ssh vm3 "cd /opt/labor && docker compose logs proxy | tail -n 3"
```

??? success "Oodatav tulemus"

    ```
    proxy-1  | ... connect() failed (111: Connection refused) while connecting to upstream, ... upstream: "http://127.0.0.1:8080/", ...
    ```

Logi tabelist K2 loengu §10-s: `111` tähendab, et sel aadressil ei kuula keegi. Vaata kolleegi faili `stack/nginx/default.conf`. See on kirjutatud ajast, kui nginx ja rakendus olid samas masinas. Miks `127.0.0.1` konteineris ei tööta ja mis on õige aadress? Paranda fail.

??? tip "Vihje"

    Proxy konteineri `127.0.0.1` on proxy ise. Rakendus on samas Compose'i võrgus teenuse nimega. Loengu §5.

    ??? example "Näide, kui ikka kinni"
        ```nginx
        upstream rakendus {
            server app:8080;
        }
        ```

Jooksuta ja vaata, mis handlerid käivituvad:

```bash
ansible-playbook site.yml --check --diff
ansible-playbook site.yml
for i in 1 2 3; do curl -s http://$VM3/; echo; done
```

??? success "Oodatav tulemus"

    ```
    TASK [stack : Proxy seaded on paigas] *************************
    changed: [vm3]
    RUNNING HANDLER [stack : proxy laeb seaded uuesti] ************
    changed: [vm3]
    ...
    Labori seis 1.0
    Vastas: 8c41e0d2a7f3
    Külastused: 8c41e0d2a7f3=1
    ```

    Muutus ainult proxy fail ja käivitus ainult proxy handler. Rakendust ega andmebaasi ei puudutatud.

??? question "Mõtle (vabatahtlik)"

    Kommenteeri ajutiselt välja `proxy laeb seaded uuesti` handleri sisu (või eemalda `notify`), muuda `default.conf`-is midagi ja jooksuta. Mida näitab `ssh vm3 "cd /opt/labor && docker compose exec -T proxy cat /etc/nginx/conf.d/default.conf"` ja mida teeb nginx? Taasta handler.

### B5 · Vaheta andmebaasi parool

Selle sammu lõpuks oled näinud, et parooli vahetus `.env`-is ei vaheta andmebaasi parooli, ja `--wait` muudab selle vea punaseks playbookiks.

Ennusta enne, mis juhtub. Muuda `group_vars/all.yml`-is `db_parool` ükskõik milleks muuks ja jooksuta:

```bash
ansible-playbook site.yml
```

??? success "Oodatav tulemus"

    ```
    RUNNING HANDLER [stack : Stack uueneb] ************************
    fatal: [vm3]: FAILED! => changed=true
      ...
      stderr: |-
        ...
        container labor-app-1 is unhealthy
    ```

    Compose lõi `db` ja `app` konteinerid uuesti, sest nende keskkonnamuutuja muutus. Rakendus ei saa andmebaasi.

Leia põhjus logist:

```bash
ssh vm3 "cd /opt/labor && docker compose logs app | grep -m1 authentication"
```

??? success "Oodatav tulemus"

    ```
    app-1  | ... password authentication failed for user "labor"
    ```

`POSTGRES_PASSWORD` kehtib ainult tühja andmekausta korral. Andmekaust on volume'is ja seal on vana parool. Taasta `group_vars/all.yml`-is vana parool ja jooksuta uuesti. Kõik paraneb.

Päris parooli vahetus käib kahes sammus: kõigepealt andmebaasis (`ALTER ROLE labor PASSWORD '...'`), siis `.env`-is. Kodutöös võid selle teha Ansible'i task'iks.

??? question "Mõtle (vabatahtlik)"

    Leht oli parooli vahetuse ajal maas. Miks? Mis peaks olema teisiti, et parooli saaks vahetada katkestuseta?

### B6 · Drift, andmed ja tulemüür

Selle sammu lõpuks on tõend, et käsitsi peatatud teenus tuleb Ansible'iga tagasi ja andmed jäävad alles ka siis, kui kõik konteinerid kustutada.

**Peatatud teenus.** Peata rakendus ja ennusta: mis task on järgmisel jooksul `changed`?

```bash
ssh vm3 "cd /opt/labor && docker compose stop app"
ansible-playbook site.yml --check
ansible-playbook site.yml
```

??? success "Oodatav tulemus"

    `Kontrolli, mis teenused käivad` näeb kahte teenust, `Kõik teenused käivad` on `changed`. Muud task'id `ok`.

Kas `stop` võttis aega? Mõõda:

```bash
ssh vm3 "cd /opt/labor && time docker compose stop app && docker compose start app"
```

??? success "Oodatav tulemus"

    `real 0m10.4s`. Docker saatis `SIGTERM`-i, rakendus PID 1-na seda ei käsitlenud, Docker ootas 10 sekundit ja saatis `SIGKILL`-i.

Lisa `compose.yaml`-is teenusele `app` rida `init: true` (loengu §4), jooksuta `site.yml` ja mõõda uuesti. Nüüd peaks `stop` võtma alla sekundi.

**Andmed.** Kustuta kõik konteinerid ja võrk. Volume jääb:

```bash
curl -s http://$VM3/ | tail -n 1
ssh vm3 "cd /opt/labor && docker compose down && docker volume ls"
ansible-playbook site.yml
curl -s http://$VM3/ | tee logid/andmed_alles.txt
```

??? success "Oodatav tulemus"

    `down` kustutab kolm konteinerit ja võrgu `labor_default`, `docker volume ls` näitab ikka `labor_andmed`. Pärast `site.yml`-i jätkab loendur sealt, kus pooleli jäi. Rakenduse konteiner on uus, seega on `Külastused:` real kaks ID-d: vana ja uus.

**Tulemüür.** K2-s avasid iga pordi firewalld-s. Siin pole keegi porti 80 avanud:

```bash
ssh -t vm3 "sudo firewall-cmd --list-all"
curl -s -o /dev/null -w "%{http_code}\n" http://$VM3/
```

Kas port 80 on nimekirjas? Kas leht vastab? Kirjuta README-sse, mida nägid ja mida see tähendab B2 küsimuse jaoks (andmebaasi port).

??? info "Loe juurde"

    - [Docker ja pakettfilter](https://docs.docker.com/engine/network/packet-filtering-firewalls/)
    - [Compose: init](https://docs.docker.com/reference/compose-file/services/#init)

### B7 · Kontroll ja tõendid

Selle sammu lõpuks kukub `site.yml` ise punaseks, kui leht läbi proxy ei vasta, ja failides on tõend, et teine jooks ei muuda midagi.

Lisa `site.yml`-i lõppu sama kontroll-play nagu K2 B4-s: `hosts: docker`, faktideta, päring `http://localhost/` kuni 5 korda, vastuses peavad olema `Vastas:` ja `Külastused:`. Kopeeri K2 repost ja muuda `hosts`.

??? example "Näide, kui ikka kinni"
    ```yaml
    - name: "Kontroll: leht vastab läbi proxy"
      hosts: docker
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

        - name: Vastuses on rakendus ja andmebaasi loendur
          ansible.builtin.assert:
            that:
              - "'Vastas:' in vastus.content"
              - "'Külastused:' in vastus.content"
            fail_msg: "Proxy vastas, aga mitte rakendus: {{ vastus.content | truncate(200) }}"
    ```

Salvesta tõendid:

```bash
ansible-playbook site.yml | tee logid/site_teine_jooks.txt
for i in 1 2 3; do curl -s http://$VM3/; done | tee logid/stack.txt
```

??? success "Oodatav tulemus"

    ```
    PLAY RECAP ****************************************************
    vm3 : ok=15  changed=0  unreachable=0  failed=0  skipped=1
    ```

    `skipped=1` on `Kõik teenused käivad`: kõik juba käivad. Täpne `ok` arv võib erineda.

??? tip "Kui mõni task on igal jooksul `changed`"

    Tavalised põhjused: kontrolltask'il puudub `changed_when: false`; `copy` kopeerib iga kord, sest rakenduse kaustas tekib `__pycache__` (vaata `ssh vm3 ls -a /opt/labor/app`); `.env` mallis on muutuja, mis muutub iga jooksuga.

Commit:

```bash
git status
git add .
git commit -m "K3: Docker ja Compose pinu Ansible'iga, teine jooks changed=0"
git push
```

`git status` ei tohi näidata ühtegi `.env` faili (ainult `stack/.env.naide`).

## Lisaülesanne · Pod tagasi YAML-ist

Selle ülesande lõpuks oled kustutanud A4 pod'i ja taastanud selle failist `logid/labor-pod.yaml`, ilma ühegi `podman run` käsuta.

1. `podman pod rm -f labor-pod` ja kontrolli, et `curl localhost:8091` ei vasta.
2. Käivita pod uuesti failist: `podman kube play logid/labor-pod.yaml`.
3. `curl localhost:8091/`: loendur jätkab, sest volume on sama.
4. Vaata `podman pod ps`: mis nime said pod ja konteinerid? Mis käsk pod'i jälle maha võtab?

??? tip "Vihje"

    `podman kube play --help`, `podman kube down`. Sama YAML-i vorming on K5-s Kubernetes'i jaoks, seal ütleb sama asja `kubectl apply -f`.

??? info "Loe juurde"

    - [podman-kube-play](https://docs.podman.io/en/latest/markdown/podman-kube-play.1.html)

## Esitamine

### README

Täida repos olev `README.md` mall: asenda nurksulgudes kohad oma tööga ja kustuta ülemine kast. `ULESANNE.md` jääb muutmata. README täitmine on boonus: kontroll BOONUS README läbib, kui täitmata kohti enam pole.

### Commit ja push

```bash
git status
git add .
git commit -m "K3: README"
git push
```

### Automaatne kontroll

Ava GitHubis oma repo → **Actions** → viimane **Autograde**:

| Kontroll | Mida vaatab |
|---|---|
| K1 | `stack/app/Containerfile`, `stack/compose.yaml`, `inventory.ini`, `site.yml`, rollid `docker` ja `stack`, `logid/podman.txt`, `logid/site_teine_jooks.txt` olemas |
| K2 | Containerfile: `FROM` täisnimega ja tag'iga (mitte `latest`), `USER`, `pip` paketi versioon kinni |
| K3 | `compose.yaml` on korrektne (`docker compose config`); kaks healthcheck'i ja `service_healthy`; `db` ei avalda porti; proxy ei suuna `127.0.0.1`-le |
| K4 | `site.yml` süntaks; `docker`-rollis paketid ja teenus; `stack`-rollis `.env` mall õigustega `0600`, handler `docker compose`-iga; `.env` pole Gitis |
| K5 | `podman.txt`, `stack.txt` ja `andmed_alles.txt`: rakendus vastas ja loendur töötas; `site_teine_jooks.txt`: vm3 `changed=0`, `failed=0` |

Hinne tuleb kontrollidest K1–K5 (45 punkti). Kontrollid nimega BOONUS (README ja kodutöö) punkte ei anna ja kukuvad seni, kuni need on tegemata.

## Kodutöö

Kodune õpe ja kodutöö on eraldi lehel: [K3 · Kodune õpe ja kodutöö](homework.md). Samasse reposse, tähtaeg Classroom 50-s.

## Veaotsing

??? info "Veaotsingu tabel: tüüpilised vead ja lahendused"

    Kontrolli järjekorras: `ps -a` (kas konteiner jookseb), `logs` (mida rakendus ütles), `inspect` (millega see käivitati).

    | Probleem | Põhjus | Lahendus |
    |---|---|---|
    | `toomanyrequests: You have reached your unauthenticated pull rate limit` | Docker Hubi piirang klassi IP-le | `mirror.gcr.io/library/<image>` või `podman login docker.io` |
    | Podman küsib `Please select an image` | lühinimi ilma registrita | kirjuta täisnimi `docker.io/library/...` |
    | `rootlessport cannot expose privileged port 80` | rootless ei saa porte alla 1024 | kasuta porti ≥ 1024 |
    | `bind: address already in use` | port on teise konteineri või teenuse all | `podman ps`, `ss -tlnp`, vali teine port |
    | `the container name "app" is already in use` | sama nimega konteiner on olemas (ka peatatuna) | `podman rm -f app` |
    | `curl: (52) Empty reply`, logis `KeyError: 'DB_HOST'` | rakendusele pole seadeid antud | `--env-file`, `-e DB_HOST=...` |
    | `/health`: `connection to server at "localhost" ... refused` | `localhost` on konteiner ise | `DB_HOST=db` (võrk) või `127.0.0.1` (pod) |
    | `could not translate host name "db"` | `db` konteiner puudub või pole samas võrgus | `podman ps`, `--network labor` mõlemal |
    | ehitus kordab iga kord `pip install`-i | `COPY` on enne `RUN pip` | pane `COPY app.py` pärast teeke |
    | `permission denied ... /var/run/docker.sock` | kasutaja pole `docker`-grupis või sessioon on vana | B1 roll, uus SSH-sessioon |
    | `dnf`: `Error: GPG check FAILED` | Dockeri võti pole usaldatud | `rpm_key` task enne paigaldust |
    | `Problem: package docker-ce ... conflicts with ... runc/podman` | Podman ja Docker samas masinas | B1 esimene task |
    | `no configuration file provided: not found` | `docker compose` käivitati vales kaustas | `chdir: "{{ stack_kaust }}"`, `cd /opt/labor` |
    | `required variable DB_PAROOL is missing` / tühjad väärtused | `.env` puudub või muutuja nimi erineb | `ssh vm3 sudo cat /opt/labor/.env`, võrdle `compose.yaml`-iga |
    | `container labor-app-1 is unhealthy` | rakendus ei saa andmebaasi või healthcheck on vale | `docker compose logs app`, B5 |
    | `password authentication failed` | `.env` parool erineb andmebaasi omast | taasta parool, B5 |
    | `failed to bind host port ... 0.0.0.0:80 ...: address already in use` | vm3 pordil 80 on K1 nginx | B3, nginx peatatud ja keelatud rollis |
    | proxy `Restarting`, logis `host not found in upstream "app:8080"` | proxy konteiner jäi katkestatud jooksust võrguta | `docker compose up -d --force-recreate proxy` |
    | `502`, logis `111: Connection refused`, upstream `127.0.0.1` | proxy otsib rakendust iseendast | `server app:8080`, B4 |
    | parandatud `default.conf`, aga ikka 502 | nginx pole seadeid uuesti laadinud | handler `proxy laeb seaded uuesti` |
    | `docker compose stop` võtab 10 s | rakendus PID 1-na ei käsitle `SIGTERM`-i | `init: true` |
    | andmed kadusid | `down -v` või volume vales kaustas | `docker volume ls`, tee `/var/lib/postgresql/data` |
