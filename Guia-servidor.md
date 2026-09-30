# Guia de supervivència al servidor


## 1. La terminal: el mínim per no perdre't

Un servidor no té escriptori: tot es fa escrivint ordres. S'hi entra per SSH des del teu ordinador.

| Què vull fer | Ordre |
| --- | --- |
| On sóc ara? | `pwd` |
| Què hi ha aquí? | `ls -l` (llista amb permisos i propietari) |
| Moure'm | `cd /var/www` · `cd ..` (enrere) · `cd ~` (a casa) |
| Crear / esborrar | `mkdir prova` · `rm fitxer` · `rm -rf directori` (**perillós**: no torna enrere) |
| Copiar / moure | `cp a b` · `mv a b` |
| Veure un fitxer | `cat fitxer` · `less fitxer` (q per sortir) · `head -20 fitxer` |
| Buscar text dins d'un fitxer | `grep "DB_NAME" wp-config.php` |
| Qui sóc? | `whoami` |
| Historial d'ordres | `history \| tail -20` |
| Autocompletar | **Tab** (dos cops si hi ha diverses opcions) |
| Tallar una ordre que va malament | `Ctrl + C` |

### `sudo`: fer les coses com a administrador

Gairebé tot el que tocarem (instal·lar, editar fitxers del sistema, reiniciar serveis) necessita permisos d'administrador:

```bash
sudo apt update
```

- Et demana **la teva contrasenya** i no la mostra mentre l'escrius. És normal.
- `sudo` **no** és una ordre: és «executa el que ve a continuació com a administrador».

### Editar fitxers amb `nano`

```bash
nano fitxer.txt
```

| Tecles | Què fan |
| --- | --- |
| Fletes / ratolí | moure's |
| `Ctrl + O`, `Enter` | desar |
| `Ctrl + X` | sortir |
| `Ctrl + W` | buscar dins del fitxer |
| `Ctrl + K` | tallar la línia |

Si surts sense desar, t'ho pregunta. Si et quedes atrapat en un editor que no coneixes (`vim`), prem `Esc` i escriu `:q!` per sortir sense desar.

---

## 2. Instal·lar programari: `apt`

| Què vull fer | Ordre |
| --- | --- |
| Actualitzar la llista de paquets (**sempre abans**) | `sudo apt update` |
| Instal·lar | `sudo apt install -y apache2` |
| Instal·lar-ne diversos | `sudo apt install -y php-mysql php-gd php-curl` |
| Tenia aquest paquet instal·lat? | `dpkg -l \| grep apache2` |
| Quina versió tinc? | `apt-cache policy apache2` |
| Actualitzar el sistema | `sudo apt update && sudo apt full-upgrade` |
| Treure un paquet | `sudo apt remove apache2` |
| Treure'l i les seves dependències òrfenes | `sudo apt purge apache2 && sudo apt autoremove` |

> `sudo apt update` **no** actualitza el sistema: actualitza la **llista** del que hi ha als repositoris. Sense això, `apt install` falla o instal·la versions velles.

---

## 3. Els serveis: `systemctl`

Un **servei** és un programa que s'executa en segon pla (Apache, MariaDB, SSH).

| Què vull fer | Ordre |
| --- | --- |
| Està actiu? | `sudo systemctl is-active apache2` → `active` o `inactive` |
| Estat detallat (i últim error) | `sudo systemctl status apache2` |
| Engegar / aturar / reiniciar | `sudo systemctl start\|stop\|restart apache2` |
| Recarregar la configuració **sense tallar el servei** | `sudo systemctl reload apache2` |
| Que arrenqui sol quan engeguis la màquina | `sudo systemctl enable --now apache2` |
| Què ha dit al registre? | `sudo journalctl -u apache2 -n 20 --no-pager` |
| Seguir el registre en directe | `sudo journalctl -u apache2 -f` |

Sortida real d'aquestes ordres:

```
$ sudo systemctl is-active apache2
active

$ sudo journalctl -u apache2 -n 2 --no-pager
Sep 23 10:14:20 servidor systemd[1]: Starting apache2.service - The Apache HTTP Server...
Sep 23 10:14:20 servidor systemd[1]: Started apache2.service - The Apache HTTP Server.
```

> **`restart` vs `reload`:** `reload` llegeix la configuració nova sense tallar les connexions. `restart` atura i torna a engegar. Sempre que puguis, `reload`.

---

## 4. Usuaris, propietaris i permisos

Cada fitxer té un **propietari**, un **grup** i uns **permisos**. A Debian, el servidor web treballa amb l'usuari **`www-data`**.

```bash
$ ls -l /var/www/torreroja
drwxr-xr-x 5 www-data www-data 4096 set 17 10:56 wp-content
-rw-r--r-- 1 www-data www-data  405 set 17 10:56 index.php
```

Com es llegeix `-rw-r--r--`:

| Posició | Significat |
| --- | --- |
| 1r caràcter | `d` = directori, `-` = fitxer |
| 2–4 (`rw-`) | què pot fer el **propietari** (llegir, escriure, executar) |
| 5–7 (`r--`) | què pot fer el **grup** |
| 8–10 (`r--`) | què pot fer **tothom** |

En numèric: `r`=4, `w`=2, `x`=1. Per això:

| Permisos | Numèric | Ús habitual |
| --- | :-: | --- |
| `rwxr-xr-x` | **755** | directoris |
| `rw-r--r--` | **644** | fitxers |
| `rw-------` | **600** | fitxers amb contrasenyes |

| Què vull fer | Ordre |
| --- | --- |
| Canviar el propietari | `sudo chown www-data:www-data fitxer` |
| Canviar el propietari de tot un arbre | `sudo chown -R www-data:www-data /var/www/torreroja` |
| Canviar permisos | `sudo chmod 755 directori` · `sudo chmod 644 fitxer` |
| Fer-ho recursivament | `sudo chmod -R 755 directori` |
| Veure el permís en numèric | `stat -c '%a %U' /var/www/torreroja` → `755 www-data` |

> **Error clàssic:** `403 Forbidden` gairebé sempre és un problema de **propietari** o de **permisos**, no del lloc web.

### `root` no és el mateix que `sudo`

- **`root`** és l'usuari administrador (pot fer-ho tot, sense preguntar).
- **`sudo`** et deixa fer una ordre concreta com a root, amb la **teva** contrasenya i deixant **rastre** al registre.
- A Debian, `sudo` no ve instal·lat: es configura a la sessió 1.

---

## 5. La xarxa: `127.0.0.1`, ports i NAT

| Concepte | Què vol dir |
| --- | --- |
| `127.0.0.1` | «aquesta mateixa màquina». També es diu *localhost* |
| **Port** | la «porta» per on escolta un servei. Web = 80, SSH = 22, MariaDB = 3306 |
| **NAT amb redirecció** | VirtualBox tradueix un port del teu ordinador a un port de la màquina virtual |

Comprovacions que faràs tota la unitat:

| Què vull saber | Ordre |
| --- | --- |
| Quins ports escolten a la MV? | `sudo ss -lntp` |
| El lloc respon? (codi d'estat) | `curl.exe -I http://127.0.0.1/` |
| Només el número | `curl.exe -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1/` |
| Ha fallat la connexió o el servidor? | `curl.exe -v http://127.0.0.1/` |
| Puc arribar per SSH? | `ssh alumne@127.0.0.1 -p 2222` |

Codis d'estat que et trobaràs:

| Codi | Què vol dir | Primer lloc on mirar |
| :-: | --- | --- |
| **200** | Tot correcte | — |
| **301 / 302** | Redirecció | enllaços permanents, HTTPS forçat |
| **403** | No tens permís | propietari i permisos dels fitxers |
| **404** | No existeix | `mod_rewrite`, `AllowOverride All`, URL mal escrita |
| **500** | Error del servidor | `sudo tail -50 /var/log/apache2/error.log` |
| **000** | Ni ha respost | el servei està aturat o falta la regla NAT |

> **`ss` sense `sudo` no et mostra el nom del procés.** Sempre amb `sudo`.

---

## 6. Com va Apache a Debian

| Ruta | Què hi ha |
| --- | --- |
| `/etc/apache2/ports.conf` | els ports on escolta (`Listen 80`) |
| `/etc/apache2/sites-available/` | els llocs virtuals **disponibles** |
| `/etc/apache2/sites-enabled/` | els llocs virtuals **actius** (enllaços simbòlics) |
| `/etc/apache2/apache2.conf` | configuració general |
| `/var/www/html` | el lloc per defecte de Debian |
| `/var/log/apache2/error.log` | **on mirar quan alguna cosa falla** |
| `/var/log/apache2/access.log` | qui ha demanat què |

| Què vull fer | Ordre |
| --- | --- |
| Comprovar la configuració abans de recarregar | `sudo apache2ctl configtest` → `Syntax OK` |
| Quin lloc virtual mana a cada port? | `sudo apache2ctl -S` |
| Activar / desactivar un lloc | `sudo a2ensite torreroja.conf` · `sudo a2dissite 000-default.conf` |
| Activar un mòdul | `sudo a2enmod rewrite headers` |
| Aplicar els canvis | `sudo systemctl reload apache2` |
| Versió | `sudo apache2ctl -v` → `Server version: Apache/2.4.68 (Debian)` |

> **Trampa que et farà perdre temps:** aquestes ordres viuen a `/usr/sbin`, que **no** és al PATH d'un usuari normal. Sense `sudo` veuràs `apache2ctl: command not found` encara que estigui instal·lat.

> **Segona trampa:** `/var/log/apache2` no es pot llegir sense `sudo`. Si et diu *Permission denied*, posa `sudo`.

---

## 7. La base de dades: MariaDB

| Què vull fer | Ordre |
| --- | --- |
| Entrar a la consola | `sudo mariadb` |
| Sortir | `exit` |
| Veure les bases de dades | `SHOW DATABASES;` |
| Crear-ne una | `CREATE DATABASE torreroja_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;` |
| Crear un usuari | `CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'UnaContrasenyaForta';` |
| Donar permisos **només** sobre aquella BD | `GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, DROP, ALTER, INDEX ON torreroja_db.* TO 'wpuser'@'localhost';` |
| Aplicar els canvis | `FLUSH PRIVILEGES;` |
| Comprovar què pot fer | `SHOW GRANTS FOR 'wpuser'@'localhost';` |
| Veure les taules | `SHOW TABLES IN torreroja_db;` |
| Comptar registres | `SELECT COUNT(*) FROM torreroja_db.tr_posts;` |
| Executar SQL sense entrar-hi | `sudo mariadb -e "SHOW DATABASES;"` |
| Fer una còpia | `sudo mariadb-dump --databases torreroja_db > copia.sql` |

Sortida real:

```
$ sudo mariadb -e "SELECT VERSION() AS versio;"
versio
11.8.6-MariaDB-0+deb13u1 from Debian
```

I per saber quin lloc virtual està responent a cada port:

```
$ sudo apache2ctl -S
VirtualHost configuration:
*:80     servidor.local (/etc/apache2/sites-enabled/000-default.conf:1)
ServerRoot: "/etc/apache2"
```

Si aquí hi veus `000-default.conf` i esperaves veure el teu lloc, ja saps per què la web no és la teva: `sudo a2dissite 000-default.conf`.

> Les ordres SQL acaben amb **punt i coma**. Si et quedes amb el prompt `->`, és que te n'has deixat un.
> `'wpuser'@'localhost'`: el `@localhost` vol dir «només des d'aquesta màquina». És el **mínim privilegi**.

---

## 8. Quan una cosa no funciona: l'ordre de comprovació

«No em funciona» no es pot diagnosticar. Segueix aquest ordre i digues **què has provat i què ha sortit**:

| # | Comprovació | Ordre |
| :-: | --- | --- |
| 1 | El servei està actiu? | `sudo systemctl is-active apache2` |
| 2 | Escolta al port que toca? | `sudo ss -lntp \| grep ':80 '` |
| 3 | Respon des de dins la MV? | `curl.exe -I http://127.0.0.1/` |
| 4 | Respon des de l'ordinador? | `curl.exe -I http://127.0.0.1/` (amb la regla NAT posada) |
| 5 | Què diu el registre d'errors? | `sudo tail -50 /var/log/apache2/error.log` |
| 6 | La configuració és vàlida? | `sudo apache2ctl configtest` |
| 7 | Quin lloc virtual està responent? | `sudo apache2ctl -S` |
| 8 | Permisos i propietari correctes? | `ls -l /var/www/torreroja` |

I amb Docker:

```bash
docker compose ps            # els contenidors estan en marxa?
docker compose logs -f web   # què diu el contenidor web?
```

> **Llegeix l'error sencer.** El 90 % de les incidències d'aquesta unitat són: permisos, mòduls de PHP que falten, credencials de la base de dades, `mod_rewrite` desactivat o una regla NAT mal posada.

---

## 9. Enllaços per ampliar

| Tema | Enllaç |
| --- | --- |
| Manual de Debian | <https://www.debian.org/doc/manuals/debian-handbook/> |
| `apt` i gestió de paquets | <https://wiki.debian.org/PackageManagement> |
| `systemd` i `systemctl` | <https://www.freedesktop.org/software/systemd/man/latest/systemctl.html> |
| Apache 2.4 — documentació | <https://httpd.apache.org/docs/2.4/> |
| Apache 2.4 — Virtual Hosts | <https://httpd.apache.org/docs/2.4/vhosts/> |
| Permisos i propietat a Unix | <https://wiki.debian.org/Permissions> |
| MariaDB — manual | <https://mariadb.com/kb/en/documentation/> |
| Codis d'estat HTTP | <https://developer.mozilla.org/ca/docs/Web/HTTP/Status> |
| `curl` | <https://curl.se/docs/manpage.html> |
| `nano` | <https://www.nano-editor.org/docs.php> |

---

## 10. El que **no** cal que sàpigues

- Programar en PHP o escriure temes i mòduls des de zero.
- Administrar xarxes, tallafocs o DNS: treballem sempre amb `127.0.0.1` i ports redirigits.
- Memoritzar ordres: la majoria de les que necessites surten en aquesta guia.

- Que la web sigui bonica. Es valora que estigui **ben instal·lada, ben configurada, segura i documentada**.
