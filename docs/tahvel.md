# Laboritahvel

Tahvel näitab, mis sinu VM-ides praegu töötab. Kontroll käib iga 5 minuti järel. Ava see menüüst **Ava laboritahvel**. Tahvel avaneb ainult koolivõrgus või VPN-iga. Tahvlil on kõigi õppijate read.

Tahvel kontrollib masinaid. Faile sinu repos kontrollib Autograde, selle punktid on tahvlil eraldi veerus.

## Ühenda VM-id tahvliga

Tee seda üks kord. Tahvli aadressi näed brauseri aadressiribal, kui avad menüüst **Ava laboritahvel** (kujul `http://192.168.x.x:8090/`). Allpool on see `<tahvel>`.

**Ansible'iga.** Jooksuta vm1-s oma repo kaustas (seal, kus on `ansible.cfg` ja inventar):

```bash
curl -sO http://<tahvel>:8090/opetaja.yml && ansible-playbook opetaja.yml
```

Playbook töötab kõigi inventari masinatega, grupi nimi pole oluline. Iga masina juures peab olema `changed` või `ok`.

**Käsitsi.** Kui Ansible ei tööta, jooksuta see käsk **igas VM-is eraldi** (vm1, vm2, vm3):

```bash
mkdir -p ~/.ssh && curl -s http://<tahvel>:8090/opetaja.pub >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && echo OK
```

Mõlemal juhul lisatakse õpetaja kontrollvõti sinu kasutajale. Tahvel saab siis su masinates olekut lugeda, aga mitte midagi muuta: tal pole sinu parooli ega sudo-õigust. Viie minuti pärast on su rida tahvlil täidetud. Kui tahad tahvli hiljem lahti ühendada, kustuta failist `~/.ssh/authorized_keys` rida, mille lõpus on `kontroll@...`.

## Mida tahvel veel näeb

- **Töötavad teenused** igas VM-is (vaheleht Masinad), nt `nginx`, `postgresql`, `labori-app`.
- **Käsuajalugu ainult arvudena:** mitu korda oled jooksutanud `ansible-playbook`, `site.yml`, `--check`, vaadanud logisid (`journalctl`, `error.log`), teinud `git push`. Käske ennast ega nende sisu tahvel ei loe ega näita. Ära kirjuta paroole käsureale, see on hea tava niikuinii.

Ajalugu salvestub faili tavaliselt alles väljalogimisel. Kui tahvel näitab „ajalugu puudub“, logi korra välja ja sisse.

## K1 · Esimene playbook

- `nginx` käib vm1-s, vm2-s ja vm3-s;
- kasutaja `saidi` on olemas;
- leht vastab väljast, st tulemüür on lahti.

## K2 · Kolmekihiline rakendus

- PostgreSQL käib vm1-s;
- `labori-app` käib vm2-s ja vm3-s;
- SELinuxi luba `httpd_can_network_connect` on sees;
- leht vastab vm1-st;
- päringud vastavad vaheldumisi vm2-st ja vm3-st.

## K3 · Podman, Docker ja Compose

Kontrollid lisanduvad enne kohtumist.

## K4 · CI/CD ja GitHub Actions

Kontrollid lisanduvad enne kohtumist.

## K5 · OpenTofu ja Kubernetes

Kontrollid lisanduvad enne kohtumist.

## Märgid

| Märk | Tähendus |
|---|---|
| ✓ | korras |
| 2/3 | korras kahes masinas kolmest |
| ✗ | puudu |
| hall rida | VM-id ei vasta: VM seisab või IP muutus |
| Git | viimase esituse Autograde'i punktid; uuenevad, kui juhendaja tahvli uuendab |
| vihje | mis masinas mis on valesti ja milline juhendi samm aitab (nt K2 A7) |
| ⏰ | labor sai Autograde'is 100% enne tähtaega |
| 🐦 | 100% vähemalt kolm päeva enne tähtaega |
| 🥇 | esimene rühmas, kes sai selle labori 100% |
| 🔁 | visa: vähemalt viis esitust, lõpuks 100% |

## Kui VM ei vasta

[![The IT Crowd GIF](https://media.giphy.com/media/v1.Y2lkPWVjZjA1ZTQ3MHh0a3djMmU1Ymk1Z3o0dW9ieWk3ZzJyZnYxODNreGZ1Z2FhMWltZyZlcD12MV9naWZzX3JlbGF0ZWQmY3Q9Zw/dlMIwDQAxXn1K/giphy.gif)](https://giphy.com/theitcrowd)

*„Kas proovisid välja ja uuesti sisse lülitada?“* Enne taaskäivitamist kontrolli, kas VM seisab või IP-aadress on muutunud. [GIF: The IT Crowd GIPHYs](https://giphy.com/theitcrowd).

Kinni? Küsi Discordis või ava oma repos issue **Vajan abi**.
