<p align="center">
  <a href="https://codezone.io/">
    <img alt="CodeZone" src="https://prismic-io.s3.amazonaws.com/codezone/5f2169a6-d854-478d-b0d4-93e8b18d0bb7_cz-lines-orange-dark.svg" height="100"><br />
  </a>
</p>
<div style="color: #666; margin-bottom: 15px" align="center"><i>Crafted by CodeZone</i></div>  





# DDEV Kinsta Provider (SSH / rsync)

This repository contains a custom **DDEV provider** for synchronizing a WordPress site hosted on **Kinsta** with a local DDEV environment using **SSH, rsync, and WP-CLI**.

It supports pulling and pushing:

- WordPress database  
- `wp-content/uploads`  
- `wp-content/plugins`

---

## Quick How-To

### Pull database + files from Kinsta
```bash
ddev auth ssh
ddev pull kinsta
```

### Pull database only
```bash
ddev auth ssh
ddev pull kinsta --skip-files
```

### Pull files only (uploads + plugins)
```bash
ddev auth ssh
ddev pull kinsta --skip-db
```

### Push local database + files to Kinsta
```bash
ddev auth ssh
ddev push kinsta
```

### Push database only
```bash
ddev auth ssh
ddev push kinsta --skip-files
```

### Push files only
```bash
ddev auth ssh
ddev push kinsta --skip-db
```

**Notes**

- `ddev auth ssh` is required once per DDEV session (or after reboot).
- The provider name comes from the filename: `kinsta.yaml` → `ddev pull kinsta`.
- **Push commands can overwrite remote data.** Read the safety notes below.

---

## Getting SSH Credentials from Kinsta

To use this provider, you need SSH access details from your Kinsta site.

### Step 1: Open Your Site in MyKinsta
1. Log in to **MyKinsta**
2. Select the site you want to connect to
3. Open the **Info** tab

### Step 2: Locate SSH Access Details
In the **SFTP / SSH** section, note the following:

- **SSH Host**  
- **Port**
- **Username**
- **Path** (this is usually the WordPress root directory)

Example values you might see:

- Host: `35.xxx.xxx.xxx`
- Port: `62497`
- Username: `example`
- Path: `/www/example_123/public`

These map directly to the provider configuration:

- `host` → SSH Host
- `port` → SSH Port
- `user` → Username
- `sitePath` → Path

### Step 3: Add Your SSH Key to Kinsta
1. In MyKinsta, go to **User Settings → SSH Keys**
2. Add your **public SSH key** (`~/.ssh/id_rsa.pub` or similar)
3. Save the key

> Kinsta does not allow password-based SSH. Keys are required.

### Step 4: Test SSH Access
From your host machine:
```bash
ssh -p <port> <user>@<host>
```

From inside DDEV:
```bash
ddev ssh
ssh -p <port> <user>@<host>
```

If both succeed, you’re ready to use the provider.

---

## What This Provider Does

This provider uses **SSH**, **rsync**, and **WP-CLI** to synchronize a WordPress site between a remote Kinsta environment and a local DDEV project.

### Pull (`ddev pull kinsta`)

**Database**
1. SSH into the remote server  
2. Export the WordPress database using WP-CLI  
3. Gzip the export on the remote server  
4. Rsync the dump into the local DDEV container  
5. Import the database locally  
6. Run `wp search-replace` to swap **remote URLs → local URLs**

**Files**
- Rsync `wp-content/uploads`
- Rsync `wp-content/plugins`

### Push (`ddev push kinsta`)

**Database**
- Export local DB, gzip it, upload to remote, import remotely
- Run `wp search-replace` to swap **local URLs → remote URLs**

**Files**
- Rsync local uploads and plugins to the remote site

---

## Requirements

- **SSH access** to the Kinsta environment (host, port, user)
- **WP-CLI installed and working**:
  - Inside the DDEV web container (local)
  - On the remote server
- Your SSH key loaded for the session:

```bash
ddev auth ssh
```

---

## Installation & Setup

1. Copy the provider file to:

```text
.ddev/providers/kinsta.yaml
```

2. Confirm the environment variables in `kinsta.yaml`:

- `host` – remote server IP or hostname  
- `port` – SSH port  
- `user` – SSH user  
- `sitePath` – path to the WordPress root on the remote server  
  (directory where `wp-config.php` lives)  
- `dbPath` – remote temp directory for database exports  
- `dbFileName` – database dump filename (without `.gz`)  

3. Verify SSH access **from inside DDEV**:

```bash
ddev ssh
ssh -p <port> <user>@<host> "cd <sitePath> && wp option get siteurl"
```

4. Run your first pull:

```bash
ddev pull kinsta
```

---

## Safety Notes (Read Before Using Push)

- `ddev push kinsta` can overwrite:
  - A live production database
  - Uploaded media
  - Installed plugins
- Treat push as a **destructive operation**:
  - Prefer pushing only to **staging** environments
  - Take a Kinsta backup or snapshot first
  - Avoid pushing to production unless absolutely necessary

---

## Common Troubleshooting

### “Please `ddev auth ssh` before running this command”
Run:
```bash
ddev auth ssh
```

### SSH works on host but fails during pull/push
Test from inside the container:
```bash
ddev ssh
ssh -p <port> <user>@<host>
```

### WP-CLI not found on remote
Ensure `wp` is installed and available in `PATH` for the SSH user.

### Wrong `sitePath`
Verify with:
```bash
ssh -p <port> <user>@<host> "cd <sitePath> && wp option get siteurl"
```

### URL replacement issues
This provider uses:

- `--precise`
- `--recurse-objects`
- `--skip-columns=guid`

Multisite or unusual setups may require adjustments.

---

## Customization Notes

- **Plugins via Composer**  
  If plugins are managed via Composer, consider removing the plugin rsync lines from the provider.

- **Exact mirroring**  
  Adding `--delete` to rsync will remove files on the destination that don’t exist locally.  
  This is powerful and dangerous — use sparingly.
