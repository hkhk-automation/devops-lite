# DevOps Lite

IT-infrastruktuuri automatiseerimine täiskasvanud õppijale: käsitsi tööst korratava, versioonihallatud ja kontrollitud muutuseni.

**Maht:** 5 kohtumist × 4 akadeemilist tundi + iseseisev töö.
**Eeldused:** Linuxi käsurida, SSH, tekstiredaktor, Git baas, GitHubi konto. Bash ja Git on eeldused, mitte teemad.

## Kohtumised

Igal kohtumisel on üks tööriist ja üks põhiidee. Klassis tehakse juhendatud labor, kodus süvendatakse ja kantakse sama oskus oma töö konteksti.

| # | Teema | Klassis | Kodus |
|---|---|---|---|
| 1 | Idempotentsus ja esimene playbook | käsitsi → halb skript → playbook → `changed=0` → drift → Git | `admin.yml` + playbook ühele oma töö korduvale tegevusele |
| 2 | Mitu hosti SSH-ga | inventari grupid, sihtimine, grupimuutujad, `--check --diff` | drift hunt, peer attack, väljakutse |
| 3 | Rollid ja saladused | Jinja2 mall, handler, roll, Ansible Vault | eelmise kohtumise playbook rolliks |
| 4 | Konteinerid | oma image, volume, Compose kahe teenusega | Ansible käivitab Compose-stacki |
| 5 | Infrastruktuur koodina | OpenTofu: plan → apply → muutus → drift → destroy | CI pipeline + lõputöö |

Lõputöö (~18 h) on iseseisev: vähemalt kaks kursuse kihti koos ja tõend, et teine jooks ei muuda midagi.

## Läbivad tööviisid

**Idempotentsus.** Sama samm annab sama tulemuse ka teisel käivitusel: `mkdir -p` → Ansible `changed=0` → OpenTofu `No changes`.

**Ennusta, siis kontrolli.** Enne olulist käsku kirjuta üles, mida ootad. Kui tulemus erineb, oled midagi valesti mõistnud ja see on kõige kasulikum koht õppimiseks.

**Viga on samm.** Mõnes labis on koht, kus midagi teadlikult ei tööta, et näha, kuidas tööriist selle lahendab.

## Keskkond

- **Kohtumine 1:** üks Ubuntu/Debian masin, kus sul on sudo: WSL2, oma VM või Linuxi sülearvuti. Sihtmasin on `localhost`.
- **Alates kohtumisest 2:** klassis antakse Proxmoxi VM-id sihtserveriteks.

## Esitamine

Iga labori jaoks on Classroom 50 ülesanne. Link tuleb kohtumisel, sellest tekib sulle oma repo.

- Tähtaeg on kirjas Classroom 50-s. Lubatud on üks hilinenud esitus, pärast seda on hinne MA.
- Kursuse läbimiseks on vaja vähemalt 80% laboritest. Muidu tuleb kirjalik eksam.
- Reposse ei lähe paroole, võtmeid ega tokeneid.

## Lisamaterjal

Kes tahab sügavamale, leiab samade teemade pikemad versioonid [IT automatiseerimise kursuselt](https://hkhk-automation.github.io/devops/).
