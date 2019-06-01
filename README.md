# EOSIO WebAuthn Browser Based Signature Example App

[![EOSIO Labs](https://img.shields.io/badge/EOSIO-Labs-5cb3ff.svg)](#about-eosio-labs)

# Overview

This example application demonstrates WebAuthn based:

- blockchain account creation for private chains
- blockchain transaction signatures for private chains

## About EOSIO Labs

EOSIO Labs repositories are experimental.  Developers in the community are encouraged to use EOSIO Labs repositories as the basis for code and concepts to incorporate into their applications. Community members are also welcome to contribute and further develop these repositories. Since these repositories are not supported by Block.one, we may not provide responses to issue reports, pull requests, updates to functionality, or other requests from the community, and we encourage the community to take responsibility for these.

# Building and running this app
Running this app will create an HTTP server listening on 0.0.0.0:8000 meaning it would typically be accessible via http://localhost:8000 (and from non-localhost as well). However, webauthn requires usage from an HTTPS origin. You will need to place an HTTPS proxy in front of the server and modify server source code with the resulting domain name and port.

## Running Locally with nodeos docker image

### Prerequisites
- Install Docker
- Install Docker Compose (included with Docker for Desktop on Mac)
- Install HAProxy
   - On Mac: `brew install haproxy`
   - On Ubuntu: `apt-get install haproxy=1.8.\*`
- Install nodejs

### Setup
The following command will generate the required SSL certificate, perform the required yarn install/setup, and execute all necessary actions against the chain.

`yarn setup`

### Starting/Stopping
To start the chain, haproxy, and the app:
`yarn start`

To stop all of the above, CTRL+C the app and then:
`yarn stop`

### Cleaning the chain
To reset the chain:
`yarn clean`

### Running any arbitrary cleos command
`yarn cleos <parameters to cleos>`

### If the wallet locks,
`yarn unlock-wallet`

## Running Locally with nodeos built from source

### Prerequisites
- Install HAProxy
- Install nodejs

### HTTPS proxy via self-signed localhost certificate:

The following describes one way of placing an HTTPS proxy in front of the server via the program `haproxy` along with a self-signed certificate that you instruct your browser to trust.

### Create a certificate and private key:

```
$ openssl req \
-x509 \
-nodes \
-new \
-newkey rsa:4096 \
-keyout localhostca.key \
-out localhostca.crt \
-sha256 \
-days 3650 \
-config <(cat <<EOF

[ req ]
prompt = no
distinguished_name = subject
x509_extensions    = x509_ext

[ subject ]
commonName = localhost

[ x509_ext ]
subjectAltName = @alternate_names
basicConstraints=CA:TRUE,pathlen:0

[ alternate_names ]
DNS.1 = localhost

EOF
)
```

### Combine the .cert file and the .key into a .pem file:

```
$ cat localhostca.crt localhostca.key > localhostca.pem
```

### Make a haproxy.cfg file which will listen on HTTPS port 7000 and forward to port 8000:

```
defaults
   timeout connect 10000ms
   timeout client  50000ms
   timeout server  50000ms

frontend https-in
   bind *:7000 ssl crt localhostca.pem
   use_backend http_backend

backend http_backend
   server server1 127.0.0.1:8000
```

### Run the program haproxy:

```
$ haproxy -f haproxy.cfg
```

You will need to add the `localhostca.crt` certificate file as a trusted certificate in your browser. The steps to accomplish this vary based on operating system and browser.

Note that it would be good practice to remove this certificate from being trusted at the conclusion of your development.

### Modification of server origin in source:

The server domain name and port must be specified in the source of this application. Modify the `socketUrl` in `src/client/ClientRoot.tsx` to be the valid HTTPS url to the HTTPS proxy. If you performed the self-signed & haproxy instructions above you would change this to `https://localhost:7000`

# Building and running the server

```
$ git submodule update --init --recursive
$ yarn
$ rm -rf node_modules/eosjs && bash -c "cd external/eosjs && yarn" && yarn add file:external/eosjs
$ rm -rf dist && yarn server
```

The server is now running and you can now access it via the HTTPS proxy you just created.

# Limitations

* server only supports a single client (browser tab) connecting to it
* server only supports a single user; it assumes all stored keys belong to that user
* server only supports a single hardware key; it assumes all stored keys came from that key
* `createKey` in `ClientRoot.tsx`
    * assumes it's served from localhost; other domains will break
    * hard codes the `user` field
    * hard codes the `challenge` field; this should be randomly generated by the server then checked by the server
* If `keys.json` (maintained by the server) is lost, then the keypairs become inaccessible. It's important to note that webauthn doesn't provide a way to recover from this.

# Create test chain with webauthn support

This starts a minimal test chain with some upgrades activated. This guide assumes you're already familiar with creating and using test chains.

**Note: If using the Docker setup, only the account creation and token transfers must be done.
All other steps are handled in the `setup` script.**

### Prerequisites:

* Install `hidapi`; a library for communicating with USB and Bluetooth HID devices
* Install `EOSIO/eos` branch `webauthn_wip`
* Install `EOSIO/eosio.cdt` branch `v1.6.x`
* Use `EOSIO/eosio.cdt` to build `EOSIO/eosio.contracts` branch `v1.7.0-rc1`

### Create a wallet, or unlock existing:

Use `$ cleos wallet create` or `$ cleos wallet unlock`

### Add the default key:

```
$ cleos wallet import --private-key 5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3
```

### Create a test chain:

This will create a fresh chain in directory `my-chain`. `chain_api_plugin` and `producer_api_plugin` enable the commands which follow.

```
$ mkdir my-chain
$ cd my-chain
$ nodeos --data-dir ./data --config-dir ./config --plugin eosio::chain_api_plugin --plugin eosio::producer_api_plugin --http-validate-host 0 --access-control-allow-origin "*" -p eosio -e 2>stderr &
```

### Preactivate feature:

Do this before installing eosio.bios:

```
$ curl -X POST http://127.0.0.1:8888/v1/producer/schedule_protocol_feature_activations -d '{"protocol_features_to_activate": ["0ec7e080177b2c02b278d5088611686b49d739925a92d9bfcacd7fc6b74053bd"]}' | jq
```

### Install contracts, activate webauthn keys, and create tokens:

```
$ cleos create account eosio eosio.token EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
$ cleos set contract eosio <path-to-eosio.contracts>/build/contracts/eosio.bios -p eosio
$ cleos set contract eosio.token <path-to-eosio.contracts>/build/contracts/eosio.token -p eosio.token
$ cleos push action eosio activate '["4fca8bd82bbd181e714e283f83e1b45d95ca5af40fb89ad3977b653c448f78c2"]' -p eosio
$ cleos push action eosio.token create '["eosio","10000000.0000 SYS"]' -p eosio.token
$ cleos push action eosio.token issue '["eosio","10000000.0000 SYS",""]' -p eosio
```

### Use app to generate 2 keys. Use them below:

```
$ cleos create account eosio usera <a-generated-public-key>
$ cleos create account eosio userb <a-generated-public-key>
$ cleos create account eosio userc <a-generated-public-key>
$ cleos create account eosio userd <a-generated-public-key>
```

![](/screenshots/screenshot-i.png?raw=true")

```
$ cleos transfer eosio usera "1000.0000 SYS" ""
$ cleos transfer eosio userb "1000.0000 SYS" ""
$ cleos transfer eosio userc "1000.0000 SYS" ""
$ cleos transfer eosio userd "1000.0000 SYS" ""
```

![](/screenshots/screenshot-ii.png?raw=true")

## Contributing

[Contributing Guide](./CONTRIBUTING.md)

[Code of Conduct](./CONTRIBUTING.md#conduct)

## License

[MIT](./LICENSE)

## Important

See LICENSE for copyright and license terms.  Block.one makes its contribution on a voluntary basis as a member of the EOSIO community and is not responsible for ensuring the overall performance of the software or any related applications.  We make no representation, warranty, guarantee or undertaking in respect of the software or any related documentation, whether expressed or implied, including but not limited to the warranties of merchantability, fitness for a particular purpose and noninfringement. In no event shall we be liable for any claim, damages or other liability, whether in an action of contract, tort or otherwise, arising from, out of or in connection with the software or documentation or the use or other dealings in the software or documentation. Any test results or performance figures are indicative and will not reflect performance under all conditions.  Any reference to any third party or third-party product, service or other resource is not an endorsement or recommendation by Block.one.  We are not responsible, and disclaim any and all responsibility and liability, for your use of or reliance on any of these resources. Third-party resources may be updated, changed or terminated at any time, so the information here may be out of date or inaccurate.  Any person using or offering this software in connection with providing software, goods or services to third parties shall advise such third parties of these license terms, disclaimers and exclusions of liability.  Block.one, EOSIO, EOSIO Labs, EOS, the heptahedron and associated logos are trademarks of Block.one.

Wallets and related components are complex software that require the highest levels of security.  If incorrectly built or used, they may compromise users’ private keys and digital assets. Wallet applications and related components should undergo thorough security evaluations before being used.  Only experienced developers should work with this software.
