# Configuration

In Starship one can define the infra required with a simple config file.
The config file one specifies is merged with the default [`values.yaml`](https://github.com/hyperweb-io/starship/blob/main/starship/charts/devnet/values.yaml)
file before the infra is spun up. All supported directives are present in the default `values.yaml`. We will go over most used ones here.

Here is a basic example that will spin up:
* 2 chains with 2 validators each
* Hermes relayer between them (by default will create the IBC-transfer ports and channels)
* Explorer instance: ping-pub with the 2 chains configuration
* Registry service: Analogous to cosmos/chain-registry, following the same schemas

```yaml
name: starship
version: 1.7.0

chains:
  - id: osmosis-1
    name: osmosis
    numValidators: 2
    ports:
      rest: 1313
      rpc: 26653
  - id: gaia-1
    name: cosmoshub
    numValidators: 2
    ports:
      rest: 1317
      rpc: 26657

relayers:
  - name: osmos-gaia
    type: hermes
    replicas: 1
    chains:
      - osmosis-1
      - gaia-1

explorer:
  enabled: true
  ports:
    rest: 8080

registry:
  enabled: true
  ports:
    rest: 8081
```

Note this is a basic configuration file. We have various other directives and operations we can perform just with
the config file directives that we will go into details into.

----------------------------------------

# Chains YAML Syntax

Here we will go into details of the `chains` top level directive in the Starship config file.

## `id`
ID of the chain, this is used as the `chain-id` of the chain
```yaml
chains:
- id: osmosis-1
  ...
- id: gaia-2
  ...
- id: juno-1
  ...
```

## `name`
Type of chain is a short hand to use the default key values for a chain found [here](https://github.com/hyperweb-io/starship/blob/main/starship/charts/devnet/values.yaml#L54...#L244)
```yaml
chains:
- id: osmosis-1
  name: osmosis
  ...
- id: gaia-2
  name: cosmoshub
  ...
```

One can override the default values supported by `defaultChain` by simply mentioning it in the chain directive.
Here is how one can set the docker image of the default values
```yaml
chains:
- id: osmosis-1
  name: osmosis
  image: ghcr.io/hyperweb-io/starship/osmosis:v25.0.0
  coins: 100000000000000uosmo
  ...
```

### `name: custom`
Optionally one can define the type to `custom`, and pass all the default params directly into the chain directive. This is useful when one is
trying setup a chain not supported in the `defaultChains`.
```yaml
chains:
- id: osmosis-1
  name: custom
  image: ghcr.io/hyperweb-io/starship/osmosis:v25.0.0
  home: /root/.osmosisd
  binary: osmosisd
  prefix: osmo
  denom: uosmo
  coins: 100000000000000uosmo,100000000000000uion
  hdPath: m/44'/118'/0'/0/0
  coinType: 118
  repo: https://github.com/osmosis-labs/osmosis
  ...
```

## `image` (optional)
Already mentioned above, but here we will go deeper into how the docker images are used for the chain. By default this value is taken from
the `name` directive. This is the standard way of running chains at specific versions.

```yaml
chains:
- id: osmosis-1
  name: osmosis
  image: "<docker-image>"
  ...
```

> One can use any docker image as long as it follows the following rules:
> * Packages `curl,make,bash,jq,sed`
> * Chain binaries installed in $PATH

We create docker images with this [Dockerfile](https://github.com/hyperweb-io/starship/blob/main/starship/docker/chains/Dockerfile), using Heighliner base
images. All supported docker images can be found [here](https://github.com/orgs/hyperweb-io/packages?repo_name=starship)

## `numValidators`
Number of validators to run for the chain. It must be greater than 1, can go up to as many resources available.

```yaml
chains:
- id: osmosis-1
  name: osmosis
  numValidators: 2
```

> Note: The first node spun up is a `genesis` node, which spins up before all other validators. Once the genesis node starts
> then all other validator nodes are spun up simultaneously

If number of validators is more than the mnemonics available in the [`keys.json`](https://github.com/hyperweb-io/starship/blob/main/starship/charts/devnet/configs/keys.json#L9) file
which is 5, then other validators are created with random mnemonic on initialization.

## `ports`
Ports directive in the `chains` directive is used for `kubectl port-forward` forwarding kubernetes service ports to local host.
```yaml
chains:
- id: osmosis-1
  name: osmosis
  numValidators: 2
  ports:
    rest: 1317     # Rest endpoint of the Genesis validator node (most used)
    rpc: 26657     # RPC endpoint of the genesis validator node (most used)
    grpc: 9091     # GRPC endpoint of the genesis validator node (less used)
    faucet: 8001   # Faucet running next to the genesis node (most used)
    exposer: 9090  # Exposer sidecar port (less used)
```

> Note: Websockets are exposed at `ws://localhost:26657/websocket` for cosmos chains.

Available endpoints for extra services:
* `exposer`: https://github.com/hyperweb-io/starship/blob/main/starship/proto/exposer/service.proto#L15
* `faucet`: https://github.com/cosmos/cosmjs/blob/main/packages/faucet/README.md#using-the-faucet

> Note: Make sure the ports are not overlapping in the config file

## `resources` (optional)
Resource directive is something with which you can control how much resources to provide to each of the chain node.
```yaml
chains:
- id: osmosis-1
  name: osmosis
  numValidators: 1
  resources:
    cpu: 1       # 1 CPU
    memory: 1Gi  # 1 GB
```

Main benefit of this directive is to closely control the resources provided for each of the validator node.
One can provide fractional cpus and memory as per the resource constraints of the system

Usually when running in the CI or locally you can provide partial resources like following:
```yaml
chains:
- id: osmosis-1
  name: osmosis
  numValidators: 1
  resources:
    cpu: "0.2"      # 0.2 CPU
    memory: "400M"  # 0.4 GB
```

For more details on the resource directive have a look at [kubernetes resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/#meaning-of-cpu)

> Note: If one provides less resources, the devnet will be stable for shorter timeframe. Nodes will start dying or restarting
> We recommend to specify resources as per your requirements

## `faucet` (optional)
Every genesis node runs a faucet. We support 2 types of faucet:
* [starship faucet](https://github.com/hyperweb-io/starship/tree/main/starship/faucet), by default
* [cosmjs faucet](https://github.com/cosmos/cosmjs/tree/main/packages/faucet)

Setting for cosmjs-faucet [here](https://github.com/hyperweb-io/starship/blob/main/starship/charts/devnet/values.yaml#L32...#L38) but can be
overridden with this directive, specially for compatible cosmjs version with the chain version.

```yaml
chains:
- id: osmosis-1
  name: osmosis
  numValidators: 1
  faucet:
    enabled: true           # optional, default is true, need to specify if want to disable faucet, set to false
    type: cosmjs            # default is cosmjs, allowed [cosmjs, starship]
    image: ghcr.io/hyperweb-io/starship/cosmjs-faucet:v0.30.1  # optional, default image set if not specified
    concurrency: 2          # optional, default is 5
    resources:
      cpu: "0.3"
      memory: "600M"
```

By default, we enable cosmjs faucet. Incase `faucet.enabled` is set to `false`, then we manually add
validator and relayer keys at gentx on genesis. This will take longer during initialization.

> Note `concurrency` in `faucet` is the number of concurrent requests the faucet can handle.
If you are running a chain with less resources, or want faster startup time,
then you can reduce the `concurrency` to a lower number.

## `build` (optional)
With the `build` directive in the chain, one can basically build chain binaries on the fly before starting the chain.
When the directive is `enabled`, then the docker image is set to a [`runner` docker image](https://github.com/hyperweb-io/starship/blob/main/starship/docker/starship/runner/Dockerfile)
which is a basic alpine image with starship dependencies.
```yaml
chains:
- id: osmosis-1
  name: osmosis
  numValidators: 2
  build:
    enabled: true
    source: v15.0.0  # can be a tag or a commit or a branch
```
We fetch the source from the `repo` defined in the `defaultChains`.

If you are using the `name: custom`, then you need to specify `repo` directive, from where to get the source.

## `upgrade` (optional)
If you want to perform a software upgrade on a chain, then this directive is here help. This will not perform the chain
upgrade, but prepare the chain nodes to be able to do an actual software upgrade.
```yaml
chains:
- id: osmosis-1
  name: osmosis
  numValidators: 2
  upgrade:
    enabled: true             # enable the directive
    type: build               # type build specifies, this will be similar to the `build` directive
    genesis: v13.0.0          # current chain version, tag, branch or commit
    upgrades:
      - name: v14             # upgrade proposal name
        version: v14.0.0      # upgrade chain version, tag, branch or commit
      - name: v15
        version: v15.0.0-rc1  # next chain upgrade version
```

We use [cosmovisor](https://docs.cosmos.network/v0.47/tooling/cosmovisor)
to run the validators when this directive is enabled, which allows for external software-upgrade-proposal.

## `genesis` (optional)
Patch `genesis.json` file directly from the config file using this directive. Once the genesis node creates the genesis file, then the
patch for genesis is applied.
```yaml
chains:
- id: osmosis-1
  name: osmosis
  numValidators: 2
  genesis:
    app_state:
      staking:
        params:
          unbonding_time: "5s"
```

## `scripts` (optional)
Scripts directive will replace the default scripts with the given scripts. In order to use this directive,
one must use [`scripts/install.sh`](https://github.com/hyperweb-io/starship/blob/main/starship/scripts/install.sh) script for running the helm chart.

> Since the local scripts are converted into configmaps that are then created in the kubernetes cluster, there is a limit of 1MBi

Types of scripts that are allowed inside the `scripts` dir are:
* `createGenesis`: Script used in the genesis node that will create the `genesis.json` file. Default: [`create-genesis.sh`](https://github.com/hyperweb-io/starship/blob/main/starship/charts/devnet/scripts/default/create-genesis.sh)
* `updateGenesis`: Script used to make custom changes to the genesis file. Default: [`update-genesis.sh`](https://github.com/hyperweb-io/starship/blob/main/starship/charts/devnet/scripts/default/update-genesis.sh)
* `updateConfig`: Script used to make custom changes to the config files. Default: [`update-config.sh`](https://github.com/hyperweb-io/starship/blob/main/starship/charts/devnet/scripts/default/update-config.sh)
* `createValidator`: Script to run the `create-validator` txn on the validator nodes after spinup. Note this script is run as part of the `PostStartHook` of the validator pods. Default: [`create-validator.sh`](https://github.com/hyperweb-io/starship/blob/main/starship/charts/devnet/scripts/default/create-validator.sh)
* `transferTokens`: Script run as part of the Starship glue code. This script is related to the faucet that we run. Default: [`transfer-tokens.sh`](https://github.com/hyperweb-io/starship/blob/main/charts/starship/devnet/scripts/default/transfer-tokens.sh)
* `buildChain`: Script that builds chain binaries, based on the `build` directive. Default: [`build-chain.sh`](https://github.com/hyperweb-io/starship/blob/main/starship/charts/devnet/scripts/default/build-chain.sh)

One can choose the sub-scripts that need to be overwritten, default scripts will be used instead.

```yaml
chains:
  - id: osmosis-1
    name: osmosis
    numValidators: 1
    scripts:
      createGenesis:
        file: scripts/create-custom-genesis.sh
      updateGenesis:
        file: scripts/update-custom-genesis.sh
      updateConfig:
        file: scripts/update-custom-config.sh
      createValidator:
        file: scripts/create-custom-validator.sh
      transferTokens:
        file: scripts/transfer-custom-tokens.sh
```

> Note `file` path is relative to the config file itself

Dir structure should look something like:
```bash
config.yaml
scripts/
  create-custom-genesis.sh
  update-custom-genesis.sh
  update-custom-config.sh
  create-custom-validator.sh
  transfer-custom-tokens.sh
```

## `cometmock` (optional)
A dedicated feature flag, that will run the chain with InformalSystem's [CometMock](https://github.com/informalsystems/CometMock)
binary. With `cometmock` enabled we will run all validators of the chain with out of process cometbft,
that connect with the cometmock pod.

Enabling cometmock is really simple with following
```yaml
chains:
- id: cosmoshub-4
  name: cosmoshub
  numValidators: 4
  cometmock:
    enabled: true
    image: ghcr.io/informalsystems/cometmock:v0.37.x  ## optional, default is 0.37.x
```

By default we use the cometbft `v0.37.x` version, but this can be set to other versions from
cometmock [releases](https://github.com/informalsystems/CometMock/pkgs/container/cometmock).
Noteable other versions are: `v0.34.x` as well as `v0.38.x`.

If `cometmock` is enabled, during port-forwarding, we forward the `ports.rpc` to the cometmock endpoint.
For in-cluster communication, make sure to connect to the cometmock service instead of validator service for rpc connections.

## `env` (optional)
The `env` directive allows you to set custom environment variables for the validator and genesis containers. This can be useful for enabling features like debug logging or configuring specific settings.

The `env` directive expects an array of objects, each containing a `name` and `value` field. Here's an example of how to use the `env` directive:
```yaml
chains:
- id: agoriclocal
  name: agoric
  numValidators: 1
  env:
    - name: DEBUG
      value: SwingSet:vat,SwingSet:ls
```

### Learn more
To more about CometMock, have a look at the [blog](https://informal.systems/blog/cometmock) that explains its working

> Note: Current support of cometmock has been tested with multiple chains with verion `0.34.x` and `0.37.x`.

## `ics` (optional)
The `ics` directive allows you to enable interchain security on the chain. This is useful when you want to test a chain which is ICS enabled

```yaml
chains:
- id: neutron-1
  name: neutron
  numValidators: 1
  ics:
    enabled: true
    provider: cosmoshub-4  # chain id of the provider chain
- id: cosmoshub-4
  name: cosmoshub
  numValidators: 1
```

> Note: To complete the ICS setup one also needs to setup the relayer with ics enabled. Have a look at the [relayer](https://docs.cosmology.zone/starship/config/relayers#ics)

## `balances` (optional)
The `balances` directive allows you to set the initial balances for the chain addresses. This is useful when you want to test a chain with specific initial balances.

```yaml
chains:
- id: osmosis-1
  name: osmosis
  numValidators: 1
  balances:
    - address: cosmos1xv9tklw7d82sezh9haa573wufgy59vmwe6xxe5
      amount: 100000000000000uosmo
    - address: cosmos1xv9tklw7d82sezh9haa573wufgy59vmwe6xxe5
      amount: 100000000000000uion
```

## `readinessProbe` (optional)
The `readinessProbe` directive allows you to set the readiness probe for the chain pods. This is useful when you want to test a chain with specific readiness probe settings.
Note, this is the same as the [kubernetes readiness probe](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)

```yaml
chains:
- id: osmosis-1
  name: osmosis
  numValidators: 1
  readinessProbe:
    exec:
      command:
      - bash
      - -c
      - |
        $CHAIN_BIN status 2>&1 | jq -e '.SyncInfo.catching_up == false'
    initialDelaySeconds: 60
    periodSeconds: 20
```

----------------------------------------

# Relayers YAML Syntax

We will go into details of the `relayers` top level directive in the Starship config file.

> Note: By default the `relayers` will open up a `transfer` port

## `name`
Name of the relayer, this is used as a unique identifier for the relayer.
```yaml
relayers:
- name: hermes-osmo-juno
  ...
```

## `type`
Type of the relayer to spin up. Currently we support:
* [hermes](https://github.com/informalsystems/hermes)
* [ts-relayer](https://github.com/confio/ts-relayer)
* [go-relayer](https://github.com/cosmos/relayer)
* [neutron-query-relayer](https://github.com/neutron-org/neutron-query-relayer)

```yaml
relayers:
- name: hermes-osmo-juno
  type: hermes
  ...
- name: ts-relayer-cosmos-juno
  type: ts-relayer
  ...
```

## `image` (optional)
If one wants to use a specific version of the relayer we support,
then it can be mentioned via this image directive.

Supported version of relayers can be found:
* [`hermes` versions](https://github.com/hyperweb-io/starship/pkgs/container/starship%2Fhermes/versions?filters%5Bversion_type%5D=tagged)
* [`ts-relayer` versions](https://github.com/hyperweb-io/starship/pkgs/container/starship%2Fts-relayer/versions?filters%5Bversion_type%5D=tagged)
* [`go-relayer` versions](https://github.com/hyperweb-io/starship/pkgs/container/starship%2Fgo-relayer/versions?filters%5Bversion_type%5D=tagged)
* [`neutron-query-relayer` versions](https://github.com/hyperweb-io/starship/pkgs/container/starship%2Fneutron-query-relayer/versions?filters%5Bversion_type%5D=tagged)

```yaml
relayers:
- name: hermes-osmo-juno
  type: hermes
  image: ghcr.io/hyperweb-io/starship/hermes:1.10.0
  ...
- name: ts-relayer-cosmos-juno
  type: ts-relayer
  image: ghcr.io/hyperweb-io/starship/ts-relayer:0.9.0
  ...
```

## `replicas`
Number of relayers to spin up. Currently we just support 1 replica for now.
```yaml
relayers:
- name: hermes-osmo-juno
  type: hermes
  replicas: 1
  ...
```

## `chains`
List of chains to connect to each other, via the Relayer.
Must be `id` of chains defined in the `chains` directive.

```yaml
chains:
- name: osmosis-1
  ...
- name: juno-2
  ...
- name: gaia-4
  ...

relayers:
- name: hermes-osmo-juno
  type: hermes
  replicas: 1
  chains:
  - osmosis-1
  - juno-2
- name: ts-relayer-osmo-gaia
  type: ts-relayer
  replicas: 1
  chains:
  - osmosis-1
  - gaia-4
```

## `config` (optional, hermes only)

For `type: hermes` relayer, we support overriding the default config params with the `config` directive.
User specified overrides are performed at the start.

> Note: Currently we only allow overrides of non chain specific values

```yaml
relayers:
- name: hermes-osmo-juno
  type: hermes
  replicas: 1
  chains:
  - osmosis-1
  - juno-1
  config:
    global:
      log_level: "error"
    rest:
      enabled: false
    telemetry:
      enabled: false
    ## event_source is a custom configuration, in hermes is set of each of the chains
    ## one just needs to mention the mode (push or pull), and we do the rest
    ## For reference: https://github.com/informalsystems/hermes/releases/tag/v1.6.0
    event_source:
      mode: "pull" # default is "push"
```

All allowed config params can be found in the [default values](https://github.com/hyperweb-io/starship/pull/185/files#diff-b6d755bc9ec9f0229fa0fc7beff17964f31889df81dbd5a16764ed8cdecbfa96R286-R310)
(Basically everything in [config.toml](https://github.com/informalsystems/hermes/blob/master/config.toml#L8...L111) file for the hermes relayer)

## `ports` (optional, hermes only)
Ports directive in the `relayers` directive is used for `kubectl port-forward` forwarding kubernetes service ports to local host.
```yaml
relayers:
- name: hermes-osmo-juno
  type: hermes
  replicas: 1
  chains:
  - osmosis-1
  - juno-1
  ports:
    rest: 3001
    exposer: 3002
```

Available endpoints on
* `rest` endpoints: https://hermes.informal.systems/documentation/rest-api.html?highlight=rest#rest-api
* `exposer` endpoint
  * `/create_channel` endpoint: Supports post request. Proto definition: https://github.com/hyperweb-io/starship/blob/main/starship/proto/exposer/service.proto#L47

One can use the exposer endpoint on the relayer to create channel from outside.
Note: Still an experimental feature on exposer

## `ics` (optional, hermes only)
ICS directive in the `relayers` directive is used for `hermes` relayer to specify the ICS setup to use

```yaml
chains:
- id: neutron-1
  name: neutron
  numValidators: 1
  ics:
    enabled: true
    provider: cosmoshub-4  # chain id of the provider chain
- id: cosmoshub-4
  name: cosmoshub
  numValidators: 1

relayers:
- name: neutron-cosmos
  type: hermes
  replicas: 1
  chains:
    - neutron-1
    - cosmoshub-4
  ics:
    enabled: true
    consumer: neutron-1
    provider: cosmoshub-4
```

If `ics` is enabled, then the `consumer` and `provider` chain must be specified. With that, we automagically open the consumer and provider ports for the relayer.

----------------------------------------

# Feature Toggles

Starship allows you to have toggles for some additional
services to run with your infra setup. A few of them are
mentioned below.

## Registry

Following the schema of the `cosmos/chain-registry` repos,
if enabled this service spins up a rest api chain-registry service
for the infra you spin up.

### Syntax

```yaml
registry:
  enabled: true  # enable registry service
  localhost: true # default: true, will set chain registry output with api endpoints pointing to localhost, using the chains[].ports for localhost endpoints
  ports:
    rest: 8081   # localhost port for redirecting traffic
  # Optional: resources directive, default cpu: 0.2, memory: 200M
  resources:
    cpu: 0.5
    memory: 200M
  # Optional: image directive, default image:ghcr.io/hyperweb-io/starship/registry:20230614-7173db2
  image: ghcr.io/hyperweb-io/starship/registry:20230614-7173db2
```

> Note: `registry.localhost` is set to true by default, meaning if the ports are specified in chains[].ports, then we set the various api endpoints of the chain registry to `localhost:<port>`. Make sure `rest`, `grpc`, `rpc` are set as [chain ports](https://starship.cosmology.tech/config/chains#ports). If set to false, then it is set to kubernetes internal DNS endpoints.

### Usage

Here is a list of available endpoints and how to use them:

| Endpoint                      | Returns
| ----------------------------- |----------------------------------------------------------------------------
| `/chain_ids`                  | List of all chain-ids in the current setup
|`/chains`                      | List of all chain items
| `/chains/{chain}`             | Chain schema for the given chain ids (`name` in the `chains` directive)
| `/chains/{chain}/assets`      | Assets of the given chain
| `/chains/{chain}/keys`        | List of mnemonics used for the setup
| `/ibc`                        | List all ibc info for all chains
| `/ibc/{chain_1}/{chain_2}`    | IBC information between the 2 chains specified

Proto definition for the service is [here](https://github.com/hyperweb-io/starship/blob/main/starship/proto/registry/service.proto)

## Explorer
In order to provide a full fledged emulation environment, we have
a handy toggle to spin up an explorer for the infra.

Currently we support only Ping Pub explorer, but we will be adding more

### Syntax

```yaml
explorer:
  enabled: true  # enable registry service
  type: ping-pub # currently only support for ping-pub explorer
  ports:
    rest: 8080   # localhost port for redirecting traffic
  # Optional: resources directive, default cpu: 1, memory: 2Gi
  resources:
    cpu: 2
    memory: 4Gi
  # Optional: image directive, default image: ghcr.io/hyperweb-io/starship/ping-pub:6b7b0d096946b6bcd75d15350c7345da0d4576db
  image: ghcr.io/hyperweb-io/starship/ping-pub:6b7b0d096946b6bcd75d15350c7345da0d4576db
```

Available versions for the explorer can be found [here](https://github.com/hyperweb-io/starship/pkgs/container/starship%2Fexposer/versions?filters%5Bversion_type%5D=tagged)

### Usage

After performing `port-forward`, open explorer at: http://localhost:8080

## Ingress
In order to get external traffic into Starship, one can use the `ingress` directive to
create ingress rules on the domain.

> NOTE: Prerequists include installing `cert-issuer` and `nginx-ingress` controller in the k8s cluster.
Domain specified, needs to be pointing to the k8s cluster in which Starship is deployed
For Starship devnets, we use: [`cluster-setup.sh`](https://github.com/hyperweb-io/starship/blob/main/starship/scripts/cluster-setup.sh)
as the setup script. Need to run just once.

### Syntax
```yaml
ingress:
  enabled: true
  type: nginx
  # host must be a wildcard entry, so that we can use the wildcard to create
  # service specific ingress rules.
  host: "*.thestarship.io"
  certManager:
    issuer: "cert-issuer"
```

Above will create following endpoints with the domain, and valid certs:
* Explorer at: `https://explorere.<host>` (if enabled)
* Registry at: `https://registry.<host>` (if enabled)
* Chains at:
  * RPC endpoint: `https://rpc.<chain-id>-genesis.<host>`
  * Rest endpoint: `https://rest.<chain-id>-genesis.<host>`
  * Chain exposer: `https://rest.<chain-id>-genesis.<host>/exposer`
  * Chain faucet: `https://rest.<chain-id>-genesis.<host>/faucet`
* Relayers at:
  * Rest endpoint: `https://rest.<relayer-type>-<relayer-name>.<host>`
  * Exposer endpoint: `https://rest.<relayer-type>-<relayer-name>.<host>/exposer`

## Frontends
Starship provides flexible frontend services that can be configured to run alongside your infrastructure.
Frontend services can be customized with different images, ports, and resource allocations.

### Syntax

```yaml
frontends:
  - name: my-frontend
    type: custom
    image: ghcr.io/hyperweb-io/starship/frontend:latest
    replicas: 1
    ports:
      rest: 3000
    env:
      API_URL: http://localhost:8080
    resources:  # optional, defaults will be used
      cpu: 0.5
      memory: 512Mi
```

### Configuration Options

| Field | Type | Description | Default |
|-------|------|-------------|---------|
| `name` | string | Name of the frontend service | Required |
| `type` | string | Type of frontend service (e.g., custom) | Required |
| `image` | string | Docker image for the frontend service | Required |
| `replicas` | integer | Number of frontend service replicas | Required |
| `ports.rest` | integer | Port for HTTP/REST traffic | Required |
| `env` | object | Environment variables for the frontend service | {} |
| `resources` | object | Resource limits and requests | Not required |

----------------------------------------

# Ethereum Chain Configuration in Starship

Ethereum support in Starship allows users to deploy Ethereum execution and consensus (beacon) nodes using simple configurations.
This section details how to configure Ethereum-based chains within Starship.

## Ethereum Execution Chain

The Ethereum Execution Chain, or Ethereum client, runs the execution layer of the Ethereum network, handling smart contract execution and maintaining state.

### Config
```yaml
chains:
  - id: 1337
    name: ethereum
    numValidators: 1
    ports:
      rest: 8545
      rpc: 8551
      ws: 8546   # optional
```

### Optional Configs
```yaml
chains:
  - id: 1337
    name: ethereum
    numValidators: 1
    ports:
      rest: 8545
      rpc: 8551
    image: ghcr.io/hyperweb-io/starship/ethereum/client-go:latest
    config:
      beacon:
        enabled: true
        image: "ghcr.io/hyperweb-io/starship/prysm/beacon-chain:v5.2.0"
        numValidator: 1
      validator:
        enabled: true
        image: "ghcr.io/hyperweb-io/starship/prysm/validator:v5.2.0"
        numValidator: 1
      prysmctl:
        image: "ghcr.io/hyperweb-io/starship/prysm/cmd/prysmctl:v5.2.0"
```