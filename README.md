# mosalam_erpnext
# Docker Compose Configuration for ERPNext

This document provides a detailed explanation of the Docker Compose configuration file for deploying an ERPNext system using Docker.

## Overview

The Compose file defines multiple services required to run the ERPNext system efficiently. It includes services for the backend, frontend, Redis caching, schedulers, workers, and Traefik reverse proxy.

---

## Services

### 1. **Backend**
Runs the ERPNext backend service.
- **Depends On**: 
  - `configurator`: Must complete successfully before starting.
- **Image**: `frappe/erpnext:${ERPNEXT_VERSION}`
- **Networks**:
  - `bench-network`
  - `mariadb-network`
- **Volumes**: 
  - `sites:/home/frappe/frappe-bench/sites`
- **Platform**: `linux/amd64`
- **Restart**: Always

---

### 2. **Configurator**
Initializes and configures the environment settings for ERPNext.
- **Entry Point**: 
  Executes a series of configuration commands.
- **Depends On**:
  - `redis-cache`: Waits for service to start.
  - `redis-queue`: Waits for service to start.
- **Environment Variables**:
  - `DB_HOST`
  - `DB_PORT`
  - `REDIS_CACHE`
  - `REDIS_QUEUE`
  - `SOCKETIO_PORT`
- **Image**: `frappe/erpnext:${ERPNEXT_VERSION}`
- **Networks**:
  - `bench-network`
  - `mariadb-network`
- **Volumes**:
  - `sites:/home/frappe/frappe-bench/sites`
- **Platform**: `linux/amd64`
- **Restart**: Always

---

### 3. **Frontend**
Handles the frontend service using an Nginx reverse proxy.
- **Depends On**:
  - `backend`: Must start before the frontend.
  - `websocket`: Must start before the frontend.
- **Environment Variables**:
  - `BACKEND`
  - `CLIENT_MAX_BODY_SIZE`
  - `FRAPPE_SITE_NAME_HEADER`
  - `SOCKETIO`
  - `UPSTREAM_REAL_IP_ADDRESS`
  - `UPSTREAM_REAL_IP_HEADER`
  - `UPSTREAM_REAL_IP_RECURSIVE`
- **Networks**:
  - `bench-network`
  - `traefik-public`
- **Labels**: 
  Traefik configurations for reverse proxying.
- **Platform**: `linux/amd64`
- **Restart**: Always

---

### 4. **Queue Workers**
Handles asynchronous task queues.

#### Long Queue Worker:
- **Command**: Processes `long`, `default`, and `short` queues.
- **Depends On**:
  - `configurator`: Must complete successfully.
- **Image**: `frappe/erpnext:${ERPNEXT_VERSION}`

#### Short Queue Worker:
- **Command**: Processes `short` and `default` queues.
- **Depends On**:
  - `configurator`: Must complete successfully.
- **Image**: `frappe/erpnext:${ERPNEXT_VERSION}`

---

### 5. **Redis Services**
Used for caching and queue management.

#### Redis Cache:
- **Image**: `redis:6.2-alpine`
- **Volumes**: `redis-cache-data:/data`

#### Redis Queue:
- **Image**: `redis:6.2-alpine`
- **Volumes**: `redis-queue-data:/data`

---

### 6. **Scheduler**
Runs scheduled tasks.
- **Command**: `bench schedule`
- **Depends On**:
  - `configurator`: Must complete successfully.
- **Image**: `frappe/erpnext:${ERPNEXT_VERSION}`
- **Restart**: Always

---

### 7. **WebSocket**
Handles real-time communication for ERPNext.
- **Command**: Executes the Socket.IO server.
- **Depends On**:
  - `configurator`: Must complete successfully.
- **Image**: `frappe/erpnext:${ERPNEXT_VERSION}`
- **Restart**: Always

---

## Networks

### 1. `bench-network`
Used for internal communication between services.
- **Driver Options**: `mtu=${NETWORK_MTU:-1400}`

### 2. `mariadb-network`
Connects services to the MariaDB database.
- **Driver Options**: `mtu=${NETWORK_MTU:-1400}`
- **External**: True

### 3. `traefik-public`
Traefik reverse proxy network.
- **Driver Options**: `mtu=${NETWORK_MTU:-1400}`
- **External**: True

---

## Volumes

### 1. `redis-cache-data`
Stores Redis cache data.

### 2. `redis-queue-data`
Stores Redis queue data.

### 3. `sites`
Stores site data for the ERPNext system.

---

## Additional Configurations

### **X-Backend-Defaults**
Default settings for backend services:
- `depends_on`: Configurator
- `image`: `frappe/erpnext:${ERPNEXT_VERSION}`
- `restart`: Always

### **X-Customizable-Image**
Reusable image configuration:
- `image`: `frappe/erpnext:${ERPNEXT_VERSION}`
- `restart`: Always

### **X-Depends-On-Configurator**
Template to ensure services depend on the successful completion of the configurator.

---

## Environment Variables and Their Descriptions

### Required Variables
These are essential for running the ERPNext system.

1. **`ERPNEXT_VERSION`**
   - Description: Specifies the version of ERPNext to use in the Docker images.
   - Example: `v15.44.0`

2. **`DB_PASSWORD`**
   - Description: The password for the database user. This is used to connect to the external MariaDB or other databases.
   - Example: `015369834aA@dbPass@`

3. **`DB_HOST`**
   - Description: The hostname or IP address of the external database. This is required only if you are using an external MariaDB database.
   - Example: `mariadb-database`

4. **`DB_PORT`**
   - Description: The port on which the external database listens. The default MariaDB port is `3306`.
   - Example: `3306`

5. **`REDIS_CACHE`**
   - Description: The hostname or IP address of the external Redis instance for caching. Leave blank if not using an external Redis instance.
   - Example: `<empty>`

6. **`REDIS_QUEUE`**
   - Description: The hostname or IP address of the external Redis instance for queue management. Leave blank if not using an external Redis instance.
   - Example: `<empty>`

7. **`LETSENCRYPT_EMAIL`**
   - Description: The email address used to register with Let's Encrypt for generating HTTPS certificates. Required for HTTPS override.
   - Example: `mail@example.com`

---

### Optional Variables
These are not mandatory but can be customized for specific setups.

1. **`FRAPPE_SITE_NAME_HEADER`**
   - Description: Overrides the default site resolution behavior. Use this to specify the site's name explicitly when accessing it via a non-standard hostname.
   - Example: `<empty>`

2. **`HTTP_PUBLISH_PORT`**
   - Description: Specifies the HTTP port to publish the backend service. The default value is `8080`.
   - Example: `<empty>`

3. **`UPSTREAM_REAL_IP_ADDRESS`**
   - Description: The trusted upstream IP address. The default value is `127.0.0.1`.
   - Example: `<empty>`

4. **`UPSTREAM_REAL_IP_HEADER`**
   - Description: The request header field whose value will replace the client address. The default is `X-Forwarded-For`.
   - Example: `<empty>`

5. **`UPSTREAM_REAL_IP_RECURSIVE`**
   - Description: Controls whether to enable recursive IP address replacement. Allowed values: `on`, `off`. The default value is `off`.
   - Example: `<empty>`

6. **`PROXY_READ_TIMEOUT`**
   - Description: Sets the maximum time for reading client requests through the proxy. Default value: `120s`. Increase this if handling long-running processes.
   - Example: `<empty>`

7. **`CLIENT_MAX_BODY_SIZE`**
   - Description: Specifies the maximum size of a request body. Default value: `50m`. Adjust this if you need to upload larger files.
   - Example: `<empty>`

8. **`SITES`**
   - Description: Lists the sites for generating Let's Encrypt certificates. Use backticks (`) and separate multiple entries with commas.
   - Example: `` `erp.a3elank.com` ``

9. **`ROUTER`**
   - Description: Sets the Traefik router name. This is used in Traefik labels for routing traffic.
   - Example: `erpnext-a3elank`

10. **`BENCH_NETWORK`**
    - Description: Defines the name of the Docker network connecting ERPNext services.
    - Example: `erpnext-a3elank`


---

## Usage Instructions

1. Clone the repository containing this Compose file.
2. Create a `.env` file with the required environment variables.
3. Start the services:
   ```bash
   docker-compose up -d

