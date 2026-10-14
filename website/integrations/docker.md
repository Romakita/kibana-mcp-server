---
description: "A Docker image and compose file that run the server over HTTP, protected by OpenID Connect, with catalogs in PostgreSQL."
---

# Docker image

This example builds an image that runs the server over HTTP, with catalogs in PostgreSQL. Users authenticate with your OpenID Connect provider, so no gateway is needed. Swap the store for any other one by changing two lines.

::: warning Version
The HTTP transport, the configuration file and the catalog stores are not part of the releases published before this documentation. Use a version of the package that includes them.
:::

## Files

```
kibana-mcp/
  Dockerfile
  kibana.mcp.config.mjs
  compose.yaml
```

### `kibana.mcp.config.mjs`

```js
import { defineKibanaMcpConfig } from "@havesomecode/kibana-mcp-server/config";
import store from "@havesomecode/kibana-mcp-server/stores/postgres";

export default defineKibanaMcpConfig({
  url: process.env.KIBANA_URL,
  user: process.env.KIBANA_USER,
  password: process.env.KIBANA_PASSWORD,
  store,
  storeOptions: { url: process.env.DATABASE_URL },
  http: {
    host: "0.0.0.0",
    port: 3000,
    oidc: {
      issuer: process.env.OIDC_ISSUER,
      resource: process.env.MCP_PUBLIC_URL,
      requiredScopes: ["kibana-mcp"],
    },
  },
});
```

Each user gets the catalog named by the `sub` claim of their token. See [OpenID Connect authentication](../deployment/oidc) for the other options and the provider setup.

A `.mjs` file loads on every supported Node.js version. A `.ts` file works too from Node.js 22.18.

### `Dockerfile`

The server and the driver are installed in the same directory as the configuration file, so that its imports resolve.

```dockerfile
FROM node:22-alpine

WORKDIR /app
RUN npm init -y >/dev/null \
 && npm install --omit=dev @havesomecode/kibana-mcp-server pg

COPY kibana.mcp.config.mjs ./

USER node
EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://127.0.0.1:3000/healthz || exit 1

CMD ["npx", "kibana-mcp-server", "serve", "--http", "--config", "kibana.mcp.config.mjs"]
```

### `compose.yaml`

```yaml
services:
  kibana-mcp:
    build: .
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      KIBANA_URL: https://kibana.example.com
      KIBANA_USER: service-account
      KIBANA_PASSWORD: ${KIBANA_PASSWORD}
      OIDC_ISSUER: https://sso.example.com/realms/acme
      MCP_PUBLIC_URL: https://kibana-mcp.example.com/mcp
      DATABASE_URL: postgres://kibana_mcp:${POSTGRES_PASSWORD}@postgres:5432/kibana_mcp
    depends_on:
      postgres:
        condition: service_healthy

  postgres:
    image: postgres:17
    environment:
      POSTGRES_USER: kibana_mcp
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: kibana_mcp
    volumes:
      - catalogs:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U kibana_mcp"]
      interval: 5s
      retries: 10

volumes:
  catalogs:
```

## Run and check

```bash
docker compose up -d --build
```

```bash
curl -s http://127.0.0.1:3000/healthz
```

```bash
curl -si http://127.0.0.1:3000/mcp \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

The call is refused with `401` and a `WWW-Authenticate` header pointing to `https://kibana-mcp.example.com/.well-known/oauth-protected-resource/mcp`. An MCP client follows it, signs the user in with the provider, and retries with an access token:

```bash
curl -s http://127.0.0.1:3000/mcp \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Accept: application/json, text/event-stream" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

## Notes

- **Listening on `0.0.0.0` requires authentication.** The server refuses to start on a non-loopback host without OpenID Connect or a shared secret.
- **The provider must be reachable at startup.** The server reads its discovery document before listening and exits if it cannot.
- **Terminate TLS in front of the container.** `MCP_PUBLIC_URL` is the HTTPS URL clients use; the server checks it against the token audience. The compose file binds the port to the loopback interface of the host for your reverse proxy.
- **Behind a gateway instead.** Replace `oidc` with `profileHeader: "x-kibana-profile"` and `authToken: process.env.KIBANA_MCP_TOKEN`, and let only the [gateway](./gateway) reach the port.
- **Catalogs must outlive the container.** They live in the `catalogs` volume here. With the file store, mount a volume at the catalog path; with SQLite, at the database path.
- **SQLite instead of PostgreSQL.** Import `stores/sqlite`, set `storeOptions: { path: "/data/catalogs.sqlite" }`, drop the `pg` install and mount a volume at `/data`.
