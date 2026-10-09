# Laboritahvel

Laboritahvel näitab, millised labori osad sinu virtuaalmasinates praegu töötavad. Kontroll käib iga viie minuti järel. Tahvel muutub koos kursusega: jooksva kohtumise kontrollid on üksikasjalikult näha, varasemad kohtumised kuvatakse ainult protsendina.

Tahvli aadressi annab õpetaja. See on nähtav koolivõrgus või VPN-iga. Tahvlil on kõigi õppijate nimed ja tulemused nähtavad.

## K1 · Esimene playbook

K1 osa näitab kolme kontrolli:

- kas `nginx` töötab masinates vm1, vm2 ja vm3;
- kas kasutaja `saidi` on olemas;
- kas veebileht vastab väljastpoolt virtuaalmasinat ehk tulemüür lubab ühenduse läbi.

## K2 · Kolmekihiline rakendus

K2 osa näitab, kas:

- PostgreSQL töötab vm1-s;
- `labori-app` töötab nii vm2-s kui ka vm3-s;
- SELinuxi luba on seadistatud;
- rakenduse leht vastab vm1-st;
- koormusjaotur suunab päringud vaheldumisi mõlemasse rakendusmasinasse.

## K3 · Podman, Docker ja Compose

Kontrollid lisanduvad selle kohtumise eel.

## K4 · CI/CD ja GitHub Actions

Kontrollid lisanduvad selle kohtumise eel.

## K5 · OpenTofu ja Kubernetes (k3s)

Kontrollid lisanduvad selle kohtumise eel.

## Tahvel ja Autograde täidavad eri ülesannet

Tahvel kontrollib virtuaalmasinate olekut ja seda, kas teenused vastavad. See ei kontrolli, millised failid on reposse esitatud. Seda teeb Autograde: see hindab koodi ja näitab labori Git-punkte Classroom 50 viimase esituse põhjal.

Masinaolekut kontrollitakse iga viie minuti järel. Git-punktid ei uuene selle intervalliga: need värskenevad siis, kui õpetaja tahvlit uuendab. Seepärast võib punktide muutumine pärast uut esitust viibida.

## Tulemuste lugemine

- ✓ tähendab, et kontroll on korras.
- Osaline tulemus, näiteks 2/3, näitab, et kontroll õnnestus osas masinatest, näiteks `nginx` töötab vm1-s ja vm2-s, aga mitte vm3-s.
- ✗ tähendab, et kontroll ei õnnestunud.
- Hall rida tähendab, et virtuaalmasinad ei vasta. Põhjuseks võib olla seisev VM või muutunud IP-aadress. Hall rida ei tähenda null punkti.
- Vihjed näitavad võimalikku vea asukohta ja juhatavad laborijuhendi sammuni, näiteks K2 A7 või B1. Need on suunad vea otsimiseks, mitte valmis lahendused.

Kui jääd hätta, küsi abi Discordis või ava oma repos issue „Vajan abi“.
