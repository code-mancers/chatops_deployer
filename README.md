# ChatopsDeployer

A lightweight Sinatra app that deploys staging apps from Github repository branches
using docker and friends.

Automatically when you submit pull requests:
![github-webhook](https://s3-ap-southeast-1.amazonaws.com/uploads-ap.hipchat.com/39906/538857/k7WU2wiVbLzQMu6/upload.png "Github Webhook")

Or whenever you feel like:
![chatops](https://s3-ap-southeast-1.amazonaws.com/uploads-ap.hipchat.com/39906/538857/YFBfOlZATG5ESNx/upload_censored.jpg "Chatops")

Features:

* Disposable environments using docker-compose
* Simple API to deploy apps in github repos
* Hubot ready
* Works with Github webhooks
* Supports multi-container environments
* Support for secret management using Vault

## Requirements

Install the following on a dedicated server with root access.

1. docker-compose - For running multi-container apps
2. nginx - For setting up a subdomain for each deployment

TODO: setup script to install requirements on Ubuntu 14.04

### Server setup

1. Add the Github OAuth token to `~/.netrc`:

        machine github.com login <oauth_token> password

2. Disable prompt when cloning repositories by adding this to `~/.ssh/config`:

        Host github.com
          StrictHostKeyChecking no

3. Create an ssh key-pair and add it to your github user:

        ssh-keygen
        # Copy ~/.ssh/id_rsa.pub and add it to keys of the github account

4. Setup frontail
Frontail is a node module which will tail your logs and expose it over an HTTP endpoint
which you can see in your browser. `npm install -g frontail` and `frontail /var/logs/chatops_deployer.log`

5. `docker login` (if pulling private docker images)

6. Cron job to cleanup unused docker containers and images

Install `docker-gc` and set up an hourly cron job to cleanup unused containers
and images. Follow instructions [here](https://github.com/spotify/docker-gc).
To prevent the cache container from getting GC'd, add an exclusion rule to prevent
GC of the container named `cache`.

```
echo cache > /etc/docker-gc-exclude-containers
```

## Usage

Set the following ENV vars:

```bash
export DEPLOYER_HOST=<hostname where nginx listens>
export DEPLOYER_WORKSPACE=<path where you want your projects to be git-cloned> # default: '/var/www'
export NGINX_SITES_ENABLED_DIR=<path to sites-enabled directory in nginx conf> # default: '/etc/nginx/sites-enabled'
export DEPLOYER_COPY_SOURCE_DIR = <path to directory containing source files to be copied over to projects> # default: '/etc/chatops_deployer/copy'
export DEPLOYER_LOG_URL = <optional URL to tail logs(if you are using something like frontail)>
export GITHUB_WEBHOOK_SECRET = <Secret used to configure github webhook (if using github webhooks to deploy)>
export GITHUB_OAUTH_TOKEN = <OAuth token which will be used to post comments on PRs (if using github webhooks)>
export DEPLOYER_DEFAULT_POST_URL = <Additional HTTP endpoint where deployment success/faulure messages are posted (optional)>
export DEPLOYER_DATA_CONTAINER_NAME = <Name of docker container that will be used for sharing data between deployments> # default: 'cache'
export DEPLOYER_DATA_CONTAINER_VOLUME = <Path of volume inside the data container that can be mounted on other containers> # default: '/cache'

# Optional to use Vault for managing and distributing secrets
export VAULT_ADDR= <address where vault server is listening>
export VAULT_TOKEN= <token which can read keys stored under path secret/*>
export VAULT_CACERT= <CA certificate file to verify vault server SSL certificate>
```
And run the server as the root user:

    $ git clone https://github.com/code-mancers/chatops_deployer.git
    $ cd chatops_deployer
    $ bundle install
    $ ruby exe/chatops_deployer

### App Configuration

To configure an app for deployment using chatops_deployer API, you need to follow the following steps:

#### 1. Dockerize the app

Add a `docker-compose.yml` file inside the root of the app that can run the app
and the dependent services as docker containers using the command `docker-compose up`.
Refer [the docker compose docs](https://docs.docker.com/compose/) to learn how
to prepare this file for your app.

Note: Any setup that is required for your app should go into either Dockerfile
or `commands` section of chatops_deployer.yml.

#### 2. Add chatops_deployer.yml

Add a `chatops_deployer.yml` file inside the root of the app.
This file will tell `chatops_deployer` about ports to expose as haikunated
subdomains, commands to run after cloning the repository and also if any files
need to be copied into the project after cloning it for any runtime configuration.

Here's an example `chatops_deployer.yml` :

```yaml
# `expose` is a hash in the format <service>:<array of ports>
# <service> : Service name as specified in docker-compose.yml
# <array of ports> : Ports on the container which should be exposed as subdomains
expose:
  web: [3000]

# `commands` is a list of commands that should be run inside a service container
# before all systems are go.
# Commands are run in the same order as they appear in the list.
commands:
  - [db, "./setup_script_in_container"]
  - [web, "bundle exec rake db:create"]
  - [web, "bundle exec rake db:schema:load"]

# `copy` is an array of strings in the format "<source>:<destination>"
# If source begins with './' , the source file is searched from the root of the cloned
# repo, else it is assumed to be a path to a file relative to
# /etc/chatops_deployer/copy in the deployer server.
# destination is the path relative to chatops_deployer.yml to which the source file
# should be copied. Copying of files happen soon after the repository is cloned
# and before any docker containers are created.
# If the source file ends with .erb, it's treated as an ERB template and gets
# processed. You have access to the following objects inside the ERB templates:
# "env", "vault"
#
# "env" holds the exposed urls. For example:
# "<%= env['urls']['web']['3000'] %>" will be replaced with "http://crimson-cloud-12.example.com"
#
# "vault" can be used to access secrets managed using Vault if you have set it up
# "<%= vault.read('secret/app-name/AWS_SECRET_KEY', 'value') %>" will be replaced with the secret key fetched from Vault
# using the command `vault read -field=value secret/app-name/AWS_SECRET_KEY`
copy:
  - "./config.dev.env.erb:config.env"
```

#### Note about caching

chatops_deployer creates an empty data only container with default name: `cache`
when it starts and creates a volume inside this container at default path `/cache`.

You can use this container and volume in your own containers in order to
persist data between commands or deployments. Here's a sample `docker-compose.yml`
for a Rails web app service:

```yaml
web:
  build: .
  command: bin/rails s -p 3000 -b '0.0.0.0'
  ports:
    - "3000"
  environment:
    - BUNDLE_PATH=/cache/tmp/bundler
  volumes_from:
    - cache
  links:
    - db
```
The above will make `/cache` available in the `web` container, so you can
read/write anything from/to this directory and subdirectories and the changes
will remain intact.

### Deployment

#### Using hubot

Use the [hubot-chatops](https://github.com/code-mancers/hubot-chatops) plugin to talk to
chatops_deployer from your chat room.

#### Using HTTP API endpoint

To deploy an app using `chatops_deployer`, send a POST request to `chatops_deployer`
like so :

```
curl -XPOST  -d '{"repository":"https://github.com/user/app.git","branch":"master","callback_url":"example.com/deployment_status"}' -H "Content-Type: application/json" localhost:8000/deploy

# If code needn't be cleaned and fetched:
curl -XPOST  -d '{"repository":"https://github.com/user/app.git","branch":"master","callback_url":"example.com/deployment_status","clean":"false"}' -H "Content-Type: application/json" localhost:8000/deploy
```

You can see that the request accepts a `callback_url`. chatops_deployer will
POST to this callback_url with the following data:

1. Success callback

Example:
```ruby
{
  "status": "deployment_success",
  "branch": "master",
  "urls": { "web" => { "3000" => "misty-meadows-123.deployer-host.com"} }
}
```

2. Failure callback

Example:
```json
{
  "status": "deployment_failure",
  "branch": "master",
  "reason": "Nginx error: Config directory /etc/nginx/sites-enabled does not exist"
}
```

#### Using Github Webhook

1. Create a Github webhook

  Follow these instructions : https://developer.github.com/webhooks/creating/ .
  Use `<host>:<port>/gh-webhook` as the payload URL, where `host:port` is where
  chatops_deployer is running. Don't forget to set a secret when configuring the
  webhook and set it in the environment variable `GITHUB_WEBHOOK_SECRET` before
  starting chatops_deployer.

2. Make sure chatops_deployer can clone the repository

  Create a github user solely for deploying your apps, or from your personal
  account, create a Personal Access Token. Make sure this user is added to the
  repository and can clone the repo and leave comments. Set this token in the
  environment variable `GITHUB_OAUTH_TOKEN` before starting chatops_deployer.

Now whenever a Pull Request is opened, updated or closed, a new deployment will be triggered
and chatops_deployer will leave a comment on the PR with the URLs to access
the services deployed for the newly deployed environment. This environment will
be destroyed when the PR is closed.

If you also want to get a message posted to a callback url, you can set a default
HTTP endpoint where the status will be updated, in the environment variable:
DEPLOYER_DEFAULT_POST_URL.

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/code-mancers/chatops_deployer.


## License

[MIT License](http://opensource.org/licenses/MIT).

