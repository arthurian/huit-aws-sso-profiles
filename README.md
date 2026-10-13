# huit-aws-sso-profiles

Run one command to get an AWS CLI profile for every account that you can access in an AWS IAM Identity Center (SSO) portal. The script is optional. The AWS CLI does all of the work, and the script only saves typing. [The plain AWS CLI commands](#the-plain-aws-cli-commands) shows how to do the same work without the script.

The defaults point at the Harvard University IT (HUIT) AWS access portal and the `HUIT*` role names. To use a different portal, see [More than one portal](#more-than-one-portal).

Requirements: AWS CLI v2 and Python 3.9 or later. The script uses only the Python standard library.

## Daily use

You run the script once to set up your profiles. After that, you use only the AWS CLI.

1. Once, and again when you get access to a new account:

   ```sh
   ./aws-sso-profiles setup
   ```

   `setup` names each profile after its account. To keep profile names that you already use, or to add a profile for another IAM role, see [Rename or add profiles](#rename-or-add-profiles). For one profile per role instead of one per account, add `--all-roles`.

2. Each day, or when your login session ends:

   ```sh
   aws sso login --sso-session huit
   ```

   One login covers every profile that uses the `huit` session. The portal sets how long the login session lasts. During that time, the CLI renews its one-hour access token without a browser.

3. Use a profile. To switch accounts, use another profile. You do not log in again.

   ```sh
   aws s3 ls --profile example-dev
   export AWS_PROFILE=example-prod
   ```

   For a profile picker, see `awsp` in [Tips](#tips).

## Usage

```sh
./aws-sso-profiles setup        # add the sso-session and a profile per account
./aws-sso-profiles list         # show every account and role
./aws-sso-profiles generate     # print profile blocks, change nothing
./aws-sso-profiles verify       # test every profile of the session
./aws-sso-profiles              # help and all options
```

`--roles` sets the role for each profile: a comma list of role names, most preferred first. The default is `HUITDevOpsAdmin,HUITGenAdmin,HUITPowerUser,HUITReadOnly`. If an account has none of these roles, `setup` uses the first role in alphabetical order. To see your role names, open the start URL in a browser.

If a profile in the same sso-session already uses the account and role, `setup` does not add another. `setup` does not check profiles in other sessions. If another profile already uses the name, `setup` skips that account and tells you. Add that profile by hand, or use `-p`.

### Rename or add profiles

The profiles are plain text in `~/.aws/config`, so you can edit them. To rename a profile, change its section header, for example `[profile example-dev]` to `[profile my-dev]`. `setup` finds profiles by session, account, and role, not by name, so it does not add the old name back. To keep both names, copy the block under the second name.

To use another IAM role from an SSO profile, for example a deploy role, add a profile that assumes it. The CLI uses the SSO profile to assume the role, so one `aws sso login` covers both:

```ini
[profile example-deploy]
role_arn = arn:aws:iam::123456789012:role/DeployRole
source_profile = example-dev
region = us-east-1
```

### More than one portal

Each AWS organization (payer account) has its own portal and start URL. Give each portal its own session name with `-s`, and a profile prefix with `-p`. Account names such as `audit` or `sandbox` can exist in more than one portal, and the prefix keeps the profile names apart. Set `--roles` to the role names of that portal, because the `HUIT*` names do not exist there.

```sh
./aws-sso-profiles setup -s other -u https://other.awsapps.com/start -p other- --roles PowerUserAccess,ReadOnlyAccess
aws sso login --sso-session other
./aws-sso-profiles verify -s other
```

Each session has its own login and token. Use `-s other` with `list`, `generate`, and `verify` too.

## Tips

These shell snippets go in your `~/.zshrc`. They use the AWS CLI only, not the script.

A shorter login command. `awsl` logs in to `huit`, and `awsl other` logs in to another session:

```sh
# awsl [session]: log in to an AWS SSO session (default: huit)
awsl() { aws sso login --sso-session "${1:-huit}"; }
```

Pick a profile with [fzf](https://github.com/junegunn/fzf) and use it in the current shell. To install fzf, run `brew install fzf`. For example, `awsp prod` selects the profile at once if only one name matches "prod". If the login session is expired, `awsp` logs you in.

```sh
# awsp [query]: pick an AWS profile with fzf, export AWS_PROFILE, log in if expired
awsp() {
  local profile
  profile=$(aws configure list-profiles | fzf --query="$1" --select-1) || return
  export AWS_PROFILE="$profile"
  aws sts get-caller-identity >/dev/null 2>&1 || aws sso login
}
```

This must be a shell function, not a script, because only a function can set `AWS_PROFILE` in your shell. With `AWS_PROFILE` set, `aws sso login` logs in to the session of that profile. The login check adds about one second to each `awsp` call.

Check which account and role the current profile uses, before you run a command that changes something:

```sh
# awsw: show the ARN of the current AWS identity
alias awsw='aws sts get-caller-identity --query Arn --output text'
```

## The plain AWS CLI commands

The examples use the Harvard HUIT portal. Change the session name, start URL, and region for your portal.

### 1. Add an sso-session to `~/.aws/config`

An sso-session (a named login to one portal) holds the portal URL and region. Many profiles can share one sso-session.

```ini
[sso-session huit]
sso_start_url = https://huitprodpayer.awsapps.com/start
sso_region = us-east-1
sso_registration_scopes = sso:account:access
```

As an alternative, run `aws configure sso-session` and answer the prompts.

### 2. Log in

```sh
aws sso login --sso-session huit
```

The CLI opens a browser and stores an access token in `~/.aws/sso/cache/`. The file name is the SHA-1 hash of the session name. The token lasts about one hour, and the CLI renews it without a browser until the login session ends.

If `AWS_PROFILE` names a profile that is not in your configuration, the login fails. In that case, run `env -u AWS_PROFILE aws sso login --sso-session huit`.

### 3. List your accounts and roles

These two commands need the access token from step 2, and the first line needs `jq`:

```sh
TOKEN=$(jq -r .accessToken ~/.aws/sso/cache/$(printf %s huit | shasum | cut -d' ' -f1).json)

aws sso list-accounts --access-token "$TOKEN" --region us-east-1 --output table
aws sso list-account-roles --access-token "$TOKEN" --region us-east-1 --account-id 123456789012 --output table
```

You can also see the same list in a browser at the start URL.

### 4. Add a profile for an account and role

```ini
[profile my-dev-account]
sso_session = huit
sso_account_id = 123456789012
sso_role_name = HUITPowerUser
region = us-east-1
```

As an alternative, run `aws configure sso --profile my-dev-account`. The command shows the accounts and roles and writes the profile for you.

### 5. Use the profile

```sh
aws sts get-caller-identity --profile my-dev-account
export AWS_PROFILE=my-dev-account
```

### 6. Log out

```sh
aws sso logout
```

## What the script adds

`setup` does steps 1 to 4 for every account in one run:

- If the configuration has no sso-session block, it adds one.
- If the cached access token is missing or expired, it runs `aws sso login`. The script checks only the one-hour token, so it can open the browser more often than the AWS CLI does.
- It runs `aws sso list-accounts`, then `aws sso list-account-roles` for up to 16 accounts at a time.
- It names each profile from the account name, for example `Example DEV` becomes `example-dev`.
- It picks one role per account from `--roles`. With `--all-roles`, it adds one profile per account and role.
- It shows the plan and asks before it writes. It only appends new profiles. It never edits or deletes a profile.

`list` does steps 2 and 3. `generate` does steps 2 to 4, but it prints the profiles and does not write them. `verify` does step 5 for every profile of the session.

## How the login works

IAM Identity Center uses OpenID Connect (OIDC, a login standard built on OAuth 2.0). `aws sso login` uses the authorization code flow with PKCE (Proof Key for Code Exchange). The CLI opens your browser and waits on a local `127.0.0.1` address for the result, so open the login URL on the same computer.

To log in with a different browser or browser profile, add `--no-browser`. The CLI then prints the URL, and you open it where you want, on the same computer.

On a computer with no browser, such as a server over SSH, add `--use-device-code`. The CLI then shows a URL and a short code. Open the URL on any device and enter the code. The links below explain both flows.

## References

- [Configuring IAM Identity Center authentication with the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html): `sso-session`, `aws configure sso`, login, and token refresh.
- [Using an IAM role in the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-role.html): `role_arn` and `source_profile`, for profiles that assume another role.
- [`aws sso login` command reference](https://docs.aws.amazon.com/cli/latest/reference/sso/login.html): all login options, including `--use-device-code` and `--no-browser`.
- [What is IAM Identity Center?](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html): the service behind the access portal.
- [IAM Identity Center OIDC API reference](https://docs.aws.amazon.com/singlesignon/latest/OIDCAPIReference/Welcome.html): the token exchange that `aws sso login` does.
- [IAM Identity Center portal API reference](https://docs.aws.amazon.com/singlesignon/latest/PortalAPIReference/): the API behind `aws sso list-accounts` and `aws sso list-account-roles`.

## Credits

Based on [aws-oidc-cli-login](https://github.com/ventz/aws-oidc-cli-login) by Ventz Petkov. This version is a smaller Python rewrite with fewer features and support for more than one portal.

## License

MIT. See [LICENSE](LICENSE).
