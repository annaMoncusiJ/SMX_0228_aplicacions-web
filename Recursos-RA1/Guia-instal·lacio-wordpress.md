# Guia d'instal·lació


> **No copiïs sense entendre.** A la defensa oral et preguntarem per què cada línia hi és.
>
> Cada pas porta un quadre de **teoria** amb el que necessites saber per entendre el que estàs fent. Llegeix-lo **abans** d'executar les ordres: és el que després hauràs d'explicar.

| Pas | Què | Sessió |
| :-: | --- | :-: |
| 3 | Instal·lar la pila (Apache + PHP + MariaDB) | S3 |
| 4 | Base de dades i usuari amb mínim privilegi | S3 |
| 5 | Lloc virtual al port 80 | S4 |
| 6 | Baixar el CMS, permisos i propietat | S5 |
| 7 | `wp-config.php` | S6 |
| 8 | Instal·lador web | S6 |
| 9 | Ajustaments, estructura, usuaris, rols i mòduls | S7–S10 |
| 10 | Pla de proves, seguretat i actualitzacions | S11–S14 |
| 11 | Còpia de seguretat, restauració i fòrums | S15–S16 |
| 12 | Contenidors en un servidor net | S17–S18 |

---

### Com vaig? Marca-ho a mesura que avances

| ✔ | Pas | Sessió | Què ha de quedar fet | Bloc de la memòria que s'omple |
| :-: | :-: | :-: | --- | :-: |
| ☐ | 3 | S3 | `apache2ctl -v`, `php -v` i `mariadb --version` responen; Apache i MariaDB en `active (running)` | **B1** (requeriments) |
| ☐ | 4 | S3 | `SHOW GRANTS` de `wpuser` només mostra `SELECT, INSERT, UPDATE, DELETE` sobre `torreroja.*` | **B1** |
| ☐ | 5 | S4 | `curl -I http://127.0.0.1/` → **200** i les 4 capçaleres de seguretat | **B1** → lliura a S4 |
| ☐ | 6 | S5 | `ls -l /var/www/torreroja` tot de `www-data`; `/wp-config.php` → **403** | **B2** |
| ☐ | 7 | S6 | `wp-config.php` amb prefix `tr_` i sals pròpies (no les d'exemple) | **B2** |
| ☐ | 8 | S6 | L'instal·lador acaba bé i `SELECT COUNT(*)` dona **12 taules** | **B2** → lliura a S6 |
| ☐ | 9 | S7–S10 | Enllaços permanents actius, RSS a `/feed/`, 4 rols creats, mòduls instal·lats | **B3** → lliura a S10 |
| ☐ | 10 | S11–S14 | 8 casos de prova executats i ≥7 mesures de seguretat aplicades | **B4** → lliura a S13 |
| ☐ | 11 | S15–S16 | Còpia feta i **restaurada a `:8081`**; fòrum amb accés per rol | **B5** → lliura a S16 |
| ☐ | 12 | S17–S18 | `docker compose up -d` al servidor net; el lloc sobreviu a `down`/`up` | **B6** → lliura a S18 |

**Si un pas no et quadra, no continuïs.** Ves a [*Si alguna cosa no funciona*](#si-alguna-cosa-no-funciona), al final de la guia, i si no ho resols, pregunta.

---

## Comprovació del punt de partida (2 minuts)

Abans de començar, confirma que les sessions 1 i 2 van quedar bé. Des del **teu ordinador**:

```bash
ssh alumne@127.0.0.1 -p 8022          # has d'entrar sense contrasenya si vas fer ssh-copy-id
```

I dins la MV:

```bash
lsb_release -a                        # Debian GNU/Linux 13 (trixie)
ip a | grep inet                      # la IP de la MV (10.0.2.15 en NAT)
sudo -v                               # el sudo configurat
df -h / ; free -h | head -2           # disc i memòria de la taula de requeriments
```

**Regles NAT que han d'estar posades** (Configuració → Xarxa → Avançat → Redirecció de ports):

| Nom | Protocol | IP amfitrió | Port amfitrió | Port convidat |
| --- | :-: | --- | :-: | :-: |
| ssh | TCP | 127.0.0.1 | 8022 | 22 |
| web | TCP | 127.0.0.1 | **80** | 80 |
| restauracio | TCP | 127.0.0.1 | **8081** | 8081 |
| ssh2 | TCP | 127.0.0.1 | **2223** | 22 |
| docker | TCP | 127.0.0.1 | **8080** | 80 |

Si alguna cosa d'aquí falla, atura't i corregeix.

---

> [!NOTE]
> **Teoria · Què estem muntant**
>
> **Una web no està desada enlloc: es fabrica a cada petició.** Quan algú obre el portal, passen cinc coses:
>
> ```
> navegador  →  port 80  →  Apache  →  PHP  →  MariaDB  →  HTML de tornada
> ```
>
> 1. El navegador fa una petició **HTTP** a una adreça i un **port**.
> 2. **Apache** és el *servidor web*: escolta al port, rep la petició i decideix qui la resol. HTTP és un protocol **sense estat**: cada petició és independent.
> 3. **PHP** és l'*intèrpret*: executa el programa que genera la pàgina. Sense intèrpret, Apache només podria servir fitxers estàtics.
> 4. **MariaDB** és la base de dades: hi viuen els continguts, els usuaris i els ajustaments.
> 5. El resultat torna com a **HTML** i el navegador el pinta.
>
> Això és una **arquitectura client-servidor** i el conjunt Apache + PHP + SGBD sobre Linux és el que es coneix com a **pila LAMP** (o LEMP si el servidor web és Nginx).
>
> **Serveis i `systemctl`.** Apache i MariaDB no són programes que obres i tanques: són **serveis**, processos en segon pla que arrenquen amb la màquina. Qui els gestiona és **`systemd`**, i per això fem servir `systemctl` (`start`, `stop`, `restart`, `reload`, `enable`, `is-active`) i llegim el registre amb `journalctl`. `enable` vol dir «arrenca sol quan engegui la màquina»; sense això, després d'un reinici el portal no hi seria.
>
> **Ports.** Dins d'una mateixa IP, el port identifica **quin** programa rep la connexió. Per convenció: web 80, HTTPS 443, SSH 22, MariaDB/MySQL 3306. Per això dues coses no poden escoltar al mateix port, i per això amb NAT «traduïm» un port de l'ordinador a un port de la màquina virtual.
>
> **Per què `apt` i no instal·lar-ho a mà?** Un paquet de Debian ve amb les **dependències resoltes** i amb **actualitzacions de seguretat** del mantenidor de la distribució. Instal·lar programari compilant-lo a mà dona més control, però deixa les actualitzacions a càrrec teu.
>
> **MariaDB o MySQL?** MariaDB és una bifurcació de MySQL mantinguda per la comunitat i és la que ve a Debian. A efectes pràctics, el SQL i les ordres són les mateixes.

## Pas 3 · Instal·lar la pila (S3)

```bash
sudo apt update
sudo apt install -y apache2 mariadb-server \
  libapache2-mod-php php-mysql php-gd php-mbstring php-xml php-curl php-zip php-intl
sudo systemctl enable --now apache2 mariadb
```

Comprovacions:

```bash
sudo systemctl is-active apache2 mariadb
sudo ss -lntp | grep -E ':80 |:3306 '
sudo apache2ctl -v
php -v | head -1
php -m | grep -Ei 'mysqli|gd|mbstring|xml|curl|zip|intl'
```

Sortida esperada:

```
$ sudo apache2ctl -v
Server version: Apache/2.4.68 (Debian)

$ php -v | head -1
PHP 8.4.24 (cli) (built: Jul 31 2026 05:11:11) (NTS)
```

> **`apache2ctl: command not found`?** Aquestes ordres viuen a `/usr/sbin`, que no és al PATH d'un usuari normal. Sempre amb `sudo`.

**Límits de PHP.** Per defecte `upload_max_filesize` és **2M** i haureu de pujar un PDF de 5 MB:

```bash
sudo nano /etc/php/8.4/apache2/php.ini
#   upload_max_filesize = 16M
#   post_max_size       = 16M
#   memory_limit        = 256M
sudo systemctl reload apache2
php -i | grep -E 'upload_max_filesize|memory_limit'
```

---

> [!NOTE]
> **Teoria · Bases de dades i mínim privilegi**
>
> Una **base de dades relacional** desa la informació en **taules** (files i columnes) que es relacionen entre elles. El **SGBD** (MariaDB) és el programa que les gestiona, i es parla amb **SQL**: `CREATE`, `GRANT`, `SELECT`, `INSERT`, `UPDATE`, `DELETE`.
>
> **El principi de mínim privilegi** és la idea més important d'aquest pas: cada compte ha de poder fer **només el que necessita**.
>
> - Si el CMS es connectés com a `root`, un error de programació o una injecció SQL donaria accés a **tot** el servidor de bases de dades.
> - Amb `wpuser@'localhost'` limitat a `torreroja_db`, el dany possible queda tancat dins d'aquella base de dades i d'aquella màquina.
>
> **Com es llegeix `'wpuser'@'localhost'`:** el que hi ha després de l'`@` és **des d'on** pot connectar-se aquell compte. `'localhost'` vol dir només des de la pròpia màquina; `'%'` voldria dir des de qualsevol lloc (no ho volem).
>
> **`utf8mb4`** és el joc de caràcters: és el que permet desar accents i símbols com cal. Sense això, els continguts en català es poden veure malament.
>
> **Per què `FLUSH PRIVILEGES`?** MariaDB carrega els permisos a memòria quan arrenca; aquesta ordre li diu que els torni a llegir.

## Pas 4 · Base de dades i usuari amb mínim privilegi (S3)

```bash
sudo mariadb <<'SQL'
CREATE DATABASE IF NOT EXISTS torreroja_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER IF NOT EXISTS 'wpuser'@'localhost' IDENTIFIED BY 'Canvia_Aquesta_Contrasenya';
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, ALTER, INDEX, REFERENCES,
      CREATE TEMPORARY TABLES, LOCK TABLES, EXECUTE, CREATE VIEW, SHOW VIEW,
      CREATE ROUTINE, ALTER ROUTINE, TRIGGER
      ON torreroja_db.* TO 'wpuser'@'localhost';
FLUSH PRIVILEGES;
SQL
```

Comprovació (és la captura que va al bloc **B1** de la memòria):

```bash
sudo mariadb -e "SHOW GRANTS FOR 'wpuser'@'localhost'\G"
```

> `@'localhost'` vol dir «només des d'aquesta màquina». **No** facis servir `root` ni `@'%'`.

---

> [!NOTE]
> **Teoria · Com decideix Apache què serveix**
>
> **Un lloc virtual** (*virtual host*) és la configuració d'un lloc web dins del servidor: quin directori arrel té (`DocumentRoot`), quines regles s'hi apliquen i on escriu els logs. Un sol Apache en pot servir molts alhora.
>
> **Com tria?** Pel parell **IP:port** i, si cal, per la capçalera `Host` de la petició. Com que aquí només tenim un lloc al port 80, mana ell. Si `000-default.conf` continua actiu, hi ha **dos** llocs virtuals al port 80 i Apache n'agafa un: per això el desactivem.
>
> **`Listen`** obre un port. Va a `ports.conf` i és **a nivell de servidor**, no d'un lloc virtual. El port 80 ja ve declarat a Debian; el 8081 (restauració) l'hem d'afegir nosaltres.
>
> **`AllowOverride All`** permet que un fitxer `.htaccess` dins del directori del lloc **reescrigui regles** del servidor. És el que necessita el CMS per convertir `/hola-mon/` en una crida interna al seu `index.php`. Sense això, les URL amigables donen 404.
>
> **`mod_rewrite`** és el mòdul que fa aquesta reescriptura; `a2enmod` activa mòduls (a Debian són fitxers a `mods-available` amb un enllaç a `mods-enabled`).
>
> **Què fan les capçaleres de seguretat:**
>
> | Capçalera | Què evita |
> | --- | --- |
> | `X-Content-Type-Options: nosniff` | que el navegador «endevini» el tipus d'un fitxer i executi alguna cosa que no tocava |
> | `X-Frame-Options: SAMEORIGIN` | que algú encasti el teu lloc dins d'un marc invisible i enganyi els usuaris (*clickjacking*) |
> | `Referrer-Policy` | quanta informació de la pàgina d'origen s'envia a tercers |
> | `ServerTokens Prod` | que Apache publiqui la seva versió exacta (informació útil per a un atacant) |
>
> **El bloc `<FilesMatch>`** denega l'accés directe a fitxers com `wp-config.php`: encara que algú en sàpiga l'URL, rebrà un **403**.
>
> **`ServerTokens` i `ServerSignature` no poden anar dins d'un `<VirtualHost>`**: si les hi poses, Apache no arrenca i dona l'error `AH00526`.
>
> **Per què `configtest` abans de `reload`?** Un error de sintaxi en un `reload` pot deixar el servei aturat. `configtest` comprova sense tocar res que estigui en marxa.

## Pas 5 · Lloc virtual al port 80 (S4)

Crea el fitxer de lloc virtual:

```bash
sudo nano /etc/apache2/sites-available/torreroja.conf
```

I enganxa-hi **exactament** això:

```apache
# torreroja.conf — portal de l'Institut Torre Roja
# Aquestes dues directives van FORA del <VirtualHost>:
ServerTokens Prod
ServerSignature Off

<VirtualHost *:80>
    ServerName 127.0.0.1
    ServerAdmin informatica@torreroja.example
    DocumentRoot /var/www/torreroja

    <Directory /var/www/torreroja>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    # Capçaleres de seguretat bàsiques
    Header always set X-Content-Type-Options "nosniff"
    Header always set X-Frame-Options "SAMEORIGIN"
    Header always set Referrer-Policy "strict-origin-when-cross-origin"

    # Denegar l'accés a fitxers sensibles
    <FilesMatch "^(\.env|wp-config\.php|composer\.(json|lock)|\.git)">
        Require all denied
    </FilesMatch>

    ErrorLog ${APACHE_LOG_DIR}/torreroja_error.log
    CustomLog ${APACHE_LOG_DIR}/torreroja_access.log combined
</VirtualHost>

# Entorn de restauració de la còpia (pas 11). S'activa quan calgui.
<VirtualHost *:8081>
    ServerName 127.0.0.1
    DocumentRoot /var/www/torreroja-restore

    <Directory /var/www/torreroja-restore>
        Options -Indexes +FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>

    ErrorLog ${APACHE_LOG_DIR}/torreroja_restore_error.log
    CustomLog ${APACHE_LOG_DIR}/torreroja_restore_access.log combined
</VirtualHost>
```

Activa'l:

```bash
sudo a2ensite torreroja.conf
sudo a2dissite 000-default.conf        # si no, el lloc per defecte es queda el port 80
sudo a2enmod rewrite headers
sudo mkdir -p /var/www/torreroja /var/www/torreroja-restore
sudo apache2ctl configtest             # → Syntax OK
sudo systemctl reload apache2
```

Comprovació **des del teu ordinador** (amb la regla NAT `80→80` posada):

```bash
curl -I http://127.0.0.1
```

Ha de tornar `200`, `Server: Apache` (no la versió completa) i les tres capçaleres de seguretat.

| Si no funciona | On mirar |
| --- | --- |
| `000` des de l'ordinador | regla NAT `80→80` a VirtualBox |
| `000` també dins la MV | `sudo systemctl is-active apache2` |
| Veus la pàgina de Debian | `sudo apache2ctl -S` (hi mana `000-default.conf`?) |
| `AH00112: DocumentRoot does not exist` | `sudo mkdir -p /var/www/torreroja-restore` |

---

> [!NOTE]
> **Teoria · Què és un CMS i per què els permisos importen**
>
> Un **gestor de continguts** (CMS) separa tres coses: el **contingut** (a la base de dades), la **presentació** (el tema) i la **gestió** (el tauler). L'alternativa seria escriure cada pàgina en HTML a mà.
>
> **Per què comprovem la suma de verificació?** Un `sha256sum` és una empremta digital del fitxer. Si la descàrrega s'ha tallat o algú ha substituït el paquet, la suma no coincideix. És la manera barata de saber que el que instal·les és el que creus que instal·les.
>
> **`.tar.gz`** és un arxiu comprimit: `tar` agrupa i `gzip` comprimeix. `tar -tzf` **llista** el contingut sense extreure'l (sempre abans de desempaquetar), i `--strip-components=1` elimina el primer nivell de directoris per no acabar amb `/var/www/torreroja/wordpress/`.
>
> **Usuari i grup.** Cada fitxer de Unix té un **propietari**, un **grup** i tres jocs de permisos. El servidor web s'executa com a **`www-data`**, un usuari **sense privilegis**: si algú aconsegueix executar codi a través del lloc, no podrà canviar el sistema.
>
> **Els permisos en numèric:** `r`=4, `w`=2, `x`=1, i es llegeixen en tres grups (propietari, grup, altres).
>
> | Permisos | Numèric | Per a què |
> | --- | :-: | --- |
> | `rwxr-xr-x` | **755** | directoris: es poden llistar i travessar |
> | `rw-r--r--` | **644** | fitxers: es poden llegir, no escriure des de fora |
> | `rw-------` | **600** | fitxers amb credencials |
>
> Un **403 Forbidden** gairebé mai és un problema del lloc web: és que Apache no pot llegir el fitxer.

## Pas 6 · Baixar el CMS, permisos i propietat (S5)

```bash
cd /tmp
curl -LO https://ca.wordpress.org/latest-ca.tar.gz
sha256sum latest-ca.tar.gz                      # anota-ho al bloc B2
tar -tzf latest-ca.tar.gz | head                # mira abans de desempaquetar
sudo tar -xzf latest-ca.tar.gz -C /var/www/torreroja --strip-components=1
sudo chown -R www-data:www-data /var/www/torreroja
sudo find /var/www/torreroja -type d -exec chmod 755 {} \;
sudo find /var/www/torreroja -type f -exec chmod 644 {} \;
```

Comprovació (valors reals):

```
$ ls -ld /var/www/torreroja
drwxr-xr-x 5 www-data www-data 4096 set 17 10:56 /var/www/torreroja

$ stat -c '%a %U' /var/www/torreroja
755 www-data
```

> **`--strip-components=1`** treu el directori `wordpress/` de dins: sense això et queda `/var/www/torreroja/wordpress/`.

---

## Si alguna cosa no funciona

1. `sudo systemctl is-active apache2 mariadb`
2. `sudo ss -lntp`
3. `curl -v http://127.0.0.1/`
4. `sudo tail -50 /var/log/apache2/error.log` (sense `sudo` no es pot llegir)
5. `sudo apache2ctl configtest` i `sudo apache2ctl -S`
6. `ls -l /var/www/torreroja`
7. `sudo mariadb -e "SELECT 1;"`

| Simptoma | Causa habitual | Solució |
| --- | --- | --- |
| `000` des de l'ordinador | Falta la regla NAT | Revisa la redirecció a VirtualBox |
| Es veu la pàgina de Debian | `000-default.conf` actiu | `sudo a2dissite 000-default.conf` + reload |
| `403 Forbidden` | Propietari o permisos | `sudo chown -R www-data:www-data` + 755/644 |
| URL amigables amb 404 | Sense `mod_rewrite` o `AllowOverride All` | `a2enmod rewrite` + reload |
| «Error establint la connexió amb la base de dades» | Credencials o `DB_HOST` | Revisa `wp-config.php` i `SHOW GRANTS` |
| `php -m` no mostra `mysqli` | Falta `php-mysql` | `sudo apt install php-mysql` |
| No es pot pujar una imatge | `upload_max_filesize` o permisos d'`uploads` | puja el límit i `chown www-data` |
| `AH00526` en arrencar Apache | `ServerTokens` dins del `<VirtualHost>` | Ha d'anar fora del bloc |
