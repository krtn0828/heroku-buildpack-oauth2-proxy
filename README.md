# heroku-buildpack-oauth2-proxy

This buildpack adds authentication against an OAuth2 provider such as
GitHub or Google to your Heroku application.

Authentication is provided by putting [oauth2-proxy](https://github.com/oauth2-proxy/oauth2-proxy)
in front of your application as a reverse proxy, allowing you to authenticate
users using OAuth2 without actually implementing OAuth2 in your applications
codebase.

One usecase is to ensure only users from your organization will be able to access static
files served with Heroku.

The following section guides you through the process of creating such a setup using
[heroku-buildpack-static](https://github.com/heroku/heroku-buildpack-static)
and this buildpack.

If you'd rather use this buildpack to secure a different application, it might still be
useful to walk through this example once to get familiar with the setup procedure. However,
you might also skip ahead to the [usage section](#usage-with-other-applications).

## Getting Started

First, clone the example branch and switch into it:

```console
git clone https://github.com/krtn0828/heroku-buildpack-oauth2-proxy -b example-static oauth2-example
cd oauth2-example
```

Then, create a new Heroku application and set the buildpacks:

```console
heroku create
heroku buildpacks:set https://github.com/heroku/heroku-buildpack-static.git
heroku buildpacks:add https://github.com/krtn0828/heroku-buildpack-oauth2-proxy.git
```

You need an account for the proxy so that it can interface with your OAuth2 provider.
For this example, we will be using GitHub as OAuth2 provider.

Create a new OAuth2 app in the [GitHub developer settings](https://github.com/settings/developers).
Ensure that the "Authorization callback URL" is set correctly. For example, if you are using
`herokuapp.com` and your application is called `ancient-woodland-33672`, you should set the
callback URL to: `https://ancient-woodland-33672.herokuapp.com/oauth2/callback`.

After having created the OAuth2 app successfully, you will be shown a _Client ID_ and a _Client Secret_,
which you need to configure on the Heroku app:

```console
heroku config:set OAUTH2_PROXY_CLIENT_ID=0123456789abcdef1234
heroku config:set OAUTH2_PROXY_CLIENT_SECRET=0123456789abcdef0123456789abcdef01234567
```

Furthermore, you need to specify that GitHub should be used as authentication provider
and provide a secret key for encrypting the session cookies:

```console
heroku config:set OAUTH2_PROXY_PROVIDER=github
heroku config:set OAUTH2_PROXY_COOKIE_SECRET=$(python -c \
    "from secrets import token_urlsafe; print(token_urlsafe(32)[:32])" \
)
```

Now, you are all set and can start the first deployment of your example application:

```console
git push heroku HEAD:master
```

You will see the buildlog for your application, followed by a message about successful
deployment.

After this, navigate to your application:

```console
heroku apps:open
```

You will be prompted to log-in with GitHub.

After successful authentication, GitHub will redirect you back to your application and a success
page will be shown.

For futher steps, you might want to have a look in the [configuration Section](#configuration) to
learn about configuration options for finer grained authentication, e.g. allowing only members
of a particular GitHub organization.

## Usage With Other Applications

For using this buildpack with your application, you need to do two things:

On the one hand, you need to set up the configuration for `oauth2-proxy`. The getting started section
describes this process for GitHub. For other providers, a look at the
[configuration Section](#configuration) and at the
[OAuth2 Provider Configuration documentation of oauth2-proxy](https://oauth2-proxy.github.io/oauth2-proxy/auth-configuration).

On the other hand, you need to ensure that `oauth2-proxy` is run as a reverse proxy in front
of your actual application worker.

For that purpose, this buildpack installs a script `start_with_oauth2_proxy.sh`. This script
can be used as the `web` process and will pass any requests to the backend process specified as
its arguments.

For example, the `heroku-buildpack-static` has `/app/bin/boot` as entrypoint. Given that,
a usable `Procfile` to run it with the proxy looks as following:

```console
web: /app/bin/start_with_oauth2_proxy.sh /app/bin/boot
```

This will take care to start both `/app/bin/boot` and `oauth2-proxy` and to route any incoming
requests correctly.

## Configuration

The following environment variables are required:

- `OAUTH2_PROXY_PROVIDER`: The provider to use. Something like `github`, `google` or `facebook`
- `OAUTH2_PROXY_CLIENT_ID`: The OAuth2 client id (generated by OAuth2 provider)
- `OAUTH2_PROXY_CLIENT_SECRET`: The OAuth2 client secret (generated by OAuth2 provider)
- `OAUTH2_PROXY_COOKIE_SECRET`: Secret key to encrypt `oauth2-proxy`'s session cookies. This string
  needs to be 32 characters long.

Any further configuration of `oauth2-proxy` can also be done via environment variables.

This is described [here](https://oauth2-proxy.github.io/oauth2-proxy/configuration#environment-variables)
in its documentation.

## Contributing

Should you encounter any issues while using this buildpack or discover any bugs, I would be glad if
you [file an issue](https://github.com/cfra/heroku-buildpack-oauth2-proxy/issues) with this GitHub project.

Also, if you feel there is anything which should be improved, go ahead and open a pull request or file an
enhancement proposal.
