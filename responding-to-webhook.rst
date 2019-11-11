Responding to Webhook Events
============================

Please first complete the setup in the previous section: :ref:`gh_app_setup`.

.. _thank_maintainer:

Thank the maintainer for installing
-----------------------------------

Let's have a bot that **thanks the maintainer who installed your Github App**.
Whenever your GitHub bot is installed, we will have the bot create an issue in the repository
it was installed at, it will say something like "thanks for installing me",
and then it will close the issue immediately.

We've learned how to create and close an issue earlier in :ref:`gh_api_command_line`.
The only thing we need to implement is webhook event handler.

Let's go back to the list of `GitHub events documentation <https://developer.github.com/webhooks/#events>`_.
There are two events related to app installations: ``installation`` event and
``installation_repositories`` event.

Let's focus with just the ``installation`` event for now.

Go to the ``__main__.py`` file, in the webservice codebase.

I've added the following lines::


    @router.register("installation", action="created")
    async def repo_installation_added(event, gh, *args, **kwargs):
        installation_id = event.data["installation"]["id"]
        pass


This is where we are subscribing to the GitHub ``installation`` event, and
specifically to the "created" issues event.

The two important parameters here are: ``event`` and ``gh``.

``event`` here is the representation of GitHub's webhook event. We can access the
event payload by doing ``event.data``.

``gh`` is the gidgethub GitHub API, which we've used in the previous section to
make API calls to GitHub.

It doesn't do anything now, so let's add the code.

We've said that the GitHub App was installed, we want to say thanks to the
maintainer by creating an issue and then close it.

From previous example, we know how to do these tasks::

    # creating an issue
    response = await gh.post(url, data={
            'title': '...',
            'body': '...',
        })

    # closing an issue (requires write permission)
    await gh.patch(url,
        data={'state': 'closed'},
    )


However, since this is a GitHub App, we can't use the personal access token.
We'll need to use the installation access token, using the ``get_installation_access_token``
utility from utils.py::

    installation_access_token =
        await utils.get_installation_access_token(gh, installation_id)


The API calls will need to be change as follows::

     response = await gh.post(url, data={},
         oauth_token=installation_access_token["token"]
     )


Let's now think about the ``url`` in this case. Previously, you might have constructed
the url manually as follows::

   url = f"/repos/mariatta/strange-relationship/issues/"

We do we know which repository your app was installed to?

Take a look at GitHub's installation event payload `example
<https://developer.github.com/v3/activity/events/types/#installationrepositoriesevent>`_.

It's a big JSON object. The portion we're interested in are::

   {
     "action": "added",
     "repositories_added": [
       {
         "id": 186853007,
         "node_id": "MDEwOlJlcG9zaXRvcnkxODY4NTMwMDc=",
         "name": "Space",
         "full_name": "Codertocat/Space",
         "private": false
       }
     ],
     ...
   }

Notice that the repository name is provided in the webhook, under the list of
"repositories". So we can iterate on it and construct the url as follows::

    for repository in event.data['repositories']:
        url = f"/repos/{repository['full_name'}/issues/"


The next piece we want to figure out is what should the comment message be. For
this exercise, we want to thank the author, and say something like
"Thanks for installing me, @author!".

Take a look again at the issue event payload::

   {
     "action": "added",
     "sender": {
       "login": "Codertocat",
          ...
   }

The installer's username can be accessed by ``event.data["sender"]["login"]``.

So now your comment message should be::

   maintainer = event.data["sender"]["login"]
   message = f"Thanks for installing me, @{maintainer}! (I'm a bot)."


Piece all of that together, and actually make the API call to GitHub to create the
comment::

    @router.register("installation", action="created")
    async def repo_installation_added(event, gh, *args, **kwargs):
        installation_id = event.data["installation"]["id"]
        installation_access_token = await utils.get_installation_access_token(
            gh, installation_id
        )
        maintainer = event.data["sender"]["login"]
        message = f"Thanks for installing me, @{maintainer}! (I'm a bot)."

        for repository in event.data["repositories_added"]:
            url = f"/repos/{repository['full_name']}/issues/"
            response = await gh.post(
                url,
                data={"title": "Mariatta's bot was installed", "body": message},
                oauth_token=installation_access_token["token"],
            )


Because our bot wants to be helpful, it wants to clean up after itself by
closing the issue right away. How do we know the issue number that was
created?

Both issue number, and the URL are returned in the response of the API call (see the
`documentation <https://developer.github.com/v3/issues/#response-3>`_)::


    issue_url = response["url"]
    await gh.patch(issue_url, data={"state": "closed"},
        oauth_token=installation_access_token["token"]
    )


Commit that file, push it to GitHub, and deploy it in Heroku.


Install your bot
''''''''''''''''

Once deployed, you can install the GitHub App in one of your repositories and
see it in action!!

From your GitHub App's settings page, click on the "Install" link on the left.
Choose one repository.

Once it's done, check out the repository where you installed it to. You should
see an issue created and closed immediately by the bot.

Congrats! You now have a bot in place!

Problems??
''''''''''

If there's any problem so far, there are a few ways you can debug this.

- Check the recent webhook deliveries under the "Advanced" link in your
  GitHub App settings page. You can see all the webhook events, the payload,
  and the status.

- Read the logs from heroku. If you have Heroku toolbelt installed, you can run::

    heroku logs -a <app name> --tail


- Add logs (or prints) to your code.

- Redeliver the webhook. After you made changes to your code, you don't have
  to re-install the App, or wait for new events to come in. You can redeliver
  the same webhook event that failed before.


.. _thank_contributor:

Thank a new contributor for the pull request
--------------------------------------------

Let's give the bot more job! Let's now have the bot **say thanks, whenever we receive
a pull request**.

For this case, you'll want to subscribe to the ``pull_request`` event, specifically
when the ``action`` to the event is ``opened``.

For reference, the relevant GitHub API documentation for the ``pull_request`` event
is here: https://developer.github.com/v3/activity/events/types/#pullrequestevent.

The example payload for this event is here: https://developer.github.com/v3/activity/events/types/#webhook-payload-example-27

Try this on your own.

I'll give you a starting hint::

    @router.register("pull_request", action="opened")
    async def pr_opened(event, gh, *args, **kwargs):
        ...

How can you tell if the person is a new contributor, or an existing member of your
organization? Perhaps you don't want this bot to be triggered if it is one
of your co-maintainers.

There are a few ways to do this.

In the pull_request webhook event, one of the data that was passed is the ``author_association``
field. It could be an ``OWNER``, ``MEMBER``, ``CONTRIBUTOR``, or ``None``,
If the ``author_association`` field is empty, you can guess that they are a
first time contributor.

.. _react_to_comments:


React to issue comments
-----------------------

Everyone has opinion on the internet. Encourage more discussion by
**automatically leaving a thumbs up reaction** for every comments in the issue.
Ok you might not want to actually do that, (and whether it can actually encourage
more discussion is questionable). Still, this can be a fun exercise.

How about if the bot always gives **you** a thumbs up?

Try it out on your own.

- The relevant documentation is here: https://developer.github.com/v3/activity/events/types/#issuecommentevent

- The example payload for the event is here: https://developer.github.com/v3/activity/events/types/#webhook-payload-example-14

- The API documentation for reacting to an issue comment is here: https://developer.github.com/v3/reactions/#create-reaction-for-an-issue-comment

.. _label_prs:

Label the pull request
----------------------

Let's make your bot do even more hard work. **Each time someone opens a pull request,
have it automatically apply a label**. This can be a "pending review" or
"needs review" label.

The relevant API call is this: https://developer.github.com/v3/issues/#edit-an-issue

