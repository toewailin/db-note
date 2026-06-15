# Enterprise Database Administration Runbook: DBngin Ecosystem Integration

**Target Platforms:** MySQL & PostgreSQL (Windows Environment)

**Role Context:** Database Administrator (DBA) Operations Manual

**Classification:** Internal Technical Reference

---

## 1. Executive Summary & Infrastructure Overview

Modern development topologies require isolated database runtimes that mimic high-availability staging environments without introducing service bloat to the host OS. **DBngin** delivers this by executing database engine binaries on demand, bypassing native Windows Services manager locks.

While DBngin provides the storage engines, administrators require intuitive visual planes to run schema migrations, perform index audits, and evaluate query plan optimizations. This manual provides detailed configuration playbooks to deploy web-based visual clients (**phpMyAdmin** for MySQL, **Adminer** for PostgreSQL) directly over DBngin's isolated listener ports.

```
+-----------------------------------------------------------------------+
|                           WINDOWS HOST NETWORK                        |
|                                (127.0.0.1)                            |
+-----------------------------------+-----------------------------------+
                                    |
          +-------------------------+-------------------------+
          |                                                   |
          ▼                                                   ▼
+----------------───+                               +----------------───+
|   MySQL Engine    |                               | PostgreSQL Engine |
|   (via DBngin)    |                               |   (via DBngin)    |
| Port: 3306 / 3307 |                               | Port: 5432 / 5433 |
+---------┬---------+                               +---------┬---------+
          ▲                                                   ▲
          │ (config.inc.php)                                  │ (Driver Mapping)
          │                                                   │
+---------┴---------+                               +---------┴---------+
|    phpMyAdmin     |                               |      Adminer      |
| Built-in PHP Host |                               | Built-in PHP Host |
| URL: :8080        |                               | URL: :8081        |
+-------------------+                               +-------------------+

```

---

## 2. PART I: MySQL Administration Lifecycle via phpMyAdmin

### Step 1: Network Socket & Port Discovery

By default, standalone instances of MySQL bind to TCP port `3306`. If your system has remnants of a previous database installation (such as XAMPP, WampServer, or native MySQL Installer), DBngin will gracefully fail over to subsequent structural socket ports (`3307`, `3308`, etc.).

1. Launch the **DBngin** controller panel.
2. Identify your active MySQL engine row.
3. Record the explicit value present in the **Port** column.
4. Ensure the execution state toggle is active (**ON**).

### Step 2: Filesystem Deployment & Staging

1. Navigate to the official [phpMyAdmin Downloads Portal](https://www.phpmyadmin.net/downloads/).
2. Extract the archive payload `phpMyAdmin-all-languages.zip`.
3. Move the extracted files into a dedicated tool subdirectory, for example: `C:\DBTools\phpmyadmin\`.

### Step 3: Enterprise Parameter Configuration

phpMyAdmin requires an active configuration file to handle session state cryptography and direct connection strings over non-standard network ports.

1. Locate `config.sample.inc.php` inside your workspace directory and clone it as **`config.inc.php`**.
2. Open `config.inc.php` using an elevated IDE or text editor.
3. Locate the `blowfish_secret` configuration block. Insert a random 32-character string to encrypt cookie arrays:
```php
$cfg['cookie_secret'] = 'a8f7b6c5d4e3f2g1h0i9j8k7l6m5n4o3p2q1'; 
// Note: Depending on your build, the variable may look like:
// $cfg['blowfish_secret'] = 'a8f7b6c5d4e3f2g1h0i9j8k7l6m5n4o3p2q1';

```


4. Scroll down to the Server Parameters array and change the default values from `localhost` to the precise loopback IPv4 address and target port:
```php
$cfg['Servers'][$i]['host'] = '127.0.0.1'; // Bypasses Windows named pipes lookup slowdowns
$cfg['Servers'][$i]['port'] = '3307';      // Replace with your explicit DBngin Port

```


5. If your local DBngin instance is provisioned with a blank root password, instruct phpMyAdmin to allow empty credentials:
```php
$cfg['Servers'][$i]['AllowNoPassword'] = true;

```



### Step 4: Web Daemon Initialization

#### Option A: Lightweight PHP Native Runtime Server (Recommended)

If your development host already possesses a PHP binary path definition, you can host phpMyAdmin directly from your terminal:

```cmd
cd C:\DBTools\phpmyadmin
php -S 127.0.0.1:8080

```

Open a browser session at `http://127.0.0.1:8080`.

#### Option B: Isolated Container Architecture via Docker

If you do not want to manage local PHP runtime environments on Windows, you can mount an ephemeral network bridge from a Docker container to the Windows host socket layer:

```cmd
docker run --name dbngin-phpmyadmin -d -e PMA_HOST=host.docker.internal -e PMA_PORT=3307 -p 8080:80 phpmyadmin

```

*Note: `host.docker.internal` allows the isolated container environment to reach back out and map to DBngin’s port on the Windows host.*

### Step 5: Secure Access & Authentication

1. Once your chosen daemon is operational, point your browser to `http://localhost:8080`.
2. Input the primary superuser identity: `root`.
3. Leave the password field entirely **blank** (or enter the custom secret defined during the initial service provisioning within DBngin).

---

## 3. PART II: PostgreSQL Administration Lifecycle

### Architectural Note for DBAs

> **Warning:** phpMyAdmin relies strictly on the `mysql` and `mysqli` extensions. It does not contain structural drivers to handle the PostgreSQL wire protocol. To deploy a unified, web-accessible database administration workspace for PostgreSQL, administrators utilize **pgAdmin 4** or **Adminer**.

### Step 1: Engine Mapping

1. Open **DBngin** and confirm your PostgreSQL instance is running.
2. Note the designated port configuration (PostgreSQL defaults to `5432`, failing over to `5433` under port conflicts).
3. The default system role is universally set to `postgres` with no password constraint applied.

### Option A: Deploying pgAdmin 4 (Enterprise Web & Desktop Paradigm)

pgAdmin 4 serves as the flagship administrative toolset for managing PostgreSQL clusters.

1. Download the executable installer package from the [pgAdmin Windows Download Matrix](https://www.pgadmin.org/download/pgadmin-4-windows/).
2. Run through the installation process and start **pgAdmin 4**. The utility launches an embedded server engine that populates a interface directly into your primary web browser window.
3. Establish a Master Security Password to restrict unauthorized software access to local configurations.
4. Locate the **Browser** pane on the left menu, right-click the **Servers** node, and navigate to **Register** $\rightarrow$ **Server...**
5. Populate the parameters exactly as specified below:
* **General Tab:** `DBngin Local Postgres Instance`
* **Connection Tab:**
* **Host name/address:** `127.0.0.1`
* **Port:** *[Your recorded DBngin Postgres port, e.g., 5432]*
* **Maintenance database:** `postgres`
* **Username:** `postgres`
* **Password:** Leave empty/blank.




6. Commit the connection configurations by selecting **Save**.

### Option B: Deeply Integrated Desktop Workflow via TablePlus

Because DBngin and TablePlus share the same core engineering team, they are designed to hook into each other automatically.

1. Ensure **TablePlus** for Windows is present on your storage volume.
2. Open the **DBngin** controller pane.
3. Locate the active PostgreSQL service line item and click the **Arrow/Open** icon.
4. DBngin will immediately pass session parameters over to TablePlus, allowing you to bypass manual login setup entirely.

### Option C: Single-File Lightweight Web Console via Adminer

Adminer is an elegant web application that serves as a direct alternative to phpMyAdmin, bundled cleanly inside a single PHP file. It provides full structural visibility for PostgreSQL engine clusters.

1. Download the compiled script distribution asset `adminer-4.x.x.php` from the official repository.
2. Create an isolated application root folder: `C:\DBTools\adminer\`.
3. Drop the executable engine download file inside the target location and rename it explicitly to **`index.php`**.
4. Spin up your local runtime interpreter to host the application asset:
```cmd
cd C:\DBTools\adminer
php -S 127.0.0.1:8081

```


5. Open your web browser and navigate to `http://127.0.0.1:8081`.
6. Use the parameters listed below to establish a connection session through the login form:
| Administrative Property | Value Specification |
| --- | --- |
| **System** | Select `PostgreSQL` from the dropdown list |
| **Server** | `127.0.0.1:5432` *(Adjust to `5433` if matching your DBngin layout)* |
| **Username** | `postgres` |
| **Password** | *[Leave Blank / Empty Field]* |
| **Database** | `postgres` |



Alternatively, you can skip setting up PHP locally and run the Adminer web console via Docker, using this terminal command:

```cmd
docker run --name dbngin-postgres-adminer -d -p 8081:8080 adminer

```

Once the container starts up, navigate to `http://localhost:8081`. Set your **Server** destination value to `host.docker.internal:5432` to connect seamlessly through to your DBngin PostgreSQL instance.
