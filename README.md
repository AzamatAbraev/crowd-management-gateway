

 >[!IMPORTANT]                                                           
 > This service is part of the [Crowd Management                        
  System](https://github.com/AzamatAbraev/crowd-management-system)         monorepo. Clone the full system from there — do not clone this repo      individually. 


# Crowd Management Gateway

The API gateway and authentication entry point for the platform. All browser traffic goes through this service. It handles the OAuth2 login flow with Keycloak, attaches the user's access token to every proxied request, and routes traffic to the backend API.

## Responsibilities

- Serve as the single entry point for the frontend
- Perform the OAuth2 Authorization Code flow with Keycloak on behalf of the user
- Attach the access token to proxied requests via the `TokenRelay` filter (Spring Cloud Gateway)
- Route API calls from the frontend to the correct backend service
- Handle logout, including redirecting to Keycloak's end-session endpoint to invalidate the Keycloak session

## Stack

- Java 25, Spring Boot 4.0.1
- Spring Cloud Gateway (WebFlux/reactive)
- Spring Security OAuth2 Client
- Spring Cloud CircuitBreaker (Resilience4j)
- Lombok

## Port

The gateway listens on port **8082**.

## Routing table

All routes proxy to `crowd-management-api` (port 8081) and prepend `/api/v1` to the path unless otherwise noted.

| External path | Forwarded to |
|---|---|
| `/resources/**` | `http://crowd-management-api:8081/api/v1/resources/**` |
| `/user/**` | `http://crowd-management-api:8081/api/v1/user/**` |
| `/analytics/**` | `http://crowd-management-api:8081/api/v1/analytics/**` |
| `/devices/**` | `http://crowd-management-api:8081/api/v1/devices/**` |
| `/buildings/**` | `http://crowd-management-api:8081/api/v1/buildings/**` |
| `/people/**` | `http://crowd-management-api:8081/api/v1/people/**` |
| `/admin/users/**` | `http://crowd-management-api:8081/api/v1/users/**` (path rewritten) |

`/people/**` is explicitly permitted without authentication so public occupancy data can be accessed without login.

## Authentication flow

1. The user visits a protected route in the browser
2. The gateway redirects to Keycloak's authorization endpoint
3. Keycloak authenticates the user and redirects back to `http://localhost:8082/login/oauth2/code/keycloak`
4. The gateway exchanges the code for tokens and stores the session
5. On subsequent requests, `TokenRelay` appends the Bearer token to the forwarded request
6. After successful login, the user is redirected to `http://localhost:5173/home`

Logout hits Keycloak's end-session endpoint directly, passing `id_token_hint` and redirecting back to `http://localhost:5173/` after the session is cleared.

## Running with Docker

The project uses a multi-stage Docker build. Java 25 and Maven run inside the build stage.

```bash
docker compose up -d
```

The container joins the `crowd-management-network` Docker network. Keycloak must be healthy before starting the gateway because the gateway fetches the OIDC discovery document from Keycloak at startup.

## Configuration

Key settings in `application.yml` and `docker-compose.yml`:

| Setting | Value |
|---|---|
| Keycloak realm | `crowd-management` |
| Keycloak client ID | `gateway` |
| Authorization URI | `http://localhost:8080/realms/crowd-management/...` (browser-facing) |
| Token/JWK URIs | `http://keycloak-app:8080/realms/crowd-management/...` (container-to-container) |
| Redirect URI | `http://localhost:8082/login/oauth2/code/keycloak` |

Note: the authorization URI uses `localhost:8080` because the browser must reach it directly. Token exchange and JWK set fetching use the Docker container name `keycloak-app` because those happen server-to-server inside the Docker network.

## Dependencies on other services

| Dependency | Container name | Reason |
|---|---|---|
| Keycloak | `keycloak-app` | OAuth2 login, token validation |
| Crowd Management API | `crowd-management-api` | All proxied API calls |

The gateway must be started after both are running. `start-all.sh` in the project root enforces this order with a 15-second wait after the API starts.
