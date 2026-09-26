# Töökeskkond

Tee see leht läbi üks kord kursuse alguses. Iga järgmine praktikum eeldab, et see on tehtud: masinatel on nimed ja sinu parool, vm1-s on Ansible ja Git, ning vm1 SSH-võti on GitHubis.

## 1 · Ühendus vm1-ga

Klassiarvuti on Windows, aga töö käib kooli Proxmoxi klastris sulle antud kolmes AlmaLinux 9 VM-is. vm1 on sinu control node: sinna paigaldad Ansible'i ja Giti ning sealt haldad kõiki kolme masinat, ka vm1 ennast. Esimestes sammudes töötad ainult vm1-s (`localhost`), hiljem lisanduvad vm2 ja vm3.

Juhendaja annab sulle kolm IP-d, kasutajanime ja parooli. Kirjuta need üles:

| Nimi | IP | Kasutaja |
|---|---|---|
| vm1 (control node) | | |
| vm2 | | |
| vm3 | | |

Ühendu Windowsist vm1-ga:

=== "VS Code (soovitatav)"

    Paigalda laiendus **Remote - SSH**. Vajuta ++f1++ → *Remote-SSH: Connect to Host* → `<kasutaja>@<vm1-ip>`. Terminal avaneb otse vm1-s (++ctrl+"ö"++), failid näed külgpaanil.

=== "PowerShell"

    ```powershell
    ssh <kasutaja>@<vm1-ip>
    ```

Esimesel ühendumisel küsitakse host key kinnitust (`yes`) ja parooli.

Allikas: [VS Code Remote-SSH](https://code.visualstudio.com/docs/remote/ssh)

Juhendajalt saadud parool on kõigil tudengitel sama ja masinad on ühes võrgus, seega vaheta see. Masinatel pole ka veel nime (prompt näitab `localhost`). Tee mõlemad asjad igas masinas.

vm1 (oled juba sees):

```bash
passwd
sudo hostnamectl set-hostname vm1
exec bash
```

`passwd` küsib vana parooli, siis kaks korda uut. Liiga lihtsa lükkab AlmaLinux tagasi (`BAD PASSWORD`): vali vähemalt 8 märki tähtede ja numbritega. Pane kõigis kolmes masinas sama uus parool, sest Ansible küsib sudo parooli ühe korra ja kasutab seda kõigil kolmel.

vm2 (vm1 terminalist):

```bash
ssh <kasutaja>@<vm2-ip>
passwd
sudo hostnamectl set-hostname vm2
exit
```

vm3: sama, `vm3` nimega.

??? success "Oodatav tulemus"

    vm1 prompt on `<kasutaja>@vm1`. `ssh <kasutaja>@<vm2-ip> hostname` vastab `vm2`, vm3 samamoodi. Kõik järgmised käsud käivad vm1-s, mitte Windowsis.

## 2 · Tööriistad vm1-s

```bash
sudo dnf install -y git ansible-core
ansible-galaxy collection install ansible.posix:1.5.4
git --version
ansible --version | head -3
```

??? success "Oodatav tulemus"

    ```
    git version 2.52.0
    ansible [core 2.14.18]
      config file = /etc/ansible/ansible.cfg
      configured module search path = [...]
    ```

Versioonid võivad veidi erineda. Oluline on, et `ansible` vastab. `ansible.posix` kollektsiooni (tulemüüri moodul) läheb vaja K1 B-osas. Versioon 1.5.4, sest uuemad ei toeta AlmaLinuxi `ansible-core 2.14`-t.

## 3 · SSH-võti

Üks võtmepaar vm1-s teeb kaks asja: sellega kloonid oma privaatse repo GitHubist ja sellega ühendub Ansible vm2 ja vm3 külge. Parooli pole kummalgi juhul vaja.

Loo võti. Vajuta kõigi küsimuste peale Enter:

```bash
ssh-keygen -t ed25519 -C "<eesnimi>@vm1"
cat ~/.ssh/id_ed25519.pub
```

??? success "Oodatav tulemus"

    Üks rida, mis algab `ssh-ed25519 AAAA...` ja lõpeb `<eesnimi>@vm1`. See on avalik võti, seda võib jagada. Fail `~/.ssh/id_ed25519` (ilma `.pub`-ita) on privaatvõti, see ei lahku kunagi vm1-st.

## 4 · Võti GitHubi ja repo kloonimine

Lisa avalik võti GitHubi:

1. GitHub → paremal üleval profiilipilt → **Settings** → **SSH and GPG keys** → **New SSH key**.
2. *Title:* `vm1`, *Key type:* Authentication Key, *Key:* kleebi `cat` väljundist kogu rida.
3. **Add SSH key**.

Kontrolli vm1-s:

```bash
ssh -T git@github.com
```

??? success "Oodatav tulemus"

    Esimesel korral kinnita `yes`, siis:

    ```
    Hi <sinu-github-kasutaja>! You've successfully authenticated, but GitHub does not provide shell access.
    ```

Seadista Git ja klooni repo. Ava Classroom 50 link, mille juhendaja jagas, ja nõustu ülesandega. Sulle tekib privaatne repo organisatsioonis `hkhk-automation`. Repo lehel vajuta **Code** → vahekaart **SSH** → kopeeri aadress (algab `git@github.com:`).

```bash
git config --global user.name "Eesnimi Perenimi"
git config --global user.email "sinu@email.ee"
cd ~
git clone git@github.com:hkhk-automation/<sinu-repo>.git
cd <sinu-repo>
ls
```

??? success "Oodatav tulemus"

    ```
    README.md  ULESANNE.md  logid
    ```

Iga praktikum on eraldi Classroom 50 ülesanne ja eraldi repo: igal kohtumisel kloonid uue repo samamoodi. `git push` töötab sama võtmega, parooli ega tokenit ei küsita.

Allikas: [GitHub: SSH-võti kontole](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)

## 5 · Kodust ligi

Kodust pääsed oma masinatele ainult kooli VPN-iga. Kui VPN on ühendatud, töötab kõik nagu klassis: VS Code Remote-SSH `<kasutaja>@<vm1-ip>` ja sealt edasi.
