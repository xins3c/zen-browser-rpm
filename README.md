# zen-browser-rpm

Signed RPM repository automation for **Zen Browser** on Fedora, published through **GitHub Actions** and **GitHub Pages**.

Repository URL:

- <https://github.com/xins3c/zen-browser-rpm>

Published repository URL:

- <https://xins3c.github.io/zen-browser-rpm/>

## What this repository does

This project:

- watches the latest release from `zen-browser/desktop`
- downloads the official `x86_64` and `aarch64` Linux tarballs
- repackages them as Fedora RPMs
- signs RPMs with GPG
- generates signed DNF repository metadata
- publishes repository contents to GitHub Pages
- publishes RPM assets to the GitHub Release for the matching upstream tag

## Security model

The workflow is designed to fail closed when core signing assumptions break.

Current protections include:

- repository metadata is signed with the same GPG identity used for RPM signing
- the published public key must match the imported private key fingerprint
- RPM publication stops if downloaded or rebuilt RPMs do not actually contain a signature
- static Pages republish can be triggered manually without waiting for a new upstream release
- `GPG_KEY_ID` is stored as a repository variable, not a secret
- packaging helper files are generated with Python instead of fragile YAML heredocs

## Repository configuration

Configure the following in:

**Settings → Secrets and variables → Actions**

### Repository secrets

- `GPG_PRIVATE_KEY`
- `GPG_PASSPHRASE`

### Repository variables

- `GPG_KEY_ID`

Use the full fingerprint for `GPG_KEY_ID`, not a short key ID.

## Public key file

Replace the tracked public key file with your real ASCII-armored public key:

- `public/RPM-GPG-KEY-zen-browser`

Example:

```bash
gpg --armor --export <FULL_FINGERPRINT> > public/RPM-GPG-KEY-zen-browser
```

The file must start with:

```text
-----BEGIN PGP PUBLIC KEY BLOCK-----
```

## GitHub Pages

Enable GitHub Pages for this repository with:

- **Source:** GitHub Actions

## Workflow behavior

The workflow supports two manual control modes through `workflow_dispatch`.

### 1. Republish Pages only

Use this when you changed:

- `public/RPM-GPG-KEY-zen-browser`
- `public/index.html`
- static repository metadata or documentation behavior

Inputs:

- `republish_pages=true`
- `force_rebuild=false`

This republishes Pages even when the upstream Zen Browser version did not change.

### 2. Force a full rebuild

Use this when the current release assets are wrong and must be replaced, for example:

- RPMs were generated incorrectly
- RPM signatures were missing or invalid
- packaging logic changed and you need fresh assets for the same upstream tag

Inputs:

- `republish_pages=true`
- `force_rebuild=true`

This rebuilds the RPMs for the current upstream tag and overwrites release assets.

## Typical operation

### Automatic path

The scheduled workflow checks upstream daily.

If a new Zen Browser release exists, it will:

1. build new RPMs
2. sign them
3. build new repository metadata
4. publish the repo to GitHub Pages
5. create or update the matching GitHub Release assets

### Manual path

Go to **Actions → Build Fedora DNF repo for Zen Browser → Run workflow** and choose the appropriate inputs.

## Fedora client installation

### Recommended import path

Import the public key from a local file first. This is more robust for testing and avoids odd client-side keyring behavior.

```bash
sudo mkdir -p /etc/pki/rpm-gpg
sudo curl -fsSL https://xins3c.github.io/zen-browser-rpm/RPM-GPG-KEY-zen-browser \
  -o /etc/pki/rpm-gpg/RPM-GPG-KEY-zen-browser
sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-zen-browser
```

Create the repo file:

```bash
sudo tee /etc/yum.repos.d/zen-browser.repo > /dev/null << 'REPO'
[zen-browser]
name=Zen Browser
baseurl=https://xins3c.github.io/zen-browser-rpm/
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-zen-browser
skip_if_unavailable=True
metadata_expire=6h
REPO
```

Then refresh metadata and install:

```bash
sudo dnf clean all
sudo dnf makecache --refresh
sudo dnf install -y zen-browser
```

## Validation checklist

After a successful publish, verify:

- <https://xins3c.github.io/zen-browser-rpm/>
- <https://xins3c.github.io/zen-browser-rpm/RPM-GPG-KEY-zen-browser>
- <https://xins3c.github.io/zen-browser-rpm/repodata/repomd.xml>
- <https://xins3c.github.io/zen-browser-rpm/repodata/repomd.xml.asc>

Then validate on a Fedora client:

```bash
sudo dnf clean all
sudo dnf makecache --refresh
sudo dnf install -y zen-browser
rpm -qi zen-browser
```

## Troubleshooting

### `repomd.xml GPG signature verification error: Signing key not found`

Usually means DNF did not import or use the repo metadata key correctly.

Use the local-file key import path from the installation section, then clean cache:

```bash
sudo dnf clean all
sudo rm -rf /var/cache/libdnf5/zen-browser-*
sudo dnf makecache --refresh
```

### `The package is not signed`

That means the published RPM asset is unsigned or stale.

Run the workflow manually with:

- `republish_pages=true`
- `force_rebuild=true`

This forces a rebuild and replaces release assets for the current upstream tag.

### Root Pages URL returns 404

That means `public/index.html` is missing from the published artifact. The repository itself may still work, but the root URL should not be considered healthy until the Pages artifact includes `index.html`.

## Operational notes

- Keep a secure backup of the private key, passphrase, and fingerprint.
- If the private key is lost, future updates cannot be signed with the same identity.
- Test installation on a second clean Fedora machine before trusting a new packaging change.
- This repository can serve as the base pattern for packaging other software later.
