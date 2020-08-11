# WooCommerce End to End Tests

Automated end-to-end tests for WooCommerce.

## Table of contents

- [Pre-requisites](#pre-requisites)
  - [Install NodeJS](#install-nodejs)
  - [Install Docker](#install-docker)
- [Configuration](#configuration)
  - [Test Environment](#test-environment)
  - [Environment Variables](#environment-variables)
  - [Jest test sequencer](#jest-test-sequencer)      
- [Running tests](#running-tests)
  - [Prep work for running tests](#prep-work-for-running-tests)
  - [How to run tests in headless mode](#how-to-run-tests-in-headless-mode) 
  - [How to run tests in non-headless mode](#how-to-run-tests-in-non-headless-mode)
  - [How to run an individual test](#how-to-run-an-individual-test)
  - [How to skip tests](#how-to-skip-tests)
- [Guide for writing e2e tests](#guide-for-writing-e2e-tests) 
  - [Tools for writing tests](#tools-for-writing-tests)
  - [Creating test structure](#creating-test-structure)
  - [Writing the test](#writing-the-test)
  - [Best practices](#best-practices)
- [Debugging tests](#debugging-tests)
- [Docker basics](#docker-basics)
  - [How to stop and restart Docker](#how-to-stop-and-restart-docker)
  - [How to stop Docker and do a clean restart](#how-to-stop-docker-and-do-a-clean-restart)

## Pre-requisites

### Install NodeJS

Install NodeJS on Mac:

```bash
brew install node 
```

### Install Docker

Install Docker Desktop if you don't have it installed:

- [Docker Desktop for Mac](https://docs.docker.com/docker-for-mac/install/)
- [Docker Desktop for Windows](https://docs.docker.com/docker-for-windows/install/)

Once installed, you should see `Docker Desktop is running` message with the green light next to it indicating that everything is working as expected.

Note, that if you install docker through other methods such as homebrew, for example, your steps to set it up will be different. The commands listed in steps below may also vary.

## Configuration

This section explains how e2e tests are working behind the scenes. These are not instructions on how to build environment for running e2e tests and run them. If you are looking for instructions on how to run e2e tests, jump to [Running tests](#running-tests). 

### Test Environment

We recommend using Docker for running tests locally in order for the test environment to match the setup on Travis CI (where Docker is also used for running tests). [An official WordPress Docker image](https://github.com/docker-library/docs/blob/master/wordpress/README.md) is used to build the site. Once the site using the WP Docker image is built, the current WooCommerce dev branch is being copied to the `plugins` folder of that newly built test site. No WooCommerce Docker image is being built or needed.

### Environment Variables

During the process of Docker building a container with test site for running tests, site URL is being set. Admin and customer users are also being created in advance with details specified in the `docker-compose.yaml` file. As a result, there is `./tests/e2e/config/default.json` file that contains pre-set variables needed to run the test:

```
{
  "url": "http://localhost:8084/",
  "users": {
    "admin": {
      "username": "admin",
      "password": "password"
    },
    "customer": {
      "username": "customer",
      "password": "password"
    }
  }
}
```

If you changed either site URL or one of the users details in the  `docker-compose.yaml` file, you'd need to copy the content of the `default.json`, paste it to `test-e2e.json` and edit it further there to match your own setup. 

### Jest test sequencer

[Jest](https://jestjs.io/) is being used to run e2e tests. By default, jest runs tests ordered by the time it takes to run the test (the test that takes longer to run will be run first, the test that takes less time to run will run last). Jest sequencer introduces tools that can be used to specify the order in which the tests are being run. In our case, they are being run in alphabetical order of the directories where tests are located. This way, tests in the new directory `activate-and-setup` will run first. 

Setup Wizard e2e test (located in `activate-and-setup` directory) will run before all other tests. This will allow making sure that WooCommerce is activated on the site and for the setup wizard to be completed on a brand new install of WooCommerce. 

## Running tests

### Prep work for running tests

- `cd` to the WooCommerce plugin folder

- `git checkout master` or checkout the branch where you need to run tests 

- Run `npm install`

- Run `npm install jest --global`

- Run `composer install --no-dev`

- Run `npm run build`

- Run the following command to build the test site using Docker: `npm run docker:up` and watch the site being built. Note that it may take a few minutes the first time you do that. The process is considered completed when the messages letting you know that WordPress was installed, WooCommerce was activated and users created will be displayed:

```
wordpress-cli_1             | Success: WordPress installed successfully.
wordpress-cli_1             | Plugin 'woocommerce' activated.
wordpress-cli_1             | Success: Activated 1 of 1 plugins.
wordpress-cli_1             | Success: Created user 2.
woocommerce_wordpress-cli_1 exited with code 0
woocommerce_wordpress-cli_1 exited with code 0
```

For more Docker commands, scroll down to [Docker basics](#docker-basics).

- Open new terminal window and `cd` to the current branch again.

- Run the following command to make sure the containers were built and running: `docker ps`. You should see the 2 following containers: 

- `woocommerce_wordpress-woocommerce-dev`
- `mariadb:10.4`

- Navigate to `http://localhost:8084/`. If everything went well, you should be able to access the site.  

### How to run tests in headless mode

To run e2e tests in headless mode use the following command:

```bash
npm run test:e2e
```

### How to run tests in non-headless mode

Tests are being run headless by default. However, sometimes it's useful to observe the browser while running tests. To do so, you can run tests in a non-headless (dev) mode:
                                    
```bash
npm run test:e2e-dev
```

The dev mode also enables SlowMo mode. SlowMo slows down Puppeteer’s operations so we can better see what is happening in the browser. You can adjust the SlowMo value by copying `/tests/e2e/env/config/jest.puppetee.config.js` to `/tests/e2e/config` and editing the value in that file. The default `PUPPETEER_SLOWMO=50` means test actions will be slowed down by 50 milliseconds.

### How to run an individual test

To run an individual test, use the direct path to the spec. For example:

```bash
npm run test:e2e ./tests/e2e/specs/wp-admin/wp-admin-product-new.test.js
``` 

### How to skip tests

To skip the tests, use `.only` in the relevant test entry to specify the tests that you do want to run. 

For example, in order to skip Setup Wizard tests, add `.only` to the login and activation tests as follows in the `setup-wizard.test.js`:

```
it.only( 'Can login', async () => {}
```

```
it.only( 'Can make sure WooCommerce is activated. If not, activate it', async () => {}
```

As a result, when you run `setup-wizard.test.js`, only the login and activate tests will run. The rest will be skipped. You should see the following in the terminal:

```
 PASS  tests/e2e/specs/activate-and-setup/setup-wizard.test.js (11.927s)
  Store owner can login and make sure WooCommerce is activated
    ✓ Can login (7189ms)
    ✓ Can make sure WooCommerce is activated. If not, activate it (1187ms)
  Store owner can go through store Setup Wizard
    ○ skipped Can start Setup Wizard
    ○ skipped Can fill out Store setup details
    ○ skipped Can fill out Payment details
    ○ skipped Can fill out Shipping details
    ○ skipped Can fill out Recommended details
    ○ skipped Can skip Activate Jetpack section
    ○ skipped Can finish Setup Wizard - Ready! section
  Store owner can finish initial store setup
    ○ skipped Can enable tax rates and calculations
    ○ skipped Can configure permalink settings
```

You can also use `.skip` in the same fashion. For example:

```
it.skip( 'Can start Setup Wizard', async () => {}
```

Finally, you can aply both `.only` and `.skip` to `describe` part of the test:

```
describe.skip( 'Store owner can go through store Setup Wizard', () => {}
```

## Guide for writing e2e tests

### Tools for writing tests

We use the following tools to write e2e tests:

- [Puppeteer](https://github.com/GoogleChrome/puppeteer) – a Node library which provides a high-level API to control Chrome or Chromium over the DevTools Protocol
- [jest-puppeteer](https://github.com/smooth-code/jest-puppeteer) – provides all required configuration to run tests using Puppeteer
- [expect-puppeteer](https://github.com/smooth-code/jest-puppeteer/tree/master/packages/expect-puppeteer) – assertion library for Puppeteer

Tests are kept in `tests/e2e/specs` folder.

The following packages are being used to write tests:

- `e2e-test-utils` - End-To-End (E2E) test utils for WordPress. You can find the full list of utils [here](https://github.com/WordPress/gutenberg/tree/master/packages/e2e-test-utils);
- `puppeteer-utils` - Utilities and configuration for running puppeteer against WordPress. See details in the [package's repository](https://github.com/Automattic/puppeteer-utils). 

### Creating test structure

It is a good practice to start working on the test by identifying what needs to be tested on the higher and lower levels. For example, if you are writing a test to verify that merchant can create virtual product, the overview of the test will be as follows:

- Merchant can create virtual product
  - Merchant can log in
  - Merchant can create virtual product
  - Merchant can verify that virtual product was created   
  
Once you identify the structure of the test, you can move on to writing it. 

### Writing the test

The structure of the test serves as a skeleton for the test itself. You can turn it into a test by using `describe()` and `it()` methods of Jest:

- [`describe()`](https://jestjs.io/docs/en/api#describename-fn) - creates a block that groups together several related tests;
- [`it()`](https://jestjs.io/docs/en/api#testname-fn-timeout) - actual method that runs the test. 

Based on our example, the test skeleton would look as follows:

```
describe( 'Merchant can create virtual product', () => {
	it( 'merchant can log in', async () => {

	} );

	it( 'merchant can create virtual product', async () => {

	} );

	it( 'merchant can verify that virtual product was created', async () => {

	} );
} );
```

Next, you can start filling up each section with relevant functions (test building blocks). Note, that we have the `@woocommerce/e2e-utils` package where many reusable helper functions can be found for writing tests. For example, `flows.js` of `@woocommerce/e2e-utils` package contains `StoreOwnerFlow` object that has `login` method. As a result, in the test it can be used as `await StoreOwnerFlow.login();` so the first `it()` section of the test will become:

```
it( 'merchant can log in', async () => {
      await StoreOwnerFlow.login();
	} );
```

Moving to the next section where we need to actually create a product. You will find that we have a reusable function such as `createSimpleProduct()` in the `components.js` of `@woocommerce/e2e-utils` package. Note that this function should not be used for this test because the way simple product is being created in this function is by using WooCommerce REST API. Because this is not how the merchant would typically create a virtual product, we would need to test it by writing actual steps for creating a product in the test. 

`createSimpleProduct()` should be used in tests where you need to test something else than creating a simple product. In other words, this function exists in order to quickly fill the site with test data required for running tests. For example, if you want to write a test that will verify that shopper can place a product to the cart on the site, you can use `createSimpleProduct()` to create a product to test the cart. 


### Best practices

- It is best to keep the tests inside `describe()` block granular as it helps to debug the test if it fails. When the test is done running, you will see the result along with the breakdown of how each of the test sections performed. If one of the tests within `describe()` block fails, it will be shown as follows:

```
FAIL ../specs/front-end/front-end-my-account.test.js (9.219s)
  My account page
    ✓ allows customer to login (2924ms)
    ✓ allows customer to see orders (1083ms)
    x allows customer to see downloads (887ms)
    ✓ allows customer to see addresses (1161ms)
    ✓ allows customer to see account details (1066ms)
```

In the example above, you can see that `allows customer to see downloads` part of the test failed and can start looking at it right away. Without steps the test goes through being detailed, it is more difficult to debug it. 

## Debugging tests 

For Puppeteer debugging, follow [Google's documentation](https://developers.google.com/web/tools/puppeteer/debugging).   

## Docker basics

### How to stop and restart Docker

- Press `Ctrl+C` in the terminal window where the containers are running 
- Stop the container(s) using the following command: `npm run docker:down`
- Restart the containers using the following command: `npm run docker:up`
