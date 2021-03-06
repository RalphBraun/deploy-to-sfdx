# SFDX Deployer

## Purpose

You have a dev hub, and an sfdx repo.  You'd like to let people spin up scratch orgs based on the repo, and these people might not feel like using a terminal, cloning the repo, loggin in to a dev hub, and executing sfdx commands.

* because they might not be developers (think admins, or even end users in a training scenario)
* because they might not be Salesforce developers (say you built an app and give your designer/CSS person github access to "make it cool")
* because you might have dev hub access and you don't want to give it to them
* because you want to let people test the app quickly
* because (like me) you're using it for workshops and demos

---

## Connected app setup

Create a connected app for JWT auth, with certificates, per the SFDX setup guide.

<https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_auth_jwt_flow.htm>

---

## Environment variables

### Required

* `HUB_USERNAME` the username from your dev hub
* `CONSUMERKEY` from your connected app

### Cloud only

* `REDIS_URL` starts with `redis://`

### Plus one of the following, depending on where you're running

* `JWTKEY` use only in heroku cloud.  Cut/paste your server.key, including the ---- lines
* `LOCAL_ONLY_KEY_PATH` don't use this in the cloud.  Put it in your local .env file, and it needs to be an absolute path

### optional

* `UA_ID` for google analytics measurement protocol
* `GITHUB_USERNAME_WHITELIST` lets you whitelist usernames.  It's a comma-separated list.  Ex: `mshanemc,andrew,bebraw`
* `GITHUB_REPO_WHITELIST` lets you whitelist username/repo combinations.  It's a comma-separated list. Ex: `mshanemc/DF17integrationWorkshops,torvalds/linux`
* `cycleTime` when the deploy queue is empty, how long, in seconds, to wait before checking again
* `HEROKU_API_KEY` lets you comine the deployer with my sfdx plugin to deploy heroku apps to a team.  If you're note using org pools, be sure to delete these heroku apps.  See [the plugin docs](https://github.com/mshanemc/shane-sfdx-plugins#sfdx-shaneherokurepodeploy) for how to use this
* org pools -- see below for details

What's whitelisting do?  Normally, this app will parse your orgInit.sh and throw an error if you're doing any funny business.  BUT if you're on the whitelist, the app owner trusts you and you can do things with bash metacharacters (think &&, |, >) and execute non-sfdx commands  (grep, rm, whatever!) etc.  BE CAREFUL!

Here's a heroku button so you can have your own instance of the Deployer

[![Deploy](https://www.herokucdn.com/deploy/button.svg)](https://heroku.com/deploy?template=https%3A%2F%2Fgithub.com%2Fmshanemc%2Fdeploy-to-sfdx)

---

## Heroku setup

The button will start this on free hobby dynos.  For real life, I'd recommend a pair of web 1x and a pair of workers (orgbuilder) at 1x.

If you're going to be doing a lot of users (imagine a workshop where lots of people are pushing the button at the same time) you can scale out more workers so the people last in line don't have to wait so long.  Otherwise, it'll spin while the workers are busy processing the deploy request queue.

The oneoffbuilder is the same as the orgbuilder but exits when it's finished with its queue item.  They do take about 30 seconds to spin up before starting to build an org, so having "live workers" is better.  These give you the ability to scale out work horizontally (booting up a new dyno will be faster than waiting in line behind several other deploys on the always-live workers)

---

## Architectural overview

Nodejs, express run the web server.
When the web server receives a request, it creates a unique deployID (user-repo-timestamp) and a message on the deploy queue (using Heroku Redis)
The server redirects the user to a web page which subscribes to a websocket

When a worker starts up, it auths to a devhub via its environment variables.
Then it listens to the deploy queue and executes jobs

* clone the repo into local filestorage
* execute the orgInit.sh script, or the default create/push/open flow if there isn't one
* drop the output results of these steps into a broadcast exchange
* delete the local folder and send the ALLDONE message

All the web servers are subscribed to the Redis pub/sub.  When they receive messages, they look at the deployID and send the messages down to the matching client.

It runs a plugin that give it powers SFDX doesn't out-of-the-box
<https://github.com/mshanemc/shane-sfdx-plugins>

Put plugins in the package.json dependencies, then linked from source in lib/hubAuth.js.  Feel free to add additional plugins using npm install some-plugin-of-yours and then add it in hubAuth.js.

## Org pools (optional, advanced!)

Building orgs that take too long?  Ever have one that doesn't get its DNS ready in time?  Know you're mostly deploying the same orgs all the time?

Org Pools are the answer.  You tell it which username/repo pairs, and how many orgs you'd like pre-built.  When the user requests one, you simply grab one from the pool.  Orgs in the pool are less than 12 hours old so they stay fresh.

There's 3 worker dynos, both off by default (leave them that way).

* If you want pools, use Heroku Scheduler to run the `poolwatcher` task up to every 10 minutes (as a one-off dyno).  If any pool orgs need to be created, it'll start up one-off dynoes to handle that
* run poolskimmer with Heroku Scheduler every hour or so--it'll check for expired orgs to help keep you within your limits.

Then, in your .env/heroku config vars, point the deployer to some url that returns json.
`POOLCONFIG_URL` = `https://where.yourstuff/is`.

Example code here, but feel free to generate it however you like.
<https://github.com/mshanemc/poolsConfig>

```json
[{
    user: 'mshanemc', //ex: lives at https://github.com/mshanemc/cg1
    repo: 'cg1',
    quantity: 4, //how many of this org to keep handy
    lifeHours: 12 //how long it should live, in hours
},
... //use 1 for each repo
]

```

There's a few more settings for pools

* `poolLoopTimeMin` how often the poolwatcher should check the pools (default=1)
* `skimmerTimeMin` how often the skimmer should run (default 60)

The worker checks every `poolLoopTimeMin` minutes to see if any pools are below their quantity and issues more deploy requests.  If the pool is empty when a request comes in, the deployer just builds an org the old-fashioned, slow way.

`poolwatcher` monitors ready and inprogress orgs, comparing them to your config targets, requests deployments, and runs the skimmer (checking for expired orgs)

`pooldeployer` should have 0 dynos running.  It runs as a one-off dyno called by *** poolwatcher* *

## Local Setup (Mac...others, who knows?)

You'll need to have a local filesystem structure that kinda replicates what we're doing on the heroku dyno.

```shell
mkdir tmp
```

NOTE: This will overwrite your dev hub default.  If you go back to doing other sfdx local work, be sure to re-auth to your hub.

Then put your JWT server.key file from the connected app in the /app/tmp folder.  (In heroku cloud, this'll load from the environment variable **JWTKEY** but heroku local doesn't load multiline variables easily, so that's why the manual placement of the key file is needed).

in your `process.env` file, put the Heroku Redis url (it can be the same as you're using in the cloud).  Don't commit this file to your repo.

then start this app with
`heroku local`

You can also run some local integration tests using mocha.  See /tests and use `mocha --watch tests/`

---

## Debugging on [non-local] Heroku

if **hosted-scratch-qa** is the name of your app

`heroku logs --tail -a hosted-scratch` will give you the logs.

`heroku redis:cli -a hosted-scratch -c hosted-scratch` will lets you look at your redis stuff and see what's in the queue

`heroku ps:exec -a hosted-scratch-qa --dyno=worker.1` lets you ssh into the dyno of your choice and take a look around, clean stuff up, etc.

---

## Testing

There's unit tests in tests/unitTests.  Run these to not break stuff.
Integration (tests/integrationTests) are harder.  You'll need some dynos running in your test environment.  Then
`export DEPLOYER_TESTING_ENDPOINT=[the url of your dev environment]`
`mocha tests/integrationTests/*Test.js`

This is deploying actual repos from github.  They're defined in the top of deployTest.js.  If you have repos you really care about, add them!

NOTE: This is using up your scratch org quotas.  The tests delete the orgs, so it's minimally wastefuly, but still expect it to take a while AND watch your daily limit.

## Setting up a repo (optional)

So you need a target repo to deploy (see examples below).  If your repo is simple, the tool will do the default behavior (create, push source, open).

But with an orgInit.sh file, you can list out all your sfdx commands and they'll be executed by the deployer.  Remember, no bash metacharacters and only sfdx commands are allowed (We can't let anyone run any arbitrary command on our servers...security, yo!)

That lets you create records, assign permsets, create users, install packages, run tests, generate passwords, and do anything you can do with an SFDX command

---

## Launcher URLs

There's not anything at / on the server.  Don't worry.  The only page you care about is `/launch` which takes 1 parameter `template`

So your path should be `https://whatever.herokuapp.com/launch?template=https://github.com/username/reponame`

Also handles branches on github, like `https://whatever.herokuapp.com/launch?template=https://github.com/username/reponame/tree/somebranch`

You can optionally pass in an `email` parameter so that the emails go to someone who's not the hub owner :)
`https://whatever.herokuapp.com/launch?template=https://github.com/username/reponame&email=shane.mclaughlin@salesforce.com`

You can pass in `&email=required` to force user to a form where they add their email.  This is useful if they need to deal with emails from the org, or password resets, or security challenges, etc.

---

## Example Repos with orgInit.sh scripts

<https://github.com/mshanemc/DF17integrationWorkshops> [![Deploy](https://raw.githubusercontent.com/mshanemc/deploy-to-sfdx/master/assets/sfdx_it_now.png)](https://hosted-scratch.herokuapp.com/launch?template=https://github.com/mshanemc/DF17integrationWorkshops)

<https://github.com/mshanemc/community-apps-workshop-df17> [![Deploy](https://raw.githubusercontent.com/mshanemc/deploy-to-sfdx/master/assets/sfdx_it_now.png)](https://hosted-scratch.herokuapp.com/launch?template=https://github.com/mshanemc/community-apps-workshop-df17)

<https://github.com/mshanemc/process-automation-workshop-df17> [![Deploy](https://raw.githubusercontent.com/mshanemc/deploy-to-sfdx/master/assets/sfdx_it_now.png)](https://hosted-scratch.herokuapp.com/launch?template=https://github.com/mshanemc/process-automation-workshop-df17)

<https://github.com/mshanemc/df17-community-content-workshop> [![Deploy](https://raw.githubusercontent.com/mshanemc/deploy-to-sfdx/master/assets/sfdx_it_now.png)](https://hosted-scratch.herokuapp.com/launch?template=https://github.com/mshanemc/df17-community-content-workshop)

<https://github.com/mshanemc/df17AppBuilding> [![Deploy](https://raw.githubusercontent.com/mshanemc/deploy-to-sfdx/master/assets/sfdx_it_now.png)](https://hosted-scratch.herokuapp.com/launch?template=https://github.com/mshanemc/df17AppBuilding)

email example:
<http://localhost:8543/launch?template=https://github.com/mshanemc/df17IntegrationWorkshops&email=shane.mclaughlin@salesforce.com>