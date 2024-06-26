# Tattletale
A Program To Tell Dom/mes When Their Subs Do Stuff On Chaster

# Architecture
Host web worker in cloudflare infra for recieving webhhoks and sending discord messages

Host html pages needed to get chaster extension working with cloudflare pages
- Main page
- Configuration page

Register chaster webhook (https://docs.chaster.app/api/extensions-api/create-your-extension/webhooks)
- Sends request to cloudflare worker
- If event payload does not contain all needed info query history endpoint to see if there has been a new event not notified of
    - https://api.chaster.app/api#/Locks/LockController_getLockHistory
- Cloudflare worker looks up chaster user's discord handle somehow
    - Register with bot command? 
    - How to verify identity?
        - Enter discord handle in extension configuraion?
        - Metadata on chaster user object?
- Open DM channel if it does not already exist (channels can live for a long time it seems?)
    - https://www.postman.com/discord-api/workspace/discord-api/request/23484324-049e3518-96a5-4412-aa76-ec8c2cd17beb
    - https://discord.com/developers/docs/resources/user#create-dm

- Send notification in DM channel
    - https://www.postman.com/discord-api/workspace/discord-api/request/23484324-eaf41e8b-35b4-4de9-95a8-df59a535d843
    - https://discord.com/developers/docs/resources/channel#create-message

## Open questions
Use discord User-Installable App? Maybe if we can only message "friends" this will work without being added to the server.


## Resources used

- [Discord Interactions API](https://discord.com/developers/docs/interactions/receiving-and-responding)
- [Cloudflare Workers](https://workers.cloudflare.com/) for hosting
- [Chaster API](https://docs.chaster.app/api/basics/introduction) to recieve notifications on behalf of the user

---

## Project structure

Below is a basic overview of the project structure:

```
├── .github/workflows/ci.yaml     -> Github Action configuration
├── public                        -> HTTP resources hosted by cloudflare pages
│   └── chaster                   
│       ├── configuration.html    -> GET  ${host}/chaster/configuration (chaster extension configuration page)
│       └── main.html             -> GET  ${host}/chaster/main          (chaster extension main page)
├── functions                     -> HTTP endpoints hosted by cloudflare pages functions
│   └── api                       
│       ├── chaster                
│       │   └── event.ts          -> POST ${host}/api/chaster/event
│       ├── discord               
│       │   └── interactions.ts   -> POST ${host}/api/discord/interactions (recieves and dispatches Discord events/commands)
│       └── index.ts              -> GET  ${host}/api 
├── src                           -> TypeScript modules that can be imported into the functions/api
│   ├── discord
│   │   ├── commands.ts           -> JSON payloads for commands
│   │   ├── client.ts             -> Discord fetch client for making requests to the discord api, typed by api.d.ts
│   │   ├── api.d.ts              -> Autogenerated types for the discord API (by typescript-openapi)
│   │   ├── messaging.ts          -> Interactions with the Discord API needed for sending DM's
│   │   └── register.ts           -> Sets up commands with the Discord API
│   ├── chaster
│   │   ├── client.ts             -> Chaster fetch client for making requests to the chaster api, typed by api.d.ts
│   │   ├── api.d.ts              -> Autogenerated types for the chaster API (by typescript-openapi)
│   │   └── fetch-api.ts          -> Chaster does not host its swagger.json spec, but embeds it in some js code, this fetches the spec to /tmp/chaster/openapi.json
│   └── env.ts
├── test
│   └── server.test.js            -> Tests for app
├── wrangler.toml                 -> Configuration for Cloudflare workers
├── package-lock.json
├── package.json
├── LICENSE
├── README.md
├── eslint.config.js
├── example.dev.vars
├── flake.lock
├── flake.nix
└── tsconfig.json
```

## Configuring project

Before starting, you'll need a [Discord app](https://discord.com/developers/applications) with the following permissions:

- `bot` with the `Send Messages` and `Use Slash Command` permissions
- `applications.commands` scope

> ⚙️ Permissions can be configured by clicking on the `OAuth2` tab and using the `URL Generator`. After a URL is generated, you can install the app by pasting that URL into your browser and following the installation flow.

## Creating your Cloudflare worker

Next, you'll need to create a Cloudflare Worker.

- Visit the [Cloudflare dashboard](https://dash.cloudflare.com/)
- Click on the `Workers` tab, and create a new service using the same name as your Discord bot

## Running locally

First clone the project:

```
git clone https://github.com/discord/cloudflare-sample-app.git
```

Then navigate to its directory and install dependencies:

```
cd cloudflare-sample-app
npm install
```

> ⚙️ The dependencies in this project require at least v18 of [Node.js](https://nodejs.org/en/)

### Local configuration

> 💡 More information about generating and fetching credentials can be found [in the tutorial](https://discord.com/developers/docs/tutorials/hosting-on-cloudflare-workers#storing-secrets)

Rename `example.dev.vars` to `.dev.vars`, and make sure to set each variable.

**`.dev.vars` contains sensitive data so make sure it does not get checked into git**.

### Register commands

The following command only needs to be run once:

```
$ npm run register
```

### Run app

Now you should be ready to start your server:

```
$ npm start
```

### Setting up ngrok

When a user types a slash command, Discord will send an HTTP request to a given endpoint. During local development this can be a little challenging, so we're going to use a tool called `ngrok` to create an HTTP tunnel.

```
$ npm run ngrok
```

![forwarding](https://user-images.githubusercontent.com/534619/157511497-19c8cef7-c349-40ec-a9d3-4bc0147909b0.png)

This is going to bounce requests off of an external endpoint, and forward them to your machine. Copy the HTTPS link provided by the tool. It should look something like `https://8098-24-22-245-250.ngrok.io`. Now head back to the Discord Developer Dashboard, and update the "Interactions Endpoint URL" for your bot:

![interactions-endpoint](https://user-images.githubusercontent.com/534619/157510959-6cf0327a-052a-432c-855b-c662824f15ce.png)

This is the process we'll use for local testing and development. When you've published your bot to Cloudflare, you will _want to update this field to use your Cloudflare Worker URL._

## Deploying app

This repository is set up to automatically deploy to Cloudflare Workers when new changes land on the `main` branch. To deploy manually, run `npm run publish`, which uses the `wrangler publish` command under the hood. Publishing via a GitHub Action requires obtaining an [API Token and your Account ID from Cloudflare](https://developers.cloudflare.com/workers/wrangler/cli-wrangler/authentication/#generate-tokens). These are stored [as secrets in the GitHub repository](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository), making them available to GitHub Actions. The following configuration in `.github/workflows/ci.yaml` demonstrates how to tie it all together:

```yaml
release:
  if: github.ref == 'refs/heads/main'
  runs-on: ubuntu-latest
  needs: [test, lint]
  steps:
    - uses: actions/checkout@v3
    - uses: actions/setup-node@v3
      with:
        node-version: 18
    - run: npm install
    - run: npm run publish
      env:
        CF_API_TOKEN: ${{ secrets.CF_API_TOKEN }}
        CF_ACCOUNT_ID: ${{ secrets.CF_ACCOUNT_ID }}
```

### Storing secrets

The credentials in `.dev.vars` are only applied locally. The production service needs access to credentials from your app:

```
$ wrangler secret put DISCORD_TOKEN
$ wrangler secret put DISCORD_PUBLIC_KEY
$ wrangler secret put DISCORD_APPLICATION_ID
```

## Questions?

Feel free to post an issue here, or reach out to [@justinbeckwith](https://twitter.com/JustinBeckwith)!

# Credits
Project skeleton based on https://github.com/discord/cloudflare-sample-app by Justin Beckwith