# Open WebUI access to Red Hat Offline Knowledge Portal (RHOKP) via okp-mcp

## Step 1: Install RHOKP and use the access key

### 1. Get the container image

You need Red Hat registry credentials: a Customer Portal login, a Red Hat Developer login, or a registry service account.

```bash
podman login registry.redhat.io
podman pull registry.redhat.io/offline-knowledge-portal/rhokp-rhel9:latest
```

_The image is large, so I pulled it ahead of time._

### 2. Generate your access key

*   Open the **Offline Knowledge Portal Access Key Generator** at [https://access.redhat.com/offline/access](https://access.redhat.com/offline/access) and click **Generate key**. _(screenshots)_
    
*   A **Red Hat Satellite subscription is needed** to generate the key, but you don't need to install Satellite.
    
*   The key is stored in your Red Hat account. Click **Generate key** again to see it.
    
*   The access key unlocks the knowledgebase content and the search index. Without it, you can still browse documentation, lifecycle pages and the Security Data API, but not search or KB articles.
    
*   Save it to a file named `rhokp_access_key` in `ACCESS_KEY=<key>` format, and restrict access to it:
    

```bash
echo 'ACCESS_KEY=<paste-key-here>' > rhokp_access_key
chmod 600 rhokp_access_key
```

The key and content are licensed for internal use only under the EULA, so don't share them publicly.

### 3. Start the container

```bash
podman run --rm -d \
  -p 8003:8080 \
  -p 8443:8443 \
  --add-host=host.docker.internal:host-gateway \
  --name rhokp \
  --env-file rhokp_access_key \
  registry.redhat.io/offline-knowledge-portal/rhokp-rhel9:latest
```

| Option | Purpose |
| --- | --- |
| `--rm -d` | Run in the background and remove the container when it stops |
| `-p 8003:8080` | HTTP on host port 8003 (the container's default is 8080) |
| `-p 8443:8443` | HTTPS using a self-signed certificate |
| `--add-host=host.docker.internal:host-gateway` | Lets the container reach the host by that name |
| `--name rhokp` | A friendly name, so you can run `podman logs -f rhokp` |
| `--env-file rhokp_access_key` | Passes the key without putting it on the command line or in shell history |

### 4. Verify

Browse to `http://<host>:8003` and try a search. _(screenshot)_
**Resources:** Minimum 1 core, 1 GB RAM and 50 GB disk. Recommended 2 cores, 2 GB RAM and 75 GB disk.

* * *

## Presenter notes for Step 1 (not for the slide)

*   The `--env-file` must contain `ACCESS_KEY=<key>`. A file holding only the raw key won't set the variable, and the portal starts with limited functionality (no search or KB content).
    
*   `--env-file` keeps the key out of `ps` output and shell history. The Red Hat docs use `--env "ACCESS_KEY=..."` instead.
    
*   Without a volume, `--rm` discards the index when the container stops, so the next start re-downloads and re-indexes (several minutes). Step 2 adds a named volume to fix this.
    
*   `--add-host=host.docker.internal:host-gateway` isn't in the Red Hat docs, and the later steps don't use it (Step 3 uses `host.containers.internal`, which podman provides automatically). Drop it from the slide unless you need it for something else.
    
*   The Red Hat docs publish `8080:8080`. The `8003` mapping is a local choice.
    
*   Blur or crop the access key field in screenshots if the key is ever visible.
    
*   This standalone container is the "simple" version. Step 2 replaces it with a pod that also runs okp-mcp. Stop this one first (`podman stop rhokp`), because the container name `rhokp` and host port `8003` would conflict.
    

* * *

## Step 2: Get, build and start okp-mcp

### 1. Get and build the okp-mcp image

```bash
git clone https://github.com/rhel-lightspeed/okp-mcp.git
cd okp-mcp/
podman build -t localhost/okp-mcp:latest -f Containerfile .
```

_The build takes a while, so I built it ahead of time._

### 2. Create a pod

```bash
podman pod create --name okp -p 8003:8080 -p 8000:8000
```

*   A pod is a group of containers that share one network namespace, so they reach each other on `127.0.0.1`.
    
*   Ports are published at the **pod** level, not per container:
    *   `8003:8080` is the RHOKP web UI.
        
    *   `8000:8000` is the okp-mcp endpoint (`/mcp`).
        
*   Solr's port **8983 is deliberately not published**. Only okp-mcp, inside the pod, can reach it.
    

### 3. Start RHOKP in the pod

```bash
podman run --rm -d --pod okp \
  --name rhokp \
  --env-file rhokp_access_key \
  -e SOLR_JETTY_HOST=0.0.0.0 \
  -v okp-solr-data:/opt/solr/server/solr/portal/data \
  registry.redhat.io/offline-knowledge-portal/rhokp-rhel9:latest
```

| Option | Purpose |
| --- | --- |
| `--pod okp` | Join the pod, so no `-p` is needed here |
| `--env-file rhokp_access_key` | Same access key file as Step 1 |
| `-e SOLR_JETTY_HOST=0.0.0.0` | Makes Solr listen on all interfaces inside the pod, so okp-mcp can reach it |
| `-v okp-solr-data:...` | Named volume that keeps the search index across restarts |

### 4. Start okp-mcp in the pod

```bash
podman run --rm -d --pod okp \
  --name okp-mcp \
  -e MCP_TRANSPORT=streamable-http \
  -e MCP_SOLR_URL=http://127.0.0.1:8983 \
  localhost/okp-mcp:latest
```

| Option | Purpose |
| --- | --- |
| `MCP_TRANSPORT=streamable-http` | The transport MCP clients (including Copilot Studio) use over HTTP |
| `MCP_SOLR_URL=http://127.0.0.1:8983` | Points okp-mcp at the Solr index in the same pod |

### 5. Verify

Watch RHOKP start up, and wait for the Solr message:

```bash
podman logs -f rhokp
# wait for: Started Solr server on port 8983
```

Check that the MCP server responds. This is the check from the okp-mcp README, and the `-4` forces IPv4:

```bash
curl -s -4 -N -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc": "2.0", "method": "initialize", "params": {"protocolVersion": "2025-11-25", "capabilities": {}, "clientInfo": {"name": "test", "version": "1.0"}}, "id": 1}'
```

Expected: a response containing `serverInfo.name: "RHEL OKP Knowledge Base"`.  
Browser check: `http://<host>:8003`.

* * *

## Presenter notes for Step 2 (not for the slide)

*   Solr isn't published on the host. That's good design: the search index is only reachable from inside the pod. It also means `curl localhost:8983` from the host won't work. Test through the MCP endpoint on 8000 instead.
    
*   `MCP_TRANSPORT` and `MCP_SOLR_URL` are close to the okp-mcp defaults (`streamable-http` and `http://localhost:8983`). Setting them explicitly makes the demo clearer. Using `127.0.0.1` also avoids the IPv6 `localhost` (`::1`) problem the README mentions.
    
*   The `okp-solr-data` volume survives `podman pod rm -f okp`. Without it, removing the pod deletes the index and the next start has to re-index.
    
*   Port 8443 (HTTPS) isn't published in this layout. Add `-p 8443:8443` to `podman pod create` if you want it.
    
*   Port 8000 is published on all host interfaces and okp-mcp has no authentication. That's fine for a demo. For production, put a gateway or reverse proxy in front (the bank/Copilot Studio discussion).
    
*   The okp-mcp README lists a prebuilt image, `quay.io/redhat-user-workloads/rhel-lightspeed-tenant/rhel-knowledge-bridge`, which skips the build. It also describes systemd quadlets for restart and boot persistence.
    
*   Open WebUI and MCPO run in a separate pod, so they reach okp-mcp through the host-published port 8000. Step 3 shows how.
    

* * *

## Step 3: Connect Open WebUI to okp-mcp through MCPO

### What MCPO does

MCPO ([https://github.com/open-webui/mcpo](https://github.com/open-webui/mcpo)) is a proxy that exposes MCP servers as OpenAPI/REST tool servers, which Open WebUI can use. Each entry in the config becomes its own route.

```
Open WebUI  --(OpenAPI)-->  MCPO (:8001/okp)  --(Streamable MCP)-->  okp-mcp (:8000/mcp)
```

### Part A: Add okp-mcp to the MCPO config

Add this entry to the `mcpServers` section of the MCPO config on the Open WebUI instance:

```json
    "okp": {
      "type": "streamable-http",
      "url": "http://host.containers.internal:8000/mcp"
    }
```

| Field | Purpose |
| --- | --- |
| `"okp"` | The server name. MCPO exposes it at the route `/okp` |
| `"type": "streamable-http"` | Matches the transport okp-mcp is running (`MCP_TRANSPORT=streamable-http`) |
| `"url"` | The okp-mcp endpoint: host-published port 8000, path `/mcp` |

`host.containers.internal` is a special hostname that podman sets up inside containers. It points back to the host, and that's how the Open WebUI/MCPO pod reaches the port published by the `okp` pod.
Restart MCPO to pick up the change:

```bash
podman restart <mcpo-container-name>
```

_MCPO reads its config at startup, so a restart is required after any change._
Check that MCPO is serving the okp tools:

```bash
curl -s http://localhost:8001/okp/openapi.json
```

### Part B: Enable access in Open WebUI

Create a new connection under **Admin Panel / Settings / Integrations**. _(screenshot: Edit Connection)_
| Field | Value |
| --- | --- |
| **Type** | `OpenAPI` |
| **Name** | `mcpo-okp` |
| **ID** (optional) | `mcpo-okp` |
| **Description** | `mcpo-okp` |
| **URL** | `http://localhost:8001/okp` |
| **Auth** | `Bearer`, with the API Key field left blank (see the notes below) |
| **Function Name Filter List** | Left empty |

Then:

1.  Use the refresh icon next to the URL to test the connection.
    
2.  Make sure the toggle next to it is **on** (enabled).
    
3.  Click **Save**.
    

Once saved, the okp tools are available to use in chats.

* * *

## Presenter notes for Step 3 (not for the slide)

*   **Correction to my earlier guess:** `host.containers.internal` is what makes the cross-pod connection work. It's built in to podman, so no `--add-host` flag is needed. The `host.docker.internal` flag from Step 1 isn't what's doing this job.
    
*   The full path is: Open WebUI, to MCPO, to the host's port 8000, to the okp pod, to okp-mcp. If this fails, check that the `okp` pod is running and that port 8000 is published.
    
*   The Open WebUI URL is `localhost:8001`, while MCPO's config uses `host.containers.internal:8000`. These are different hops. Open WebUI reaches MCPO on its own local network (probably the same pod, since `localhost` works). MCPO then goes out through the host to the `okp` pod.
    
*   The Auth field is set to Bearer with no key. That works if MCPO isn't started with an API key. If MCPO uses one (`--api-key`), enter it there. There's also an option for no auth, which may be cleaner for the demo.
    
*   The Access Control button controls which Open WebUI users or groups can use this tool server. Check it for your demo account.
    
*   Naming: MCPO's route `/okp` and the connection name `mcpo-okp` are independent. The name is just a label in Open WebUI, and the URL is what matters.
    
*   MCPO is only needed for Open WebUI, which wants OpenAPI tool servers. Copilot Studio can talk Streamable MCP to okp-mcp directly, with no MCPO layer in between. That's the bridge to the bank discussion.
    
*   Restarting MCPO affects every MCP server behind it. Do it before the demo, not during.
    
*   Neither the MCPO entry nor okp-mcp has authentication. That's fine for a demo, but not for production.
    

* * *

