What's Next?
============

You now have built yourself a fully functional GitHub App! Congratulations!!

However, the App you've built today might not be the GitHub bot you really want.
That's fine. The good thing is you've learned how to build one yourself, and you
have access to all the libraries, tools, documentation needed in order to build
another GitHub bot.

Additional ideas and inspirations
---------------------------------

Automatically delete a merged branch
''''''''''''''''''''''''''''''''''''

Related API: https://developer.github.com/v3/git/refs/#delete-a-reference.

The branch name can be found in the pull request webhook event.

Monitor comments in issues / pull request
'''''''''''''''''''''''''''''''''''''''''

Have your bot detect blacklisted keywords (e.g. offensive words, spammy contents) in
issue comments. From there you can choose if you want to delete the comment,
close the issue, or simply notify you of such behavior.

Automatically merge PRs when all status checks passed
'''''''''''''''''''''''''''''''''''''''''''''''''''''

Folks using GitLab have said that they wished that this is available on GitHub.
You can have a bot that does this! We made `miss-islington <https://github.com/python/miss-islington/blob/master/miss_islington/backport_pr.py>`_
do this for CPython.

Detect when PR has merge conflict
'''''''''''''''''''''''''''''''''

When merge conflict occurs in a pull request, perhaps you can apply a label or
tell the PR author about it, and ask them to rebase. You might have to do this
as a scheduled task or a cron job.


Other topics
------------

Rate limit
''''''''''

You have a limit of 5000 API calls per hour using the OAuth token.
The `Rate Limit API <https://developer.github.com/v3/rate_limit/>`_ docs have
more info on this.

Caching the Installation Access Token
'''''''''''''''''''''''''''''''''''''

When we receive an installation access token, it returns an expiry date. You can
use the same installation access token until it becomes expired. You can think
of ways to cache this token somehow, instead of requesting it each time we need it.

Unit tests with pytest
''''''''''''''''''''''

`bedevere <https://github.com/python/bedevere>`_ has 100% test coverage using
`pytest <https://docs.pytest.org/en/latest/>`_ and
`pytest-asyncio <https://pypi.org/project/pytest-asyncio/>`_.
You can check the source code to find out how to test your bot.


Error handling
''''''''''''''

`gidgethub exceptions <https://gidgethub.readthedocs.io/en/latest/__init__.html>`_ documentation.


