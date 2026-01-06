# Twingate Linux Headless Client (Docker Image)

This image provides a lightweight, self-contained way to run the **Twingate Linux Client in headless mode** using a Service Key. It exists to simplify deployments where the regular Connector is not appropriate—such as lightweight gateways, utility containers, or environments where the client must run alongside other services. Instead of mounting a Service Key file, you paste the multi-line JSON directly into `docker-compose.yml`, and the container configures itself automatically on startup.

The idea of this container is that you can run it as a sidecar container alongside other services that need Twingate access. For example, you might run [Uptime Kuma](https://github.com/louislam/uptime-kuma) or [Gatus](https://gatus.io/), services that are used to remotely monitor your infrastructure, and need Twingate access to reach internal resources.

For more information on Twingate services and headless clients, see the [Twingate Services documentation](https://www.twingate.com/docs/services).

**Note**: If you are not familiar with third party container registries such as GHCR (GitHub Container Registry), please see the [GHCR Authentication Guide](#ghcr-github-container-registry-authentication-guide) at the bottom of this README.

---

## Usage

### 1. Get your Service Key

In the Twingate Admin Console, create a **Service** and generate a **Service Key**. Copy the JSON exactly as provided.

### 2. Example `docker-compose.yml`

```yaml
version: "3.8"

services:
  tg-headless-client:
    image: ghcr.io/twingate-solutions/twingate-custom-client-container:latest
    privileged: true

    environment:
      TWINGATE_SERVICE_KEY: | # Replace with your Twingate service key JSON
        {
          "host": "example.twingate.com",
          "client_id": "YOUR_CLIENT_ID",
          "client_secret": "YOUR_CLIENT_SECRET",
          "id": "YOUR_SERVICE_KEY_ID"
        }

    devices:
      - /dev/net/tun
    cap_add:
      - NET_ADMIN
    tty: true
    restart: unless-stopped
```

### 3. Start the client

```bash
docker compose up -d
docker compose logs -f tg-headless-client
```

### 4. Access the container

```bash
docker compose exec -it tg-headless-client bash
twingate status
```

---

## Health Checks and Container Health

The container utilizes a `healthcheck` that will run through any scripts in the `/healthchecks.d/` directory every 90 seconds. Each of the scripts should return a `0` exit code if the check passes, or a non-zero exit code if it fails. If any of the checks fail, the container will be marked as unhealthy.

Two example health checks are provided:
- `00-twingate-status-healthcheck.sh`: Checks that the Twingate client is running and connected.
- `10-curl-resource-healthcheck.sh`: Attempts to curl a specified internal Resource to ensure connectivity. This is set to `http://example.internal` by default, but you can modify it to point to a resource that makes sense for your environment. Alternatively, you can add a Resource in your Twingate Admin Console with the alias `example.internal` that points to anything (such as google.com) and assign access to the Service. The health check will then attempt to use Twingate to reach that site, and as long as it doesn't fail or return an error, the check will pass.

Any additional checks can be added by placing executable scripts in the `/healthchecks.d/` directory. Make sure they follow the same convention of returning `0` for success and non-zero for failure. Use either of the existing ones as templates, or create your own from scratch.

---

## Forking and Customization

The purpose of this repository is as an example of how to build a custom Twingate headless client container. In its current state that's all it will do, you can load it as a [sidecar container to provide Twingate access to other services](https://www.twingate.com/docs/linux-headless#sharing-networking-stacks). However, you could also fork this repository and customize the Dockerfile to add additional services or functionality as needed. You could also self host the image if there is some form of security or compliance requirement to do so.

It has one action currently, to build and push the image to GHCR on a monthly basis. This will tag the image with `latest` and the version of the Twingate client installed in the image. This is done so that you can always pull the latest Twingate client image if you want to stay up to date.

---

## GHCR (GitHub Container Registry) Authentication Guide

### Adding GHCR (GitHub Container Registry)

GHCR (hosted at `ghcr.io`) is GitHub's container registry. To use images stored on GHCR from Docker (pull or push), you need to authenticate Docker and reference images with the `ghcr.io/OWNER/IMAGE:TAG` name.

1) Create a Personal Access Token (PAT)

  - Go to GitHub and create a PAT with the appropriate scopes. For pulling only, `read:packages` is sufficient. To push images you will also need `write:packages` (and `repo` if you're working with private repositories). See [GitHub's PAT docs for details](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token).

2) Authenticate Docker to GHCR

   - Using an environment variable called `GITHUB_PAT` (recommended) you can log in without exposing the token in your shell history.

     - PowerShell (Windows):

       ```powershell
       echo $env:GITHUB_PAT | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
       ```

     - Bash (macOS / Linux):

       ```bash
       echo "$GITHUB_PAT" | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
       ```

     - CMD (Windows cmd.exe):

       ```cmd
       echo %GITHUB_PAT% | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
       ```

- You can also run `docker login ghcr.io -u YOUR_GITHUB_USERNAME` and paste the PAT when prompted. Docker Desktop will store credentials in the OS credential store by default.

3) Pulling images from GHCR

   - Pull directly with Docker:

     ```bash
     docker pull ghcr.io/OWNER/IMAGE:TAG
     ```

   - Use the same image reference in `docker-compose.yml`:

     ```yaml
     services:
       myservice:
         image: ghcr.io/OWNER/IMAGE:TAG
     ```

4) Links and further reading

- About GHCR: https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry/about-the-container-registry
- Authenticating to GHCR: https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry/authenticating-to-github-container-registry
- Pushing & pulling: https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry/pushing-and-pulling-containers

Notes:
- Use `read:packages` for pulls only; add `write:packages` (and `repo` for private repos) for pushes.
- When running on CI (GitHub Actions) prefer `GITHUB_TOKEN` or a repository/organization PAT with minimal scopes.

---