# Apache Airflow 3 Deployment on EC2 (Ubuntu)

A guide to deploying **Apache Airflow 3.0.6** on a single Ubuntu EC2 instance with PostgreSQL as the metadata database and `LocalExecutor`.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [1. Update & Install System Packages](#1-update--install-system-packages)
- [2. Configure PostgreSQL](#2-configure-postgresql)
- [3. Create Python Virtual Environment](#3-create-python-virtual-environment)
- [4. Install Airflow 3](#4-install-airflow-3)
- [5. Initialize Metadata DB & Config](#5-initialize-metadata-db--config)
- [6. Start Airflow Services](#6-start-airflow-services)
- [7. Health Check](#7-health-check)
- [8. Set Admin Password](#8-set-admin-password)
- [9. Persist Environment Variables](#9-persist-environment-variables)
- [10. Start / Stop Airflow (After Reboot)](#10-start--stop-airflow-after-reboot)
- [Accessing the Web UI](#accessing-the-web-ui)
- [Notes & Security Recommendations](#notes--security-recommendations)

---

## Overview

| Component        | Value                          |
|-------------------|---------------------------------|
| Airflow Version   | 3.0.6                          |
| Executor          | LocalExecutor                  |
| Metadata DB       | PostgreSQL                     |
| Auth Manager      | Simple Auth Manager (default)  |
| Airflow Home      | `$HOME/airflow3`               |
| API/Web UI Port   | `8080`                         |

---

## Prerequisites

- An EC2 instance running Ubuntu (20.04/22.04/24.04)
- A user with `sudo` privileges
- Security group inbound rule allowing TCP traffic on port `8080` from your IP
- Basic familiarity with the Linux shell

---

## 1. Update & Install System Packages

```bash
sudo apt update && sudo apt -y upgrade
sudo apt install -y python3 python3-venv python3-pip build-essential \
  libpq-dev postgresql postgresql-contrib wget curl unzip
```

Set the Airflow home directory:

```bash
export AIRFLOW_HOME="$HOME/airflow3"
mkdir -p "$AIRFLOW_HOME"
cd "$AIRFLOW_HOME"
```

---

## 2. Configure PostgreSQL

Enable and start the PostgreSQL service:

```bash
sudo systemctl enable --now postgresql
```

Create the Airflow database role and database:

```bash
export AF_DB=airflow_db
export AF_DB_USER=airflow_user
export AF_DB_PASS="password"

sudo -u postgres psql -c "CREATE ROLE ${AF_DB_USER} LOGIN PASSWORD '${AF_DB_PASS}';"
sudo -u postgres psql -c "CREATE DATABASE ${AF_DB} OWNER ${AF_DB_USER};"
sudo -u postgres psql -d ${AF_DB} -c "GRANT ALL ON SCHEMA public TO ${AF_DB_USER};
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON TABLES TO ${AF_DB_USER};
ALTER DEFAULT PRIVILEGES IN SCHEMA public GRANT ALL ON SEQUENCES TO ${AF_DB_USER};"
```

Verify the connection string:

```bash
echo "postgresql+psycopg2://${AF_DB_USER}:${AF_DB_PASS}@localhost:5432/${AF_DB}"
```

> ⚠️ **Replace `password` with a strong, unique password before using this in any real environment.**

---

## 3. Create Python Virtual Environment

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip setuptools wheel
```

---

## 4. Install Airflow 3

```bash
export AIRFLOW_VERSION="3.0.6"
export PYTHON_VERSION=$(python -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")')
export CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"

pip install "apache-airflow[postgres]==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"
```

---

## 5. Initialize Metadata DB & Config

Generate a Fernet key (used to encrypt connection credentials):

```bash
export FERNET_KEY=$(python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())")
```

Set core environment variables:

```bash
export AIRFLOW__DATABASE__SQL_ALCHEMY_CONN="postgresql+psycopg2://${AF_DB_USER}:${AF_DB_PASS}@localhost:5432/${AF_DB}"
export AIRFLOW__CORE__EXECUTOR="LocalExecutor"
export AIRFLOW_HOME
```

Run the metadata database migration:

```bash
airflow db migrate
```

---

## 6. Start Airflow Services

```bash
mkdir -p $AIRFLOW_HOME/logs

export AIRFLOW__API__HOST=0.0.0.0
export AIRFLOW__API__PORT=8080

airflow scheduler      > "$AIRFLOW_HOME/logs/scheduler.log"      2>&1 &
airflow dag-processor  > "$AIRFLOW_HOME/logs/dag-processor.log"  2>&1 &
airflow triggerer      > "$AIRFLOW_HOME/logs/triggerer.log"      2>&1 &
airflow api-server     > "$AIRFLOW_HOME/logs/api-server.log"     2>&1 &
```

Airflow 3 runs as four separate components:

| Component      | Purpose                                             |
|-----------------|------------------------------------------------------|
| `scheduler`     | Schedules and triggers DAG task instances             |
| `dag-processor` | Parses DAG files and serializes them to the metadata DB |
| `triggerer`     | Handles deferrable/async tasks                        |
| `api-server`    | Serves the REST API and the web UI                     |

---

## 7. Health Check

```bash
airflow version && airflow info | head -n 20
curl -s http://localhost:8080/api/v2/monitor/health | jq .
```

---

## 8. Set Admin Password

Airflow 3's default **Simple Auth Manager** generates a password file on first run.

```bash
ls -l $AIRFLOW_HOME/*auth*.json*
cat  $AIRFLOW_HOME/simple_auth_manager_passwords.json.generated
```

To change it:

```bash
nano $AIRFLOW_HOME/simple_auth_manager_passwords.json.generated
```

Then access the UI at:

```
http://<EC2-PUBLIC-IP>:8080
```

---

## 9. Persist Environment Variables

Exit the virtual environment and add variables to `.bashrc` so they persist across sessions/reboots:

```bash
deactivate
nano .bashrc
```

Append the following:

```bash
# Airflow persistent environment variables
export AIRFLOW_HOME="$HOME/airflow3"

# Database credentials
export AF_DB=airflow_db
export AF_DB_USER=airflow_user
export AF_DB_PASS="password"

# Airflow database connection string
export AIRFLOW__DATABASE__SQL_ALCHEMY_CONN="postgresql+psycopg2://${AF_DB_USER}:${AF_DB_PASS}@localhost:5432/${AF_DB}"

# LocalExecutor
export AIRFLOW__CORE__EXECUTOR="LocalExecutor"

# Airflow API / Webserver configs
export AIRFLOW__API__HOST=0.0.0.0
export AIRFLOW__API__PORT=8080
```

Reload the shell config:

```bash
source .bashrc
```

---

## 10. Start / Stop Airflow (After Reboot)

Whenever the EC2 instance is stopped and restarted from the AWS console, bring Airflow back up with:

```bash
cd ~/airflow3
source .venv/bin/activate

sudo systemctl start postgresql
sudo systemctl status postgresql

airflow scheduler      > "$AIRFLOW_HOME/logs/scheduler.log"      2>&1 &
airflow dag-processor  > "$AIRFLOW_HOME/logs/dag-processor.log"  2>&1 &
airflow triggerer      > "$AIRFLOW_HOME/logs/triggerer.log"      2>&1 &
airflow api-server     > "$AIRFLOW_HOME/logs/api-server.log"     2>&1 &
```

To stop all Airflow processes:

```bash
pkill -f "airflow scheduler"
pkill -f "airflow dag-processor"
pkill -f "airflow triggerer"
pkill -f "airflow api-server"
```

---

## Accessing the Web UI

1. Ensure your EC2 security group allows inbound traffic on port `8080` from your IP.
2. Open a browser and navigate to:

   ```
   http://<EC2-PUBLIC-IP>:8080
   ```

3. Log in using the admin credentials from `simple_auth_manager_passwords.json.generated`.

---

## Notes & Security Recommendations

- **Do not use `password` as a real database password.** Replace `AF_DB_PASS` with a strong, randomly generated secret and consider storing it in a secrets manager rather than plain environment variables or `.bashrc`.
- **Restrict port 8080** to trusted IPs only, or place Airflow behind a reverse proxy (e.g., Nginx) with TLS and proper authentication.
- **Simple Auth Manager** is intended for development/testing. For production, consider integrating an external auth provider (e.g., FAB with LDAP/OAuth, or a supported enterprise auth manager).
- Consider running Airflow components as **systemd services** instead of background (`&`) processes for better reliability, auto-restart, and log management.
- Rotate the `FERNET_KEY` carefully — rotating it will invalidate existing encrypted connection passwords in the metadata DB unless a proper key-rotation procedure is followed.
- Back up the PostgreSQL metadata database regularly.

---

## License

Add your project's license here (e.g., MIT, Apache 2.0).
