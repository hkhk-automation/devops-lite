# K4 · Kodune õpe ja kodutöö

Kodutöö on vabatahtlik boonus: hinne tuleb klassitöö kontrollidest K1–K5. Kodutöö läheb samasse repositooriumisse, kuhu klassitöö. Tähtaeg on kirjas Classroom 50-s. Kodutöös viid pipeline'i edasi: image'i suitsutest enne registrit, turvaaukude skaneerimine, tagasipööramine eelmisele versioonile ja action'ite versioonid kinni. Juhiseid on vähem kui klassis. Kui jääd kinni, küsi Discordis või ava oma repositooriumis issue **Vajan abi** ja kirjuta sinna käsk ning veateade.

Main haru on kaitstud, seega läheb iga muudatus harusse ja pull request'iga main harusse ([praktikum B4](lab.md#b4-toota-kaitstud-main-haruga)). Autograde hindab main haru.

| Osa | Mida teed | Esitad | Hinne |
|---|---|---|---|
| [I](#i-lugemine-ja-kusimused) | loed, uurid, vastad küsimustele | `markmed.md`, `vastused.md` | boonus |
| [H1](#h1-suitsutest-enne-registrit-boonus) | image käivitub CI-s koos PostgreSQL-iga enne `docker push`-i | `ci.yml` | boonus |
| [H2](#h2-skanneeri-image-boonus) | Trivy otsib image'ist turvaauke | `ci.yml`, `logid/trivy.txt` | boonus |
| [H3](#h3-poora-tagasi-eelmisele-versioonile-boonus) | tagasipööramine eelmisele räsile | `logid/tagasi.txt` | boonus |
| [H4](#h4-actionite-versioonid-kinni-boonus) | action'id commit'i räsiga, Dependabot | `ci.yml`, `.github/dependabot.yml` | boonus |
| [H5](#h5-leia-kolleegi-vead-boonus) | neli istutatud viga kaustas `vead/` | `vead.md` | boonus |
| [III](#iii-eneseanaluus) | eneseanalüüs | `vastused.md` lõpus | – |

Tulemust näed pärast iga liitmist: **Actions** → **Autograde**. Kodutöö kontrollide nimi algab sõnaga BOONUS ja need punkte ei anna: iga läbitud boonuskontroll annab [laboritahvlil](../tahvel.md) tähe ja XP-d. Punane rist Autograde'is tähendab ainult, et boonus on tegemata. **CI** peab jääma roheliseks ka kodutöö järel.

## I · Lugemine ja küsimused

Enne küsimusi loe läbi:

??? note "Lugemisnimekiri"

    | Mida | Kus |
    |---|---|
    | Loeng: klassis käsitlemata osad §3, §6, §8 | [K4 loeng](lecture.md) |
    | Workflow süntaks: `jobs.<id>.services`, `if`, `permissions` | [docs.github.com](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax) |
    | Actionsi turvalisus | [docs.github.com](https://docs.github.com/en/actions/reference/security/secure-use) |
    | gitleaks: kuidas reeglid töötavad | [github.com/gitleaks](https://github.com/gitleaks/gitleaks#configuration) |

### Uuri → `markmed.md`

1. Ava oma viimase rohelise käivituse `build` job → **Set up job**. Kirjuta üles runneri image'i versioon ja `GITHUB_TOKEN Permissions` nimekiri. Võrdle seda `lint` jobi omaga. Mis on erinev ja miks?
2. Ava [gitleaksi vaikereeglid](https://github.com/gitleaks/gitleaks/blob/master/config/gitleaks.toml) ja leia reegel `generic-api-key`. Mis võtmesõnu see otsib? Miks praktikumi A4-s parooli ei leitud?

### Vasta küsimustele → `vastused.md`

Vasta lühidalt oma sõnadega, üks-kaks lauset igale.

1. A1-s oli runneris kaust tühi. Mida see ütleb runneri kohta ja miks on see hea? (lab A1, loeng §2)
2. B1-s õnnestus sisselogimine, aga `docker push` kukkus. Miks on õige anda `packages: write` ainult `build` jobile? (lab B1, loeng §5)
3. B4-s jäi `lint` punaseks ka pärast tokeni kustutamist. Mida peaksid tegema, kui token oleks olnud kehtiv? (lab B4, loeng §3)

## II · Harjutused

!!! tip "Soovitatav töökäik"

    1. `git switch -c <haru>`
    2. muuda, commit, `git push -u origin <haru>`, ava PR
    3. vaata PR-i kontrolle, paranda samas harus
    4. roheline PR → liida, `git switch main && git pull`

### H1 · Suitsutest enne registrit · boonus

Selle ülesande lõpuks käivitab `build` job ehitatud image'i koos PostgreSQL-iga ja `docker push` toimub ainult siis, kui rakendus vastab `/health` päringule.

Lint kontrollib faile, aga mitte seda, kas image käivitub. Suitsutest on lühike kontroll, et rakendus üldse töötab.

- `build` jobil on teenuskonteiner (`services:`) `postgres`, sama image mis `stack/compose.yaml`-is, ja tervisekontroll;
- pärast ehitust käivitab step rakenduse konteineri, mis saab andmebaasiga ühendust;
- step kontrollib `curl`-iga `/health`-i ja kukub, kui vastus pole 200;
- suitsutest on enne `docker push`-i.

??? tip "Vihje: teenuskonteiner ja võrk"

    [Teenuskonteinerid](https://docs.github.com/en/actions/tutorials/use-containerized-services/create-postgresql-service-containers): kui job töötab otse runneris (mitte konteineris), on teenus aadressil `localhost` ja port tuleb avaldada (`ports: ["5432:5432"]`). Rakenduse konteiner jõuab runneri `localhost`-ile `--network host` abil. Tervisekontroll: `options: >- --health-cmd pg_isready --health-interval 5s --health-retries 10`. `curl --retry 10 --retry-connrefused --fail`.

Valmis, kui
{ .silt }

- [ ] PR-is on `build` roheline ja suitsutesti step'i logis on `ok`
- [ ] kui `DB_HOST` on vale, on `build` punane ja push'i ei toimu
- [ ] `ci.yml`-is on `services:` ja `postgres` ning `/health` (Autograde otsib neid)

Esitad
{ .silt }

`.github/workflows/ci.yml`

### H2 · Skanneeri image · boonus

Selle ülesande lõpuks otsib pipeline image'ist teadaolevaid turvaauke ja sa tead, kui palju neist on sinu koodis ja kui palju alusimage'is.

- uus step `build` jobis pärast ehitust ja enne `docker push`-i (image on ainult selle jobi runneris);
- [Trivy](https://trivy.dev/) skannib sama image'it, mis ehitati;
- pipeline kukub ainult kriitiliste aukude peale, millel on parandus olemas;
- käivita sama skannimine vm3-s registris oleva image'i peal ja salvesta tulemus.

??? tip "Vihje: Trivy ilma paigalduseta"

    Trivy on ka image: `docker run --rm -v /var/run/docker.sock:/var/run/docker.sock docker.io/aquasec/trivy:0.76.1 image --severity CRITICAL --ignore-unfixed --exit-code 1 <image>`. Kinnita versioon nagu teistel tööriistadel. Kolmanda osapoole action'it kasutades kinnita see commit'i räsiga (H4).

Valmis, kui
{ .silt }

- [ ] CI on roheline või punane põhjusega, mille oskad selgitada
- [ ] `ci.yml`-is on `trivy` (Autograde otsib seda)
- [ ] `logid/trivy.txt`-is on skannimise tulemus ja `vastused.md`-s kaks lauset: mitu auku leiti ja kas need on alusimage'is või sinu koodis

Esitad
{ .silt }

```bash
ssh -t vm3 "sudo docker run --rm -v /var/run/docker.sock:/var/run/docker.sock docker.io/aquasec/trivy:0.76.1 image --severity HIGH,CRITICAL <image>:<räsi>" | tee logid/trivy.txt
```

### H3 · Pööra tagasi eelmisele versioonile · boonus

Selle ülesande lõpuks oled tarninud vm3-sse eelmise versiooni ja jälle uue, ilma ühegi uue ehituseta.

Registris on iga commit'i image alles. Tagasipööramine on eelmise räsi tarnimine.

- leia **Packages** lehelt või `git log --oneline`-ist kaks räsi, millel on image;
- tarni vanem, kontrolli lehte, tarni uuem;
- mõlema järel on leht roheline ja näitab õiget räsi.

??? question "Mõtle (vabatahtlik)"

    Mis juhtuks tagasipööramisega, kui uus versioon oleks muutnud andmebaasi tabelit? Mida see tähendab andmebaasi muudatuste kohta?

Valmis, kui
{ .silt }

- [ ] `logid/tagasi.txt`-is on kaks erinevat `Labori seis <räsi>` rida (Autograde loeb neid)

Esitad
{ .silt }

```bash
VM3=$(ssh -G vm3 | awk '/^hostname /{print $2}')
ansible-playbook site.yml -J -e app_versioon=<vanem> && curl -s http://$VM3/ | tee logid/tagasi.txt
ansible-playbook site.yml -J -e app_versioon=<uuem> && curl -s http://$VM3/ | tee -a logid/tagasi.txt
```

### H4 · Action'ite versioonid kinni · boonus

Selle ülesande lõpuks on kõik action'id `ci.yml`-is kinnitatud commit'i räsiga ja Dependabot teeb PR-i, kui neist tuleb uus versioon.

`actions/checkout@v7` on tag ja tag'i saab liigutada. Kui action'i omaniku konto kaaperdatakse, võib `v7` homme näidata teisele koodile, mis näeb sinu `GITHUB_TOKEN`-it. Commit'i räsi ei muutu.

- iga `uses:` rida on kujul `omanik/nimi@<40-märgine räsi> # v7.1.0`;
- `.github/dependabot.yml` kontrollib kord nädalas ökosüsteemi `github-actions`;
- CI on roheline.

??? tip "Vihje: kust räsi saada"

    `git ls-remote https://github.com/actions/checkout refs/tags/v7.0.1`. Dependabot: [konfiguratsioonifaili näide](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/keeping-your-actions-up-to-date-with-dependabot). Kommentaar räsi järel jääb inimesele loetavaks ja Dependabot uuendab ka seda.

Valmis, kui
{ .silt }

- [ ] `ci.yml`-is on iga `uses:` 40-märgise räsiga (Autograde kontrollib)
- [ ] `.github/dependabot.yml`-is on `github-actions` (Autograde otsib seda)

Esitad
{ .silt }

`.github/workflows/ci.yml`, `.github/dependabot.yml`

### H5 · Leia kolleegi vead · boonus

Selle ülesande lõpuks oled leidnud neli viga, mille kolleeg jättis kausta `vead/`, ja kirjutanud iga kohta, kuidas sa selle leidsid.

Kaustas `vead/` on neli workflow faili. Igaüks on töötava `ci.yml`-i koopia, milles on üks viga. Iga faili jaoks:

1. võrdle faili oma `ci.yml`-iga (`diff vead/1-ci.yml .github/workflows/ci.yml`) ja ennusta, mis juhtub;
2. tee haru, kopeeri fail `.github/workflows/ci.yml`-iks, push, ava PR ja vaata, mis tegelikult juhtub;
3. sulge PR liitmata ja kustuta haru.

Kolm viga on PR-is näha. Üks avaldub ainult main harus, sest PR ei pane registrisse. Selle tagajärje leiad loengust ja B1-st.

!!! warning "Tähelepanu: kolm viga ei anna veateadet"

    Ainult üks viga teeb pipeline'i punaseks. Ülejäänud kolm jätavad kõik roheliseks või ei käivita midagi ja on ohtlikud alles hiljem: image jõuab registrisse kontrollimata, pipeline ei käivitu üldse või saladus jääb leidmata. Kontrolli, mis toimus: kas käivitus on olemas, mis järjekorras jobid käisid, mitu commit'i `gitleaks` vaatas.

Kirjuta `vead.md`-sse iga vea kohta pealkiri `## <faili nimi>` ja selle alla kolm rida: mida nägid (sümptom), mis oli põhjus, kuidas selle leidsid.

Valmis, kui
{ .silt }

- [ ] `vead.md`-is on neli `##` pealkirja (Autograde loeb neid)
- [ ] main harus on sinu õige `ci.yml` ja CI on roheline

Esitad
{ .silt }

`vead.md`

## III · Eneseanalüüs

`vastused.md` lõppu kolm vastust, igaüks üks lause:

1. Mis töötas esimese korraga?
2. Kus jäid kinni ja mis aitas edasi?
3. Mida tahad järgmisel kohtumisel küsida?

??? note "Vabatahtlik: paigaldus self-hosted runneriga"

    Pidev paigaldus tähendaks, et pärast rohelist `build`-i käivitab `site.yml`-i keegi muu kui sina. GitHubi runner kooli võrku ei pääse, aga vm1-sse paigaldatud [self-hosted runner](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/add-runners) ühendub ise GitHubiga ja ootab töid. Runneri lisamiseks peab olema repositooriumi administraator, seega klassis seda ei tee.

    Mõtle läbi ja kirjuta `vastused.md`-sse paar lauset: mis saladusi vajaks job `tarne` (`runs-on: self-hosted`, `needs: build`) ja kus neid hoiaksid? Kes saaks pärast seda vm3-s käske käivitada? Miks on kaitstud main haru siis veel olulisem?
