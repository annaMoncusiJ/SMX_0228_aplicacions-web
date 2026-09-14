# Simulador d'un servidor Debian amb VirtualBox

Guia pas a pas per crear una màquina virtual amb **Debian** a **VirtualBox** que simuli un servidor i connectar-s'hi des de l'ordinador local (amfitrió) mitjançant **SSH**.

---

## Índex

1. [Requisits previs](#1-requisits-previs)
2. [Crear la màquina virtual](#2-crear-la-màquina-virtual)
3. [Instal·lar Debian](#3-installar-debian)
4. [Configurar la xarxa NAT amb redireccionament de ports](#4-configurar-la-xarxa-nat-amb-redireccionament-de-ports)
5. [Connectar-se via SSH des de l'amfitrió](#5-connectar-se-via-ssh-des-de-lamfitrió)
6. [Resolució de problemes](#7-resolució-de-problemes)

---

## 1. Requisits previs

- **VirtualBox** instal·lat: <https://www.virtualbox.org/wiki/Downloads>
- **Imatge ISO de Debian** (versió *netinst*): <https://www.debian.org/distrib/netinst.en.html>
- Un client SSH a l'ordinador local

---

## 2. Crear la màquina virtual

1. Obre VirtualBox i fes clic a **«Nova»** (New).
2. Emplena l'assistent:

   | Camp | Valor recomanat |
   |---|---|
   | Nom | `servidor-debian` |
   | ISO Image | Selecciona la imatge descarregada anteriorment |
   | Tipus | `Linux` |
   | Versió | `Debian (64-bit)` |
   | Memòria RAM | `2048 MB` (mínim 1024 MB) |
   | Processors | `2` (mínim 1) |
   | Disc virtual | `VDI`, dinàmic, **32 GB** minim |

> [!CAUTION]
> Activa el checkbox "Skip Unattended Installation" de la primera part de la configuració.

3. Fes clic a **«Acaba»** per crear la màquina.

> [!NOTE]
> Un servidor sense entorn gràfic consumeix molt pocs recursos; amb 1 CPU i 1 GB de RAM ja funciona, però 2 GB donen més marge.

---

## 3. Instal·lar Debian

### 3.1. Arrencar

Arrenca la màquina (**Inicia**) i tria **«Install»** (mode text) o **«Graphical install»** segons prefereixis.

### 3.2. Passes de la instal·lació

1. **Idioma, país i teclat**: tria els que prefereixis (p. ex. Catalan / Espanya).
2. **Nom de la màquina (hostname)**: `servidor-debian`.
3. **Contrasenya de root**: defineix-ne una recomano que sigui `alumne` per a ser fàcil de recordar.
4. **Usuari normal**: crea el teu usuari (p. ex. `alumne`) amb la seva contrasenya.
5. **Particionat del disc**: si és la primera vegada, tria **«Guiat − utilitzar tot el disc»** → **«Tots els fitxers en una partició»** → confirma escrivint els canvis al disc (**«Sí»**).
6. **Gestor de paquets**: tria una rèplica (mirror) propera quan te la demani.

### 3.3. Selecció de programari (pas clau)

Quan aparegui la pantalla **«Selecció de programari»**:

- ❌ **Desmarca** *«Entorn d'escriptori Debian»*  i tots els altes (volem un servidor, no un escriptori).
- ✅ **Marca** *«Servidor SSH»* (**openssh-server**).
- ✅ **Marca** *«Utilitats estàndard del sistema»*.

### 3.4. Finalitzar instal·lació

1. Instal·la el carregador d'arrencada **GRUB** al disc virtual (`/dev/sda`) quan ho demani.
2. Acaba la instal·lació i reinicia.
3. Inicia sessió a la consola amb el teu usuari per comprovar que tot funciona.

### 3.5. Configuració

1. Iniciem sessió amb usuari: `root` contrasenya: `alumne` o la que hagueu ficat.
2. Instal·lem sudo:
   ```bash
   apt-get install sudo -y
   ```
3. Afegim el nostre usuari com a sudo
   ```bash
   sudo visudo
   ```
   ```bash
   # User privilege specification
   root    ALL=(ALL:ALL) ALL
   ibc     ALL=(ALL:ALL) ALL
   ```
4. sortim de l'usuari
   ```bash
   exit
   ```

Amb aixó, ara ja podrem realitzar comandes com a administrador des del nostre usuari sense necessitat de ser root.

---

## 4. Configurar la xarxa NAT amb redireccionament de ports

Amb xarxa **NAT**, la màquina virtual surt a Internet a través de l'amfitrió, però **no és accessible des de fora per defecte**. La solució és redirigir un port de l'amfitrió cap al port 22 (SSH) de la màquina virtual.

### 4.1. Des de la interfície gràfica

1. Amb la màquina **aturada** ves de la màquina virtual a:
   **Configuració → Xarxa → Adaptador 1**.
2. Comprova que està **habilitat** i que *«Connectat a»* diu **«NAT»**.
3. Desplega la secció **«Avançat»** i fes clic a **«Redirecció de ports»**.
4. Afegeix una regla nova amb la icona **+** :

   | Nom | Protocol | IP amfitrió | Port amfitrió | IP convidat | Port convidat |
   |-----|----------|-------------|---------------|-------------|----------------|
   | `ssh` | `TCP` | `127.0.0.1` | `8022` | *(en blanc)* | `22` |

5. Desa amb **Acceptar**.

### 4.2. Comprovar que SSH funciona dins de la VM

Arrenca la màquina i, dins de la consola, verifica que el servei està actiu:

```bash
sudo systemctl status ssh
```

Si no estigués instal·lat (no vas marcar l'opció a la instal·lació), instala-ho:

```bash
sudo apt update
sudo apt install openssh-server
sudo systemctl enable --now ssh
```
Un cop instal·lat comprova que s'ha instal·lat correctament

---

## 5. Connectar-se via SSH des de l'amfitrió

Amb la màquina virtual engegada, obre un terminal a l'ordinador local:

```bash
ssh alumne@127.0.0.1 -p 8022
```

- `usuari` → l'usuari que vas crear a la instal·lació de Debian.
- `-p 8022` → el port redirigit.

La primera connexió et demanarà confirmar la petjada digital del servidor.

Escriu `yes`, introdueix la contrasenya i... ja ets dins! 🎉

## 6. Resolució de problemes

| Problema | Possible solució |
|---|---|
| `Connection refused` | Comprova que la VM està engegada i que `systemctl status ssh` està *active (running)* dins de la VM. |
| `Connection timed out` | Revisa la regla de redirecció: el port convidat ha de ser `22` i el protocol `TCP`. |
| El port 8022 ja està ocupat | Canvia el port amfitrió (p. ex. `2222`) a la regla i fes `ssh -p 2200 ...`. |