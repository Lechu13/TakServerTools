# TAK Server 4.9 AWS poradnik
instrukcja na podstawie oficjalnej isntrukcji TAK
**Wymagania**
Konto na AWS z podpiętą kartą kredytową
[Jest to niezbędne aby uruchomić instancję serwera w chmurze]

**Przygotowanie**
pobieramy pliki ze strony [https://tak.gov/](https://tak.gov/) do folderu

    takserver_4.9-RELEASE23_all.deb
    takserver-public-gpg.key
    deb_policy.pol

**Uruchomienie instancji c5.xlarge 40gb**
**Nazwa:** [dowolna] np. TAK-Server
**OS:** Ubuntu 22.04
Instance type: c5.xlarge (sugerowana przez TAK koszt 0,386$/h)
**Key pair:** (jeśli nie posiadasz, pamiętaj o pobraniu go do folderu z plikami) create new key pair (jeśli posiadasz wybierz z listy)
**Network:** zostawiamy, wrócimy do konfiguracji później
**Storage:** 40gb gp2 (sugerowane przez TAK)
klikamy **Launch instance**

 - Przechodzimy do ekranu naszej instancji
 - Zaznaczamy instancję i klikamy Connect
 - Z zakładki SSH client kopiujemy swój adres serwera, przyda się
   później
   [ubuntu@ec2-3-9-179-24.eu-west-2.compute.amazonaws.com](mailto:ubuntu@ec2-3-9-179-24.eu-west-2.compute.amazonaws.com)
 - Wracamy do zakładki EC2 Instance Connect i klikamy Connect bez
   edytowania niczego

**Wysyłka plików do instancji**

 Otwieramy terminal
 Idziemy do folderu, w którym znajdują się pobrane pliki np:

    cd /Users/lechu/projects/TAK

wysyłamy pliki za pomocą scp, schemat komendy:

    scp -i "<Nazwa Klucza>" <Nazwa Pliku> ubuntu@<Adres instancji>.compute.amazonaws.com:<Miejsce docelowe na serwerze>

Przykład (pamiętaj aby klucz .pem znajdował się w folderze z którego wykonujesz komendę) :  

    scp -i "TAK_KEY.pem" takserver_4.9-RELEASE23_all.deb ubuntu@ec2-3-8-118-190.eu-west-2.compute.amazonaws.com:/home/ubuntu

Jeśli w procesie zostaniesz poproszony o potwierdzenie połączenia wpisz **yes**
W razie problemów z uprawnieniami klucza aws  
linux/mac:

    chmod 400 [Nazwa klucza].pem

Windows:  
[https://www.youtube.com/watch?v=P1erVo5X3Bs](https://www.youtube.com/watch?v=P1erVo5X3Bs)  
użyj powrshell!

**Pamiętaj o wysłaniu wszystkich 3 plików:**

 - takserver_4.9-RELEASE23_all.deb
 - takserver-public-gpg.key
 - deb_policy.pol

**konfiguracja sieciowa instancji:**
w bocznym menu wybieramy 

    Network > Security > Security Groups

Wybieramy Create security group nadajemy jej dowolną nazwę i dodajemy poniższe inbound rules  
![](https://lh5.googleusercontent.com/Dw_H61B_3hK6FTu3uphQkS3daiqV7-q4op3CoAWz9mrtd5VYJUqRHEwjY6ES4qyjQ58jNbYxM7O9sv3jo238Mc9S9O8UdaXJbFDvA8RwpAT-jrTM1F-xctaOQsUyHaZWITQGQ2EJoN0QdQNB17hKbGM)

**Sprawdzenie sygnatur**
wracamy do konsoli instancji
Otwieramy plik **deb_policy.pol** w dowolnym edytorze tekstu i kopiujemy swoje ID i podmieniamy w komendach. **Przykład F06237936851F5B5**

    sudo apt-get update
    sudo apt install debsig-verify
    sudo mkdir /usr/share/debsig/keyrings/[ID]
    sudo mkdir /etc/debsig/policies/[ID]
    sudo touch /usr/share/debsig/keyrings/[ID]/debsig.gpg
    sudo gpg --no-default-keyring --keyring /usr/share/debsig/keyrings/[ID]/debsig.gpg --import takserver-public-gpg.key
    sudo cp deb_policy.pol /etc/debsig/policies/[ID]/debsig.pol
    debsig-verify -v takserver_4.9-RELEASE23_all.deb

**zwiększamy limit połączeń**

    echo -e "* soft nofile 32768\n* hard nofile 32768" | sudo tee -- append /etc/security/limits.conf > /dev/null

**instalacja bazy danych**

    sudo mkdir -p /etc/apt/keyrings
    sudo curl https://www.postgresql.org/media/keys/ACCC4CF8.asc --output /etc/apt/keyrings/postgresql.asc
    sudo sh -c 'echo "deb [signed-by=/etc/apt/keyrings/postgresql.asc] http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/postgresql.list'
    sudo apt update

**instalacja serwera TAK**

    sudo apt install ./takserver_4.9-RELEASE23_all.deb

Jeśli zostaniesz zapytany o coś w procesie wpisz **y** i zatwierdź

**Podstawowa konfiguracja bazy danych**

    sudo /opt/tak/db-utils/takserver-setup-db.sh
    sudo systemctl daemon-reload

**Przydatne komendy:**
Uruchamiaj server TAK przy starcie instancji

    sudo systemctl enable takserver

uruchom/zatrzymaj server TAK

    sudo systemctl [start|stop] takserver

**konfiguracja certyfikatów:**

    sudo su tak
    nano /opt/tak/certs/cert-metadata.sh

ustawiamy dowolne wartości w STATE,CITY,ORGANIZATIONAL_UNIT podmieniając ${nazwa}

Aby opóścić edytor nano **ctl + x** potem **y** i zatwierdzamy enter tą samą nazwę pliku

    cd /opt/tak/certs
    ./makeRootCa.sh --ca-name takserver-CA
    ./makeCert.sh server takserver

Certyfikat dla użytkownika [user] jako nazwa

    ./makeCert.sh client user
  
Certyfikat dla admina

    ./makeCert.sh client admin
    exit
    sudo systemctl restart takserver

**poczekaj minutę aż serwer się uruchomi**

    sudo java -jar /opt/tak/utils/UserManager.jar certmod -A /opt/tak/certs/files/admin.pem

jeśli wyskoczy **java.lang.reflect.InvocationTargetException** poczekaj kolejną minutę i spróbuj ponownie

Konfigurujemy serwer

    sudo su tak
    cd /opt/tak
    nano CoreConfig.xml

w tagu **<input** usuwamy **coreVersion=”2”** i dodajemy **auth=”x509”**

w tagu **<tls** zmieniamy ścieżki do **serverKeystore** i **truststore** dodając na początku: **/opt/tak/**
zapisz plik **ctrl + x y i enter**

    exit

pobieramy certyfikat do panelu administracyjnego oraz dla użytkowników

    cd /opt/tak/certs/files
    sudo cp truststore-root.p12 lechu.p12 test.p12 admin.p12 /home/ubuntu
    cd /home/ubuntu
    sudo chmod -R o+rw truststore-root.p12 lechu.p12 test.p12 admin.p12
    
w terminalu na naszym komputerze

    scp -i "TAK_KEY.pem" -r ubuntu@ec2-3-9-179-24.eu-west-2.compute.amazonaws.com:/home/ubuntu/<Nazwa Pliku> /Users/lechu/projects/TAK-TUTORIAL

Pobierz wszytskie pliki:
 - truststore-root.p12  
 - user.p12  
 - admin.p12

Dodajemy certyfikat do przeglądarki lub keystore
https://support.google.com/chrome/a/answer/3505249?hl=pl
https://support.apple.com/pl-pl/guide/keychain-access/kyca2431/mac

**Konfiguracja z poziomu panelu administratora:**
Podstaw swój publicIp instancji np. **3.9.179.24**

https://[publicIP]:8443/setup  
Jeśli przeglądarka oznaczy stronę jako niebezpieczna rozwiń opcje zaawansowane i zaakceptuj ryzyko
Wybierz swój zaimportowany certyfikat administratora, podaj hasło systemu jeśli uprawnienia do certyfikatu tego wymagają

  

**Przejdź przez konfigurację:**

 - Do you want to set up a secure configuration?
 - Yes
 - Configure Secure Inpunt
 - Next
 - Next
 - Skip
 - No
 - TAK Home

**Zapraszam na nasz serwer discord proobronni gdzie poruszamy również tematy związane z TAK**

Link: https://discord.gg/Y2E5tkaeBp

Lechu











