---
id: end-to-end-test
title: Write an end to end test
---

## **Introduction**

Here we discuss the process of writing end to end (e2e) tests for your wrapper. By end-to-end, we mean tests that make
calls to the wrapper through a Polywrap Client. E2e testing is both necessary and useful. It is necessary because calls 
between wrappers depend on the Polywrap Client. It is useful because such tests can more accurately represent the user
experience of your wrapper when it is integrated into an application.

We will focus on setting up a test environment and a Polywrap Client. Once these tasks are completed, testing
a wrapper is similar to testing a traditional SDK.

## **Prerequisites**

This guide uses [TypeScript](https://www.typescriptlang.org/) to implement the Polywrap test environment. We will also
use the [Jest](https://jestjs.io/) testing framework, though you can use any framework and follow without difficulty.

:::tip
In the future, TypeScript will be one of many supported languages for implementing plugins. As more Polywrap Clients are released in various languages, implementing plugins in those languages will be supported as well.
:::

You'll need the following installed as developer dependencies before testing your wrapper:

- [@web3api/cli](../../reference/cli/polywrap-cli)
- `@web3api/test-env-js`
- [@web3api/client-js](../../reference/clients/js/client-js)
- `jest`
- `@types/jest`

These can be added to your TypeScript project using the npm or yarn package managers:
```
npm install --save-dev @web3api/cli @web3api/test-env-js @web3api/client-js jest @types/jest
```
```
yarn add -D @web3api/cli @web3api/test-env-js @web3api/client-js jest @types/jest
```

## **The Polywrap Test Environment**

A Polywrap test environment is most useful when it integrates an IPFS node and an Ethereum test network, so that incomplete 
wrappers can be deployed and queried on your local machine.

The Polywrap CLI can be used to start a Polywrap-ready test environment:

```shell
w3 test-env up
```

The CLI can also be used to stop the test environment:

```shell
w3 test-env down
```

The test environment is a docker container with:
- A test server at **http://localhost:4040**
- A standard Ganache Ethereum test network at **http://localhost:8545**
- A Ganache Ethereum mainnet fork test network at **http://localhost:8546**
- An IPFS node at **http://localhost:5001**

It also sets up an ENS contract at initialization, so you can build wrappers and deploy them to an ENS URI on your locally hosted testnet.

:::tip
The test server can be queried to obtain an IPFS provider, Ethereum provider, and ENS registry address.

```typescript
async function getProviders(): { ipfs: string; ethereum: string; ensAddress: string } {
  const { data: { ipfs, ethereum } } = await axios.get("http://localhost:4040/providers");
  const { data: { ensAddress } } = await axios.get("http://localhost:4040/deploy-ens");
  return { ipfs, ethereum, ensAddress };
}
```

However, this guide will use the `@web3api/test-env-js` package instead. The `@web3api/test-env-js` can be used to start 
and stop this same test environment programmatically.
:::

## **Starting and stopping a Polywrap Test environment with @web3api/test-env-js**

The `@web3api/test-env-js` package includes functions for starting and stopping a Polywrap test environment.

The `initTestEnvironment` function takes no arguments and returns a `TestEnvironment` object, which contains
five strings that can be used to interact with the test environment:

```typescript
interface TestEnvironment {
  ipfs: string; // IPFS provider
  ethereum: string; // Ethereum provider
  ensAddress: string; // ENS registry address
  registrarAddress: string; // ENS registrar address
  reverseAddress: string; // ENS reverse registrar address
  resolverAddress: string; // ENS resolver address
}
```

The `stopTestEnvironment` function takes no arguments and tears down a test environment if one is running.

To see these in action, let's start a new file where we will write our first test. Like many testing frameworks in
JavaScript, the Jest framework includes hooks that run before and after all tests. I've added a call to `initTestEnvironment`
in the `beforeAll` hook, so we can start our test environment before running our tests. I added a call to `stopTestEnvironment` 
in the `afterAll` hook to make sure the test environment does not continue running on our system after we finish testing.

```typescript
$snippet: js-e2e-test-init
```

## **Building and deploying a wasm wrapper for testing with @web3api/test-env-js**

To query a wasm wrapper in a test, we first need to build and deploy it. We can do this with `@web3api/test-env-js` using
the `buildAndDeployApi` function. The `buildAndDeploy` function requires an absolute path to the directory of your wrapper
project that contains the `web3api.yaml` manifest file, the values that were returned by `initTestEnvironment`, and 
optionally an ENS domain name.

The `buildAndDeployApi` function returns an ENS domain name and an IPFS CID. Either of these outputs can be used to call
your wrapper. If an ENS domain name was not provided when calling `buildAndDeployApi`, a name will be randomly selected for you.

Here we obtain the absolute path to our wrapper project in three steps. First we get the directory of the folder containing
our test script, using the node.js `__dirname` global variable. We then append the path from our test script file to the
manifest directory. Finally, we import the node.js feature `path`, and use `path.resolve` to get direct absolute path to
our wrapper project.

```typescript
$snippet: js-e2e-test-build
```

## **Setting up a Polywrap Client**

Now that we have deployed our wasm wrapper to a local test environment, we need to set up a Polywrap Client that can
query it. We can make sure our client is ready to query the wrapper by configuring three of the default plugin wrappers.

- The `ipfsPlugin` can be configured using the test environment's local IPFS node.
- The `ethereumPlugin` can be configured with Ethereum providers on the Ganache Ethereum test network and the Ganache
mainnet fork network from our test environment.
- The `ensPlugin` can be configured using the Ethereum address of the test environment's ENS registry contract.

### **Configure the IPFS Plugin**

The `ipfsPlugin` can be configured with a default IPFS provider and an array of fallback providers that will be tried if
the default provider fails.

```typescript
$snippet: js-e2e-test-config-ipfs-import

$snippet: js-e2e-test-config-ipfs
```

### **Configure the Ethereum Plugin**

The `ethereumPlugin` can be configured with Ethereum providers and signers. It accepts JSON RPC provider strings, 
Providers from the ethers.js package, and providers from the web3.js package.

To configure the plugin, we need to provide it with an object containing network names along with a provider. We can
optionally provide a signer as well, but by default the plugin will attempt to obtain a signer from the provider if it
is needed.

We can give the networks any name we want in our configuration. We can later use these names to choose which network to 
query. Here we will use the names "testnet" and "mainnet" to help us remember which test network we are using.

We also need to tell the Ethereum plugin which network should be used when a network is not specified. We do this by
setting the value of the `defaultNetwork` property to one of our network names.

```typescript
$snippet: js-e2e-test-config-ethereum-import

$snippet: js-e2e-test-config-ethereum
```

### **Configure the ENS Plugin**

The `ensPlugin` accepts an Ethereum address for the ENS registry contract on each of the Ethereum networks declared
in our Ethereum plugin configuration. We will only use the ENS registry deployed on our test environment's Ethereum
network.

```typescript
$snippet: js-e2e-test-config-ens-import

$snippet: js-e2e-test-config-ens
```

### **Create the Polywrap Client Instance**

Now we are ready to add the plugins to our Polywrap Client configuration and create the client. The URI's we assign in
our client configuration will redirect to the plugin instances when queried.

```typescript
$snippet: js-e2e-test-config
```

## **Deploy the Smart Contract**

The Simple Storage wrapper is designed to interact with a Simple Storage smart contract. We need to deploy an
instance of the contract to work with. While there are many tools developers can use to deploy a smart contract on a
local network, the Simple Storage API has a method `deployContract` we can use to get the job done. We'll call it before
running our test to make sure the contract is ready.

```typescript
$snippet: js-e2e-test-deploy
```

## **Generate TypeScript Types**

Since we are using TypeScript, we will want types to work with. It is possible to automatically generate TypeScript
types from a GraphQL schema using the Polywrap CLI's `app` command. 

Let's set up a `web3api.app.yaml` manifest in a new folder called `types`.

Before building our wrapper, we have a GraphQL schema for each of our modules. The `app` command is intended to be used 
with built wrappers, which have only one schema. We will provide the manifest with the path to the composed schema in 
our build folder. This means we need to build our wrapper before running tests.

```yaml title="web3api.app.yaml"
$start: yaml-e2e-test-app-manifest
```

We can then call the `app` command of the Polywrap CLI.

```shell
yarn w3 app codegen -m ./src/__tests__/types/web3api.app.yaml -c ./src/__tests__/types/w3
```

The generated output includes TypeScript types for the Simple Storage wrapper and its imports.

```typescript title="types.ts"
...
$snippet: js-e2e-test-types
...
```

## **Testing a Wrapper Method**

From this point, testing a function in your wrapper is no different from testing a traditional SDK. Instead of calling
a method in a traditional SDK, you will invoke your wrapper.

We will test the `setIpfsData` method we added to the SimpleStorage API in [Adding new mutation functions](./mutation-functions). 
For arguments, the `setIpfsData` method takes the Ethereum address of a deployed Simple Storage contract and the data 
the user wants to add to IPFS. It returns the IPFS hash of the data.

We will test the `setIpfsData` method using the string `"Hello from IPFS!"` as the data for our test case. To be sure
our method returns the correct IPFS hash, we can compare the method's return value to the value we get from the free 
[*File CID Checker*](https://app.pinata.cloud/cidchecker) service provided by [Pinata](https://www.pinata.cloud/).

```typescript title="Final test file"
$snippet: js-e2e-test-final
```