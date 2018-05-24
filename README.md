---
title: Building a CI server
redirect_from: /guides/building-a-ci-server/
---

# Building a CI server

{:toc}

_Continuous Integration_ (or "CI") is a team software development practice in which contributions to the team code are merged together ("integrated") at frequent intervals, to keep every developer's copy of the codebase in sync with each others, and to make sure that what everyone is working on is jointly harmonious.

While Git and {{ site.data.variables.product.product_name }} are ideal tools for merging code from multiple authors, a CI server is a tool that helps check the all code contributions work together harmoniously. Depending on the nature of the project, this might include running test suites, performing security checks, ensuring that the code builds, or even deploying to staging or production servers.

The [Checks API][checks API] lets your CI servers receive triggers from {{ site.data.variables.product.product_name }} that code is ready for testing (for example, when a new pull request is opened), and gives your CI server the ability to give detailed status reports directly back to the developers in the {{ site.data.variables.product.product_name }} interface.

In this guide, we'll learn how to use the Checks API by building out the framework for a CI server  that you can use in your development process. We'll learn:

* What is a **run suite**?
* What is a **run check**?
* How can we use these tools to create rich and helpful status reports in {{ site.data.variables.product.product_name }}?

At the end of the guide, we'll have the framework for a CI server that will:

* Listen for requests to run a test suite, for example when a Pull Request is opened
* Inform  when a new run has begun
* Update {{ site.data.variables.product.product_name }} with a detailed report when the run has complete.

This guide builds upon the {{ site.data.variables.product.product_name }} App boilerplate code that is featured in [Building Your First {{ site.data.variables.product.product_name }} App][my first app]. If you haven't yet run through that guide, or you aren't sure where to begin building a {{ site.data.variables.product.product_name }} App, we recommend that you read through that one first.

(TODO the above paragraph should be a re-usable snippet)

Note: you can download the complete source code for this project [from the platform-samples repo][platform samples].

## First steps: App permissions

We're going to pick up immediately where the [Building Your First {{ site.data.variables.product.product_name }} App][my first app] guide left off. We assume that you have the [boilerplate code][app boilerplate] running and ready to go, that you have [ngrok][ngrok] running and pointed at your app, and your app [registered with {{ site.data.variables.product.product_name }}][app settings]. If you don't, please feel free to go back and review the [Building Your First {{ site.data.variables.product.product_name }} App][app boilerplate] guide.

Now that we want to leverage specific features available to GitHub apps, we need to update our app's permissions. That is, we need to instruct GitHub that our app requires access to particular things in the repositories where it is installed. Visit [your app's settings page][app settings] and click **Permissions & webhooks**

![The **Permissions & webhooks** tab in the app settings page](images/permissions.png)

To use the Checks API, scroll to the bottom of this page, and select the **Read & write** access level for Checks permissions.

![checkbox for granting read and write permissions for checks](images/checks%20permissions.png)

Then, just below, add event subscriptions for **Check suites** and **Check runs**

![checkboxes for check suites and check runs](images/events%20permissions.png)

Then click **Save changes**

### Installation settings

What happens next depends on whether you have already installed your app into your account.

1. You have probably installed your app on a repository already. If so, you'll need to tell GitHub that you (as the app _installer_) accept these new permissions. Visit [the app installations settings page][installation settings], click on **Configure** next to your app's name.
![app installation configuration page](images/installation%20configuration.png) 
At the top, you'll see that your app's permissions have changed (because we just updated them!. Click **Details**.
![view permission details](images/accept%20permissions.png) 

2. If you _haven't_ yet installed your sample app, you'll need to do that now. Choose a repository, or create a new, empty repository. Return to the [app settings page][app settings], click on the name of your app, and then **Install App** on the left. Then follow the prompts to select an organization and repository.
![app installation screen](images/install%20selection.png)


In either case, you'll be prompted to allow your app access to some or all of your repositories. This is of course necessary so that our app can inspect our code. Choose as you like, but we recommend that you have an empty sandbox repository set up to play around in, and limit your app to that repository.

![repository selection screen](images/repository%20access.png)


## How will the Checks API help us?

OK, so now we've got all the permissions necessary lined up! Enough prep work, let's get down to nuts and bolts.

Imagine: You're working on a project with other engineers. You're ready to check in some of your code, so you open a pull request on {{ site.data.variables.product.product_name }}.

What happens now is that {{ site.data.variables.product.product_name }} notices that this is an opportune moment to involve the CI server. In the nomenclature of the Checks API, a **check** is an action that an external service can provide by inspecting your code in some way, and deciding whether your code _passes_ or _fails_ some test. A simple example is an integration test suite: A test suite _checks_ your code to see if it breaks any of the tests, then indicates whether all the tests pass, or at least one test fails. But that's just one example. 

Anyway, when you create your pull request, {{ site.data.variables.product.product_name }} creates a **Check Suite**. A Check Suite is a (for the moment empty) collection of checks. {{ site.data.variables.product.product_name }} then notifies all apps installed for that repository that are interested that a new Check Suite has been created.

At this stage, the notified apps are given the chance to add one or more _Check Runs_ to the Check Suite. A Check Run represents one check—it might be an entire entire test suite, or it might be an individual test (to continue the integration testing example).

Each Check Run can have one of three statuses: **queued** (the default), **running**, or **complete**. In turn, each status can have additional detail. For example, a **complete** Check Run might be marked as **passing** or **failing**.

Each app is free to update the status of each Check Run it creates asyncronously, or even add more Check Runs if needed. But once each Check Run is marked as complete, {{ site.data.variables.product.product_name }} will mark the entire Check Suite as complete, closing the loop.

![Checks API timing diagram illustrating the above flow](images/timing%20diagram.png)

## Receiving check events

Let's update our server to *just* handle the check suite created event right now:

``` ruby
post '/event_handler' do
  payload = JSON.parse(params[:payload])

  case request.env['HTTP_X_GITHUB_EVENT']
  when "check_suite"
    if payload['action'] == 'requested' || payload['action'] == 'rerequested'
      create_check_run(payload)
    end
  end
end

helpers do
  def create_check_run(payload)
    puts "It's #{payload['repository']['name']}"
  end
end
```

What's going on? Every event that {{ site.data.variables.product.product_name }} sends out includes an `X-{{ site.data.variables.product.product_name }}-Event`
HTTP header. We'll only care about the check suite events for now. From there, we'll
take the payload of information, and return the repository name field. In this scenario,
our server is concerned not just with the check suite create, but when someone requests
the check suite be re-run, including when new commits are pushed to the triggering pull request.

To test out this proof-of-concept, make some changes in a branch in your test
repository, and open a pull request. Your server should respond accordingly!

## Working with check suits and check runs

TODO

With our server in place, we're ready to start our first requirement, which is
setting (and updating) CI statuses. Note that at any time you update your server,
you can click **Redeliver** to send the same payload. There's no need to make a
new pull request every time you make a change!

Since we're interacting with the {{ site.data.variables.product.product_name }} API, we'll use [Octokit.rb][octokit.rb]
to manage our interactions. We'll configure that client with
[a personal access token][access token]:

``` ruby
# Never, ever, hardcode app tokens or other secrets in your code!
# Always extract from a runtime source, like an environment variable.
#
PRIVATE_KEY = OpenSSL::PKey::RSA.new(ENV['GITHUB_PRIVATE_KEY'].gsub('\n', "\n")) # convert newlines
APP_IDENTIFIER = ENV['GITHUB_APP_IDENTIFIER']

# Before each request, instantiate an Octokit client using a JWT that we generate with our private key
before do
  payload = {
      # issued at time
      iat: Time.now.to_i,
      # JWT expiration time (10 minute maximum)
      exp: Time.now.to_i + (10 * 60),
      # The {{ site.data.variables.product.product_name }} App's identifier
      iss: APP_IDENTIFIER
  }
  JWT.encode(payload, PRIVATE_KEY, 'RS256')
  @client ||= Octokit::Client.new(bearer_token: get_jwt)
end
```

TODO we need to create the check run
After that, we'll just need to update the pull request on {{ site.data.variables.product.product_name }} to make clear
that we're processing on the CI:

``` ruby
def create_check_run(payload)
  # First, we need to exchange our JWT for an installation token against the repository that triggered this check suite
  token = get_installation_token(payload)
  installation_client = Octokit::Client.new(bearer_token: token)

  # Octokit doesn't yet support the Checks API, but it does provide generic HTTP methods we can use!
  # https://developer.github.com/v3/checks/runs/#create-a-check-run
  result = installation_client.post("#{payload['repository']['url']}/check-runs", {
      accept: 'application/vnd.github.antiope-preview+json', # This header is necessary for beta access to Checks API
      name: 'Awesome CI',
      head_branch: payload['check_suite']['head_branch'],
      head_sha: payload['check_suite']['head_sha'],
      status: :in_progress
  })

  # Assuming that this notifcation goes through, we can start our actual build run here.

  result.attrs
end

```

We're doing three very basic things here:

TODO no we're not.
* we're looking up the full name of the repository
* we're looking up the last SHA of the pull request
* we're setting the status to "pending"

That's it! From here, you can run whatever process you need to in order to execute
your test suite. Maybe you're going to pass off your code to Jenkins, or call
on another web service via its API, like [Travis][travis api]. After that, you'd
be sure to update the status once more. In our example, we'll just set it to `"success"`:

``` ruby
   .
   .
   .

   case request.env['HTTP_X_GITHUB_EVENT']
    when 'check_suite'
      # A new check_suite has been created or rerequested. Create a new check_run with status "running"
      if payload['action'] == 'requested' || payload['action'] == 'rerequested'
        create_check_run(payload)
      end

    when 'check_run'
      # GH confirms our new check_run has been created, or rerequested. Update it to "completed"
      if payload['action'] == 'created' || payload['action'] == 'rerequested'
        update_check_run(payload)
      end
    end

    .
    .
    .
```

``` ruby
def update_check_run(payload)
  token = get_installation_token(payload)
  installation_client = Octokit::Client.new(bearer_token: token)
  # Update the check run to a success state. We could include other information like line numbers, comments,
  # or other things to help in the case of failure
  # Also, normally, we would make this call when we were actually done with our CI, not as an artificial
  # side effect of a check run being initiated.

  # Octokit doesn't yet support the Checks API, but it does provide generic HTTP methods we can use!
  # https://developer.github.com/v3/checks/runs/#update-a-check-run
  # notice the verb! PATCH!
  result = installation_client.patch(payload['check_run']['url'], {
      accept: 'application/vnd.github.antiope-preview+json', # This header is necessary for beta access to Checks API
      name: 'Awesome CI',
      status: :completed,
      conclusion: :success,
      completed_at: Time.now.utc.iso8601
  })

  result.attrs
end
```

## Going further

Check suites and runs are powerful entities, and you can do a lot more with them than we have here in this demo. For example, on failing runs, you can indicate files and line numbers to pinpoint the exact cause of the failure, You can also provide useful commentary on why a check run passed or failed, for example the total run time, memory consumption, or the results of upstream operations. Read more about how to use these features in the [Checks API documentation][checks API].

## Conclusion

At {{ site.data.variables.product.product_name }}, we've used a version of [Janky][janky] to manage our CI for years.
The basic flow is essentially the exact same as the server we've built above.
At {{ site.data.variables.product.product_name }}, we:

* Fire to Jenkins when a pull request is created or updated (via Janky)
* Wait for a response on the state of the CI
* If the code is green, we merge the pull request

All of this communication is funneled back to our chat rooms. You don't need to
build your own CI setup to use this example.
You can always rely on [{{ site.data.variables.product.product_name }} integrations][integrations].



private keys:
```bash
export GITHUB_PRIVATE_KEY=`awk '{printf "%s\\n", $0}' awesome-ci-2.2018-05-15.private-key.pem`
```

```bash
export GITHUB_WEBHOOK_SECRET="my secret"
```

```bash
export GITHUB_APP_IDENTIFIER=12247
```

https://github.com/settings/apps  app settings

TODO

[checks API]: /v3/checks
[app boilerplate]: https://github.com/github/platform-samples/tree/master/apps/ruby/building-a-ci-server
[my first app]: /apps/guides/TODO
[ngrok]: https://ngrok.com/
[app settings]: https://github.com/settings/apps
[installation settings]: https://github.com/settings/installations











[ngrok]: https://ngrok.com/
[using ngrok]: /webhooks/configuring/#using-ngrok
[platform samples]: https://github.com/github/platform-samples/tree/master/api/ruby/building-a-ci-server-app
[Sinatra]: http://www.sinatrarb.com/
[octokit.rb]: https://github.com/octokit/octokit.rb
[access token]: https://help.github.com/articles/creating-an-access-token-for-command-line-use
[travis api]: https://api.travis-ci.org/docs/
[janky]: https://github.com/github/janky
[heaven]: https://github.com/atmos/heaven
[hubot]: https://github.com/github/hubot
[integrations]: https://github.com/integrations
