# Guide: Frigate, Home Assistant, and MQTT with Docker Compose

This guide is designed to set up a complete stack with Frigate, Home Assistant, and a secure Mosquitto (MQTT) broker using Docker Compose.

This guide proactively addresses common pitfalls:
* **Permission Issues:** Prevents "read-only" files by creating the directory structure with the correct user permissions beforehand.
* **Crash Loops:** Avoids the Mosquitto "chicken-and-egg" problem by using a two-stage "bootstrap" configuration.
* **Version Mismatch:** Includes the step to update Frigate to v0.16+ to be compatible with the HACS integration.
* **Integration:** Explains how to install HACS and the correct Frigate integration (not the built-in one).

---

### Step 1: Create the `.env` File

In your Docker directory, create a file named `.env`. This file will store all your secrets.

**You must change the placeholder passwords** to your own strong, unique passwords.

    # .env
    # Passwords and users for the Docker stack
    
    # Frigate password (single quotes are important)
    FRIGATE_RTSP_PASSWORD='your_frigate_rtsp_password'
    
    # MQTT Users
    FRIGATE_MQTT_USER="frigate_user"
    FRIGATE_MQTT_PASSWORD="YOUR_STRONG_FRIGATE_MQTT_PASSWORD"
    
    HA_MQTT_USER="ha_user"
    HA_MQTT_PASSWORD="YOUR_STRONG_HA_MQTT_PASSWORD"

---

### Step 2: Directory Structure and Permissions (Crucial!)

Before starting Docker, create all necessary directories. This ensures you are the owner of the files, preventing "read-only" permission problems.

    # Set your base docker directory
    DOCKER_BASE="/path/to/your/docker"
    
    # Create all necessary directories
    mkdir -p $DOCKER_BASE/homeassistant
    mkdir -p $DOCKER_BASE/mosquitto/config
    mkdir -p $DOCKER_BASE/mosquitto/data
    mkdir -p $DOCKER_BASE/mosquitto/log
    
    # (You likely already have Frigate's config dirs, like $DOCKER_BASE/config)

---

### Step 3: Mosquitto Initial Config (The "Bootstrap")

To prevent the "crash loop," we will first start Mosquitto *without* requiring a password.

Create the following configuration file. **This is a minimal, temporary config.** Do **not** add the `password_file` line yet, as this will cause the container to crash.

**File:** `/path/to/your/docker/mosquitto/config/mosquitto.conf`

    # INITIAL CONFIGURATION (TEMPORARY)
    # This file MUST NOT contain the 'password_file' line yet.
    
    persistence true
    persistence_location /mosquitto/data/
    log_dest file /mosquitto/log/mosquitto.log
    
    # Temporary setting to allow the container to start:
    allow_anonymous true

---

### Step 4: The `docker-compose.yml` File

Now, create the `docker-compose.yml` file in your base Docker directory. This file will read the variables from your `.env` file.

**Note:** You must update all volume paths (e.g., `/path/to/your/docker/...`) to match your system.

    services:
      frigate:
        container_name: frigate
        privileged: true
        restart: unless-stopped
        stop_grace_period: 30s
        image: ghcr.io/blakeblackshear/frigate:stable
        shm_size: "512mb"
        devices:
          - /dev/bus/usb:/dev/bus/usb
          - /dev/apex_0:/dev/apex_0
          - /dev/video11:/dev/video11
          - /dev/dri/renderD128:/dev/dri/renderD128
        volumes:
          - /etc/localtime:/etc/localtime:ro
          # This path MUST match where your config.yml is
          - /path/to/your/docker/config:/config
          - /path/to/your/docker/storage:/media/frigate
          - type: tmpfs
            target: /tmp/cache
            tmpfs:
              size: 1000000000
        ports:
          - "8971:8971"
          - "8554:8554"
          - "8555:8555/tcp"
          - "8555:8555/udp"
        environment:
          FRIGATE_RTSP_PASSWORD: ${FRIGATE_RTSP_PASSWORD}
          FRIGATE_MQTT_ENABLED: "True"
          FRIGATE_MQTT_HOST: "mqtt"
          FRIGATE_MQTT_PORT: "1883"
          # Passwords in YAML with quotes
          FRIGATE_MQTT_USER: '${FRIGATE_MQTT_USER}'
          FRIGATE_MQTT_PASSWORD: '${FRIGATE_MQTT_PASSWORD}'
        networks:
          - pi-services
        depends_on:
          - mqtt
    
      homeassistant:
        container_name: homeassistant
        image: "ghcr.io/home-assistant/home-assistant:stable"
        volumes:
          - /path/to/your/docker/homeassistant:/config
          - /etc/localtime:/etc/localtime:ro
        restart: unless-stopped
        privileged: true
        network_mode: host
        depends_on:
          - mqtt
    
      mqtt:
        container_name: mqtt
        image: eclipse-mosquitto:latest
        restart: unless-stopped
        ports:
          - "1883:1883"
          - "9001:9001"
        volumes:
          - /path/to/your/docker/mosquitto/config:/mosquitto/config
          - /path/to/your/docker/mosquitto/data:/mosquitto/data
          - /path/to/your/docker/mosquitto/log:/mosquitto/log
        networks:
          - pi-services
    
    networks:
      pi-services:
        driver: bridge

---

### Step 5: First Start and User Creation

We will use `docker compose` (without the hyphen).

1.  **Start MQTT only:**
    Because the config (from Step 3) allows anonymous users, the container will now start correctly.

        docker compose up -d mqtt

2.  **Load the `.env` variables:**
    Run this command in your terminal. This makes the passwords available as variables (e.g., `$HA_MQTT_PASSWORD`).

        source .env

3.  **Create the MQTT users:**
    This method ensures that **all special characters** are processed correctly.
    **Note the `-c` flag on the FIRST command** to create the file.

        # Create the Home Assistant user (-c creates the file)
        docker exec -it mqtt mosquitto_passwd -b -c /mosquitto/config/passwordfile "${HA_MQTT_USER}" "${HA_MQTT_PASSWORD}"
        
        # Create the Frigate user (no -c, appends to the file)
        docker exec -it mqtt mosquitto_passwd -b /mosquitto/config/passwordfile "${FRIGATE_MQTT_USER}" "${FRIGATE_MQTT_PASSWORD}"

---

### Step 6: Secure Mosquitto (The "Lockdown")

Now that the password file exists, we can disable anonymous access.

1.  **Edit `mosquitto.conf`:**
    Open `/path/to/your/docker/mosquitto/config/mosquitto.conf` and **replace** its *entire contents* with this **final, secure** configuration.

        # FINAL CONFIGURATION (SECURE)
        
        persistence true
        persistence_location /mosquitto/data/
        log_dest file /mosquitto/log/mosquitto.log
        
        # --- Authentication (SECURE) ---
        # We disable anonymous access
        allow_anonymous false
        
        # And we NOW add the password_file line
        password_file /mosquitto/config/passwordfile
        # ---------------------------------
        
        # Listeners
        listener 1883
        listener 9001
        protocol websockets

2.  **Restart Mosquitto:**
    The container will now restart, read the secure config, and use the `passwordfile` (which now exists).

        docker compose restart mqtt

---

### Step 7: Update Frigate Container (Important!)

The latest Home Assistant Frigate integration (from HACS) requires Frigate version 0.16 or newer. We must force an update to prevent `400 Bad Request` errors.

1.  **Stop all containers:**

        docker compose down

2.  **Pull the latest "stable" Frigate image:**

        docker pull ghcr.io/blakeblackshear/frigate:stable

3.  **Start all containers again:**
    This will re-create Frigate using the new `0.16+` image.

        docker compose up -d

---

### Step 8: Configure Frigate (config.yml)

Frigate needs to be told to enable MQTT in its own `config.yml` file.

1.  **Edit your Frigate `config.yml` file.**
    This is located at the path you mapped as a volume, e.g., `/path/to/your/docker/config/config.yml`.

2.  **Add the following `mqtt:` block** to the file. This tells Frigate to enable MQTT and to get the details from the environment variables you defined in `docker-compose.yml`.

        mqtt:
          enabled: True
          host: mqtt
          port: 1883
          user: "{FRIGATE_MQTT_USER}"
          password: "{FRIGATE_MQTT_PASSWORD}"

3.  **Save the `config.yml` file.**

4.  **Restart Frigate** to apply the new config:

        docker compose restart frigate

---

### Step 9: Install HACS

The official-looking "Frigate" integration in Home Assistant is only for the add-on. For a Docker setup, we must use the custom integration from HACS.

1.  If you do not have HACS, follow the official guide to install it:
    [https://hacs.xyz/docs/installation/container](https://hacs.xyz/docs/installation/container)

---

### Step 10: Install the HACS Frigate Integration

1.  In your Home Assistant sidebar, go to **HACS**.
2.  Go to **Integrations**.
3.  Click **+ EXPLORE & ADD REPOSITORIES**.
4.  Search for **"Frigate"** and install it.
5.  **Restart Home Assistant** (Go to Settings > System > Restart, or run `docker compose restart homeassistant`).

---

### Step 11: Configure the Frigate Integration

1.  After Home Assistant restarts, go to **Settings > Devices & Services**.
2.  Click **+ ADD INTEGRATION**.
3.  Search for and select **Frigate**.
4.  You will be prompted for your Frigate URL and credentials.

* **URL:** **`https://[YOUR_SERVER_IP]:8971`** (e.g., `https://mail:8971` or `https://192.168.178.118:8971`).
* **Validate SSL:** **Uncheck this box** (because Frigate uses a self-signed certificate).
* **Username:** The user you created in the Frigate Web UI.
* **Password:** The password you created in the Frigate Web UI.

5.  Click **Submit**. The integration should now connect and add all your cameras and devices.

---

### Step 12: How to Update Home Assistant

You cannot update from the HA web interface. You must update it from the command line.

1.  **Pull the new image:**

        docker compose pull homeassistant

2.  **Re-create the container:**

        docker compose up -d homeassistant

3.  **(Optional) Clean up old images:**

        docker image prune
