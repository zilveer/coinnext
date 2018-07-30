Our servers:

1. Bastion - can be accessed through SSH by whitelisted IPs on AWS. All the other servers can be accessed through SSH only from Bastion.

2. MySQL server
It hosts the database for the application (not the engine) and the queue: coinnext_prod, coinnext_queue_prod

3. Engine server - runs nodejs and a mysql instance only for the engine.
You should clone the coinnext_engine repo on this one and set up the config.json that would look smth like this:

{
  "production" : {
    "mysql": {
      "db": "coinnext_engine_production",
      "user": "",
      "password": "",
      "port": 3306,
      "host": "localhost",
      "logging": false
    },
    "queue": {
      "db": "coinnext_queue_prod",
      "user": "",
      "password": "",
      "port": 3306,
      "host": "",
      "logging": false
    },
    "salt": ""
  }
}

To start the engine: NODE_ENV=production node engine.js

4. Core API server
Runs transactions loading tasks, processes payments, aggregates stats.
You should clone the coinnext repo in here and start the following processes: core_api.js , queue.js , cron.js
The Web app communicates with the Core API and sends it new order data also asks it to generate a new wallet address.
The Admin app asks the Core API about the wallets statuses.
Only the Core API can access the wallets.
The config would look like this:

{
  "production" : {
    "mysql": {
      "db": "coinnext_prod",
      "user": "",
      "password": "",
      "port": 3306,
      "host": "",
      "logging": false
    },
    "queue": {
      "db": "coinnext_queue_prod",
      "user": "",
      "password": "",
      "port": 3306,
      "host": "",
      "logging": false
    },
    "redis": {
      "host": "",
      "port": "",
      "user": "",
      "pass": "",
      "db": "cnx-prod"
    },
    "slackalerts": {
      "enabled": true,
      "domain": "",
      "token": "",
      "channel": "#alerts"
    },
    "app_host": "",
    "wallets_host": "",
    "users": {
      "hostname": "https://coinnext.com"
    },
    "wallets": {
    }
  }
}

You should also configure the wallets data in the configs/wallets_config/ for each wallet you add. BTC.json would look like this:

{
  "client": {
    "host": "",
    "port": 8332,
    "user": "",
    "pass": "",
    "ssl": true,
    "sslStrict": false,
    "sslCa": false
  },
  "wallet": {
    "address": "",
    "account": "cnx_bank",
    "passphrase": ""
  },
  "confirmations": 3,
  "currency": "BTC"
}

You also need a Redis instance for the web sockets to scale and to keep the sessions. It’s safe to get one at openredis.com .

The cron task checks the wallet balances, processes payments and aggregates stats.

The queue executes commands that the Core API and the Engine will add to it. It’s basically a way for the engine and the app to talk to each other.

5. Webb App
It’s the frontend app that renders the site. The config would look like this:

{
  "production" : {
    "mysql": {
      "db": "coinnext_prod",
      "user": "",
      "password": "",
      "port": 3306,
      "host": "",
      "logging": false
    },
    "redis": {
      "host": "",
      "port": "",
      "user": "",
      "pass": "",
      "db": "cnx-prod"
    },
    "session": {
      "cookie_secret": "",
      "session_key": "cnxw",
      "cookie": {
        "maxAge": 43200000,
        "path": "/",
        "domain": ".coinnext.com",
        "httpOnly": true,
        "secure": true
      }
    },
    "salt": "",
    "emailer": {
      "enabled": true,
      "host": "https://coinnext.com",
      "from": "Coinnext <no-reply@coinnext.com>",
      "transport": {
        "host": "email-smtp.eu-west-1.amazonaws.com",
        "port": 465,
        "secureConnection": true,
        "auth": {
          "user": "",
          "pass": ""
        },
        "debug": false
      }
    },
    "app_host": "localhost:5000",
    "wallets_host": "",
    "users": {
      "hostname": ""
    },
    "assets_host": "https://d293nu39zb1o23.cloudfront.net",
    "assets_key": "1398130967890",
    "recaptcha": {
      "public_key": "",
      "private_key": ""
    }
  }
}

You need a AWS SES account for sending emails and a Recaptcha for rendering captchas. You can set up a CDN URL for your assets as well.

5. Admin
The Admin app runs on another server the has access to the DB and the Core API. The access to port 443 is whitelisted through the AWS Firewall only for some IPs.

{
  "production" : {
    "mysql": {
      "db": "coinnext_prod",
      "user": "",
      "password": "",
      "port": 3306,
      "host": "",
      "logging": false
    },
    "redis": {
      "host": "",
      "port": "",
      "user": "",
      "pass": "",
      "db": ""
    },
    "session": {
      "admin": {
        "cookie_secret": "",
        "session_key": "cnxadmin",
        "cookie": {
          "maxAge": 2592000000,
          "path": "/",
          "httpOnly": true,
          "secure": true
        }
      }
    },
    "emailer": {
      "enabled": true,
      "host": "https://coinnext.com",
      "from": "Coinnext <no-reply@coinnext.com>",
      "transport": {
        "host": "email-smtp.eu-west-1.amazonaws.com",
        "port": 465,
        "secureConnection": true,
        "auth": {
          "user": "",
          "pass": ""
        },
        "debug": false
      }
    },
    "salt": "",
    "app_host": "",
    "wallets_host": ""
  }
}


Cake tasks

We use Cake tasks to run migrations and populate admin users. You have to install the coffee-script module globally for it: sudo npm install coffee-script -g
You can see all the available tasks if you hit the cake command (or NODE_ENV=production cake) in the app root folder.
cake db:create_tables - Populate tables
cake db:seed_market_stats - Add market stats in the database
cake db:migrate - Run the SequelizeJS migrations
cake -e your_email@host.com -p pass admin:generate_user - it will generate and admin user together with an GAuth Token. You should open the generated URL in your browser to get the QR Code. You don’t need it in dev mode.

We use PM2 (https://github.com/unitech/pm2) to run the node servers and PM2-web (https://www.npmjs.org/package/pm2-web) to monitor them. You can use Forever or Monit if you prefer. We also use Zabbix to monitor all the servers and have metrics and alerts.

I would recommend to use Amazon RDS for MySQL to be easier to maintain the DBs.
