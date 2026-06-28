# JHU WebSec, Zero Trust and Identity Assignment Files

This repository contains the configuration files for the assignment.

Use these files to build the lab environment in your Linux sandbox so you can focus on the main goal of the assignment: applying Zero Trust and identity management principles to a web application, then evaluating the result using AAA and CI4A.

## Local lab status

The running stack is:

- Nginx Proxy Manager reverse proxy on ports `80`, `81`, and `443`
- Authelia identity layer on the internal Docker network
- Ghost CMS and MySQL behind the proxy
- self-signed TLS certificates for `auth.home.local`, `blog.home.local`, and `admin.home.local`

The local runtime files that contain generated secrets, database state, certificates, or credentials are intentionally ignored by git. Do not submit `.env`, `.local/`, `authelia/config/`, `certs/`, `ghost/`, or `nginx-proxy-manager/`.

The report and captured evidence are in:

- `submission/Module11_Zero_Trust_Identity_Management_Report.md`
- `submission/evidence/01-baseline-and-admin-redirect.log`
- `submission/evidence/02-after-ghost-block.log`
- `submission/evidence/03-authelia-identity-evidence.log`

Local testing used `curl --resolve <host>:443:127.0.0.1` instead of editing `/etc/hosts`, because privileged host-file edits were not available in this environment.

## Files in this repository

- `JHUWebSec_IDZT_files.zip`  
  Zip archive containing all assignment files.

The ZIP archive contains:

- `docker-compose.yml`  
  Starter Docker Compose file for the lab stack.

- `configuration.yml`  
  Main Authelia configuration file.

- `users_database.yml`  
  Local user database for the lab.

- `admin_adv.config`  
  Reverse proxy configuration used to protect the admin path with the identity layer.

- `ghostBlock.config`  
  Reverse proxy configuration used to block direct access to the `/ghost` path.

## How to get the files in your Linux sandbox

You may either download the ZIP file or download the individual files.

### Option 1, download the ZIP file

Download `JHUWebSec_IDZT_files.zip` into your Linux sandbox, then unzip it.

Example in bash:

```bash
mkdir -p ~/IDZT
cd ~/IDZT
wget https://raw.githubusercontent.com/drcjhu/JHU_WebSec_ZT_ID_Module/main/JHUWebSec_IDZT_files.zip
unzip JHUWebSec_IDZT_files.zip
```
If your instructions place files in slightly different locations, follow the assignment guidance. The main goal is to keep the environment organized and easy to troubleshoot.

The ZIP file structure:

```text
~/IDZT/
├── docker-compose.yml
├── Advanced Config/
│   ├── admin.home.local/
│   │   └── admin_adv.config
│   └── blog.home.local/
│       └── ghostBlock.config
└── authelia/
    └── config/
        ├── configuration.yml
        └── users_database.yml
```
