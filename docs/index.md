# DevOps Lite

IT-infrastruktuuri automatiseerimine täiskasvanud õppijale: käsitsi tööst korratava, versioonihallatud ja kontrollitud muutuseni.

See on [IT automatiseerimise kursuse](https://hkhk-automation.github.io/devops/) (13 nädalat) kokkusurutud versioon. Iga kohtumine katab 2–3 sealset nädalat. Pikemad selgitused ja lisaharjutused on viidatud iga kohtumise juures.

**Maht:** 5 kohtumist × 4 akadeemilist tundi (loeng kuni 30 min, ülejäänu praktikum) + iseseisev töö.
**Eeldused:** Linuxi käsurida, SSH, tekstiredaktor, Git baas, GitHubi konto. Bash ja Git on eeldused, mitte teemad.

---

## K1 · Idempotentsus ja esimene playbook

**Õpid:** miks käsitsi seadistus triivib; käsk vs soovitud olek; idempotentsus ja `changed=0` kui tõend; inventar, ad-hoc käsud, moodulid ja faktid; esimene playbook; `--check --diff`; drift; SSH-võtmed; sama playbook mitmele masinale; blast radius (`--limit`).

- **Klassis:** käsitsi → halb skript → playbook `localhost`-il → teine jooks `changed=0` → dry run → drift → SSH-võtmed → sama playbook kolmele VM-ile.
- **Kodus:** `admin.yml` kolmele masinale; playbook ühele oma töö korduvale tegevusele; teooria küsimused.
- **Pikemalt:** [N1](https://hkhk-automation.github.io/devops/week01/lecture/), [N3](https://hkhk-automation.github.io/devops/week03/lecture/)

## K2 · Serveripark

**Õpid:** inventari grupid ja pesastatud grupid; `group_vars` ja `host_vars`; muutujate eelistusjärjekord; Jinja2 mall (`template`-moodul); handler; üks kood, kaks keskkonda (test/prod).

- **Klassis:** grupid ja sihtimine → grupimuutujad → mall, mis näitab hosti ja keskkonda → handler, mis taaskäivitab teenuse ainult muutuse korral → dry run enne päris muutust.
- **Kodus:** drift hunt (leia ja paranda istutatud vead), peer attack paarilisega, väljakutse.
- **Pikemalt:** [N4](https://hkhk-automation.github.io/devops/week04/lecture/), [N12](https://hkhk-automation.github.io/devops/week12/lecture/)

## K3 · Rollid ja saladused

**Õpid:** rolli struktuur (`tasks`, `handlers`, `templates`, `defaults`); `site.yml`; `defaults` vs `vars`; Ansible Vault; saladused Gitis krüptituna; tagid.

- **Klassis:** K2 playbook rolliks → `defaults/main.yml` → Vaultis parool → `site.yml`, mis kasutab mitut rolli.
- **Kodus:** teine roll (nt `chrony` või kasutajad) ja mõlemad rollid kolmel masinal.
- **Pikemalt:** [N4](https://hkhk-automation.github.io/devops/week04/lecture/), [N11](https://hkhk-automation.github.io/devops/week11/lecture_ansible_roles/)

## K4 · Konteinerid

**Õpid:** image vs konteiner; Dockerfile ja kihid; cache; portide avamine; volume ja andmete püsivus; Compose: mitu teenust, nimega võrk, healthcheck, `depends_on`; konteiner vs Ansible'iga seadistatud VM.

- **Klassis:** valmis image → oma image → volume'iga andmebaas → Compose kahe teenusega.
- **Kodus:** Ansible paigaldab Dockeri ja käivitab Compose-stacki sihtserveris, teine jooks `changed=0`.
- **Pikemalt:** [N5](https://hkhk-automation.github.io/devops/week05/lecture/), [N6](https://hkhk-automation.github.io/devops/week06/lecture/)

## K5 · Infrastruktuur koodina ja CI

**Õpid:** deklaratiivne vs imperatiivne infra; OpenTofu provider, resource, state; `plan` → `apply` → muutus (`~` vs `-/+`) → drift → `destroy`; mis läheb Giti ja mis mitte; CI pipeline, mis kontrollib koodi igal pushil.

- **Klassis:** OpenTofu elutsükkel Dockeri provideriga; lühidemo GitHub Actionsist.
- **Kodus:** pipeline, mis jooksutab `ansible-lint` ja `tofu validate`; lõputöö plaan.
- **Pikemalt:** [N7](https://hkhk-automation.github.io/devops/week07/lecture/), [N9](https://hkhk-automation.github.io/devops/week09/async_task/), [N10](https://hkhk-automation.github.io/devops/week10/lecture/)

## Lõputöö (~18 h, iseseisev)

Probleem sinu töökohast või kodulaborist. Vähemalt kaks kursuse kihti koos (nt Ansible roll + Compose, või OpenTofu + Ansible), saladused krüptitud, README-s käivitusjuhis ja tõend, et teine jooks ei muuda midagi.

**Välja jäetud** võrreldes pika kursusega: automaatne image'i ehitamine ja registry (N8), OpenTofu moodulid (N11 valik 1), Kubernetes.

---

## Läbivad tööviisid

**Idempotentsus.** Sama samm annab sama tulemuse ka teisel käivitusel: `mkdir -p` → Ansible `changed=0` → OpenTofu `No changes`.

**Ennusta, siis kontrolli.** Enne olulist käsku kirjuta üles, mida ootad. Kui tulemus erineb, oled midagi valesti mõistnud, ja just see on kõige kasulikum koht õppimiseks.

**Viga on samm.** Mõnes labis on koht, kus midagi teadlikult ei tööta, et näha, kuidas tööriist selle lahendab.

## Keskkond

- **Control node:** sinu masin (WSL2, oma VM või Linux), kus on Ansible ja Git.
- **Sihtmasinad:** alguses `localhost`, seejärel klastri VM-id, mille aadressid annab juhendaja.

## Esitamine

Iga praktikumi jaoks on Classroom 50 ülesanne. Link tuleb kohtumisel, sellest tekib sulle oma repo.

- Tähtaeg on kirjas Classroom 50-s. Lubatud on üks hilinenud esitus, pärast seda on hinne MA.
- Kursuse läbimiseks on vaja vähemalt 80% praktikumidest. Muidu tuleb kirjalik eksam.
- Reposse ei lähe paroole, võtmeid ega tokeneid.
