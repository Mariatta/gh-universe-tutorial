.. _gh_app_setup:

Building a GitHub App
=====================


About webhooks
--------------

In the previous example, we've been interacting with GitHub by performing actions:
we make requests to GitHub. And we've been doing that locally on our own machine.

It's not so much of a "bot" that actually respond to anything. We've been running
the code by ourselves.

Now let's learn about webhooks so we can build a bot.

Webhook events
--------------

When an event is triggered in GitHub, GitHub can notify you about the event by
sending you a POST request along with the payload.

Some example ``events`` are:

- issues: any time an issue is assigned, unassigned, labeled, unlabeled, opened,
  edited, closed, reopened, etc.

- pull_request: any time a pull request is opened, edited, closed, reopened,
  review requested, etc.

- status: any time there's status update.

The complete list of events is listed `here <https://developer.github.com/webhooks/#events>`_.

Since GitHub needs to send you POST requests for the webhook, it can't send them
to your personal laptop. So we need to create a webservice that's open on the internet.
This webservice will become our GitHub App.

There are plenty of options for hosting your webservice. For this tutorial, we
will be deploying our webservice to Heroku.

How does authentication works with a GitHub App?
------------------------------------------------

In the previous section, we used a personal access token when we executed our
scripts. The token was how GitHub identifies you.

A GitHub App has a different mechanism for authentication. It is possible to
build a GitHub App that can be installed in various repositories, and installed
by multiple people. So how can we identify who's who in a GitHub App? Do we
need to ask for everyone's access token?

When a GitHub App is installed in a repository, you get assigned an ``installation_id``.
When GitHub is sending you a webhook event, it will include the ``installation_id``
as the payload. You can then use the ``installation_id`` to create an ``installation
access token``. The ``installation access token`` can be used to make API
calls, similar to how you use a personal access token.

More reading: https://developer.github.com/apps/building-github-apps/authenticating-with-github-apps/

How exactly can we create the ``installation access token`` from an ``installation_id``?
The documentation linked above has more details, but the process is as follows.
We will be creating a JWT (JSON web token with the GitHub App's ID, and GitHub
App's Private Key. We will then pass in the JWT and ``installation_id`` to GitHub,
and GitHub will provide us with an ``installation_access_token``.

I've created `a couple convenience functions
<https://github.com/Mariatta/gh_app_starter/blob/master/webservice/utils.py>`_
to abstract these out, which you can
later use or copy paste.

Create a webservice
-------------------

Let's create a webservice, that will become our bot. I've created a repository
that you can fork and clone, as a starting point.

1. Go to https://github.com/Mariatta/gh_app_starter.

2. Fork and clone the repository::

       $ git clone git@github.com:{your GitHub username}/gh_app_starter.git

Let's go over the repository content quickly.

Procfile
    This file is specifically for Heroku. Heroku will use this file to know
    what kind of dyno to run for your web service, and what command to run.

requirements.txt
    This lists the dependencies needed for the webservice. Heroku will read
    this file and install the dependencies.

runtime.txt
    This is also for Heroku. It tells Heroku which Python runtime environment to
    run for your webservice. We'll be using Python 3.7.5.

webservice/utils.py
    This file contains a couple convenience functions, to help us with GitHub
    App's authentication flow. There are two items there: ``get_jwt()``
    and ``get_installation_access_token`` coroutine.

webservice/__main__.py
    This is the code of our web service. It runs an aiohttp server. I've defined
    a couple request handlers. The "/" endpoint (the ``handle_get`` coroutine)
    simply returns the text "Hello world".

    The "/webhook" endpoint (the ``webhook`` coroutine) accepts a POST method.
    This is where we will receive all webhook events from GitHub.


You can try running the webservice locally first. Create and activate a virtual
environment, and install the dependencies. From the root of the repository::

    $ python3.7 -m venv venv
    $ source venv/bin/activate
    (venv) $ python3.7 -m pip install -U pip -r requirements.txt

Start the server::

    (venv) $ python3.7 -m webservice

Note that this is the same command you see in the ``Procfile``.

You should now see the following output::

    ======== Running on http://127.0.0.1:8080 ========
    (Press CTRL+C to quit)

Open your browser and point it to http://127.0.0.1:8080.  Alternatively,
you can open another terminal and type::

    curl -X GET http://127.0.0.1:8080

Whichever method you choose, you should see the output: "Hello World".

Deploy the webservice to Heroku
-------------------------------

Login to your account on Heroku. You should land at https://dashboard.heroku.com/apps.

Click **"New"** > **"Create a new app"**. Type in the app name, choose the United States region,
and click **"Create app"** button.
If you leave it empty, Heroku will assign a name for you.

Once your web app has been created, go to the **Deploy** tab. Under **"Deployment method"**,
choose GitHub. Connect your GitHub account if you haven't done that.

Under "Search for a repository to connect to", enter ``gh_app_starter`` (assuming
you forked my repo). Press "Search". Once it found the right repo, press "Connect".

Scroll down. Under **Deploy a GitHub branch**, choose "master", and click **"Deploy Branch"**.
(You may also want to "Enable Automatic Deploys").

Watch the build log, and wait until it finished.

When you see "Your app was successfully deployed", click on the "View" button.

You should see "Hello world.".

Tip: Install Heroku toolbelt to see your logs. Once you have Heroku toolbelt installed,
you can watch the logs by::

   heroku logs -a <app name> --tail



Create a GitHub App
-------------------

Create a GitHub App by going to https://github.com/settings/apps. You can
also get there by going to GitHub, click on your avatar, choose **Settings**, and
**Developer Settings**.

Click the **New GitHub App** button.

You will be presented with a form. Choose a name for your app. This will
become the name of your bot! (It can also be changed later). I suggest something
descriptive. I already have a bot named ``mariatta-bot``, so this time I will go
with ``mariatta-bot-again``.

Enter a description. You can leave most of the other fields empty. For this
tutorial, the two important fields are: **Webhook URL** and **Webhook Secret**.

In the **Webhook URL** field, enter the url of your heroku website, ended with
``/webhook``. For example ``https://{yourappname}.herokuapp.com/webhook``.

In the **Webhook secret**, enter a passphrase (or any text). This secret
will be used by our webservice. We need a way to know that the webhook we
receive is indeed from GitHub, and it is meant for our bot. If other bot or
other webservice somehow made a POST request to your endpoint, you don't
really want to do anything about it. Therefore this secret should be known
only by your webservice and your GitHub App (and yourself!) Whatever secret
you put in here, remember it (or copy it somewhere), we will use it later.

Scroll down to the **Permissions** section. For this tutorial, set the permission
for both **Issues** and **Pull requests** to **"Read & Write"**.

Scroll down to the **Subscribe to events** section. Tick the **Issues** and
**Pull request** boxes.

For the question **Where can this GitHub App be installed?**, for now let's
limit this to yourself, since we're still learning and developing it. You can
always change this to **Any account** later on.

Click the **Create GitHub App** button!

Set up config vars in Heroku
----------------------------

We need to create the following config variables in Heroku. This is similar
as if we're setting an environment variable in our Terminal. There are several
values that are "secret and confidential" that we do not want to hardcode
or commit to our codebase.

The first config var to create is ``GH_APP_ID``. You'll see this in your GitHub
App's settings page.

The next config var is ``GH_SECRET``, which is the **webhook secret** from
when you created the GitHub App in the previous step.

The last config var is ``GH_PRIVATE_KEY``. This will be used for generating
your bearer token (in ``get_jwt()``).  From your GitHub App's settings page,
scroll down and click the **Generate Private Key** button. It will generate
a private key, and automatically be downloaded as a ``.pem`` file. Copy
the content of that file to the ``GH_PRIVATE_KEY`` config var.


Now that we have a webservice running, and config vars setup, we can start
building our bot!









