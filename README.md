# Guide: Frigate, Home Assistant, and MQTT with Docker Compose

This guide is designed to set up a complete stack with Frigate, Home Assistant, and a secure Mosquitto (MQTT) broker using Docker Compose.

This guide proactively addresses common pitfalls:
* **Permission Issues:** Prevents "read-only" files by creating the directory structure with the correct user permissions beforehand.
* **Crash Loops:** Avoids the Mosquitto "chicken-and-egg" problem by using a two-stage "bootstrap" configuration.
* **Special Characters:** Uses `.env` variables in a way that safely handles special characters in passwords.

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
    This method (using variables with double quotes) ensures that **all special characters** are processed correctly.

        # Create the Home Assistant user
        docker exec -it mqtt mosquitto_passwd -b -c /mosquitto/config/passwordfile "${HA_MQTT_USER}" "${HA_MQTT_PASSWORD}"
        
        # Create the Frigate user
        docker exec -it mqtt mosquitto_passwd -b /mosquitto/config/passwordfile "${FRIGATE_MQTT_USER}" "${FRIGATE_MQTT_PASSWORD}"

    The file `/mosquitto/config/passwordfile` has now been created *inside* the container (and on your host).

---

### Step 6: Secure Mosquitto (The "Lockdown")

Now that the password file exists, we can disable anonymous access.

1.  **Using sudo Edit`mosquitto.conf`:**
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
