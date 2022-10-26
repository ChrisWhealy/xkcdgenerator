# xkcdgenerator Actor

This project implements a wasmCloud actor that listens for a simply HTTP GET request and responds with a random XKCD cartoon.

A complete guided walkthough of this project is available on [YouTube](https://www.youtube.com/watch?v=7OE__0thnK4).

## Important Technical Detail!

If your GitHub user name contains uppercase letters (as mine does), then when you try to deploy your signed Wasm module to an OCI registry, that deployment will fail with the cryptic error message `invalid reference format`.

To fix the problem:

* Open the file [./.github/workflows/release.yaml](https://github.com/ChrisWhealy/xkcdgenerator/blob/main/.github/workflows/release.yml)
* Line 97 contains the following long line:
   ```yaml
   wash reg push ghcr.io/${{ github.REPOSITORY }}:${{ env.actor-version }} build/${{ env.actor-name }}_s.wasm -a org.opencontainers.image.source=https://github.com/${{ github.REPOSITORY }} --allow-latest
   ```

   Both occurrences of `${{ github.REPOSITORY }}` must be replaced with `<your_lowercase_user_name>/xkcdgenerator`.

## The Implementation

Since all wasmCloud actors are sand-boxed, they can only perform pure, CPU-bound computations.
If additional functinoality is needed (such as listening for or making HTTP requests), then the actor must be explicitly granted this capability.

This is the role of a capability provider: it provides the wasmCloud host with a safety mechanism for ensuring that an actor does not suddenly perform any unexpected (and possibly malicious) actions.

| Capability Provider | Needed for...
|---|---
| [wasmcloud:httpserver](https://github.com/wasmCloud/interfaces/tree/main/httpserver) | listening for incoming HTTP requests
| [wasmcloud:httpclient](https://github.com/wasmCloud/interfaces/tree/main/httpclient) | fetching the metadata of the selected XKCD cartoon
| [wasmcloud:builtin:numbergen](https://github.com/wasmCloud/interfaces/tree/main/numbergen) | generating a random cartoon number

The implementation is in the file [src/lib.rs](./src/lib.rs)

## Local Deployment

1. Type `make` to compile the actor and generate a signed WebAssembly module
1. Start a local wasmCloud server by typing `wash up -d`
1. In your browser, go to the wasmCloud dashboard that should now be running on <http://localhost:4000>
1. Start a new actor by selecting "From File" in the drop down list then selecting `build/xkcdgenerator_s.wasm`.  Only one replica is needed for this demo
1. Start 2 new [capability providers](https://github.com/wasmCloud/capability-providers):
   * The HTTP server
      * The "Desired Host" is already prefilled
      * The OCI reference is `wasmcloud.azurecr.io/httpserver:0.16.2`
      * The link name can be left at `default`
   * The HTTP client
      * The "Desired Host" is already prefilled
      * The OCI reference is `wasmcloud.azurecr.io/httpclient:0.5.3`
      * The link name can be left at `default`

   You do not need to create a provider for the number generator as this functionality is built in to wasmCloud server
1. Now that you have an actor and two capability providers, these need to be linked together.

   To link the actor to the HTTP server, click on "Define Link":

   * The actor is `xkcdgenerator`
   * The provider is `HTTP server`
   * The link name can be left at `default`
   * The contract id is `wasmcloud:httpserver`
   * In this case, the HTTP server needs to know what port to run on, so in the Values field, enter `address=0.0.0.0:8080`
   * Press Submit and after a few seconds, a new capability provider is spun up

   To link the actor to the HTTP client, click on "Define Link":

   * The actor is `xkcdgenerator`
   * The provider is `HTTP client`
   * The link name can be left at `default`
   * The contract id is `wasmcloud:httpclient`
   * Leave the Values field empty
   * Press Submit and after a few seconds, a new capability provider is spun up

### Local Execution

Visit <http://localhost:8000>

### Deployment to the Cosmonic Platform

Some Github Actions configuration is included with this repo that allows you to deploy this app to the Cosmic platform.

If you store your source code on Github, we've included two actions: `build.yml` and `release.yml` under `.github/workflows`.

The build action will automatically build, lint, and check formatting for your actor.

The release action will automatically release a new version of your actor whenever code is pushed to `main`, or when you push a tag that starts with the lowercase letter `v`.  E.G. `v0.1.0`.

These actions require 3 secrets
1. `WASH_ISSUER_KEY`

   To generate this secret value, type `wash keys gen account`

   ```bash
   wash keys gen account

   Public Key: ACSIDMSYNPGHEQZLJWVJKF5ZYRG2UR4BW6WSY0ZZZZ2S39YMDNBFWKB3
   Seed: SAAF4DHEHKVI6ESGFAY3BP7UDNC17GACNTZX8DBYQ72TL4BJWEWAGEV6D4

   Remember that the seed is private, treat it as a secret.
   ```

   Add a new secret called `WASH_ISSUER_KEY` and use the `Seed` value

1. `WASH_SUBJECT_KEY`

   To generate this secret value, type `wash keys gen module`

   ```bash
   wash keys gen module

   Public Key: MDYZ1FUDTSTQSZJWGDIPJSNRCXUAB6ANHUJPW4HGYWP93FD4BZP0ETY1
   Seed: SMADXCWVB7PDBQVLMPV7VXWU2MCQYLUDSB4A2VXW9I4E5T2FQPN7KNL3AU

   Remember that the seed is private, treat it as a secret.
   ```

   Add a new secret called `WASH_SUBJECT_KEY` and use the `Seed` value


1. `WASMCLOUD_PAT`, which can be created by following the [Github PAT instructions](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) and ensuring the `write:packages` permission is enabled
