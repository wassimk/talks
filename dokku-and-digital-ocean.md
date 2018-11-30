# Dokku with DigitalOcean

[Dokku](http://dokku.viewdocs.io/dokku/) is a mini-Heroku just for you. Run you rown PaaS.

In this set of notes we deploy a demo Ruby on Rails app that uses PostgreSQL, Redis, Memcached and Let's Encrypt. You can see it at [suggestotron.apps.tulsarb.org](https://suggestotron.apps.tulsarb.org/).

### Sign-up for DigitalOcean:

Use [referral link](https://m.do.co/c/8f0f950f85db) for $10 credit.

### Create a [Droplet](https://cloud.digitalocean.com/droplets):

1. **One-click apps:** `Dokku 0.11.3 on 16.04`

2. **Size:** 2 GB / 1 vCPU for $10 a month
   * Size shouldn't matter to get started
   * Since base $5 droplets are now 1GB
3. **Datacenter:** Typically closest to your customer base.
4. **Additional options:** Monitoring
   * It pre-installs the agent, can do later too.
5. **Upload SSH Key:** `cat ~/.ssh/id_ras | pbcopy`
6. **Name:** *tulsarb* - doesn't matter too much.

### Firewall

[Firewalls](https://cloud.digitalocean.com/networking/firewalls) are under *Networking* in the dashboard:

1. **Name:** doesn't matter, _web-app-firewall_
2. **Inbound Rules:** Turn on SSH, HTTP and HTTPS.
3. **Outbound Rules:** Default are okay.
4. **Remove IPv6:** Dokku no like.
5. Apply to Droplets, search for *tulsarb*.

You should also turn on Ubuntu firewall which has good default rules. It's `uft enable` after _ssh_ into the server. You can see the rules via `ufw status`.

### DNS

*Optional but makes it easier to work with!*

A record of `apps.tulsarb.org` to the IP address.

CNAME record of `*.apps.tulsa.org` to `apps.tulsarb.org`.

This let's all of our apps automatically get a hostname of `<app-name>.apps.tulsarb.org`

### Update the OS Packages

``` shell
apt update # fetch the updates
apt upgrade # apply the updates
```

### Configure Dokku

**Initial Setup via the Web:**

1. Visit the IP address via HTTPS

2. SSH key should be pre-filled

3. Set the hostname to: `apps.tulsarb.org`

4. Enable virtual host naming for pretty URL's like: `https://<app-name>.apps.tulsarb.org`

**Continue Setup via SSH:**

SSH into the server via: `ssh root@ssh.apps.tulsarb.org`

**Install Plugins:**

``` shell
sudo dokku plugin:install https://github.com/dokku/dokku-postgres.git
sudo dokku plugin:install https://github.com/dokku/dokku-redis.git
sudo dokku plugin:install https://github.com/dokku/dokku-memcached.git
sudo dokku plugin:install https://github.com/dokku/dokku-letsencrypt.git
```

**Configure Global E-mail Address for all SSL Certificates**
```shell
dokku config:set --global DOKKU_LETSENCRYPT_EMAIL=my@email-address.com
```

### Configure your Application

Have your application ready to deploy!

**Create the application in Dokku:**

``` shell
dokku apps:create suggestotron
```

**Create Services and Link them to Your App:**

A service is a Docker container running the specific service such as PostgreSQL or Redis. A link is connecting your application to that service. It let's them communicate plus sets ENV variables for your application to use for communication. There is some work to allow re-using one database service for multiple apps but right now think of it as a new service per application.

``` shell
# Databases
dokku postgres:create suggestotron-db
dokku postgres:link suggestotrone-db suggestotron # sets DATABASE_URL ENV for you
```

``` shell
# Redis
dokku redis:create suggestotron-redis
dokku redis:link suggestotron-redis suggestotron # sets REDIS_URL ENV for you
```

``` shell
# Memcached
dokku memcached:create suggestotron-memcached
dokku memcached:link suggestotron-memcached suggestotron # sets MEMCACHED_URL ENV for you
```

**Set any other needed environmental variables:**

```shell
dokku config:set suggestotron AWS_ACCESS_KEY_ID=thekey
dokku config:set suggestotron AWS_REGION=us-east-1
dokku config:set suggestotron MAILER_HOST=smtp.server.com
# etc...
```

### Deploy your Application

**Configure git remotes:**

``` shell
git remote add dokku dokku@apps.tulsarb.org:suggestotron
```
**Deploy:**

This deploy process looks like Heroku because it detects your app type and uses the appropriate open-sourced Heroku [build pack](https://devcenter.heroku.com/articles/buildpacks). It uses the `Procfile` to know how to start processes.

``` shell
git push dokku master
```

**Setup SSL certificates:**

Do this after initial deploy and any domain adds and removes.

``` shell
dokku letsencrypt suggestotron
```

**Start the Worker Process too:**

``` shell
dokku ps:scale suggestotron web=1 worker=1
```

### Tips

**One Off Commands:**

Things like Rails database migrations or rake tasks:

``` shell
dokku run suggestotron rake db:migrate
```

**Deploy Hooks:**

You can do things like run migrations automatically. Read [more](http://dokku.viewdocs.io/dokku/advanced-usage/deployment-tasks/).

**Tweak Dokku default scale size:**

Dokku only starts one web worker by default, if you don't want to run the scale command _(once)_ you can set the default to start a worker by creaking a `DOKKU_SCALE` file in your apps root directory:

```shell
# /DOKKU_SCALE
web=1
worker=1
```

**Test your app and fix issues:**

``` shell
dokku logs suggestotron -t # tail all logs
dokku logs suggestotron -p web # tail just web process
dokku logs suggestotron -p worker # tail just worker process
```

**Backround worker doesn't auto-restart:**

After deploying I noticed the background worker wasnt' restarting. You can restart the whole app with:

``` shell
dokku ps:restart suggestotron
```

**Run recurring jobs and tasks:**

Dokku supports cron for scheduling recurring tasks and jobs. Just read through the [documentation](http://dokku.viewdocs.io/dokku/deployment/one-off-processes/#general-cron-recommendations) for configuration help.

**Run `dokku` commands locally:**

Now you don't need to type the app name each time. The `dokku` command looks for the Git remote and extrapolates the app name. Plus you don't have to be ssh'd into the server for most things. There are many [different packages](https://github.com/dokku/dokku/blob/master/docs/community/clients.md) for this client.

``` shell
gem install dokku-cli
```

**Automatically Upgrade Unbuntu OS:**

Run through the [installation instructions](https://help.ubuntu.com/community/AutomaticSecurityUpdates#Using_the_.22unattended-upgrades.22_package) for automatic uanttended upgrades. Make sure to turn on e-mail notifications. Then when they work turn it on only for errors.