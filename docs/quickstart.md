# Quickstart

This document will help you get started with Unleash Edge locally.

## Why

You can think of Unleash Edge as a lightweight proxy layer between your SDKs and your Unleash instance. Its goal is to expose the same HTTP interface while providing higher throughput.

### Performance & UX

Especially for frontend clients, Unleash Edge helps you reduce latency for flag resolution by running closer to your users. For example, Unleash Edge could run on a global CDN (Content Delivery Network) or as part of your on-premises infrastructure.

Setting up one or more Edge nodes helps you distribute traffic across multiple nodes and reduce the load on your Unleash instance. By default, Unleash Edge relies on in-memory caching, but you can configure it to use Redis or the local filesystem.

### Security & resilience

From a security standpoint, Unleash Edge lets you expose a single, SDK-compatible endpoint without opening your Unleash instance to the public, thus reducing the attack surface.

At the same time, Unleash Edge provides an additional layer of resilience, so brief upstream hiccups don't disrupt feature delivery.

Edge can also run in offline mode, so it doesn't need a connection to an upstream Unleash instance. This can simplify local development or in environments with limited connectivity.

## Prerequisites

Here's what you need before getting started:

1. An [Unleash instance](https://github.com/Unleash/unleash) running locally or remotely (version `4.15` or later)
2. A valid [API token](https://docs.getunleash.io/reference/api-tokens-and-client-keys) for your Unleash instance
3. [Docker](https://www.docker.com/get-started/) installed and running
4. Your preferred Unleash SDK in a [sample app](https://github.com/Unleash/unleash-sdk-examples)


## How to run Unleash Edge locally

First, make sure your Unleash instance is running (locally or remotely) and generate a new client token.

<details>
  <summary>Using the Rust toolchain</summary>

  If you're comfortable with the Rust toolchain, install the CLI with cargo binstall (or [from source](https://github.com/Unleash/unleash-edge/blob/main/docs/development-guide.md)):

  ```shell
  cargo binstall unleash-edge
  ```

  Then launch it:

  ```shell
  unleash-edge edge \
    --strict \
    --upstream-url <your_unleash_instance> \
    --tokens '<your_client_token>'
  ```

</details>

<details>
  <summary>Using Docker</summary>

  Launch Unleash Edge locally with Docker:

  ```shell
  docker run -it \
    -p 3063:3063 \
    -e STRICT=true \
    -e UPSTREAM_URL=<your_unleash_instance> \
    -e TOKENS='<your_client_token>' \
    unleashorg/unleash-edge \
    edge
  ```

  By default, the command above uses the latest `unleashorg/unleash-edge` tag. Feel free to pick a specific version from the list of [tags on Docker Hub](https://hub.docker.com/r/unleashorg/unleash-edge). 

</details>

### Required parameters

Let's break down the parameters and placeholders in the command above.

#### `<your_unleash_instance>`

This is the URL of your Unleash instance. Use the base URL, e.g. `https://app.unleash-hosted.com/testclient` or `http://localhost:4242`.

<details>
  <summary>⚠️ Important note when using Docker</summary>

  When using Docker with a local Unleash instance, `localhost` will refer to the container itself so you cannot simply use `http://localhost:4242`.
  You'll need to use a different hostname, depending on where the Unleash instance is running.

  #### Solution 1: use host.docker.internal to reach the host

  The easiest solution when using Docker Desktop is to reference the Unleash instance as `http://host.docker.internal:4242`.

  On Linux, you also need to define the host with `--add-host=host.docker.internal:host-gateway`, or just use `--network=host`.

  #### Solution 2: run on the same user-defined network

  If Unleash runs locally on Docker as well, you could run both containers on the same network.

  When launching Unleash with `docker compose up`, you could use the [default Compose network](https://docs.docker.com/compose/how-tos/networking/) named `unleash_default` and reference the instance as `web`. In your Edge launch command, it will look like `--network unleash_default -e UPSTREAM_URL=http://web:4242`.

  If you're not using the default Compose setup or if you prefer a bit of decoupling, you can create a custom network:

  ```shell
  # define custom network
  docker network create unleash-net

  # launch Unleash
  docker run -d --name unleash --network unleash-net ....

  # launch Unleash Edge
  docker run -it --network unleash-net -e UPSTREAM_URL=http://unleash:4242 ....
  ```

</details>


#### `<your_client_token>`

This API token is required when launching Unleash Edge in strict mode, which is recommended starting with `v19.2+`.

You can generate a new API token by visiting your Unleash instance, under **Admin settings → Access control → API access**.
Click **New API token**, give it a name, and confirm the default values.

Note: make sure you keep the single quotes in `-e TOKENS='...'` so the `*` isn't expanded by your shell. 

## How to point your SDK at Unleash Edge

**Checkpoint**: Before you continue, make sure Unleash Edge is running on port `3063`.

You can verify this by fetching `http://localhost:3063/internal-backstage/health`. This endpoint should respond with `{"status":"OK"}`.

Once everything is running smoothly, updating your application code is very straightforward:

1. Identify the code or configuration file where the Unleash instance URL is defined.
2. Update it to use the local Unleash Edge on `http://localhost:3063`.
3. Restart your app and test it.


For example, if you're using the [React example](https://github.com/Unleash/unleash-sdk-examples/tree/main/React), update your `index.tsx` file as follows:

```tsx
<FlagProvider
    config={{
    url: "http://localhost:3063/api/frontend/", // Unleash Edge running locally
    clientKey:
        "default:development.unleash-insecure-frontend-api-token", 
    refreshInterval: 15, 
    appName: "codesandbox-react",
    }}
>
```

Similarly, if you're using a backend language such as Go, Python, or Rust, update the `UNLEASH_API_URL` environment variable in your `.env` file:

```
UNLEASH_API_URL=http://localhost:3063/api
```

## Verification

Should you encounter any issues while connecting your SDK to Unleash Edge, use the following commands to help identify the problem.

```shell
# is the local Unleash instance running correctly?
curl http://localhost:4242/health

# is the local Unleash Edge instance running correctly?
curl http://localhost:3063/internal-backstage/health
# or via CLI
unleash-edge health

# is data going through? is my token valid?
curl -H "Authorization: <your_token>" http://localhost:3063/api/client/features
```

You might encounter some of these common issues:

- If Unleash Edge logs show "connection refused" to `127.0.0.1:4242` within Docker, you're pointing at `localhost` inside the container. Use `host.docker.internal` or a shared Docker network instead.
- If you get "401/invalid token", ensure you're using a valid token from your Unleash instance that matches the environment and project you expect.

## Next steps

Congratulations, you successfully set up Unleash Edge locally!

Unleash Edge comes with a lot of flexibility and advanced configuration options that are worth exploring next:

1. [Offline mode](https://github.com/Unleash/unleash-edge/blob/main/docs/concepts.md#offline) - Learn how to configure Unleash Edge to work without an Unleash instance, using a local features file.
2. [Pretrusted tokens](https://github.com/Unleash/unleash-edge/blob/main/README.md#pretrusted-tokens) - Learn how to explicitly authorize known frontend tokens without upstream validation.
3. [Security considerations in production](https://github.com/Unleash/unleash-edge/blob/main/docs/deploying.md) - Learn how to run Unleash Edge in production with the best practices for CORS, health checks, and sensitive endpoints.
4. [Persistent cache storage](https://github.com/Unleash/unleash-edge/blob/main/docs/CLI.md#unleash-edge-edge) - Learn how to enable persistent storage for caching with options such as `--backup-folder` and `--redis-url`.
5. [Advanced CLI config](https://github.com/Unleash/unleash-edge/blob/main/docs/CLI.md) - Learn how to customize the CLI behavior with options such as `--base-path`, `--workers`, `--allow-list`, `--edge-request-timeout`, or `--edge-auth-header`.
