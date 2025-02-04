- Feature Name: Credential helper for bundle install
- Start Date: 2025-02-04
- RFC PR: (leave this empty)
- Bundler Issue: (leave this empty)

# Summary

Implement a mechanism to securely pass authentication credentials for private Gem registries in Bundler.

# Motivation

- Enable users to execute automated installation processes without storing authentication credentials in plaintext
- Allow shorter expiration periods for API tokens in private Gem registries

# Guide-level explanation

You are a developer using a private artifact registry (such as GitHub Packages) instead of [RubyGems.org](http://rubygems.org/), and you're looking for a secure way to install gems using Bundler.

You have external processes (like GitHub CLI or 1Password) where authentication credentials are securely stored, and you want to use these credentials to install Gems.

To accomplish this, follow these steps:

1. Create an authentication helper script in a location like `/usr/local/bin/gh-helper.sh`

```bash
#!/bin/bash
atpons:$(gh auth token)
```

2. Configure Bundler to use the authentication helper with the following command:

```bash
$ bundle config credential-helper.rubygems.pkg.github.com /usr/local/bin/gh-helper.sh
```

3. When installing with a Gemfile like this, Bundler will execute the credential helper to obtain the token and use it as `https://atpons:$(/usr/local/bin/gh-helper.sh)@rubygems.pkg.github.com/`:

```ruby
source "https://rubygems.pkg.github.com/atpons" do
  gem "test-gem"
end
```

# Reference-level explanation

While Gem registries support API access via Basic authentication, private registries often issue short-lived API keys (like AWS CodeArtifact).

Additionally, services like GitHub Packages issue tokens that should be stored securely.

Storing these in configuration files and updating them periodically degrades experience and poses security risks.

Other SCM and package registries support credential helpers for authentication:

- Git implements a Credential Helper mechanism to call external processes for passing credentials
    - https://git-scm.com/doc/credential-helpers
- Docker implements a similar Credential Helper mechanism that can use OS credential stores
    - https://github.com/docker/docker-credential-helpers
- pnpm implements a Git-like Credential Helper mechanism
    - https://pnpm.io/ja/npmrc#urltokenhelper
- pip supports keyring integration to use OS credential stores
    - https://pip.pypa.io/en/stable/topics/authentication/#keyring-support

# Drawbacks

This proposal won't affect existing authentication mechanisms.

However, it may create some confusion since RubyGems now supports OpenID Connect for pushing gems, even though this proposal specifically addresses download authentication in Bundler.

# Rationale and Alternatives

Although Bundler supports temporary credential settings, this method is insecure because it stores API keys directly in the configuration.

Environment variables offer another option, but this leads to inconsistent credential handling and potential organizational confusion.

Some RubyGems-compatible package registries currently use a temporary configuration write as a workaround.

https://docs.aws.amazon.com/codeartifact/latest/ug/configure-use-rubygems-bundler.html#configure-ruby-gem

For AWS specifically, while it issues temporary credentials that require periodic command updates, a credential helper would allow dynamic credential retrieval from the AWS CLI with just one initial setup.

This proposal enables storing credential retrieval methods in a configuration file with a unified interface. Furthermore, the credential helper interface could eventually allow RubyGems and Bundler to utilize the OS's standard credential store for API key storage.

# Unresolved questions

- Credential helper interface specification
- Whether to support this for RubyGems pushes
