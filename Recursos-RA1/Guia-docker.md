# Pràctica S17–S18 · WordPress amb Nginx, PHP-FPM i MariaDB (Docker Compose)

UD1/RA1 · MP 0228 Aplicacions web · **servidor net** · bloc **B6**

> **Objectiu:** desplegar el portal de l'Institut Torre Roja amb tres contenidors independents: Nginx, PHP-FPM/WordPress i MariaDB. Compartir els fitxers del web mitjançant una carpeta `www`, comprovar la persistència, fer una còpia i restaurar-la.

## Què has de tenir abans de començar

- La fase tradicional acabada: WordPress funciona a `http://127.0.0.1` i tens la còpia de seguretat verificada.
- Una segona màquina virtual Debian 13, sense Apache, PHP ni MariaDB instal·lats manualment.
- Accés a VirtualBox i a un terminal de l'ordinador amfitrió.
- La màquina tradicional (MV1) es conserva: **no hi facis proves destructives** durant aquesta pràctica.

La MV2 ha de ser un Debian net. Si fas servir una màquina clonada, clona una instal·lació base anterior a la instal·lació d'Apache, PHP i MariaDB. **No clonïs la MV1 ja configurada.**

## 1. Preparar la segona màquina virtual (S17)

Crea o clona una màquina Debian 13 sense entorn gràfic. Recomanació: 2 GB de RAM, 2 CPU i 20 GB de disc. Instal·la-hi només el servidor SSH i les utilitats estàndard del sistema.

A VirtualBox, amb la MV2 apagada, configura l'adaptador de xarxa com a **NAT** i crea aquestes regles:

| Nom | Protocol | IP amfitrió | Port amfitrió | IP convidat | Port convidat |
|---|---|---|---:|---|---:|
| `ssh-docker` | TCP | `127.0.0.1` | `2223` | *(buit)* | `22` |
| `web-docker` | TCP | `127.0.0.1` | `8080` | *(buit)* | `80` |

La MV1 conserva la regla SSH del projecte (`2222 → 22`) i la web tradicional (`80 → 80`). No canviïs aquestes regles.

Arrenca la MV2 i comprova'n el nom i l'adreça:

```bash
hostname
ip -br a
sudo systemctl is-active ssh
```

Des de l'ordinador amfitrió:

```bash
ssh alumne@127.0.0.1 -p 2223
```

Canvia `alumne` pel teu usuari real.

## 2. Comprovar que el servidor és net

A la MV2, executa:

```bash
command -v apache2 || echo "Apache no instal·lat"
command -v nginx || echo "Nginx no instal·lat"
command -v php || echo "PHP no instal·lat"
command -v mariadb || echo "MariaDB client no instal·lat"
sudo ss -lntp
```

No hi ha d'haver cap servidor web escoltant al port 80. Si has clonat la màquina equivocada, atura't i prepara una MV2 neta: no desinstal·lis paquets a l'atzar.

## 3. Instal·lar Docker Engine i el connector Compose

Docker s'instal·la una sola vegada a la màquina virtual. **No instal·larem Nginx, PHP ni MariaDB amb `apt`**: aquests serveis s'executaran dins dels contenidors.

```bash
sudo apt update
sudo apt install -y ca-certificates curl
```

Afegeix la clau oficial del repositori de Docker:

```bash
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg \
  -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
```

Afegeix el repositori oficial per a la versió de Debian detectada:

```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Instal·la el motor, el client i el connector Compose:

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

Activa el servei i comprova'n l'estat:

```bash
sudo systemctl enable --now docker
sudo systemctl is-active docker
sudo docker version
sudo docker compose version
sudo docker run --rm hello-world
```

La darrera ordre descarrega una imatge de prova, crea un contenidor, l'executa i l'elimina.

### Permisos per executar Docker

Per no haver d'escriure `sudo` davant de cada ordre, pots afegir el teu usuari al grup `docker`:

```bash
sudo usermod -aG docker "$USER"
```

Tanca la sessió SSH i torna-hi a entrar. Comprova-ho amb `docker run --rm hello-world`. Pertànyer al grup `docker` equival, a efectes pràctics, a tenir privilegis molt elevats sobre la màquina. Si no vols afegir-t'hi, executa les ordres amb `sudo`.

## 4. Preparar la carpeta del projecte

Crea el directori de treball i la carpeta `www`:

```bash
mkdir -p ~/torreroja-docker/www
cd ~/torreroja-docker
```

L'estructura serà aquesta:

```text
torreroja-docker/
├── docker-compose.yml
├── default.conf
├── .env
└── www/
    ├── index.php
    ├── wp-admin/
    ├── wp-content/
    ├── wp-includes/
    └── ...
```

**La carpeta `www` és al mateix nivell que `docker-compose.yml`.** No cal crear-la dins del contenidor. La muntarem als dos contenidors web:

- Nginx: `./www:/var/www/html:ro` (només lectura).
- PHP-FPM: `./www:/var/www/html` (lectura i escriptura).

Això és un **bind mount**: els fitxers són a la màquina Debian, dins de `~/torreroja-docker/www`, i els dos contenidors veuen la mateixa carpeta.

> **Important:** deixa `www` buida abans del primer arrencament. La imatge oficial de WordPress copiarà els fitxers inicials a la carpeta muntada durant la inicialització. Si ja hi has copiat WordPress manualment, no cal que la buidis: comprova que hi ha `index.php`, `wp-admin`, `wp-content` i `wp-includes`.

## 5. Crear el fitxer de variables `.env`

Crea'l:

```bash
nano .env
```

Escriu valors propis. Les contrasenyes són d'exemple: no reutilitzis les de la MV1 ni contrasenyes personals.

```dotenv
MARIADB_DATABASE=torreroja_db
MARIADB_USER=wpuser
MARIADB_PASSWORD=Canvia_Aquesta_Clau_1
MARIADB_ROOT_PASSWORD=Canvia_Aquesta_Clau_Root_2
```

Desa'l i limita'n els permisos:

```bash
chmod 600 .env
```

El fitxer `.env` conté credencials. **No el lliuris, no el pugis a Git i no en facis captures on es puguin llegir les contrasenyes.**

## 6. Crear `docker-compose.yml`

Crea el fitxer:

```bash
nano docker-compose.yml
```

Copia-hi aquesta configuració i respecta la indentació:

```yaml
services:
  nginx:
    image: nginx:1.28-alpine
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./www:/var/www/html:ro
      - ./default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - php
    networks:
      - torreroja_net

  php:
    image: wordpress:php8.3-fpm
    restart: unless-stopped
    environment:
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_USER: ${MARIADB_USER}
      WORDPRESS_DB_PASSWORD: ${MARIADB_PASSWORD}
      WORDPRESS_DB_NAME: ${MARIADB_DATABASE}
      WORDPRESS_CONFIG_EXTRA: |
        define('WP_HOME', 'http://127.0.0.1:8080');
        define('WP_SITEURL', 'http://127.0.0.1:8080');
    volumes:
      - ./www:/var/www/html
    depends_on:
      db:
        condition: service_healthy
    networks:
      - torreroja_net

  db:
    image: mariadb:11.4
    restart: unless-stopped
    environment:
      MARIADB_DATABASE: ${MARIADB_DATABASE}
      MARIADB_USER: ${MARIADB_USER}
      MARIADB_PASSWORD: ${MARIADB_PASSWORD}
      MARIADB_ROOT_PASSWORD: ${MARIADB_ROOT_PASSWORD}
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - torreroja_net
    healthcheck:
      test: ["CMD", "healthcheck.sh", "--connect", "--innodb_initialized"]
      interval: 10s
      timeout: 5s
      retries: 10

volumes:
  db_data:

networks:
  torreroja_net:
```

### Què has d'identificar al fitxer?

- `services`: defineix els tres serveis que formen l'aplicació.
- `image`: indica quina imatge es fa servir per crear cada contenidor.
- `ports`: publica el port 80 de Nginx al port 80 de Debian. VirtualBox el redirigeix al port 8080 de l'ordinador.
- `volumes`: comparteix la carpeta `www` amb Nginx i PHP-FPM, i desa la base de dades en un volum Docker anomenat `db_data`.
- `environment`: passa a WordPress i MariaDB les dades de connexió que es llegeixen del fitxer `.env`.
- `depends_on`: estableix l'ordre d'arrencada. PHP espera que MariaDB estigui preparada; Nginx espera que s'hagi iniciat PHP-FPM.
- `networks`: connecta els tres serveis a una xarxa privada. Compose també podria crear una xarxa per defecte, però aquí la declarem explícitament.

**No publiquem el port 9000 de PHP-FPM ni el 3306 de MariaDB a l'ordinador.** Només Nginx rep connexions des de fora. Els altres serveis es comuniquen per la xarxa interna de Docker.

## 7. Crear la configuració de Nginx: `default.conf`

Crea el fitxer:

```bash
nano default.conf
```

Copia-hi:

```nginx
server {
    listen 80;
    server_name localhost;

    root /var/www/html;
    index index.php index.html;
    client_max_body_size 64M;

    location / {
        try_files $uri $uri/ /index.php?$args;
    }

    location ~ \.php$ {
        try_files $uri =404;
        include fastcgi_params;
        fastcgi_pass php:9000;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }

    location ~ /\. {
        deny all;
    }
}
```

### Com funciona aquesta configuració?

- `listen 80`: Nginx escolta al port 80 dins del contenidor.
- `root`: indica on són els fitxers del web dins del contenidor.
- `try_files`: busca el fitxer sol·licitat i, si cal, envia la petició a WordPress.
- `location ~ \.php$`: identifica les peticions de fitxers PHP.
- `fastcgi_pass php:9000`: envia el processament PHP al servei `php`, al port intern 9000.
- `SCRIPT_FILENAME`: indica a PHP-FPM quin fitxer ha d'executar.
- `deny all`: impedeix accedir a fitxers ocults, com els que comencen per punt.

**Nginx no executa PHP.** Nginx rep la petició HTTP i, quan és una petició PHP, la passa a PHP-FPM mitjançant FastCGI.

## 8. Validar i iniciar els contenidors

Comprova que els fitxers són al directori correcte:

```bash
ls -la
```

Hi has de veure `docker-compose.yml`, `default.conf`, `.env` i la carpeta `www`.

Valida la configuració:

```bash
docker compose config
```

Si no mostra errors, descarrega les imatges i inicia els serveis:

```bash
docker compose up -d
```

Consulta'n l'estat:

```bash
docker compose ps
docker compose logs --tail=50
```

La primera inicialització pot trigar una mica. Comprova que WordPress ha preparat els fitxers a `www`:

```bash
ls -la www
```

Hi has de trobar `index.php`, `wp-admin`, `wp-content` i `wp-includes`. Si encara no hi són, consulta els registres de PHP-FPM:

```bash
docker compose logs --tail=100 php
```

Comprova que Nginx pot resoldre el nom del servei PHP-FPM:

```bash
docker compose exec nginx getent hosts php
```

I que PHP-FPM pot resoldre MariaDB:

```bash
docker compose exec php getent hosts db
```

Obre al navegador de l'ordinador:

`http://127.0.0.1:8080`

Completa l'instal·lador de WordPress. Utilitza el títol **Portal de l'Institut Torre Roja** i no reutilitzis les contrasenyes de la instal·lació tradicional.

Comprova també des de Debian:

```bash
curl -I http://127.0.0.1
```

Des de l'ordinador amfitrió, el port és 8080:

```powershell
curl.exe -I http://127.0.0.1:8080
```

## 9. Entendre els contenidors, la xarxa i els volums

Executa:

```bash
docker compose ps
docker image ls
docker container ls
docker volume ls
docker network ls
```

Completa aquesta taula a la memòria:

| Element | En aquest projecte |
|---|---|
| Imatge | Plantilla utilitzada per crear cada contenidor |
| Contenidor | Instància en execució de Nginx, PHP-FPM o MariaDB |
| Bind mount | `./www`, compartida entre Nginx i PHP-FPM |
| Volum Docker | `db_data`, on MariaDB desa les dades |
| Xarxa | `torreroja_net`, que connecta els tres serveis |
| Registre | Servei des d'on es descarreguen imatges, com Docker Hub |

### Per què hi ha dos tipus de muntatge?

`./www:/var/www/html` és un **bind mount**: relaciona una carpeta concreta de Debian amb una ruta del contenidor. Els canvis que PHP-FPM faci als fitxers es veuen a la carpeta `www` de Debian. Nginx hi accedeix en mode només lectura (`:ro`).

`db_data:/var/lib/mysql` és un **volum Docker amb nom**: Docker gestiona aquest espai i MariaDB hi desa les dades. No depèn del cicle de vida del contenidor.

### Com es comuniquen els serveis?

- El navegador de l'ordinador arriba a VirtualBox pel port 8080.
- VirtualBox envia la connexió al port 80 de Debian.
- Docker publica aquest port al 80 del contenidor Nginx.
- Nginx envia les peticions PHP a `php:9000`.
- WordPress/PHP-FPM es connecta a MariaDB amb `db:3306`.

Els noms `php` i `db` són els noms dels serveis a Compose. Docker els resol dins de la xarxa. No cal configurar IPs fixes.

## 10. Prova de persistència obligatòria

Entra al portal i crea una entrada amb un títol fàcil de reconèixer, per exemple **Prova de persistència Docker**. Publica-la i comprova que es veu a la portada.

A Debian, atura i elimina els contenidors:

```bash
docker compose down
```

Aquesta ordre elimina els contenidors i la xarxa del projecte, però conserva:

- La carpeta `www`, perquè és una carpeta de Debian muntada als contenidors.
- El volum `db_data`, perquè `down` no elimina els volums amb nom.

Torna a crear els contenidors:

```bash
docker compose up -d
docker compose ps
```

Actualitza el navegador i comprova que l'entrada continua publicada.

> **Evidència B6:** captura de l'entrada abans i després de `down` i `up -d`. A la memòria explica per què s'han conservat tant els fitxers com la base de dades.

## 11. Còpia de seguretat

La còpia ha d'incloure els fitxers de WordPress i la base de dades. Crea un directori de còpies fora del projecte:

```bash
mkdir -p ~/torreroja-backups
```

### 11.1. Còpia dels fitxers

Com que `www` és una carpeta de Debian, pots copiar-la directament des de l'amfitrió:

```bash
tar -czf ~/torreroja-backups/wordpress-files.tar.gz -C ~/torreroja-docker/www .
```

Comprova que l'arxiu existeix i que conté fitxers:

```bash
ls -lh ~/torreroja-backups/wordpress-files.tar.gz
tar -tzf ~/torreroja-backups/wordpress-files.tar.gz | head
```

### 11.2. Còpia de la base de dades

El bolcat s'executa dins del contenidor MariaDB i es desa a Debian:

```bash
docker compose exec -T db sh -c \
  'mariadb-dump -uroot -p"$MARIADB_ROOT_PASSWORD" --add-drop-database --databases "$MARIADB_DATABASE"' \
  | gzip > ~/torreroja-backups/database.sql.gz
```

Comprova'l:

```bash
ls -lh ~/torreroja-backups/database.sql.gz
gzip -t ~/torreroja-backups/database.sql.gz
```

### 11.3. Verificar la integritat

```bash
cd ~/torreroja-backups
sha256sum database.sql.gz wordpress-files.tar.gz > checksums.sha256
sha256sum -c checksums.sha256
```

Les dues comprovacions han d'acabar amb `OK`. La suma comprova la integritat dels fitxers, però la prova definitiva és restaurar-los.

## 12. Restaurar la còpia

**Atenció:** la restauració substitueix els fitxers i les dades actuals. Fes-la només després d'haver creat i verificat la còpia.

Ves al directori del projecte i atura Nginx i PHP-FPM:

```bash
cd ~/torreroja-docker
docker compose stop nginx php
```

### 12.1. Restaurar els fitxers

Elimina el contingut actual de `www` i extreu-hi la còpia:

```bash
find ~/torreroja-docker/www -mindepth 1 -maxdepth 1 -exec rm -rf -- {} +
tar -xzf ~/torreroja-backups/wordpress-files.tar.gz -C ~/torreroja-docker/www
```

Aquesta operació només afecta els fitxers de la carpeta `www`, no la base de dades.

### 12.2. Restaurar la base de dades

El bolcat s'ha generat amb `--add-drop-database`, per tant inclou les ordres necessàries per substituir la base de dades existent. Envia'l a MariaDB com a usuari root:

```bash
gunzip -c ~/torreroja-backups/database.sql.gz \
  | docker compose exec -T db sh -c \
    'mariadb -uroot -p"$MARIADB_ROOT_PASSWORD"'
```

Torna a iniciar els serveis web:

```bash
docker compose start php nginx
docker compose ps
docker compose logs --tail=50
```

Obre `http://127.0.0.1:8080` i comprova que el portal mostra el contingut restaurat. Verifica també que pots iniciar sessió amb l'usuari administrador.

## 13. Prova destructiva: `down -v`

Compara aquestes ordres:

```bash
docker compose down
```

Elimina contenidors i xarxa, però conserva `www` i `db_data`.

```bash
docker compose down -v
```

Elimina també els volums Docker amb nom del projecte, inclòs `db_data`. **No elimina la carpeta `www`**, perquè és una carpeta de Debian muntada amb un bind mount.

Fes aquesta prova només després d'haver verificat la còpia i la restauració. Executa:

```bash
docker compose down -v
docker volume ls
docker compose up -d
```

Ara els fitxers de WordPress continuen a `www`, però la base de dades s'ha inicialitzat de nou i ja no conté els continguts, usuaris ni ajustaments anteriors. WordPress mostrarà l'instal·lador o un error de configuració fins que es torni a crear/restaurar la base de dades.

A la memòria explica per què `down -v` elimina la base de dades però no els fitxers de `www`.

## 14. Ordres bàsiques que has de saber utilitzar

| Ordre | Funció |
|---|---|
| `docker compose config` | Valida i mostra la configuració resultant |
| `docker compose up -d` | Crea i inicia els serveis en segon pla |
| `docker compose down` | Elimina contenidors i xarxa, però conserva volums |
| `docker compose ps` | Mostra l'estat dels serveis |
| `docker compose logs -f nginx` | Segueix els registres de Nginx |
| `docker compose logs -f php` | Segueix els registres de PHP-FPM |
| `docker compose exec php bash` | Obre una shell dins del contenidor PHP-FPM |
| `docker compose restart` | Reinicia els serveis |
| `docker volume ls` | Mostra els volums Docker |
| `docker network ls` | Mostra les xarxes Docker |

Quan acabis, deixa els contenidors aturats amb `docker compose down`. No utilitzis `-v` llevat que estiguis fent expressament la prova destructiva.

## 15. Resolució de problemes

| Símptoma | Comprovació / solució |
|---|---|
| `Permission denied` en executar Docker | Torna a entrar per SSH després d'afegir l'usuari al grup `docker`, o utilitza `sudo` |
| No es poden descarregar les imatges | Comprova que la MV2 té Internet i revisa els registres |
| El port 80 està ocupat | `sudo ss -lntp`; la MV2 ha de ser neta i no tenir cap servidor web escoltant |
| Nginx mostra `502 Bad Gateway` | Comprova `docker compose ps`, `docker compose logs php` i que `fastcgi_pass` sigui `php:9000` |
| Nginx mostra `403` o `404` | Comprova que `www` conté els fitxers de WordPress i que Nginx munta `./www` a `/var/www/html` |
| WordPress no connecta amb MariaDB | Revisa les variables del `.env`, `WORDPRESS_DB_HOST: db:3306` i els registres de `db` |
| PHP-FPM no veu els fitxers | Comprova el muntatge `./www:/var/www/html` del servei `php` |
| Nginx no veu els fitxers | Comprova el muntatge `./www:/var/www/html:ro` del servei `nginx` |
| La web no s'obre des de l'ordinador | Comprova el port `80:80` i la regla NAT `8080 → 80` |
| `docker compose config` mostra variables buides | Comprova que `.env` és al mateix directori que `docker-compose.yml` |
| Després de `down -v` falten les dades | És esperat: el volum `db_data` s'ha eliminat; la carpeta `www` continua existint |

## 16. Lliurament B6 i defensa oral (S18)

Al bloc **B6** de la memòria tècnica incorpora:

1. El fitxer `docker-compose.yml` comentat i adjunt al lliurament.
2. El fitxer `default.conf` de Nginx.
3. Una captura de `docker compose ps` amb els tres serveis en funcionament.
4. Una captura de l'estructura del projecte on es vegi `docker-compose.yml`, `default.conf` i `www`.
5. Les captures de la prova de persistència: abans i després de `docker compose down` i `up -d`.
6. Una evidència de la còpia: els dos fitxers generats i la verificació SHA-256.
7. Una evidència de la restauració: el portal torna a mostrar el contingut i permet iniciar sessió.
8. La comparativa entre instal·lació tradicional i contenidors de l'apartat 8.8 de l'enunciat.
9. Una conclusió pròpia de tres línies.

No incloguis el fitxer `.env`, contrasenyes ni captures que les mostrin. No cal lliurar la carpeta `www` completa si el portal ja està documentat a la memòria; el que s'avalua és la configuració i les evidències.

### Preguntes de defensa oral

Has de poder explicar, mirant els teus fitxers:

- Quina diferència hi ha entre una imatge i un contenidor?
- Per què hem separat Nginx, PHP-FPM i MariaDB en tres contenidors?
- Quina diferència hi ha entre Nginx i PHP-FPM? Quin dels dos executa PHP?
- Què fa `fastcgi_pass php:9000`?
- Per què Nginx i PHP-FPM comparteixen `www`?
- Quina diferència hi ha entre un bind mount i un volum Docker amb nom?
- Per què Nginx munta `www` amb `:ro`?
- Com arriba una petició des del navegador fins a PHP-FPM?
- Per què WordPress es connecta a `db:3306` i no a `localhost`?
- Per què no publiquem els ports 9000 i 3306?
- Quina diferència hi ha entre `docker compose down` i `docker compose down -v` en aquest projecte?
- Per què cal copiar tant els fitxers com la base de dades?
- Quina diferència pràctica has observat entre la instal·lació tradicional i la de contenidors?
