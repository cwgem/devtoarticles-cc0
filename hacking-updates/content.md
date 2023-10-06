{%- # TOC start (generated with https://github.com/derlin/bitdowntoc) -%}

- [Hacktoberfest](#hacktoberfest)
- [DJango Issue Tracking](#django-issue-tracking)
- [GitHub Actions Runner Deep Dive](#github-actions-runner-deep-dive)
- [GitHub Actions Autoscale Runner (k8)](#github-actions-autoscale-runner-k8)
- [GitHub Composite Action For Pester](#github-composite-action-for-pester)
- [Conclusion](#conclusion)

{%- # TOC end -%}

Giving some updates since several things have been happening which delay my articles in progress! It's all hacking related and probably content for later articles so don't worry. I went to go add my [Mastodon account](https://floss.social/@cwprogram) for people that prefer SNS updates, but it looks like there's only Twitter integration. Would love if dev.to had an option for ActivityPub related accounts since they have a Mastodon presence and all (not to mention the weird stuff that happens at X/Twitter these days). As another bonus I'm also posting the MD content of articles to a [GitHub repository](https://github.com/cwgem/devtoarticles) (which also acts as a backup and the site's editor doesn't have an auto-save).

## Hacktoberfest

That's a really weird word to type out... One thing that seems a bit weird is the process to join hacktoberfest. The pledge articles seem like unnecessary noise and it feels like it would have been better off as a comment thread for badge purposes. I also think it's difficult to find things as many folks have different ways they want to contribute. Also there's the question on if the maintainer wants help with them in the first place. Some examples I could think of for labels that would help:

- `needs-help-readme`: While technically a document, it's the first view many have into projects so having it called out for contributions might help
- `needs-help-docs`: Needs documentation help, with maybe a `DOCS_CONTRIBUTION.md` file or something to give specifics on what documentation help is needed
- `needs-help-tests`: Either improving tests or getting tests up and running for projects
- `needs-help-issues`: Needs help with bug wrangling (ie. issue validation) 
- `needs-help-prs`: This is more a volume management label so folks can get time to deal with existing PRs before taking on new ones

## DJango Issue Tracking

An interesting thing I came across while looking into repositories that needed help with contributions was [DJango's repository](https://github.com/django/django). In particular I noticed that they didn't have an Issues tab. After a quick look I found they host their own [issue tracker](https://code.djangoproject.com/). As for the reason it's primarily relate to [issue history](https://forum.djangoproject.com/t/why-django-doesnt-have-an-issues-tab-in-its-github-reposistory/9870/2). That said, I sometimes wonder if the ease of opening issues is a good thing. While it may seem like you would want anyone to be able to file an issue, I feel like signal/noise becomes a thing due to issues like:

- Misconception that issues are a forum for questions (assuming the project uses discussions/doesn't explicitly have a questions label)
- Users who simply dump an error log with no method of reproduction
- Users who give very vague reports without any logging information

Not only that but many bug trackers allow for specific bug workflows. GitHub Issues on the other hand is simply a free style text field. Compare this to things that actual issue trackers can deal with:

- A specific workflow UI with separate fields like OS, version, etc
- Issue search based on said fields
- Auto assignment based on components
- Required fields to ensure crucial information is not left out

and so forth. This kind of information and workflows are crucial to getting high quality bug reports, which is important in keeping volunteer maintainers from being burned out.

## GitHub Actions Runner Deep Dive

My last article on [GitHub Actions Runner Deep Dive](https://dev.to/cwprogram/github-actions-runner-deep-dive-registration-and-setup-1ojb) had a decent amount of response so I'm working on the continuation. That said it takes time as I'll be looking how GitHub Actions Runner deals with actual job execution. This is more time consuming because there's a [decent amount of things you can do with workflows](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions). The main areas I want to touch are:

- How shell execution happens
- How actions are downloaded/executed
- How reusable workflow execution happens
- How containers work (Docker actions/service containers/build containers)

## GitHub Actions Autoscale Runner (k8)

I also plan to do a piece on the [actions-runner-controller](https://github.com/actions/actions-runner-controller/) for Kubernetes. From what I've seen from the documentation it's essentially a somewhat split up version of the [actions runner](https://github.com/actions/runner). While there is an option to use [GitHub webhooks](https://docs.github.com/en/webhooks) for getting jobs, that would mean I'd have to worry about GitHub being able to reach my infra via opening an ports on my router / using some kind of proxy forwarding service (I really don't want to do that). Instead the other option is it uses the same long polling connection as the standard runner to listen for messages. Once it gets a message it will adjust auto-scale values and start the respective containers to handle jobs. The helm charts in the repository seem pretty straightforward, though I might also do a deep dive on the code base as well.

## GitHub Composite Action For Pester

I'm working on taking my hand at attempting to develop GitHub Actions composite action for [Pester5 tests and coverage](https://pester.dev/). Pester is a Powershell testing framework which I use in my [PyPyInstaller project](https://pester.dev/docs/usage/configuration). Part of the issue I found is that Pester5 introduces a configuration object for handling various options, which I didn't find supported by many of the existing actions. I was mainly inspired to write something like this after I was reading Microsoft's [action-python](https://github.com/microsoft/action-python/blob/main/action.yml) composite action.

## Conclusion

That's all the fun stuff I've been working on so far. All said on the hacktoberfest I hack **every day**. Don't need a specific month for me! Might file a GitHub feature request on issues if there isn't one already... Also can't wait to try out that GitHub Actions autoscaler.