# zen-browser-rpm

Automatiza la construcción y publicación de paquetes RPM firmados para **Zen Browser** en Fedora usando **GitHub Actions** + **GitHub Pages**.

## Qué hace

- consulta el último release de `zen-browser/desktop`
- descarga assets oficiales para `x86_64` y `aarch64`
- reempaqueta los binarios como RPM
- firma los RPM con GPG
- genera un repositorio DNF con `createrepo_c`
- firma `repodata/repomd.xml`
- publica el repo vía GitHub Pages
- crea un GitHub Release con los RPM adjuntos

## Estado inicial

El repo quedó preparado, pero **no debe ejecutarse todavía** hasta que reemplaces el archivo público de GPG y configures los secrets.

## Secrets requeridos

En **Settings → Secrets and variables → Actions** agrega:

### Secrets
- `GPG_PRIVATE_KEY`
- `GPG_PASSPHRASE`

### Variables
- `GPG_KEY_ID`

`GPG_KEY_ID` no necesita ser secret.

## Archivo público requerido

Reemplaza este archivo con tu llave pública real:

- `public/RPM-GPG-KEY-zen-browser`

Ejemplo:

```bash
gpg --armor --export <TU_KEY_ID> > public/RPM-GPG-KEY-zen-browser
```

## Habilitar GitHub Pages

En **Settings → Pages** selecciona:

- **Source:** GitHub Actions

## Disparo manual

Cuando ya estén listos la llave pública, secrets y Pages:

- ve a **Actions**
- abre el workflow **Build Fedora DNF repo for Zen Browser**
- ejecuta **Run workflow**

## Instalación esperada para usuarios

La URL final de Pages será:

```text
https://xins3c.github.io/zen-browser-rpm/
```

Comandos esperados de instalación:

```bash
sudo rpm --import https://xins3c.github.io/zen-browser-rpm/RPM-GPG-KEY-zen-browser

sudo tee /etc/yum.repos.d/zen-browser.repo > /dev/null << 'REPO'
[zen-browser]
name=Zen Browser
baseurl=https://xins3c.github.io/zen-browser-rpm/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://xins3c.github.io/zen-browser-rpm/RPM-GPG-KEY-zen-browser
skip_if_unavailable=True
metadata_expire=6h
REPO

sudo dnf clean all
sudo dnf install -y zen-browser
```

## Notas de seguridad

- La publicación se detiene si el archivo público `RPM-GPG-KEY-zen-browser` no coincide con la llave privada usada para firmar.
- El workflow usa `vars.GPG_KEY_ID` en lugar de guardar el key ID como secret.
- Los archivos auxiliares del RPM se generan con Python para evitar problemas de indentación/heredocs en YAML.
