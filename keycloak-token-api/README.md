# keycloak-token-api

MuleSoft 4 application that exposes two REST endpoints for OAuth 2.0 token operations via Keycloak.

| Endpoint | Method | Description |
|---|---|---|
| `/gettoken` | POST | Generate an access token using Client Credentials grant |
| `/validatetoken` | POST | Validate an access token using Token Introspection (RFC 7662) |

---

## Project Structure

```
keycloak-token-api/
├── pom.xml                                          # Maven dependencies
├── mule-artifact.json                               # Mule artifact metadata
├── src/
│   ├── main/
│   │   ├── mule/
│   │   │   ├── global.xml                           # Global configs, HTTP configs, error handler
│   │   │   └── keycloak-token-api.xml               # Flow definitions
│   │   └── resources/
│   │       ├── config.yaml                          # Non-sensitive properties
│   │       ├── config-secured.yaml                  # Encrypted credentials (AES)
│   │       ├── log4j2.xml                           # Logging configuration
│   │       └── api/
│   │           └── keycloak-token-api.raml          # RAML 1.0 specification
│   └── test/
│       └── munit/                                   # MUnit test suites
└── postman/
    └── Keycloak-Token-API.postman_collection.json   # Postman test collection
```

---

## Prerequisites

| Requirement | Version |
|---|---|
| Mule Runtime | 4.11.2 |
| Java (JDK) | 17 |
| Maven | 3.8+ |
| Keycloak | 21+ |

---

## Secure Properties Setup

The client credentials are encrypted with AES-CBC. Before running:

### Step 1: Encrypt credentials using Anypoint Secure Properties Tool

Download: https://docs.mulesoft.com/mule-runtime/latest/secure-configuration-properties#secure_props_tool

```bash
# Encrypt client ID
java -jar secure-properties-tool.jar string encrypt AES CBC MyEncryptionKey16 "Salesforce-dev"

# Encrypt client secret
java -jar secure-properties-tool.jar string encrypt AES CBC MyEncryptionKey16 "HvgPTaUVCqlrLBJKczi5IsuTQLfSL5fb"
```

### Step 2: Update `config-secured.yaml`

Replace the placeholder values with the actual encrypted output:
```yaml
keycloak:
  client:
    id: "![<encrypted-client-id>]"
    secret: "![<encrypted-client-secret>]"
```

### Step 3: Set the encryption key

For local development (add to Anypoint Code Builder run configuration):
```
-Dsecure.key=MyEncryptionKey16
```

For production (CloudHub): set as a Secure Application Property in Runtime Manager.

> **Security Note:** Never commit real encryption keys or plaintext credentials to source control.
> The `secure.key` in `config.yaml` is a **development placeholder only**.

---

## Local Development Setup

### 1. Configure `config.yaml`

```yaml
http:
  port: "8081"

keycloak:
  host: "localhost"
  port: "8080"
  realm: "Muletest"

secure:
  key: "MyEncryptionKey16"   # development only - use JVM arg in production
```

### 2. Keycloak Setup

Ensure Keycloak is running at `http://localhost:8080` with:
- Realm: `Muletest`
- Client: `Salesforce-dev` (Client Credentials grant enabled)
- Client secret configured

Refer to the [Keycloak quick setup guide](https://www.keycloak.org/getting-started/getting-started-docker) if needed.

### 3. Build the Application

```bash
cd keycloak-token-api

# Set JAVA_HOME to JDK 17
export JAVA_HOME=/path/to/jdk17

# Build
mvn clean package -DskipTests
```

**Expected output:**
```
[INFO] BUILD SUCCESS
[INFO] keycloak-token-api-1.0.0-SNAPSHOT-mule-application.jar
```

### 4. Run Locally (Anypoint Code Builder)

In VS Code with Anypoint Code Builder:
1. Open `keycloak-token-api` project
2. Add run configuration with JVM arg: `-Dsecure.key=MyEncryptionKey16`
3. Click **Run** or use `F5`

Or via Maven:
```bash
mvn mule:run -Dsecure.key=MyEncryptionKey16
```

---

## API Reference

### POST /gettoken

Generates an OAuth 2.0 access token from Keycloak using Client Credentials grant.

**No request body required.**

**Positive Response (200):**
```json
{
  "access_token": "eyJraWQiOi...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "muleapi"
}
```

**Negative Scenarios:**

| Scenario | Status | Error |
|---|---|---|
| Invalid client ID/secret in secure properties | 401 | `INVALID_CLIENT` |
| Keycloak unreachable / timeout | 503 | `SERVICE_UNAVAILABLE` |
| Unexpected internal error | 500 | `INTERNAL_SERVER_ERROR` |

---

### POST /validatetoken

Validates an OAuth 2.0 access token using Keycloak Token Introspection (RFC 7662).

**Request Body:**
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Positive Response - Active Token (200):**
```json
{
  "exp": 1780321602,
  "iat": 1780321542,
  "iss": "http://localhost:8080/realms/Muletest",
  "sub": "1b32e1a0-34ee-4e05-88fe-ee6d41c698a7",
  "azp": "Salesforce-dev",
  "scope": "email profile",
  "client_id": "Salesforce-dev",
  "active": true
}
```

**Positive Response - Inactive Token (200, RFC 7662):**
```json
{
  "active": false
}
```

**Negative Scenarios:**

| Scenario | Status | Error |
|---|---|---|
| Missing `access_token` field in body | 400 | `INVALID_REQUEST` |
| Empty `access_token` value | 400 | `INVALID_REQUEST` |
| Invalid/expired/revoked token | 200 | `{"active": false}` |
| Invalid client credentials | 401 | `INVALID_CLIENT` |
| Keycloak unreachable / timeout | 503 | `SERVICE_UNAVAILABLE` |

---

## Flow Explanation

### keycloak-get-token-flow

```
HTTP POST /gettoken
  └─ Log request
  └─ HTTP Request → Keycloak /token
       ├─ Body: grant_type=client_credentials
       ├─ Content-Type: application/x-www-form-urlencoded
       ├─ Auth: Basic (client_id:client_secret from secure properties)
       └─ Response Validator: 200..299 only
  └─ Choice: access_token present?
       ├─ YES → Log + Transform → return {access_token, token_type, expires_in, scope}
       └─ NO  → raise APP:VALIDATION → 400 INVALID_REQUEST
```

### keycloak-validate-token-flow

```
HTTP POST /validatetoken
  └─ Log request
  └─ Choice: access_token in body?
       ├─ YES:
       │    └─ Transform: JSON → "token=<value>" (form-urlencoded)
       │    └─ HTTP Request → Keycloak /token/introspect
       │         ├─ Body: token=<access_token>
       │         ├─ Content-Type: application/x-www-form-urlencoded
       │         ├─ Auth: Basic (client_id:client_secret)
       │         └─ Response Validator: 200..299 only
       │    └─ Choice: active == true?
       │         ├─ TRUE  → Log + return full JWT claims
       │         └─ FALSE → Log WARN + return {"active": false}
       └─ NO → raise APP:VALIDATION → 400 INVALID_REQUEST
```

### global-error-handler

| Error Type | HTTP Status | Response |
|---|---|---|
| `APP:VALIDATION`, `HTTP:BAD_REQUEST`, `MULE:EXPRESSION` | 400 | `INVALID_REQUEST` |
| `HTTP:UNAUTHORIZED` | 401 | `INVALID_CLIENT` |
| `HTTP:FORBIDDEN`, `APP:INVALID_TOKEN` | 403 | `INVALID_TOKEN` |
| `HTTP:CONNECTIVITY`, `HTTP:TIMEOUT`, `HTTP:SERVICE_UNAVAILABLE` | 503 | `SERVICE_UNAVAILABLE` |
| `ANY` | 500 | `INTERNAL_SERVER_ERROR` |

---

## Testing

### Import Postman Collection

1. Open Postman
2. Import `postman/Keycloak-Token-API.postman_collection.json`
3. Run tests in order:
   - **Folder 01**: Direct Keycloak connectivity tests
   - **Folder 02**: MuleSoft `/gettoken` tests (positive + negative)
   - **Folder 03**: MuleSoft `/validatetoken` tests (positive + negative)

### cURL Quick Test

**Get Token:**
```bash
curl -X POST http://localhost:8081/gettoken \
  -H "Content-Type: application/json"
```

**Validate Token:**
```bash
curl -X POST http://localhost:8081/validatetoken \
  -H "Content-Type: application/json" \
  -d '{"access_token": "eyJhbGci..."}'
```

**Negative - Missing access_token:**
```bash
curl -X POST http://localhost:8081/validatetoken \
  -H "Content-Type: application/json" \
  -d '{}'
# Expected: 400 {"error":"INVALID_REQUEST","description":"..."}
```

---

## Deployment

### CloudHub 2.0

1. **Encrypt credentials** with your production key:
   ```bash
   java -jar secure-properties-tool.jar string encrypt AES CBC <PROD_KEY> "<client_id>"
   java -jar secure-properties-tool.jar string encrypt AES CBC <PROD_KEY> "<client_secret>"
   ```

2. **Update `config-secured.yaml`** with production-encrypted values

3. **Build:**
   ```bash
   mvn clean package -DskipTests
   ```

4. **Deploy via Anypoint Platform:**
   - Navigate to Runtime Manager → Deploy Application
   - Upload `target/keycloak-token-api-1.0.0-SNAPSHOT-mule-application.jar`
   - Set environment: Production
   - Add Secure Property: `secure.key` = `<your-production-encryption-key>`
   - Add Property: `keycloak.host` = `<your-keycloak-host>`
   - Add Property: `keycloak.port` = `8080` (or `443` for HTTPS)
   - Add Property: `keycloak.realm` = `<your-realm>`

5. **Or via Maven:**
   ```bash
   mvn deploy -DmuleDeploy \
     -Dcloudhub.application.name=keycloak-token-api-prod \
     -Dcloudhub.environment=Production \
     -Dsecure.key=<PROD_KEY>
   ```

### Environment-Specific Properties

For multiple environments, create profile-specific config files and reference via Maven profiles:

| File | Environment |
|---|---|
| `config.yaml` | All environments (non-sensitive) |
| `config-secured.yaml` | All environments (encrypted credentials) |
| JVM `-Dsecure.key` / Runtime Manager Secure Property | Must be set per-environment |

---

## Troubleshooting

| Problem | Cause | Fix |
|---|---|---|
| `401 INVALID_CLIENT` on /gettoken | Wrong client ID/secret | Re-encrypt correct credentials; verify Keycloak client config |
| `503 SERVICE_UNAVAILABLE` | Keycloak not running | Start Keycloak; verify `keycloak.host` and `keycloak.port` |
| `400 INVALID_REQUEST` on /validatetoken | Missing `access_token` field | Ensure JSON body contains `"access_token"` key |
| `{"active": false}` | Token expired or revoked | Call /gettoken first to get a fresh token |
| Secure properties decrypt error | Wrong `secure.key` | Use same key that was used for encryption |
| App starts but port 8081 in use | Port conflict | Change `http.port` in `config.yaml` |

---

## References

- [MuleSoft HTTP Connector](https://docs.mulesoft.com/http-connector/latest/)
- [Mule Secure Properties](https://docs.mulesoft.com/mule-runtime/latest/secure-configuration-properties)
- [Keycloak Token Introspection (RFC 7662)](https://www.rfc-editor.org/rfc/rfc7662)
- [Keycloak Client Credentials Grant](https://www.keycloak.org/docs/latest/server_admin/index.html#_client-credentials)
- [MuleSoft Documentation](https://docs.mulesoft.com/general/)