# UD1 · Projecte «Portal web de l'Institut Torre Roja»

| | |
| --- | --- |
| **Resultat d'aprenentatge** | RA1. Instal·la gestors de continguts, identificant-ne les aplicacions i configurant-los segons requisits |
| **Hores** | **20 hores** = 18 hores de classe + 2 hores de prova final. |
| **Treball** | Individual. Cadascú amb la seva màquina virtual |
| **Entorn** | Debian 13 (Trixie) a VirtualBox, xarxa NAT amb redirecció de ports. **Tot es fa per `127.0.0.1`** |
| **Lliurament final** | Sessió 18 |
| **Què es lliura** | **Un sol fitxer**: la memòria tècnica en PDF (6 blocs, 6–8 pàgines). Més el `compose.yml` |

**Ports que fareu servir durant tota la unitat**

Treballem amb els **ports per defecte** del servei. 

| `127.0.0.1` (el vostre ordinador) | Màquina | Per a què |
| --- | --- | --- |
| 2222 | 22 | SSH del **servidor del projecte** (MV1) |
| 80 | 80 | La web del projecte → `http://127.0.0.1` |
| 8081 | 8081 | La restauració de la còpia |
| 2223 | 22 | SSH del **servidor net** (MV2, fase Docker) |
| 8080 | 80 | El desplegament amb Docker → `http://127.0.0.1:8080` |

El port 80 de l'ordinador ja el fa servir el servidor tradicional, per això el lloc en contenidors surt pel **8080**.

---

## Índex

1. [La situació](#1-la-situació)
2. [Què haureu fet al final de les 18 hores](#2-què-haureu-fet-al-final-de-les-18-hores)
3. [Les 20 sessions](#3-les-20-sessions)
4. [Què ha de tenir el projecte](#4-què-ha-de-tenir-el-projecte)
   - 4.1 [La infraestructura (fase tradicional)](#41-la-infraestructura-fase-tradicional)
   - 4.2 [El gestor de continguts](#42-el-gestor-de-continguts)
   - 4.3 [Abast mínim del lloc web](#43-abast-mínim-del-lloc-web)
   - 4.4 [Usuaris i rols](#44-usuaris-i-rols)
   - 4.5 [Fòrums](#45-fòrums)
   - 4.6 [Abast ampliat (opcional)](#46-abast-ampliat-opcional)
5. [Seguretat, actualitzacions, proves i còpies](#5-seguretat-actualitzacions-proves-i-còpies)
6. [La part amb Docker](#6-la-part-amb-docker)
7. [Com es qualifica](#7-com-es-qualifica)
   - 7.1 [Els 5 aspectes (100 % de la UD1)](#71-els-5-aspectes-100--de-la-ud1)
   - 7.2 [La prova final (sessions 19 i 20)](#72-la-prova-final-sessions-19-i-20)
   - 7.3 [Què no es valora](#73-què-no-es-valora)
8. [L'únic lliurable: la memòria tècnica](#8-lúnic-lliurable-la-memòria-tècnica)
   - 8.1 [Els 6 blocs](#81-els-6-blocs)
   - 8.2 [Taula de requisits (bloc B1)](#82-taula-de-requisits-bloc-b1)
   - 8.3 [Matriu de rols i capacitats (bloc B3)](#83-matriu-de-rols-i-capacitats-bloc-b3)
   - 8.4 [Llista de verificació de seguretat (bloc B4)](#84-llista-de-verificació-de-seguretat-bloc-b4)
   - 8.5 [Registre d'actualitzacions (bloc B4)](#85-registre-dactualitzacions-bloc-b4)
   - 8.6 [Pla de proves de funcionament (bloc B5)](#86-pla-de-proves-de-funcionament-bloc-b5)
   - 8.7 [Registre de còpia i verificació (bloc B5)](#87-registre-de-còpia-i-verificació-bloc-b5)
   - 8.8 [Comparativa dels dos models (bloc B6)](#88-comparativa-dels-dos-models-bloc-b6)
   - 8.9 [Comprovació final abans de lliurar](#89-comprovació-final-abans-de-lliurar)
9. [Si alguna cosa no us surt](#9-si-alguna-cosa-no-us-surt)
10. [Consells pràctics](#10-consells-pràctics)
    - 10.1 [Errors freqüents](#101-errors-freqüents)

---

## 1. La situació

L'Institut Torre Roja necessita un **portal web** que faci dues coses alhora:

1. **Informar** les famílies i l'alumnat: qui som, quina oferta educativa tenim, documents, secretaria, contacte.
2. **Servir d'eina de treball intern**: notícies, documents restringits per nivells, una borsa de treball i fòrums amb accessos diferents per a cada col·lectiu.

El departament d'informàtica us encarrega la feina completa: **muntar el servidor, instal·lar-hi el gestor de continguts, administrar-lo, assegurar-lo, fer-ne còpies de seguretat i preparar un segon desplegament amb contenidors.** Haureu de documentar-ho tot i defensar les vostres decisions tècniques.

>[!WARNING]
> **Atenció a l'enfocament.** Això **no és un projecte de disseny web**. No es valora que la web sigui bonica. Es valora que **el servei estigui ben instal·lat, ben configurat, ben administrat, segur, amb còpies verificades i desplegable de dues maneres**, a més, l'heu de poder **explicar i justificar** amb documentació tècnica.

---

## 2. Què haureu fet al final de les 18 hores

1. Un servidor **Debian** instal·lat per vosaltres, accessible per SSH.
2. **Apache (o Nginx) + PHP + MariaDB** instal·lats i configurats **a mà**, amb el lloc al **port 80** (el port per defecte).
3. Un **WordPress** instal·lat sobre aquesta pila, amb l'estructura del portal, menús, usuaris amb rols diferents i els mòduls necessaris.
4. El gestor **endurit** (seguretat) i **actualitzat** seguint un procediment responsable.
5. Un **pla de proves** executat amb resultats, i una **còpia de seguretat restaurada i verificada** al port 8081.
6. **Fòrums** amb regles d'accés per col·lectius.
7. El **mateix servei desplegat amb Docker Compose** al port 8080, amb xarxa, volums i variables d'entorn.
8. Una **comparativa tècnica** entre els dos models i **una sola memòria tècnica** amb totes les evidències.

---

## 3. Les 20 sessions

Les sessions són d'**una hora** amb aquesta estructura: 10 min d'esquema i objectiu, 10 min de demostració, 35 min de treball pràctic (un únic punt de control per sessió) i 5 min de tancament. La prova final està repartida en les dues últimes sessions.

| Sessió | Què farem | Què va a la memòria | Lliurement |
| :-: | --- | --- | :-: |
| **S1** | **Instal·lem el servidor**: Debian a VirtualBox, NAT amb redirecció de ports, SSH, `sudo`, actualització | captures per al bloc **B1** | — |
| **S2** | Arquitectura d'una aplicació web i requisits del CMS | bloc **B1** | — |
| **S3** | Pila LAMP: Apache, PHP, MariaDB, base de dades i usuari | bloc **B1** | — |
| **S4** | Lloc virtual al **port 80** amb `mod_rewrite` i capçaleres | bloc **B1** | **B1** |
| **S5** | Instal·lem el CMS a mà: descàrrega, permisos i propietat | bloc **B2** | — |
| **S6** | `wp-config.php` i estructura de directoris | bloc **B2** | **B2** |
| **S7** | Ajustaments, enllaços permanents, tema i personalització | bloc **B3** | — |
| **S8** | Pàgines, menús, continguts i canal RSS | bloc **B3** | — |
| **S9** | Usuaris i rols amb matriu de capacitats | bloc **B3** | — |
| **S10** | Mòduls: contacte, galetes i multidioma | bloc **B3** | **B3** |
| **S11** | Disseny del pla de proves (8 casos) | bloc **B5** (disseny) | — |
| **S12** | Seguretat del gestor (10 mesures) | bloc **B4** | — |
| **S13** | Cicle d'actualització amb còpia prèvia i registre | bloc **B4** | **B4** |
| **S14** | Execució del pla de proves | bloc **B5** | — |
| **S15** | Còpia de seguretat i restauració al port 8081 | bloc **B5** | — |
| **S16** | Verificació de la restauració i fòrums amb regles d'accés | bloc **B5** | **B5** |
| **S17** | **Servidor net** + motor de contenidors; imatges, xarxes, volums i persistència | bloc **B6** | — |
| **S18** | Docker Compose, comparativa, tancament i defensa oral | bloc **B6** | **B6 + tota la memòria** |
| **S19** | **Prova final, part 1** — M1 (entorn, serveis, BD, CMS) i M2 (usuaris, seguretat, còpia) + 3 preguntes orals → **5 punts** | res (es corregeix a la pantalla) | — |
| **S20** | **Prova final, part 2** — M3 (proves, restauració verificada, Compose, persistència, actualització) + 3 preguntes orals → **5 punts** | res (es corregeix a la pantalla) | — |

Cada sessió té un **mínim obligatori**. Qui acaba abans fa l'abast ampliat (secció 4.6). Qui necessita més temps arriba al mínim: **és el mínim el que s'avalua**.

**Només hi ha 6 lliuraments** (B1 a S4, B2 a S6, B3 a S10, B4 a S13, B5 a S16 i la memòria completa a S18). Cada bloc és **una pàgina** de la memòria: el que lliureu abans és la mateixa pàgina que després formarà part del PDF final. No hi ha carpetes, ni fitxers separats, ni índexs de captures.

---

## 4. Què ha de tenir el projecte

### 4.1 La infraestructura (fase tradicional)

- **Debian 13 (Trixie)** a la vostra màquina virtual, sense entorn d'escriptori, amb **NAT i redirecció de ports**.
- **Apache 2.4** (o Nginx) amb un lloc al **port 80** (per defecte), accessible des del vostre ordinador a `http://127.0.0.1`.
- **PHP** amb les extensions que demana el CMS (`mysqli`, `gd`, `mbstring`, `xml`, `curl`, `zip`, `intl`). Comproveu els requisits vigents a la documentació oficial abans d'instal·lar: és el que es demana al bloc **B1**.
- **MariaDB** amb una base de dades `torreroja_db` i un usuari propi (`wpuser@localhost`) amb permisos **només** sobre aquella base de dades.
- WordPress a `/var/www/torreroja`, amb propietat i permisos correctes (`www-data`, `755`/`644`).
- Serveis habilitats a l'arrencada.

**Instal·lació sense Docker.** Aquesta fase es fa tota a mà: és la part de sistemes de la unitat i la que us permet entendre què fa després el Docker.

### 4.2 El gestor de continguts

- **WordPress** (opció comuna del grup).
- Instal·lació amb **prefix de taules propi** (no `wp_`).
- Enllaços permanents actius (URL amigables).

### 4.3 Abast mínim del lloc web

**És el mínim. No cal fer una web enorme.**

| Element | Mínim |
| --- | --- |
| Apartats amb menú visible (7) | El centre · Qui som · Oferta educativa · Documents · Secretaria · Contacte · Borsa de treball |
| Pàgines | 8 |
| Entrades (notícies) | 4 |
| Formulari de contacte | 1, funcional |
| Avís i gestió de galetes | Actiu |
| Idiomes | Català i castellà en **una secció** del lloc |
| Usuaris | 6 amb rols diferents |
| Fòrums | 2 amb regles d'accés diferents |
| Contingut restringit | 1 pàgina només per al professorat + 1 per a un nivell d'alumnat |
| Canal RSS | Actiu i validat |

### 4.4 Usuaris i rols

| Rol | Qui el faria servir | Què pot fer |
| --- | --- | --- |
| Administrador | Departament d'informàtica | Tot |
| Editor | Equip directiu | Publicar i editar qualsevol contingut, moderar |
| Professor | Claustre | Crear i publicar el seu contingut, moderar el seu fòrum |
| Alumne ESO / BATX / FP | Alumnat | Esborranys i comentaris, llegir el seu fòrum i el seu contingut restringit |
| Convidat | Públic | Llegir el contingut públic i el fòrum obert |

Com a mínim heu de crear **un rol personalitzat** (per exemple *Alumne FP*). La matriu que heu d'omplir és al bloc **B3** (secció 8.3).

### 4.5 Fòrums

| Fòrum | Qui el llegeix | Qui hi escriu | Qui el modera |
| --- | --- | --- | --- |
| Fòrum general del centre | Tothom, inclòs convidat | Usuaris registrats | Editor i administrador |
| Fòrum de Formació Professional | Alumne FP, professorat, administrador | Alumne FP i professorat | Professorat |

Heu de poder **demostrar amb una captura** que un alumne d'ESO no veu el fòrum de FP.

### 4.6 Abast ampliat (opcional)

Altres apartats (Consell escolar, Biblioteca, ESO, Batxillerat, FP, Escola/empresa, qui és qui), entorn de proves separat, `Dockerfile` propi per afegir una extensió de PHP, tercer fòrum, cerca avançada. **No dona nota extra.**

---

## 5. Seguretat, actualitzacions, proves i còpies

Aquests quatre blocs tenen sessions pròpies (S11–S16) i són **un 20 % de la nota**. Són els que més sovint queden fluixos: no els deixeu per al final.

- **Seguretat (bloc B4):** apliqueu **com a mínim 7** de les 10 mesures de la secció 8.4. Amb 10 mesures opteu a la puntuació màxima d'aquest aspecte.
- **Actualitzacions (bloc B4):** el cicle és sempre **còpia prèvia → actualitzar → verificar → registrar**. Prioritat a les actualitzacions de seguretat.
- **Proves (bloc B5):** dissenyareu 8 casos de prova a la S11 i els executareu a la S14, amb resultat esperat i resultat obtingut.
- **Còpies (bloc B5):** heu de copiar **els fitxers i la base de dades**, restaurar la còpia a `http://127.0.0.1:8081` i **verificar** que el contingut, els usuaris i els ajustaments són correctes. Una còpia sense verificar no compta.

---

## 6. La part amb Docker

A les sessions 17 i 18 desplegareu el mateix servei amb **Docker Compose**, però **en un servidor net**: una màquina Debian on **no hi ha ni Apache, ni PHP, ni MariaDB instal·lats a mà**.

> Per què? Perquè és com es treballa en les empreses avui en dia

El servidor net es prepara a la sessió 17 (duplicació mv) amb les regles `2223→22` i `8080→80`, i l'únic que s'hi instal·la és el **motor de contenidors**. El guió és a `practica_S17_servidor_net_docker.md`.

Com a mínim heu de saber fer i explicar:

- Què és una **imatge** i què és un **contenidor**, i què és un registre.
- Un servei per al CMS i un per a la base de dades, connectats per una **xarxa**.
- **Volums** per a la persistència: demostrar que el contingut sobreviu a un `docker compose down`.
- **Variables d'entorn** i fitxer `.env` per a les dades delicades (el `.env` real **no** es penja mai).
- `compose.yml`: llegir-lo, completar-lo i justificar cada bloc.
- Ordres bàsiques: `up -d`, `down`, `ps`, `logs -f`, `exec`, `restart`, `docker volume ls`, `docker network ls`.
- Còpia i restauració **des del contenidor**.

**Prova de persistència obligatòria:** creeu contingut → `docker compose down` → `docker compose up -d` → el contingut hi és. Després feu la prova destructiva (`down -v`) i expliqueu en una línia què s'ha perdut.

El lloc en contenidors es publica al **port 80 del servidor net** i surt a l'ordinador per `http://127.0.0.1:8080` (el port 80 de l'ordinador ja el fa servir el servidor tradicional).

---

## 7. Com es qualifica

### 7.1 Els 5 aspectes (100 % de la UD1)

| # | Aspecte | Pes | D'on surt l'evidència |
| :-: | --- | :-: | --- |
| 1 | Instal·lació i configuració de la infraestructura | 20 % | bloc B1 + comprovació a l'aula + script de verificació |
| 2 | Administració del CMS: usuaris, rols, menús, mòduls i continguts | 20 % | blocs B2 i B3 |
| 3 | Seguretat, actualització, proves i còpies de seguretat | 20 % | blocs B4 i B5 + prova final |
| 4 | Desplegament amb Docker Compose | 20 % | bloc B6 + prova final |
| 5 | Funcionament, documentació i justificació tècnica | 20 % | memòria completa + defensa oral |

Cada aspecte es valora amb 4 nivells:

| Nivell | Què vol dir |
| :-: | --- |
| **4** | Fet amb autonomia, sense errors, amb evidències i justificació tècnica |
| **3** | Fet correctament, amb algun detall sense justificar |
| **2** | Fet amb ajuda del guió o amb evidències incompletes |
| **1** | No funciona o no hi ha evidència |

### 7.2 La prova final (sessions 19 i 20, individual i dins de les 20 hores)

- Com que les sessions són d'una hora, la prova es fa en **dues sessions de 5 punts**

### 7.3 Què **no** es valora

- L'estètica de la web, el logotip bonic, les animacions o tenir molts apartats.
- Fer la web més gran del mínim.
- Programar temes o mòduls des de zero.
- Una memòria llarga: **es valora que sigui curta, clara i amb les captures justes**.

Es valora **instal·lar, configurar, administrar, assegurar, provar, copiar, desplegar i documentar**.

---

## 8. L'únic lliurable: la memòria tècnica

**Un fitxer: `cognom_nom_UD1.pdf`**.
S'hi afegeix **un segon fitxer**: `compose.yml`. Res més.

- Les captures van **dins del PDF**, retallades al que importa. **No** es lliuren fitxers d'imatge separats.
- Les captures han de mostrar **la comanda i la seva sortida**, no només la pantalla del navegador.
- **Cap contrasenya real**: tapeu-les o feu servir valors falsos.
- Hi ha una captura que no aporta res? Fora.

### 8.1 Els 6 blocs

| Bloc | Què hi va (màxim) | Pàg. | CA | Lliurament |
| :-: | --- | :-: | :-: | :-: |
| **B1** Infraestructura i requisits | taula de requisits (8.2) + 2 captures: `SHOW GRANTS` i `curl -I http://127.0.0.1` | 1 | 1.1 | S4 |
| **B2** Instal·lació i configuració del CMS | 3 línies sobre l'estructura de directoris + 2 captures: `ls -l` amb permisos i `wp-config.php` amb les dades tapades | 1 | 1.1 | S6 |
| **B3** Administració del portal | matriu de rols (8.3) + llista de mòduls amb 1 línia de justificació cadascun + 1 captura d'una prova d'accés en incògnit | 1 | 1.2, 1.3, 1.6 | S10 |
| **B4** Seguretat i actualitzacions | llista de les mesures aplicades (8.4) + registre d'actualitzacions de 3 files (8.5) | 1 | 1.5, 1.7 | S13 |
| **B5** Proves, còpies i fòrums | pla de proves de 8 casos amb resultats (8.6) + 5 línies del registre de còpia i verificació (8.7) + 1 captura del fòrum restringit | 1–2 | 1.4, 1.8, 1.9, 1.10 | S16 |
| **B6** Contenidors | `compose.yml` comentat + captura de la persistència abans/després + comparativa (8.8) amb conclusió de 3 línies | 1 | 1.1, 1.5, 1.9 | S18 |

**Més:** la **defensa oral individual (5 minuts)** a la sessió 18. No és un document: us preguntarem per la vostra màquina i pel vostre `compose.yml`.

### 8.2 Taula de requisits (bloc B1)

Llegiu la documentació oficial del CMS i contrasteu-ho amb la vostra màquina.

| Requisit | Mínim segons la documentació | Recomanat | El meu sistema (comprovat) |
| --- | --- | --- | --- |
| Sistema operatiu | | | `lsb_release -a` |
| Servidor web i versió | | | `apache2ctl -v` |
| Intèrpret i extensions | | | `php -v` · `php -m` |
| Gestor de base de dades i versió | | | `mariadb --version` |
| Memòria i espai | | | `free -h` · `df -h /var/www` |
| Límits de PHP | | | `php -i \| grep -E 'memory_limit\|upload_max'` |
| Port del lloc i redirecció NAT | | | `ss -lntp` + `curl -I http://127.0.0.1` |

### 8.3 Matriu de rols i capacitats (bloc B3)

| Rol del gestor | Usuari de prova | Crea | Publica | Modera | Gestiona usuaris | Instal·la mòduls |
| --- | --- | :-: | :-: | :-: | :-: | :-: |
| Administrador | | ☐ | ☐ | ☐ | ☐ | ☐ |
| Editor | | ☐ | ☐ | ☐ | ☐ | ☐ |
| Professor (rol personalitzat) | | ☐ | ☐ | ☐ | ☐ | ☐ |
| Alumne ESO / BATX / FP | | ☐ | ☐ | ☐ | ☐ | ☐ |
| Convidat (sense compte) | — | ☐ | ☐ | ☐ | ☐ | ☐ |

A sota, **una línia** per justificar cada rol i **una captura** d'una prova d'accés en finestra d'incògnit.

### 8.4 Llista de verificació de seguretat (bloc B4)

Marqueu les aplicades i escriviu **una línia** de comanda o constant al costat. Mínim 7.

| # | Mesura | Aplicada? | Com ho demostres (una línia) |
| --- | --- | :-: | --- |
| 1 | Compte d'administrador sense nom previsible i contrasenya forta | ☐ | |
| 2 | Actualitzacions automàtiques de seguretat activades | ☐ | |
| 3 | Limitació d'intents d'entrada | ☐ | |
| 4 | Doble factor per a comptes amb privilegis | ☐ | |
| 5 | Edició de fitxers des del tauler desactivada (`DISALLOW_FILE_EDIT`) | ☐ | |
| 6 | Mode de depuració desactivat en producció (`WP_DEBUG false`) | ☐ | |
| 7 | Prefix de taules no predefinit | ☐ | |
| 8 | Versió del gestor no exposada | ☐ | |
| 9 | Permisos de fitxers mínims | ☐ | |
| 10 | Còpies automàtiques programades i verificades | ☐ | |

### 8.5 Registre d'actualitzacions (bloc B4)

| Data | Element | Versió abans → després | Tipus | Còpia prèvia | Verificació |
| --- | --- | --- | --- | :-: | --- |
| | Nucli | | | ☐ | |
| | Tema | | | ☐ | |
| | Mòdul | | | ☐ | |

### 8.6 Pla de proves de funcionament (bloc B5) — 8 casos

A la S11 ompliu les tres primeres columnes; a la S14, les dues últimes.

| ID | Objectiu | Com es comprova | Resultat obtingut | OK? |
| :-: | --- | --- | --- | :-: |
| P01 | El lloc respon des de l'ordinador | `curl -I http://127.0.0.1` → 200 | | ☐ |
| P02 | Enllaços permanents | Obrir una entrada per URL amigable → 200, no 404 | | ☐ |
| P03 | Canal RSS | Obrir `/feed` → XML vàlid | | ☐ |
| P04 | Formulari de contacte | Enviar el formulari → missatge rebut | | ☐ |
| P05 | Restricció per rol | Entrar com a alumne ESO → no veu el contingut d'FP | | ☐ |
| P06 | Fòrum restringit | Entrar com a alumne ESO → no veu el fòrum d'FP | | ☐ |
| P07 | Galetes i multidioma | Obrir en incògnit i canviar d'idioma | | ☐ |
| P08 | Restauració | `http://127.0.0.1:8081` → contingut i usuaris idèntics | | ☐ |

### 8.7 Registre de còpia i verificació (bloc B5) — 5 línies

| Camp | Valor |
| --- | --- |
| Mètode i data de la còpia | |
| Fitxers generats amb la seva mida i `sha256sum` | |
| Entorn de restauració i què s'ha comprovat (contingut / usuaris / ajustaments) | |
| Incidències durant la restauració | |
| Conclusió: la còpia és vàlida? Per què? | |

### 8.8 Comparativa dels dos models (bloc B6)

| Criteri | Instal·lació tradicional | Amb contenidors |
| --- | --- | --- |
| On és la configuració | | |
| Gestió dels serveis | | |
| Dependències i versions | | |
| Persistència de les dades | | |
| Actualitzacions i còpies | | |
| Reproductibilitat del desplegament | | |

**Conclusió pròpia (3 línies):** quan faries servir cada model, què hi guanyes i què hi perds.

### 8.9 Comprovació final abans de lliurar

| Bloc | Hi és? | Criteris que acredita |
| :-: | :-: | --- |
| B1 Infraestructura i requisits | ☐ | 1.1 |
| B2 Instal·lació del CMS | ☐ | 1.1 |
| B3 Administració: rols, menús, mòduls | ☐ | 1.2, 1.3, 1.6 |
| B4 Seguretat i actualitzacions | ☐ | 1.5, 1.7 |
| B5 Proves, còpies i fòrums | ☐ | 1.4, 1.8, 1.9, 1.10 |
| B6 Contenidors + `compose.yml` adjunt | ☐ | 1.1, 1.5, 1.9 |
| Defensa oral feta a la S18 | ☐ | tots |

---

## 9. Si alguna cosa no us surt

Si no podeu seguir el ritme, parlem i pactem un pla de treball amb les sessions pendents. **No es redueix el nombre de criteris a assolir.**

---

## 10. Consells pràctics

1. **Documenteu mentre feu la feina, no al final.** Feu la captura just després de cada pas i enganxeu-la al bloc corresponent: després no es pot reconstruir.
2. **Còpia abans de tocar res.** Abans d'actualitzar i abans d'experiments: captura de virtual box.
3. **Llegiu els errors.** El 90 % de les incidències d'aquesta unitat són permisos, mòduls de PHP que falten, credencials de la base de dades, el mòdul de reescriptura desactivat o una regla NAT mal posada.
4. **Tot passa per `127.0.0.1` i un port.** Si la web no es veu des del vostre ordinador, comproveu dues coses abans de tocar l'Apache: que escolti al port (`ss -lntp` dins la MV) i que existeixi la regla de redirecció de VirtualBox.
5. **Pregunteu amb la comanda a la pantalla.** «No em funciona» no es pot diagnosticar; «aquesta comanda em dona aquesta sortida» sí.
6. **Contrasenyes fora del PDF.** Tapeu-les a les captures.
7. **No copieu res d'Internet sense entendre-ho.**

### 10.1 Errors freqüents

| Simptoma | Causa habitual | Solució |
| --- | --- | --- |
| `Connection refused` per SSH | MV aturada o servei `ssh` inactiu | `sudo systemctl status ssh` dins la MV |
| `Connection timed out` per SSH | Regla de redirecció malament | Port convidat `22`, protocol `TCP` |
| `sudo: command not found` | A Debian `sudo` no ve instal·lat | `apt-get install sudo` com a root |
| «not in the sudoers file» | Falta la línia al `visudo` | `alumne ALL=(ALL:ALL) ALL` |
| `apt` no troba els paquets | Falta `apt update` | `sudo apt update` |
| El lloc no es veu al port 80 | El lloc per defecte de Debian s'ho queda tot | `a2dissite 000-default.conf` i `systemctl reload apache2` |
| La restauració no va al 8081 | Falta `Listen 8081` a `ports.conf` | `echo "Listen 8081" \| sudo tee -a /etc/apache2/ports.conf` |
| Apache no arrenca | `ServerTokens` dins del `<VirtualHost>` | Ha d'anar fora del bloc |
| La web no es veu des de l'ordinador | Falta la regla NAT 80→80 | Afegiu la redirecció a VirtualBox |
| `php -m` no mostra `mysqli` | Falta `php-mysql` | `sudo apt install php-mysql` |
| `403 Forbidden` | Propietari o permisos | `chown -R www-data:www-data` + 755/644 |
| URL amigables amb 404 | Sense `mod_rewrite` o `AllowOverride All` | `a2enmod rewrite` + `AllowOverride All` + reload |
| «Error establint la connexió amb la base de dades» | Credencials o `DB_HOST` | Reviseu `wp-config.php` i `SHOW GRANTS` |
| No es pot pujar una imatge | Permisos de `wp-content/uploads` | `chown www-data` al directori |
| `compose.yml` no arrenca | Indentació amb tabuladors | Només espais |
| El contenidor web no troba la BD | S'ha aixecat abans que la BD estigués a punt | `depends_on` amb `condition: service_healthy` |