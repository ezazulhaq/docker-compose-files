# Self-Hosted Supabase: Comprehensive Multi-Project & SSL Guide (Dokploy & Oracle Cloud)

Self-hosted Supabase (unlike Supabase Cloud) is designed with a 1-to-1 architecture: one deployment equals one project, and the Studio dashboard can only manage a single database instance at a time. 

If you want to host multiple applications on the same server to save memory and CPU, you have two choices: deploy a completely new Supabase stack (heavy), or logically separate them using **PostgreSQL schemas** (efficient). 

This guide covers the entire journey of creating an isolated logical project (schema), securing it, exposing it to the internet through Oracle Cloud's strict firewalls, giving it custom subdomains in Dokploy, and finally, encrypting the database connection with SSL.

---

## 1. Creating an Isolated "Project" (Schemas & Roles)
We will create a new schema and a dedicated user. This ensures that an application using this database user cannot read or write data belonging to your other applications (like `public` or `project_b`).

**Run this script in your Supabase SQL Editor:**
```sql
-- 1. Create the new schema (your new "project")
CREATE SCHEMA new_schema;

-- 2. Grant PostgREST roles access so the Supabase REST API works
-- This is strictly required if you intend to use the supabase-js client to query this schema.
GRANT USAGE ON SCHEMA new_schema TO anon, authenticated, service_role;
GRANT ALL ON ALL TABLES IN SCHEMA new_schema TO anon, authenticated, service_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA new_schema GRANT ALL ON TABLES TO anon, authenticated, service_role;
ALTER DEFAULT PRIVILEGES IN SCHEMA new_schema GRANT ALL ON SEQUENCES TO anon, authenticated, service_role;

-- 3. Create a dedicated Database User with a password
CREATE ROLE new_username WITH LOGIN PASSWORD 'your_new_password';

-- 4. Set the default schema for the user
-- By setting the search_path, any time this user connects, they will automatically query the 'new_schema' schema without needing to prefix their tables (e.g., SELECT * FROM users).
ALTER ROLE new_username SET search_path TO new_schema, public, extensions;

-- 5. Grant the new user full access to their specific schema
GRANT CONNECT ON DATABASE postgres TO new_username;
GRANT ALL ON SCHEMA new_schema TO new_username;
GRANT ALL ON ALL TABLES IN SCHEMA new_schema TO new_username;
GRANT ALL ON ALL SEQUENCES IN SCHEMA new_schema TO new_username;
ALTER DEFAULT PRIVILEGES IN SCHEMA new_schema GRANT ALL ON TABLES TO new_username;
ALTER DEFAULT PRIVILEGES IN SCHEMA new_schema GRANT ALL ON SEQUENCES TO new_username;

-- 6. Lock down access to other schemas
-- Prevent this user from accidentally touching the default public schema or other apps.
REVOKE CREATE ON SCHEMA public FROM new_username;
```

---

## 2. Setting up Web Subdomains in Dokploy (Studio & API)
Before we expose the raw database port, you likely want to access your Supabase Studio and API via clean subdomains (like `studio.yourdomain.com`). Because these are HTTP/HTTPS services, Dokploy can handle this entirely through its UI using its built-in Traefik router.

1. In **Hostinger DNS**, create `A Records` for your subdomains pointing to your VPS IP.
2. In **Dokploy**, go to your Supabase Application and click the **Domains** tab.
3. Click **Add Domain** and map them:
   * **For the Supabase Dashboard (UI):** Add `studio.yourdomain.com` -> Select the `studio` service -> Port `3000`.
   * **For the Supabase API Gateway:** Add `api.yourdomain.com` -> Select the `kong` service -> Port `8000`.
4. Save. Dokploy will automatically generate a free Let's Encrypt SSL certificate for these web interfaces.

---

## 3. Exposing the Raw Database Port
Unlike the Studio or API, the database itself uses a raw TCP protocol. Traefik (Dokploy's router) handles HTTP/HTTPS web traffic, so we must expose the database port directly to the host network instead of using the Dokploy "Domains" tab.

### The Port Conflict Problem
By default, Postgres uses port `5432`. If your server already has a native Postgres installation or another Dokploy database running, Docker will throw a `Bind for 0.0.0.0:5432 failed: port is already allocated` error.

**The Fix:** Map a custom external port (e.g., `5431`) to the internal container port (`5432`).

**In Dokploy > Supabase Application > docker-compose.yml:**
```yaml
services:
  db:
    # ... other configurations ...
    ports:
      # Format is "HostPort:ContainerPort"
      - "5431:5432" 
```

---

## 4. Oracle Cloud Networking & DNS (The Double Firewall)
Oracle Cloud Infrastructure (OCI) has notoriously strict networking. You must pierce **two** separate firewalls to allow external traffic to hit port `5431`. If you only do one, your connection tests will hang and say `Operation timed out`.

### Layer 1: Oracle Cloud Web Console (Security Lists)
1. Go to **Networking** -> **Virtual Cloud Networks (VCN)**.
2. Select your VCN -> **Public Subnet** -> **Default Security List**.
3. Add an Ingress Rule: **TCP**, Destination Port **5431**, Source **0.0.0.0/0** (Allow all).

### Layer 2: Oracle OS Firewall (iptables)
Oracle's official Ubuntu images come with a heavily locked-down internal firewall. Appending rules to the end of the chain fails because Oracle puts a "block everything" rule at the end. You must *insert* the rule before the block.
SSH into your server and run:
```bash
# Insert rule at position 6 (before the final REJECT rule)
sudo iptables -I INPUT 6 -m state --state NEW -p tcp --dport 5431 -j ACCEPT
# Save rules to survive reboots
sudo netfilter-persistent save
```

### DNS Setup
Create an **A Record** in Hostinger for `db.yourdomain.com` pointing to your Oracle VPS Public IP. 
*(Warning: Ensure it is "DNS Only". If it is proxied via a CDN like Cloudflare's orange cloud, the CDN will block port 5431).*

---

## 5. Enabling SSL for PostgreSQL
To secure your database credentials in transit, you must encrypt the connection. Because raw TCP bypasses Dokploy's Traefik SSL generation, we must manually give the Postgres container certificates.

### Step 5a: Generate Self-Signed Certificates
SSH into your server and run:
```bash
sudo mkdir -p /opt/supabase/ssl

# Generate a 10-year self-signed certificate
sudo openssl req -new -x509 -days 3650 -nodes -text \
  -out /opt/supabase/ssl/server.crt \
  -keyout /opt/supabase/ssl/server.key \
  -subj "/CN=db.yourdomain.com"
```

### Step 5b: The UID Permissions Fix (Crucial)
PostgreSQL is incredibly strict about security. If `server.key` is owned by the wrong user, it crashes on startup with the error: `FATAL: private key file must be owned by the database user or root`.
Because custom images (like `supabase/postgres`) constantly change their internal User ID (e.g., UID `105` vs standard `999`), guessing the UID is error-prone.

Instead, force Docker to look up the exact internal user and fix the host files automatically:
```bash
# Replace 'supabase/postgres:15.1.1.0-84' with your exact image tag from Dokploy
sudo docker run --rm -u root -v /opt/supabase/ssl:/ssl --entrypoint /bin/sh supabase/postgres:15.1.1.0-84 -c "chown postgres:postgres /ssl/server.key /ssl/server.crt && chmod 0600 /ssl/server.key"
```

### Step 5c: Update Dokploy Compose for SSL
Go back to your `docker-compose.yml` in Dokploy and update the `db` service. You must mount the files as volumes and append the SSL flags to the existing `command` array:

```yaml
services:
  db:
    # ...
    volumes:
      - /opt/supabase/ssl/server.crt:/var/lib/postgresql/server.crt:ro
      - /opt/supabase/ssl/server.key:/var/lib/postgresql/server.key:ro
    command:
      [
        "postgres",
        "-c",
        "config_file=/etc/postgresql/postgresql.conf",
        "-c",
        "log_min_messages=fatal",
        # Added SSL Commands below:
        "-c",
        "ssl=on",
        "-c",
        "ssl_cert_file=/var/lib/postgresql/server.crt",
        "-c",
        "ssl_key_file=/var/lib/postgresql/server.key"
      ]
```
Redeploy the application in Dokploy.

---

## 6. Final Connection String
Your database is now fully operational, isolated, and secured!

You can connect using the following JDBC URL:
```text
jdbc:postgresql://db.yourdomain.com:5431/postgres?user=new_username&password=auditagent_devDB01&currentSchema=new_schema&sslmode=require
```

**Understanding the parameters:**
* `5431`: Our custom exposed port to avoid conflicts.
* `currentSchema=new_schema`: Explicitly tells the driver to query your new project schema.
* `sslmode=require`: Forces the driver to encrypt the traffic using the self-signed certificates we generated, keeping your data completely safe from network sniffers.
