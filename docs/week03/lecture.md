# K3 · Konteinerid: Podman, Docker ja Compose

Selle loengu jaoks peaksid tundma K2 pinu: nginx koormusjaotur, kaks rakendusserverit ja PostgreSQL, kõik neljas rollis. Täna paned sama rakenduse konteineritesse. Klassis räägime peatükkidest 1, 2, 4, 6 ja 7. Ülejäänu loe kodus läbi, peatükid 8 ja 9 on praktikumi B-osa ja kodutöö jaoks.

## Õpiväljundid

Pärast seda loengut oskad:

- selgitada, mis on konteiner Linuxi tuuma vaates ja millal on õige valik VM, millal konteiner;
- kirjutada Containerfile'i (Dockerfile'i), mille kihid on vahemälu jaoks õiges järjekorras ja mis ei jookse root'ina;
- valida andmetele õige koha (kirjutatav kiht, nimega volume, bind mount) ja selgitada, miks `localhost` tähendab konteineris konteinerit ennast;
- võrrelda Dockeri ja Podmani arhitektuuri ning selgitada, mida tähendab rootless;
- kirjeldada mitme teenusega rakendust Compose-failina koos tervisekontrollide ja käivitusjärjekorraga;
- jagada töö Ansible'i ja konteinerite vahel: kumb haldab serverit, kumb rakendust, ja kuidas jääb Compose'i paigaldus idempotentseks.

## 1. Sama rakendus, teine pakend

K2 lõpus töötas „Labori seis“ kolmes masinas. Selleks oli vaja nelja rolli: `common`, `db`, `app` ja `lb`. Rakenduse roll pidi teadma, et masinas on Python 3.9, et draiver tuleb paketist `python3-psycopg2`, et teenus jookseb kasutajana `labori` ja et systemd loeb seaded failist `/etc/labori-app.env`. Kõik see on teadmine selle kohta, **kuidas rakendus käib**, mitte selle kohta, kuidas server on seadistatud.

Nüüd tuleb arendajalt teade: järgmine versioon vajab Python 3.13-t ja uuemat draiverit. AlmaLinux 9 vaikimisi Python on 3.9. Valikud on ebameeldivad: paigaldad serverisse teise Pythoni ja hakkad jälgima, kumb teenus kumba kasutab; või uuendad serveri OS-i rakenduse pärast; või ütled arendajale „ei saa“.

Konteiner lahendab selle teisiti. Arendaja pakib rakenduse koos Pythoni, teekide ja käivituskäsuga ühte **image'isse**. Server peab oskama ainult image'eid käivitada. Pythoni versioon on image'i sees ja serveri oma ei puutu asjasse. Sama image jookseb arendaja sülearvutis, testis, CI-s ja tootmises ühtemoodi.

Tänane jaotus:

| Kes | Mida haldab | Millega |
|---|---|---|
| Ansible | server: paketid, Docker või Podman, kasutajad, kaustad, tulemüür | rollid nagu K1–K2 |
| image | rakendus: Python, teegid, kood, käivituskäsk | Containerfile |
| Compose | teenuste kooslus: mis image, mis seaded, kes kellest sõltub, kus on andmed | `compose.yaml` |

Kui kahtled, kuhu miski kuulub, küsi: kas see on osa serverist (tulemüür, ketas, kasutajad) või osa rakendusest (Pythoni versioon, teek, käivituskäsk)? Esimene on Ansible'i töö, teine image'i töö.

??? question "Kordamisküsimus"

    Vaata K2 rolli `app` task'e. Millised neist kaovad, kui rakendus on image'is, ja millised jäävad alles serveri poolele?

## 2. Konteiner on protsess

Konteiner ei ole väike virtuaalmasin. See on tavaline Linuxi protsess, mida tuum näitab teisiti. Kui käivitad konteineris `python3 app.py`, näed hostis `ps -ef` väljundis sama `python3` protsessi. Vahe on selles, mida see protsess ise näeb.

Tuum annab protsessile kaks asja:

- **namespace'id** määravad, mida protsess näeb: oma protsessipuu (konteineris on rakendus PID 1), oma võrguliidesed ja IP, oma failisüsteemi juur, oma hostname, oma kasutajate tabel;
- **cgroup'id** määravad, kui palju protsess tohib võtta: protsessor, mälu, protsesside arv.

Need kaks koos ongi konteiner. Docker ja Podman on tööriistad, mis seavad need üles, toovad failisüsteemi image'ist ja käivitavad protsessi.

Sellest tulevad kaks praktilist järeldust:

- **tuum on ühine**. Konteineris võib olla Debian, Alpine või Ubuntu, aga `uname -r` näitab hosti AlmaLinuxi tuuma. Windowsi rakendust Linuxi konteinerisse ei pane ja tuuma turvaauk puudutab kõiki konteinereid korraga;
- **konteineri elu on protsessi elu**. Kui protsess lõpetab, on konteiner peatunud. Konteiner, mis käivitab `bash`-i ilma terminalita, lõpetab kohe, sest `bash` ei saa sisendit.

<figure style="max-width:690px;margin:.8em auto" class="lx" markdown="0">
<svg viewBox="0 0 690 250" role="img" aria-label="VM toob kaasa oma tuuma, konteinerid jagavad hosti tuuma" xmlns="http://www.w3.org/2000/svg">
<style>.lx svg{width:100%;height:auto;font-family:var(--md-text-font-family,sans-serif)}.lx .box{fill:var(--md-code-bg-color);stroke:var(--md-default-fg-color--lighter);stroke-width:1.2}.lx .hi{fill:var(--md-primary-fg-color);fill-opacity:.16;stroke:var(--md-primary-fg-color);stroke-width:1.5}.lx .ok{fill:#2e7d32;fill-opacity:.14;stroke:#2e7d32;stroke-width:1.3}.lx .bad{fill:#c62828;fill-opacity:.12;stroke:#c62828;stroke-width:1.3}.lx .t{fill:var(--md-default-fg-color);font-size:15px;font-weight:700}.lx .b{fill:var(--md-default-fg-color);font-size:14.5px;font-weight:600}.lx .s{fill:var(--md-default-fg-color--light);font-size:13px}.lx .c{fill:var(--md-default-fg-color);font-size:12.5px;font-family:var(--md-code-font-family,monospace)}.lx .a{stroke:var(--md-default-fg-color--light);stroke-width:1.5;fill:none}</style>
<text class="t" x="165" y="20" text-anchor="middle">Virtuaalmasinad</text>
<rect class="box" x="20" y="34" width="95" height="44" rx="6"/><text class="b" x="67" y="61" text-anchor="middle">rakendus</text>
<rect class="box" x="120" y="34" width="95" height="44" rx="6"/><text class="b" x="167" y="61" text-anchor="middle">rakendus</text>
<rect class="box" x="220" y="34" width="95" height="44" rx="6"/><text class="b" x="267" y="61" text-anchor="middle">rakendus</text>
<rect class="bad" x="20" y="84" width="95" height="44" rx="6"/><text class="s" x="67" y="104" text-anchor="middle">OS + tuum</text><text class="s" x="67" y="120" text-anchor="middle">GB</text>
<rect class="bad" x="120" y="84" width="95" height="44" rx="6"/><text class="s" x="167" y="104" text-anchor="middle">OS + tuum</text><text class="s" x="167" y="120" text-anchor="middle">GB</text>
<rect class="bad" x="220" y="84" width="95" height="44" rx="6"/><text class="s" x="267" y="104" text-anchor="middle">OS + tuum</text><text class="s" x="267" y="120" text-anchor="middle">GB</text>
<rect class="hi" x="20" y="134" width="295" height="40" rx="6"/><text class="b" x="167" y="159" text-anchor="middle">hüperviisor (Proxmox)</text>
<rect class="box" x="20" y="180" width="295" height="40" rx="6"/><text class="b" x="167" y="205" text-anchor="middle">riistvara</text>
<text class="t" x="525" y="20" text-anchor="middle">Konteinerid</text>
<rect class="box" x="380" y="34" width="95" height="44" rx="6"/><text class="b" x="427" y="61" text-anchor="middle">rakendus</text>
<rect class="box" x="480" y="34" width="95" height="44" rx="6"/><text class="b" x="527" y="61" text-anchor="middle">rakendus</text>
<rect class="box" x="580" y="34" width="95" height="44" rx="6"/><text class="b" x="627" y="61" text-anchor="middle">rakendus</text>
<rect class="ok" x="380" y="84" width="95" height="44" rx="6"/><text class="s" x="427" y="104" text-anchor="middle">teegid</text><text class="s" x="427" y="120" text-anchor="middle">MB</text>
<rect class="ok" x="480" y="84" width="95" height="44" rx="6"/><text class="s" x="527" y="104" text-anchor="middle">teegid</text><text class="s" x="527" y="120" text-anchor="middle">MB</text>
<rect class="ok" x="580" y="84" width="95" height="44" rx="6"/><text class="s" x="627" y="104" text-anchor="middle">teegid</text><text class="s" x="627" y="120" text-anchor="middle">MB</text>
<rect class="hi" x="380" y="134" width="295" height="40" rx="6"/><text class="b" x="527" y="152" text-anchor="middle">üks Linuxi tuum</text><text class="s" x="527" y="167" text-anchor="middle">namespace'id + cgroup'id</text>
<rect class="box" x="380" y="180" width="295" height="40" rx="6"/><text class="b" x="527" y="205" text-anchor="middle">VM või riistvara</text>
<text class="s" x="345" y="244" text-anchor="middle">Meil on mõlemad: konteinerid jooksevad Proxmoxi VM-i sees.</text>
</svg>
</figure>

### Millal VM, millal konteiner

| Olukord | Valik | Miks |
|---|---|---|
| teine OS samas riistvaras (Windows Linuxi kõrval) | VM | konteiner ei too oma tuuma |
| võõras või vaenulik kood, eri kliendid | VM | eraldi tuum, väiksem ründepind |
| terve vana server koos OS-iga | VM | see pole üks rakendus |
| veebirakendus, API, taustatöö | konteiner | kerge, kiire, sama pakend igal pool |
| CI testid puhtas keskkonnas | konteiner | tekib ja kaob sekunditega |
| rakendus vajab teist Pythoni või teegi versiooni kui server | konteiner | sõltuvused on image'is |

Praktikas ei vali sa üht. Proxmox jagab riistvara VM-ideks, Ansible seadistab VM-id ja VM-ides jooksevad konteinerid. Täna on just nii.

??? question "Kordamisküsimus"

    Kolleeg ütleb: „Konteiner on turvaline, sest see on isoleeritud.“ Mis selles on õige ja mis mitte? Mis juhtub, kui tuumas on turvaauk?

??? info "Loe juurde"

    - [Red Hat: mis on Linuxi konteiner](https://www.redhat.com/en/topics/containers/whats-a-linux-container)
    - [man 7 namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html), [man 7 cgroups](https://man7.org/linux/man-pages/man7/cgroups.7.html)

## 3. Image, kihid ja register

**Image** on kirjutuskaitstud failisüsteem koos metaandmetega: mis käsk käivitub, mis kasutajana, mis pordid. **Konteiner** on image'ist käivitatud protsess, millele lisatakse õhuke kirjutatav kiht. Ühest image'ist saab käivitada mitu konteinerit, nagu ühest klassist mitu objekti.

Image koosneb **kihtidest**. Iga Containerfile'i käsk, mis muudab faile, teeb uue kihi. Kihid on jagatud: kui kümme image'it põhinevad samal `python:3.13-slim`-il, on see kettal üks kord. Kihi muutmisel ehitatakse uuesti see ja kõik selle järel olevad kihid, eelmised võetakse vahemälust. See on §4 kõige olulisem reegel.

### Nimi, tag ja digest

Image'i täisnimi on neljas osas:

```
docker.io/library/postgres:17-alpine
│         │       │        └── tag: versioon või variant
│         │       └── repo
│         └── nimeruum (library = Docker Hubi ametlikud image'id)
└── register
```

- **register** on server, kust image tuleb: `docker.io` (Docker Hub), `quay.io`, `ghcr.io` (GitHub, K4), `registry.access.redhat.com`. `localhost/` tähendab, et image on ehitatud selles masinas ja registris seda pole;
- **tag** on silt, mitte versioon. `postgres:17-alpine` näitab täna versioonile 17.x ja homme uuemale 17.y-le. `latest` on lihtsalt vaikimisi tag'i nimi, mitte „uusim“;
- **digest** (`postgres@sha256:3f1a…`) on sisu räsi. See ei muutu kunagi. Kui vajad täpselt sama image'it ka poole aasta pärast, kasuta digest'i.

Ansible'is kinnitasid kollektsiooni versiooni (`ansible.posix:1.5.4`), et sama repo töötaks ka hiljem. Image'iga on sama: `python:3.13-slim` on mõistlik, `python` ilma tag'ita on homme teine Python.

### Lühinimed: Docker eeldab, Podman küsib

Docker teeb `nginx`-ist alati `docker.io/library/nginx`. Podman ei eelda ühtegi registrit. AlmaLinuxis on fail `/etc/containers/registries.conf.d/000-shortnames.conf`, kus on populaarsetele nimedele aliased, ja `/etc/containers/registries.conf`, kus on registrid, mida lühinime puhul proovida. Kui alias puudub, küsib Podman terminalis, millist registrit kasutada, ja skriptis kukub. Põhjus on turvalisus: kui keegi paneb esimesse registrisse samanimelise pahatahtliku image'i, tõmbaks lühinimi selle.

Reegel nii Dockeri kui ka Podmani jaoks: kirjuta failidesse alati täisnimi koos registriga.

### Piirangud ja peeglid

Docker Hub piirab anonüümseid tõmbeid IP-aadressi kohta. Klassis on viisteist inimest ühe välise IP taga, nii et piir võib täis saada. Siis on veateade `toomanyrequests`. Lahendusi on kolm: logida Docker Hubi sisse (`podman login docker.io`), kasutada peeglit (Google hoiab Docker Hubi ametlike image'ide koopiat aadressil `mirror.gcr.io`, näiteks `mirror.gcr.io/library/postgres:17-alpine`) või pidada oma registrit. Ettevõtetes on tavaliselt oma register (Harbor, Nexus, GitLab, GHCR), mis hoiab ka koopiaid välistest image'itest.

??? question "Kordamisküsimus"

    Kaks serverit käivitasid `postgres:17-alpine` kuu aja vahega ja käituvad veidi erinevalt. Kuidas see võimalik on? Kuidas tagaksid, et mõlemas on bitt-bitilt sama image?

??? info "Loe juurde"

    - [OCI image'i spetsifikatsioon](https://github.com/opencontainers/image-spec)
    - [Docker Hub: tõmbepiirangud](https://docs.docker.com/docker-hub/usage/pulls/)
    - [containers-registries.conf(5)](https://github.com/containers/image/blob/main/docs/containers-registries.conf.5.md)

## 4. Containerfile

Image'i retsept on tekstifail. Dockeri maailmas on selle nimi `Dockerfile`, Podmani maailmas `Containerfile`. Sisu ja süntaks on samad. Podman leiab mõlemad, Docker otsib vaikimisi `Dockerfile`-i, seega Compose'is ütled faili nime välja (`dockerfile: Containerfile`).

Tänase rakenduse Containerfile:

```dockerfile
FROM docker.io/library/python:3.13-slim

RUN pip install --no-cache-dir psycopg2-binary==2.9.13 \
 && useradd --system --no-create-home labori

WORKDIR /app
COPY app.py .

USER labori
ENV APP_PORT=8080
EXPOSE 8080
CMD ["python3", "app.py"]
```

| Käsk | Mida teeb | Kiht? |
|---|---|---|
| `FROM` | alusimage, millele ehitad | alusimage'i kihid |
| `RUN` | käivitab ehituse ajal käsu | jah |
| `WORKDIR` | töökaust järgmistele käskudele ja käivitusele | metaandmed |
| `COPY` | kopeerib faili ehituskontekstist image'isse | jah |
| `USER` | mis kasutajana järgmised käsud ja rakendus jooksevad | metaandmed |
| `ENV` | keskkonnamuutuja vaikeväärtus | metaandmed |
| `EXPOSE` | dokumentatsioon: rakendus kuulab sellel pordil | metaandmed |
| `CMD` | käsk, mis käivitub konteineri startimisel | metaandmed |

### Kihtide järjekord

Ehituse ajal võrdleb tööriist iga käsku ja selle sisendit eelmise ehitusega. Kui midagi ei muutunud, võetakse kiht vahemälust. Esimene muutunud käsk ja kõik järgnevad ehitatakse uuesti.

Seepärast pane alla see, mis muutub harva, ja üles see, mis muutub tihti. Teekide paigaldus võtab kümneid sekundeid ja muutub harva. Kood muutub iga päev. Kui `COPY app.py` oleks enne `pip install`-i, paigaldaks iga koodimuudatus kõik teegid uuesti. Praktikumi A2-s mõõdad seda.

`.dockerignore` (Podmanis ka `.containerignore`) ütleb, mis faile ehituskontekst ei sisalda: `__pycache__`, `.git`, `.env`. Ilma selleta võib kogemata image'isse minna fail, mida seal olla ei tohi.

### Mitte root

Vaikimisi jookseb protsess konteineris root'ina. Dockeriga on see konteineri root sama UID 0, mis hostis. Kui rakenduses on auk ja ründaja pääseb konteinerist välja, on ta hostis root. `USER labori` paneb rakenduse jooksma tavakasutajana. See on üks rida ja peaks olema igas image'is, mille ise teed.

### Saladused ei kuulu image'isse

`ENV DB_PAROOL=...` või `COPY .env .` paneb parooli image'i kihti. Kihid on loetavad igaühele, kellel on image: `podman image history` ja `podman save` näitavad kõike. Isegi kui järgmine käsk faili kustutab, on see eelmises kihis alles. Seaded ja paroolid antakse konteinerile käivitamisel, mitte ehitamisel.

### `CMD` kujul nimekiri ja PID 1

`CMD ["python3", "app.py"]` (exec-kuju) käivitab Pythoni otse ja see on konteineris PID 1. `CMD python3 app.py` (shell-kuju) käivitab `/bin/sh -c`, mille all on Python, ja signaalid ei jõua rakenduseni.

PID 1-l on Linuxis eriroll: tuum ei rakenda talle signaalide vaikimisi käitumist. Kui rakendus ise `SIGTERM`-i ei käsitle, siis `stop` ei peata seda. Tööriist ootab 10 sekundit ja tapab protsessi `SIGKILL`-iga. Meie `app.py` ei käsitle `SIGTERM`-i. Lahendus on väike init-protsess, mis jookseb PID 1-na ja edastab signaalid: Compose'is `init: true`, käsureal `--init`. Praktikumi B-osas mõõdad vahet.

??? question "Kordamisküsimus"

    Kolleegi Containerfile'is on järjekord `COPY . .`, siis `RUN pip install -r requirements.txt`. Mis juhtub iga koodimuudatusega? Kuidas paned read ümber ja miks peab `requirements.txt` eraldi kopeerima?

??? info "Loe juurde"

    - [Dockerfile'i viide](https://docs.docker.com/reference/dockerfile/)
    - [Docker: ehituse vahemälu](https://docs.docker.com/build/cache/)
    - [Podman: podman-build](https://docs.podman.io/en/latest/markdown/podman-build.1.html)

## 5. Andmed ja võrk

### Kus andmed on

Konteineri kirjutatav kiht kaob koos konteineriga. `stop` ja `start` jätavad selle alles, `rm` kustutab. Uue versiooni paigaldus on alati `rm` ja uus konteiner uuest image'ist, seega kõik, mis on ainult kirjutatavas kihis, kaob esimese uuendusega.

| Kus | Kuidas | Milleks |
|---|---|---|
| kirjutatav kiht | vaikimisi | ajutised failid; mitte midagi olulist |
| nimega volume | `-v andmed:/var/lib/postgresql/data` | andmebaas, üleslaadimised. Tööriist haldab kausta ise |
| bind mount | `-v ./nginx/default.conf:/etc/nginx/conf.d/default.conf:ro` | hosti fail konteinerisse, tavaliselt seaded |

Volume elab konteinerist kauem. `docker compose down` kustutab konteinerid ja võrgu, aga mitte volume'eid. `docker compose down -v` kustutab ka volume'id ja koos nendega andmebaasi. Seda `-v`-d ei kirjuta keegi kogemata kaks korda.

Volume tuleb ühendada sinna, kus rakendus andmeid päriselt hoiab. Kui ühendad PostgreSQL-i volume'i vale kausta, töötab kõik, kuni konteiner esimest korda uuesti luuakse. Siis on andmebaas tühi. Õige tee on image'i dokumentatsioonis (ametlikus `postgres:17` image'is `/var/lib/postgresql/data`).

SELinuxiga hostis peab Podman bind mount'i failile panema õige märgise, muidu keelab SELinux konteineril seda lugeda. Selleks on volume'i lõpus `:Z` (ainult see konteiner) või `:z` (jagatud). Nimega volume'id saavad märgise ise.

### Kuidas konteinerid üksteist leiavad

Igal konteineril on oma võrgu namespace: oma IP ja oma `localhost`. **`localhost` konteineris tähendab konteinerit ennast.** K2-s oli `DB_HOST` vm1 IP. Konteineris rakendus ühenduda `localhost:5432`-ga ei saa, sest seal ei kuula keegi, isegi kui andmebaas jookseb samas masinas teises konteineris.

Konteinerid leiavad üksteist nime järgi, kui nad on samas **kasutaja loodud võrgus**. Selles võrgus on sisseehitatud DNS, mis vastab konteineri (või Compose'i teenuse) nimele. `DB_HOST=db` töötab, sest Compose pani teenuse `db` võrku nimega `db`. Vaikimisi võrgus (`bridge` Dockeris, `podman` Podmanis) nimelahendust pole.

### Pordid väljapoole

Konteineri port ei ole väljast nähtav enne, kui see avaldatakse: `-p 8090:8080` tähendab „hosti port 8090 suunatakse konteineri pordile 8080“. Avalda ainult see, mida peab väljast kasutama. Tänases pinus on avaldatud ainult proxy port. Andmebaas ja rakendus on ainult konteinerite võrgus.

Dockeriga on üks üllatus, mida K2 järel tasub teada. Docker kirjutab pakettfiltri reeglid ise ja avaldatud port on väljast kättesaadav ka siis, kui `firewall-cmd --list-all` seda ei näita. Kui avaldad andmebaasi pordi „ainult testiks“, on see võrgus lahti. Praktikumi B-osas kontrollid seda ise.

??? question "Kordamisküsimus"

    Rakenduse konteineris on `DB_HOST=localhost`, andmebaas jookseb samas masinas teises konteineris ja selle port on avaldatud (`-p 5432:5432`). Miks rakendus ikkagi ühendust ei saa? Mis on kaks õiget lahendust ja kumb on parem?

??? info "Loe juurde"

    - [Docker: volume'id](https://docs.docker.com/engine/storage/volumes/)
    - [Docker: võrgud](https://docs.docker.com/engine/network/)
    - [Docker ja pakettfilter](https://docs.docker.com/engine/network/packet-filtering-firewalls/)

## 6. Docker ja Podman

Mõlemad käivitavad samu OCI image'eid ja käsud on peaaegu samad (`build`, `run`, `ps`, `logs`, `exec`). Erinevus on arhitektuuris.

<figure style="max-width:690px;margin:.8em auto" class="lx" markdown="0">
<svg viewBox="0 0 690 226" role="img" aria-label="Docker: klient, root-deemon, konteinerid. Podman: iga käsk on kasutaja protsess, deemonit pole" xmlns="http://www.w3.org/2000/svg">
<style>.lx svg{width:100%;height:auto;font-family:var(--md-text-font-family,sans-serif)}.lx .box{fill:var(--md-code-bg-color);stroke:var(--md-default-fg-color--lighter);stroke-width:1.2}.lx .hi{fill:var(--md-primary-fg-color);fill-opacity:.16;stroke:var(--md-primary-fg-color);stroke-width:1.5}.lx .ok{fill:#2e7d32;fill-opacity:.14;stroke:#2e7d32;stroke-width:1.3}.lx .bad{fill:#c62828;fill-opacity:.12;stroke:#c62828;stroke-width:1.3}.lx .t{fill:var(--md-default-fg-color);font-size:15px;font-weight:700}.lx .b{fill:var(--md-default-fg-color);font-size:14.5px;font-weight:600}.lx .s{fill:var(--md-default-fg-color--light);font-size:13px}.lx .a{stroke:var(--md-default-fg-color--light);stroke-width:1.5;fill:none}.lx .h{fill:var(--md-default-fg-color--light)}</style>
<defs><marker id="lxd" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path class="h" d="M0,0 L10,5 L0,10 z"/></marker></defs>
<text class="t" x="170" y="20" text-anchor="middle">Docker</text>
<rect class="box" x="20" y="34" width="130" height="44" rx="6"/><text class="b" x="85" y="54" text-anchor="middle">docker CLI</text><text class="s" x="85" y="70" text-anchor="middle">sina</text>
<line class="a" x1="150" y1="56" x2="186" y2="56" marker-end="url(#lxd)"/><text class="s" x="168" y="48" text-anchor="middle">sokkel</text>
<rect class="bad" x="188" y="34" width="140" height="44" rx="6"/><text class="b" x="258" y="54" text-anchor="middle">dockerd</text><text class="s" x="258" y="70" text-anchor="middle">root, alati käib</text>
<line class="a" x1="258" y1="78" x2="258" y2="104" marker-end="url(#lxd)"/>
<rect class="box" x="188" y="106" width="140" height="36" rx="6"/><text class="s" x="258" y="129" text-anchor="middle">containerd + runc</text>
<line class="a" x1="258" y1="142" x2="258" y2="166" marker-end="url(#lxd)"/>
<rect class="box" x="148" y="168" width="100" height="36" rx="6"/><text class="s" x="198" y="191" text-anchor="middle">konteiner</text>
<rect class="box" x="258" y="168" width="100" height="36" rx="6"/><text class="s" x="308" y="191" text-anchor="middle">konteiner</text>
<text class="t" x="530" y="20" text-anchor="middle">Podman</text>
<rect class="box" x="400" y="34" width="130" height="44" rx="6"/><text class="b" x="465" y="54" text-anchor="middle">podman</text><text class="s" x="465" y="70" text-anchor="middle">sina, sinu UID</text>
<line class="a" x1="465" y1="78" x2="465" y2="104" marker-end="url(#lxd)"/>
<rect class="ok" x="400" y="106" width="130" height="36" rx="6"/><text class="s" x="465" y="129" text-anchor="middle">conmon + crun</text>
<line class="a" x1="465" y1="142" x2="465" y2="166" marker-end="url(#lxd)"/>
<rect class="box" x="415" y="168" width="100" height="36" rx="6"/><text class="s" x="465" y="191" text-anchor="middle">konteiner</text>
<rect class="box" x="560" y="106" width="110" height="98" rx="6"/><text class="b" x="615" y="132" text-anchor="middle">pod</text><text class="s" x="615" y="152" text-anchor="middle">mitu konteinerit,</text><text class="s" x="615" y="168" text-anchor="middle">üks võrk,</text><text class="s" x="615" y="184" text-anchor="middle">ühine localhost</text>
<text class="s" x="345" y="222" text-anchor="middle">Dockeris teeb töö deemon. Podmanis on iga konteiner sinu protsessi järglane.</text>
</svg>
</figure>

**Docker** on klient-server. `docker` käsk saadab päringu soklisse `/var/run/docker.sock`, kus kuulab root'ina töötav deemon `dockerd`. Deemon tõmbab image'id, loob võrgud ja käivitab konteinerid. Kes pääseb soklile, see saab käivitada konteineri, mis ühendab enda sisse hosti `/` ja kirjutab sinna, mida tahab. **`docker`-grupi liige on sisuliselt root**, nii et grupp on sama tundlik kui `wheel`.

**Podman** deemonit ei kasuta. `podman run` on tavaline protsess, mis seab konteineri üles ja jätab selle jälgimiseks väikese `conmon` protsessi. Podman töötab vaikimisi **rootless**: konteineri protsess jookseb sinu UID-ga.

### Rootless

Konteineris võib olla `root`, aga hostis on see sinu kasutaja. Seda teeb kasutajate namespace: konteineri UID 0 vastendatakse sinu UID-ga ja konteineri UID-d 1, 2, 3… vastendatakse hosti UID-dega vahemikust, mis on sulle antud failis `/etc/subuid`. Kui keegi pääseb konteinerist välja, on ta hostis sina, mitte root.

Rootless'il on oma piirangud, mida praktikumis näed:

- pordid alla 1024 on privilegeeritud. Tavakasutaja ei saa avaldada porti 80 (`rootlessport cannot expose privileged port 80`). Kasuta kõrgemat porti või pane ette hosti nginx;
- image'id, konteinerid ja volume'id on kasutaja kodukaustas (`~/.local/share/containers`). Teine kasutaja ega `sudo podman` neid ei näe;
- kui kasutaja logib välja, lõpetab systemd vaikimisi tema protsessid, ka konteinerid. Teenusena käivitamiseks on vaja `loginctl enable-linger`.

### Pod

Podmani nimi tuleb sõnast **pod**: konteinerite rühm, millel on üks võrgu namespace. Pod'i konteinerid räägivad omavahel `localhost`-i kaudu ja port avaldatakse pod'ile, mitte konteinerile. See on sama mõiste mis Kubernetes'is (K5). `podman kube generate` kirjutab pod'ist Kubernetes'i YAML-i ja `podman kube play` käivitab YAML-ist pod'i.

### Taaskäivitus ja systemd

Dockeris taaskäivitab konteinerid deemon (`restart: unless-stopped`). Podmanis deemonit pole, seega on taaskäivitus systemd töö. Tänapäevane viis on **Quadlet**: kirjutad faili `~/.config/containers/systemd/labori-app.container`, mis näeb välja nagu systemd unit, ja systemd teeb sellest teenuse. Konteiner on siis teenus nagu iga teine: `systemctl --user status`, `journalctl --user -u`. Kodutöös proovid.

### Kumb valida

| | Docker | Podman |
|---|---|---|
| arhitektuur | root-deemon `dockerd` | deemonit pole, iga konteiner on protsess |
| õigused | deemon on root; `docker`-grupp ≈ root | vaikimisi rootless |
| AlmaLinux/RHEL | eraldi repo `download.docker.com` | baasrepos, Red Hat toetab |
| Compose | `docker compose` (ametlik plugin) | `podman compose` kutsub välist tööriista (`podman-compose` või `docker-compose`) |
| taaskäivitus | deemon, `restart:` | systemd, Quadlet |
| pod'id ja Kubernetes YAML | ei | jah |
| kus kohtad | arendajate sülearvutid, CI (GitHub Actions), enamik juhendeid | RHEL-i perekonna serverid, turvatundlikud keskkonnad |

Täna kasutad mõlemat: vm1-s Podmani, sest see on AlmaLinuxi oma ja rootless, ja vm3-s Dockerit koos Compose'iga, sest Compose on seal kõige küpsem ja K4 CI kasutab sama Dockerit. Sama image ja sama Containerfile töötavad mõlemas.

??? question "Kordamisküsimus"

    Kolleeg lisab teenusekasutaja `deploy` `docker`-gruppi, et CI saaks konteinereid käivitada, ja ütleb, et `sudo`-t `deploy` ei saa. Kas see on turvaline? Kuidas sama teha Podmaniga?

??? info "Loe juurde"

    - [Podman: rootless](https://github.com/containers/podman/blob/main/docs/tutorials/rootless_tutorial.md)
    - [Docker: rünnakupind ja `docker`-grupp](https://docs.docker.com/engine/security/#docker-daemon-attack-surface)
    - [Podman: Quadlet (podman-systemd.unit)](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)
    - [Podman: pod'id](https://docs.podman.io/en/latest/markdown/podman-pod.1.html)

## 7. Compose: mitu teenust ühes failis

K2 pinu kolm kihti olid kolmes VM-is ja neljas rollis. Compose kirjeldab sama kooslust ühes failis: mis teenused, mis image'ist, mis seadetega, mis järjekorras ja kus on andmed.

<figure style="max-width:690px;margin:.8em auto" class="lx" markdown="0">
<svg viewBox="0 0 690 214" role="img" aria-label="Compose projekt labor: proxy, app ja db ühes võrgus, avaldatud ainult proxy port" xmlns="http://www.w3.org/2000/svg">
<style>.lx svg{width:100%;height:auto;font-family:var(--md-text-font-family,sans-serif)}.lx .box{fill:var(--md-code-bg-color);stroke:var(--md-default-fg-color--lighter);stroke-width:1.2}.lx .hi{fill:var(--md-primary-fg-color);fill-opacity:.16;stroke:var(--md-primary-fg-color);stroke-width:1.5}.lx .ok{fill:#2e7d32;fill-opacity:.14;stroke:#2e7d32;stroke-width:1.3}.lx .b{fill:var(--md-default-fg-color);font-size:14.5px;font-weight:600}.lx .s{fill:var(--md-default-fg-color--light);font-size:13px}.lx .c{fill:var(--md-default-fg-color);font-size:12.5px;font-family:var(--md-code-font-family,monospace)}.lx .a{stroke:var(--md-default-fg-color--light);stroke-width:1.5;fill:none}.lx .d{stroke-dasharray:4 3}.lx .h{fill:var(--md-default-fg-color--light)}</style>
<defs><marker id="lxc" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto"><path class="h" d="M0,0 L10,5 L0,10 z"/></marker></defs>
<rect class="box" x="10" y="70" width="110" height="50" rx="6"/><text class="b" x="65" y="92" text-anchor="middle">brauser</text><text class="s" x="65" y="108" text-anchor="middle">http://vm3/</text>
<line class="a" x1="120" y1="95" x2="176" y2="95" marker-end="url(#lxc)"/><text class="s" x="148" y="88" text-anchor="middle">:80</text>
<rect class="hi d" x="170" y="20" width="510" height="150" rx="10"/><text class="s" x="182" y="38">võrk labor_default · vm3 · docker compose</text>
<rect class="box" x="180" y="66" width="140" height="58" rx="6"/><text class="b" x="250" y="90" text-anchor="middle">proxy</text><text class="s" x="250" y="108" text-anchor="middle">nginx:1.30-alpine</text>
<line class="a" x1="320" y1="95" x2="356" y2="95" marker-end="url(#lxc)"/><text class="c" x="338" y="88" text-anchor="middle">app:8080</text>
<rect class="box" x="358" y="66" width="140" height="58" rx="6"/><text class="b" x="428" y="90" text-anchor="middle">app</text><text class="s" x="428" y="108" text-anchor="middle">labori-app (build)</text>
<line class="a" x1="498" y1="95" x2="534" y2="95" marker-end="url(#lxc)"/><text class="c" x="516" y="88" text-anchor="middle">db:5432</text>
<rect class="box" x="536" y="66" width="134" height="58" rx="6"/><text class="b" x="603" y="90" text-anchor="middle">db</text><text class="s" x="603" y="108" text-anchor="middle">postgres:17-alpine</text>
<rect class="ok" x="536" y="138" width="134" height="26" rx="6"/><text class="s" x="603" y="156" text-anchor="middle">volume andmed</text>
<line class="a" x1="603" y1="124" x2="603" y2="136"/>
<text class="s" x="345" y="200" text-anchor="middle">Väljast on nähtav ainult proxy port. app ja db leiavad üksteist nime järgi.</text>
</svg>
</figure>

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
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      retries: 10

  app:
    build:
      context: ./app
      dockerfile: Containerfile
    image: localhost/labori-app:${APP_VERSIOON}
    environment:
      DB_HOST: db
      # ...
    depends_on:
      db:
        condition: service_healthy

volumes:
  andmed:
```

Mõned asjad, mis failis kohe silma ei hakka:

- **projekti nimi** (`name: labor`) on kõigi objektide eesliide: konteiner `labor-db-1`, võrk `labor_default`, volume `labor_andmed`. Kahe eri nimega projekti saab samas masinas kõrvuti hoida;
- **võrk** tekib ise. Iga teenus on selles oma nimega, seega `DB_HOST: db`;
- **`${DB_PAROOL}`** asendatakse failist `.env`, mis on samas kaustas. `.env` ei lähe Giti. `$${POSTGRES_USER}` (kaks dollarit) tähendab, et Compose jätab muutuja rahule ja selle asendab konteineri shell;
- **`build`** ja **`image`** koos: Compose ehitab image'i kaustast `./app` ja paneb sellele nime `localhost/labori-app:1.0`. Kui versioon muutub, ehitatakse uue nimega image.

### Järjekord ja tervis

`depends_on: [db]` ütleb ainult, et `db` konteiner käivitatakse enne `app`-i. Käivitatud ei tähenda valmis: PostgreSQL vajab esimesel korral mitu sekundit andmekausta loomiseks. Seepärast on `db`-l **healthcheck** (käsk, mida jooksutatakse iga 5 sekundi järel) ja `app` ootab `condition: service_healthy`. Sama muster on `proxy` ja `app` vahel.

`docker compose ps` näitab tervist veerus `STATUS` (`healthy`, `unhealthy`, `starting`). `docker compose up -d --wait` ootab, kuni kõik teenused on käivitunud ja terved, ja kukub, kui mõni jääb haigeks. See on Ansible'i jaoks oluline: punane task tähendab, et rakendus päriselt ei tööta.

Healthcheck käsk jookseb konteineri sees, seega peab vajalik tööriist seal olemas olema. `python:3.13-slim`-is pole `curl`-i, aga Python on, nii et `app` kontrollib ennast Pythoni ühe reaga.

### Käsud

```bash
docker compose up -d                 # loo ja käivita, mis puudub või muutus
docker compose up -d --build --wait  # ehita, käivita ja oota tervist
docker compose ps                    # teenused ja tervis
docker compose logs -f app           # ühe teenuse logi
docker compose exec db psql -U labor # käsk töötavas konteineris
docker compose down                  # konteinerid ja võrk maha, volume'id jäävad
docker compose config                # fail pärast muutujate asendamist
```

`up -d` on ise idempotentne: kui midagi ei muutunud, jäävad konteinerid puutumata. Kui `.env`-is muutus muutuja, mida teenus kasutab, luuakse just see konteiner uuesti.

### Üks lõks: andmebaasi parool

`POSTGRES_PASSWORD` kasutatakse ainult siis, kui andmekaust on tühi, ehk esimesel käivitusel. Kui hiljem muudad `.env`-is parooli, saab `app` uue parooli, aga andmebaasis on vana. Tulemus on `password authentication failed`. Parooli vahetus käib andmebaasis (`ALTER ROLE`) ja alles siis `.env`-is. K2 kodutöö vabatahtlikus osas nägid sama probleemi teise nurga alt. Praktikumi B-osas teed selle läbi.

### Compose ja Podman

Compose'i failivorming on avatud standard (Compose Specification). `docker compose` on Dockeri ametlik plugin. `podman compose` on Podmani alamkäsk, mis ise faili ei loe, vaid kutsub välja kas `podman-compose` (eraldi Pythoni tööriist, AlmaLinuxis EPEL-ist) või `docker-compose` binaari, mis räägib Podmani sokliga. Lihtsate failide puhul töötavad kõik, keerulisemate puhul erinevad detailid. Seepärast kasutame Compose'i vm3-s koos Dockeriga ja vm1-s Podmani käsurida ja pod'e.

??? question "Kordamisküsimus"

    `app` kukub esimesel `docker compose up`-il, teisel korral töötab. Mis on tõenäoline põhjus ja mis kaks rida Compose-failis selle parandavad?

??? info "Loe juurde"

    - [Compose Specification](https://compose-spec.io/)
    - [Docker: Compose-faili viide](https://docs.docker.com/reference/compose-file/)
    - [Docker: käivitusjärjekord](https://docs.docker.com/compose/how-tos/startup-order/)
    - [postgres image'i dokumentatsioon](https://hub.docker.com/_/postgres)

## 8. Ansible ja konteinerid

Compose kirjeldab rakenduse. Keegi peab ikka paigaldama Dockeri, looma kausta, panema sinna Compose-faili ja `.env`-i ning käivitama. Käsitsi tehtuna on see sama käsitsi töö, mida K1-s kaotasime. Ansible teeb selle kahe rolliga:

| Roll | Mida teeb | Sarnane K2-ga |
|---|---|---|
| `docker` | Dockeri repo ja võti, paketid, teenus, kasutaja `docker`-gruppi | `common` |
| `stack` | kaust `/opt/labor`, Compose-fail ja rakenduse kood, `.env` mallist, käivitus | `app` + `lb` + `db` korraga |

### Idempotentne käivitus

`docker compose up -d` on ise idempotentne, aga `command`-moodul on Ansible'i jaoks alati `changed`. Lahendus on sama, mis K2 andmebaasi kasutajaga: muutus käivitab käsu ja kontroll otsustab, kas midagi on vaja teha.

```yaml
# tasks/main.yml
- name: Compose-fail, rakendus ja proxy seaded on paigas
  ansible.builtin.copy:
    src: "{{ playbook_dir }}/stack/{{ item }}"
    dest: "{{ stack_kaust }}/"
  loop: [compose.yaml, app, nginx]
  notify: Stack uueneb

- name: Muudatused rakenduvad kohe
  ansible.builtin.meta: flush_handlers

- name: Kontrolli, mis teenused käivad
  ansible.builtin.command: docker compose ps --services --status running
  args:
    chdir: "{{ stack_kaust }}"
  register: stack_kaivad
  changed_when: false
  check_mode: false

- name: Kõik teenused käivad
  ansible.builtin.command: docker compose up -d --wait --force-recreate
  args:
    chdir: "{{ stack_kaust }}"
  when: stack_kaivad.stdout_lines | length < 3

# handlers/main.yml
- name: Stack uueneb
  ansible.builtin.command: docker compose up -d --build --wait
  args:
    chdir: "{{ stack_kaust }}"
```

- kui fail muutus, ehitab ja käivitab handler uuesti;
- kui keegi peatas konteineri käsitsi (drift) või see on katki, näeb kontroll vähem kui kolme teenust ja loob konteinerid uuesti (andmed on volume'is);
- kui kõik on paigas, on teine jooks `changed=0`.

### Moodul või `command`

Kollektsioonis `community.docker` on moodul `docker_compose_v2`, mis teeb sama ja oskab ise öelda, kas midagi muutus. See loeb Compose'i väljundit ja peab olema Compose'i versiooniga kooskõlas. Kollektsiooni uuemad versioonid nõuavad uuemat `ansible-core`-i, kui AlmaLinux 9-s on (2.14). Viimane `ansible-core 2.14`-ga töötav haru on `community.docker` 3.x. Kodutöös H4 proovid moodulit ja võrdled.

Sama loogika kehtib Podmani kohta: kollektsioonis `containers.podman` on moodulid `podman_container`, `podman_pod` ja Quadlet'i failide jaoks.

Reegel jääb K2-st samaks: kui on moodul, mis sobib sinu versioonidega, eelista moodulit. Kui pole, tee `command` idempotentseks ise.

??? question "Kordamisküsimus"

    Mis juhtub tänase rolliga, kui keegi teeb vm3-s käsitsi `docker compose down`? Mis siis, kui ta teeb `docker compose down -v`? Kumba Ansible parandab ja kumba mitte?

??? info "Loe juurde"

    - [Docker: paigaldus RHEL-i perekonda](https://docs.docker.com/engine/install/rhel/)
    - [community.docker.docker_compose_v2](https://docs.ansible.com/ansible/latest/collections/community/docker/docker_compose_v2_module.html)
    - [containers.podman kollektsioon](https://github.com/containers/ansible-podman-collections)

## 9. Konteiner tootmises: kontrollnimekiri

Need ei ole eraldi teema, vaid harjumused, mida tänases praktikumis järgid:

| | Mida | Miks |
|---|---|---|
| versioon | `postgres:17-alpine`, mitte `postgres` või `latest`; digest, kui vaja täpset kordust | homme sama tulemus |
| register | täisnimi koos registriga | lühinimede rünnak, Podmani küsimine |
| kasutaja | `USER` image'is | konteinerist välja pääsenu ei ole root |
| saladused | `.env` või saladuste haldur, mitte `ENV`/`COPY` image'is | kihid on loetavad |
| pordid | avalda ainult see, mida peab väljast kasutama | Docker möödub firewalld-st |
| andmed | nimega volume õiges kaustas | konteiner on ajutine |
| tervis | healthcheck + `depends_on: condition: service_healthy` + `--wait` | käivitatud ei tähenda töötav |
| signaalid | exec-kujul `CMD`, vajadusel `init: true` | `stop` ei tohi oodata 10 sekundit |
| skaneerimine | `trivy image` enne tootmist (K4) | alusimage'i turvaaugud on sinu turvaaugud |

??? info "Loe juurde"

    - [Docker: ehituse head tavad](https://docs.docker.com/build/building/best-practices/)
    - [OWASP: Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
    - [Trivy](https://trivy.dev/)

## 10. Kokkuvõte

??? note "Põhimõtted ühes tabelis"

    | Põhimõte | Mida see tähendab |
    |---|---|
    | konteiner on protsess | namespace'id + cgroup'id; tuum on hostiga ühine |
    | Ansible serverile, image rakendusele | Pythoni versioon on image'is, Docker ja kaustad on Ansible'i rollis |
    | täisnimi ja versioon | `docker.io/library/postgres:17-alpine`; tag on silt, digest on sisu |
    | kihid: harva muutuv üles | teegid enne koodi; `.dockerignore` |
    | mitte root, mitte saladusi | `USER` image'is; paroolid käivitamisel, mitte ehitusel |
    | `localhost` on konteiner ise | teenused leiavad üksteist nime järgi kasutaja loodud võrgus |
    | andmed volume'is | `down` jätab alles, `down -v` kustutab; õige tee image'i dokumentatsioonist |
    | Docker: root-deemon | `docker`-grupp ≈ root; avaldatud port möödub firewalld-st |
    | Podman: rootless, pod'id | pordid alla 1024 ei käi; systemd ja Quadlet taaskäivituseks; `kube generate` |
    | Compose | üks fail kogu pinule; `.env`; healthcheck + `service_healthy`; `up -d --wait` |
    | idempotentne paigaldus | failid → handler `up --build --wait`; kontroll `ps --status running` → `up -d` |

??? info "Loe juurde"

    - [pikem materjal: Docker (IT-24)](https://hkhk-automation.github.io/devops/week05/lecture/)
    - [pikem materjal: Docker Compose (IT-24)](https://hkhk-automation.github.io/devops/week06/lecture/)

*Järgmine: [praktikum](lab.md). Juhendatud osas paned rakenduse vm1-s Podmaniga konteinerisse, iseseisvas osas paigaldab Ansible vm3-sse Dockeri ja kogu pinu Compose'iga. Seejärel [kodune õpe ja kodutöö](homework.md).*
