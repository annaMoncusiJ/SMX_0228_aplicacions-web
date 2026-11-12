# Guia de còpia de seguretat i restauració

UD1/RA1 · MP 0228 Aplicacions web · **S15** · bloc **B5**

## Què aprendràs

En acabar, has de poder explicar què cal copiar d’un lloc WordPress, crear-ne una còpia, comprovar que els fitxers no s’han corromput i **restaurar-la en un entorn separat** al port `8081`. La còpia només es considera verificada quan has provat la restauració.

> **Idea clau:** una còpia de seguretat no és només un fitxer guardat. És una còpia que pots recuperar i que has provat de recuperar.

## 1. La teoria que necessites

### Un lloc web té dues parts que han d’anar juntes

| Part | Què hi ha | Com la copiem en aquesta pràctica |
| --- | --- | --- |
| **Fitxers** | WordPress, el tema, les extensions, les pujades i els fitxers de configuració | Un arxiu `tar.gz` del directori del lloc |
| **Base de dades** | Pàgines, entrades, usuaris, rols, comentaris i ajustaments | Un bolcat SQL comprimit amb `mariadb-dump` |

Si només copies la base de dades, et poden faltar imatges i extensions. Si només copies els fitxers, et poden faltar les pàgines, els usuaris i la configuració guardats a MariaDB. **Per tant, necessites totes dues parts.** En aquest treball, `backup.sh` desa el directori complet `/var/www/torreroja` i la base de dades `torreroja_db`.

### Tipus de còpia

- **Completa:** copia totes les dades seleccionades. És fàcil d’entendre i restaurar; és la que fem servir aquí.
- **Incremental:** desa només els canvis des de l’última còpia de qualsevol tipus. Estalvia espai, però restaurar-la requereix la completa i tota la cadena d’increments.
- **Diferencial:** desa els canvis des de l’última còpia completa. Per restaurar-la cal la completa i l’última diferencial.

Per a aquest projecte, una còpia completa és més senzilla i redueix els errors de restauració.

### Integritat, recuperació i seguretat

- **Suma SHA-256:** és una empremta del fitxer. `sha256sum -c` comprova si el fitxer encara coincideix amb l’empremta registrada. Si no coincideix, no el donis per bo.
- **La suma no és una prova de restauració:** una suma correcta no demostra que WordPress arrenqui ni que les dades siguin coherents. Només una restauració provada ho demostra.
- **Consistència:** fes la còpia quan ningú no estigui editant el lloc. Els fitxers i la base de dades s’han de correspondre al mateix estat, tant com sigui possible.
- **Confidencialitat:** la còpia conté dades personals, comptes i credencials de `wp-config.php`. No la publiquis, no l’adjuntis a Git i no comparteixis contrasenyes ni captures que les mostrin.
- **Retenció:** conserva més d’una còpia i elimina les antigues de manera controlada. La política depèn de quant de temps calgui poder retrocedir.
- **Regla 3-2-1 (per a un entorn real):** tres còpies de les dades, en dos tipus de suport, amb una còpia fora del lloc. Les còpies d’aquesta pràctica són dins la mateixa MV; si es perd el disc virtual, també es podrien perdre. Serveixen per demostrar i provar la recuperació, però no són per si soles un pla de desastre complet.

**RPO** vol dir quantes dades com a màxim acceptes perdre (per exemple, les de l’últim dia). **RTO** vol dir quant de temps pots trigar a recuperar el servei. La freqüència de les còpies i el procediment de restauració s’escullen segons aquests dos objectius.

## 2. Crear i comprovar la còpia

Fes-ho quan el lloc estigui en un estat que vulguis poder recuperar. `backup.sh` desa la còpia a `/var/backups/torreroja/` i crea tres fitxers amb el mateix segell de data i hora: bolcat de la base de dades, arxiu dels fitxers i sumes SHA-256.

> Abans d’executar-lo, assegura’t que `backup.sh` és al servidor i que tens la contrasenya de `wpuser`. Si el tens a l’ordinador de l’aula, obre un terminal a la carpeta on hi ha el fitxer i copia’l amb SSH (canvia `alumne` pel teu usuari):
>
> ```bash
> scp -P 8022 backup.sh alumne@127.0.0.1:/home/alumne/backup.sh
> ```
>
> Introdueix la contrasenya de la base de dades quan se’t demani; no l’escriguis en una captura ni al PDF.

```bash
cd ~
read -rsp 'Contrasenya de wpuser: ' DB_PASSWORD; printf '\n'
sudo DB_PASSWORD="$DB_PASSWORD" bash ./backup.sh
unset DB_PASSWORD
```

La data dels fitxers és del tipus `AAAAMMDD_HHMMSS`. Utilitza sempre **els tres fitxers del mateix segell horari**: no barregis el bolcat d’una còpia amb els fitxers d’una altra.

Comprova el resultat des del directori de còpies, fent servir el nom exacte del fitxer de sumes que ha creat l’script:

```bash
cd /var/backups/torreroja
ls -lh
sha256sum -c checksums_AAAAMMDD_HHMMSS.sha256
```

Les dues comprovacions han d’acabar amb `OK`. A més, verifica que l’arxiu es pot llegir:

```bash
tar -tzf fitxers_AAAAMMDD_HHMMSS.tar.gz | head
```

Si les sumes no coincideixen, falta algun fitxer o `tar` dona error, **atura’t**: no restauris una còpia que no has pogut validar.

## 3. Restaurar en un lloc separat

La restauració es fa al directori `/var/www/torreroja-restore`, amb una base de dades nova i el port `8081`. **No sobreescriguis** `/var/www/torreroja` ni la base de dades original: aquesta separació et permet provar la còpia sense arriscar el portal principal.

### Prepara les dades restaurades

1. Tria el segell horari de la còpia verificada i identifica els dos fitxers corresponents.
2. Crea una base de dades buida i un usuari de restauració amb permisos només sobre aquella base. No facis servir l’usuari `root` a `wp-config.php`.
3. Carrega-hi el bolcat SQL comprimit. Per exemple, substitueix el nom del fitxer i de la base de dades pels teus:

```bash
gunzip -c /var/backups/torreroja/db_AAAAMMDD_HHMMSS.sql.gz | sudo mariadb torreroja_restore
```

4. Extreu els fitxers al directori separat. `--strip-components=1` treu el primer directori `torreroja/` de l’arxiu perquè `wp-config.php` quedi directament dins del `DocumentRoot`:

```bash
sudo mkdir -p /var/www/torreroja-restore
sudo tar -xzf /var/backups/torreroja/fitxers_AAAAMMDD_HHMMSS.tar.gz \
  -C /var/www/torreroja-restore --strip-components=1
sudo chown -R www-data:www-data /var/www/torreroja-restore
```

5. Edita `/var/www/torreroja-restore/wp-config.php` perquè apunti **a la base de dades i a l’usuari de restauració**. No canviïs la configuració del lloc original.

### Activa el port de restauració

El fitxer [`torreroja.conf`](torreroja.conf) ja conté un lloc virtual al port `8081`, amb `DocumentRoot /var/www/torreroja-restore`. Debian ha de tenir també `Listen 8081` a `/etc/apache2/ports.conf`, el lloc virtual activat i la regla NAT `8081 → 8081` a VirtualBox.

Comprova la configuració abans de recarregar Apache:

```bash
sudo apache2ctl configtest
sudo systemctl reload apache2
```

Si el port o el lloc virtual encara no estan activats, revisa la guia d’instal·lació i la configuració de referència; no afegeixis repetidament directives `Listen 8081`.

### Ajusta l’adreça només a la còpia

WordPress desa `home` i `siteurl` a la base de dades. Si la còpia restaurada manté l’adreça de producció, alguns enllaços poden tornar al port `80`. A la **base de dades restaurada**, ajusta les dues opcions al port de prova. Aquest projecte fa servir el prefix `tr_`:

```bash
sudo mariadb -e "UPDATE torreroja_restore.tr_options SET option_value='http://127.0.0.1:8081' WHERE option_name IN ('home','siteurl');"
```

Aquesta ordre és per a la base de dades de restauració, no per a `torreroja_db`.

## 4. Verifica que la restauració serveix

Obre al navegador de l’ordinador:

```text
http://127.0.0.1:8081/
```

Comprova, com a mínim:

- la portada respon i es veu el contingut esperat;
- una pàgina i una entrada antigues hi són;
- les imatges pujades es carreguen;
- els usuaris i els rols esperats existeixen;
- els ajustaments principals són correctes;
- la pàgina no et redirigeix al port `80`.

Des del servidor també pots comprovar el codi HTTP:

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8081/
```

S’espera `200`. Un `200` tot sol **no** prova que les dades estiguin completes: comprova també el contingut, els usuaris i les imatges. No entris al mateix temps a editar l’original i la còpia com si fossin un sol lloc: són dues instal·lacions independents.

## 5. Problemes típics

| Símptoma | Què revisar |
| --- | --- |
| `Access denied` en fer el bolcat | Usuari, contrasenya i permisos de `wpuser` sobre `torreroja_db` |
| `sha256sum` diu `FAILED` | Que els tres fitxers siguin del mateix segell; torna a generar la còpia si cal |
| `tar` diu que no pot obrir el fitxer | Nom exacte, ruta i permisos; no facis servir un comodí que pugui triar una còpia diferent |
| Apache no arrenca després d’afegir el 8081 | `sudo apache2ctl configtest`; comprova que `Listen 8081` no estigui duplicat |
| `Connection refused` al 8081 | Apache escoltant al 8081, lloc virtual activat i servidor en marxa |
| Surt el lloc original o torna al port 80 | `DocumentRoot`, base de dades del `wp-config.php` i `home`/`siteurl` de la base restaurada |
| La portada es veu però falten imatges o usuaris | Arxiu de fitxers o bolcat SQL incorrecte/incomplet; comprova que pertanyin al mateix segell horari |

## 6. Què anotes al bloc B5

Segueix la plantilla de l’enunciat, §8.7. Deixa constància de:

1. mètode i data de la còpia;
2. noms dels fitxers generats, mides i sumes SHA-256;
3. entorn de restauració i comprovacions de contingut, usuaris i ajustaments;
4. incidències i com les has resolt;
5. conclusió: per què consideres vàlida —o no— la còpia.

**No cal lliurar un fitxer més:** aquesta informació va al bloc **B5** del PDF únic.
