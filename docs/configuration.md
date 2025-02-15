# Configuration

## General

<!-- TODO AUTH_ENABLE and STORAGE_ENABLE -->
<!-- TODO SERVER_URL, HOST, PORT -->
<!-- TODO REDIRECT_URL_SUCCESS, REDIRECT_URL_ERROR -->

## Connect to Hasura

In order to connect HBP to Hasura, you need to provide the Hasura GraphQL endpoint in the `HASURA_ENDPOINT` environment variable. Note that this should include the full path of the GraphQL endpoint, usually ending with `/v1/graphql`.
For example, in the [default docker-compose file of the HBP repository](https://github.com/nhost/hasura-backend-plus/blob/master/docker-compose.yaml), `HASURA_ENDPOINT` equals `http://graphql-engine:8080/v1/graphql`.

You also need to provide a valid Hasura admin secret key in the `HASURA_GRAPHQL_ADMIN_SECRET` environment variable. Note that this variable is mandatory for HBP to work, i.e. HBP won't work if your Hasura instance is not secured with such an admin key. You can find further reading about admin secret keys in the [Hasura documentation](https://hasura.io/docs/1.0/graphql/manual/deployment/production-checklist.html#set-an-admin-secret).

The last point of attention is to make sure both HBP and Hasura are using the same JWT configuration: as HBP will generate the JWT used for authentication in Hasura, it is very important that JWT is configured in a way that Hasura understands it. You will find more information on how to configure JWT in HBP.

## Configure JWT

<!-- TODO - JWKS endpoint -->

## Migrations

::: warning
Before running migrations on any sort, it is HIGHLY recommended to make a backup of your database.
:::

To get HBP running correctly we must configure Hasura and PostgreSQL. This configurations is done using Hasura migrations and will add the correct database configuration and apply the correct Hasura metadata.

### Apply migrations for a new installation

For a complete new installation you can have Hasura apply the migrations automatically for you, using Docker. This is an example in docker-compose.

```yaml
graphql-engine:
  image: hasura/graphql-engine:v1.2.1.cli-migrations-v2
  depends_on:
    - 'postgres'
  restart: always
  env_file: .env
  ports:
    - '8080:8080'
  volumes:
    - [path-to]/hasura-backend-plus/migrations:/hasura-migrations
    - [path-to]/hasura-backend-plus/metadata:/hasura-metadata
```

### Migrating from HBP v1

Hasura Backend Plus v2 introduces some brand new features, coming with some breaking changes:

- While all the former v1 features exist in v2, the [API endpoints](api) have been modified, and some may behave slightly differently. You may need to change your frontend applications accordingly.
- The Storage module have been completely rewritten. <!-- TODO link to storage -->
- The refresh token is now stored in an [HTTP cookie](https://developer.mozilla.org/en-US/docs/Web/HTTP/Cookies) <!-- TODO link to refresh token / cookies system -->

To upgrade from v1 to v2:

::: tip
We'll be using the [Hasura CLI](https://hasura.io/docs/1.0/graphql/manual/hasura-cli/index.html). Make sure it's installed on your computer.
:::

::: tip
All user and account data will be copied to the new v2 schema and work out of the box.
:::

1. Make sure you have backed up your database!

2. Download HBP locally and change directory

```sh
git clone git@github.com:nhost/hasura-backend-plus.git
cd hasura-backend-plus
```

3. Create empty `config.yaml` file.

```sh
touch config.yaml
```

4. Re order the directories so we'll be using the `migrations-v1` directory.

```sh
rm -rf migrations
mv migrations-v1 migrations
```

5. Apply migrations

Note: `[endpoint]` should not include `/v1/graphql`.

```sh
hasura migrate apply --endpoint=[endpoint] --admin-secret=[admin-secret]
```

6. Clean up

```sh
cd ..
rm -rf hasura-backend-plus
```

You have now migrated from HBP v1 to HBP v2 schema. You can no start using HBP v2!

### Experimental

HBP can apply migrations automatically on startup. Both from a zero to version 2. And from version 1 to version 2. However, this is a experimental feature and it's not recommended to be used in production since it could lead to unwanted side effects.

::: tip
The HBP migration system relies on [Hasura CLI](https://hasura.io/docs/1.0/graphql/manual/hasura-cli/index.html) and uses a [v1 migrations/metadata configuration](https://hasura.io/docs/1.0/graphql/manual/migrations/config-v1/index.html), as the config v2 doesn't allow metadata incremental change (yet?).
:::

By default, HBP does not checks when starting if its schema is already present in the database. If not, it runs the necessary SQL migrations and loads the related Hasura metadata, while keeping the existing database and Hasura metadata unchanged.

<!-- TODO link to the database schema -->

You can disable this automatic check and migration system by setting `AUTO_MIGRATE=false`.

If you want to upgrading HBP v1 to v2, you can set `AUTO_MIGRATE=v1` and restart HBP. HBP will then upgrade the database and Hasura for HBPv2. All user and account data will be placed correctly in the new v2 schema.

## Registration

### Activate accounts

By default, accounts are automatically activated on registration. You may want to change this so you add a step to the registration process.

To deactivate autoactivation, set the environment variable `AUTO_ACTIVATE_NEW_USERS=false`

In addition to this, you can send a verification email with an activation link. You will then need to [configure the connection to a SMTP server](#enable-emails).

If SMTP is enabled, then the user will receive an email with an activation link. If the activation succeeds, the user is redirected to the url found in the `REDIRECT_URL_SUCCESS` environment variable. If it fails, they will redirected to the url given by the `REDIRECT_URL_ERROR` environment variable.

You can change the default email templates. In order to do so, you can mount [custom configuration files](#custom-configuration-files) when using docker, or change files in the [custom directory](https://github.com/nhost/hasura-backend-plus/tree/master/custom) when running HBP from the source code.
Other email templates are available and described [here](#email-templates)

### Limit email domains

You can limit registration to ranges of emails that are only part of a whitelist. For instance, you may want to limit registration only to the email addresses of your own organisation. You can pass a list of comma-separated email domains to the `ALLOWED_EMAIL_DOMAINS` environment variable, for instance:

```
ALLOWED_EMAIL_DOMAINS=gmail.com,yourorganisation.com
```

### Password constraints

By default, clients can register with a password of at least three characters. You can change this in setting a higher value:

```
MIN_PASSWORD_LENGTH=6
```

You can ask HBP to check on [Have I Been Pwned](https://haveibeenpwned.com/Passwords) if the password has been previously exposed in data breaches. If so, the registration will fail. This option is disabled by default. You can change it to:

```
HIBP_ENABLE=true
```

### Additional registration fields

You may want to extend the `public.users` table with your own fields and relations, and to expect the client to set some of them when registering. It is possible to set a list of columns in the `REGISTRATION_CUSTOM_FIELDS` environment value.

<!-- TODO link to the page on schema -->

Here is an example on the way to proceed to add a `nickname` value to the registration:

1. Add a column `nickname` of type text to the `public.users` table
2. Set the environment variable `REGISTRATION_CUSTOM_FIELDS=nickname`
3. The registration endpoint now expects a `nickname` value in addition to `email` and `password`

::: warning
Any given field must exist in the `users` GraphQL type that corresponds to the `public.users` PostgreSQL table, or registration will fail.
:::

<!-- TODO link to JWT custom claims -->

## Authentication

### OAuth Providers

### Two-factor Authentication

## Enable emails

## Custom configuration files

<!-- TODO explain the contents of the configuration files, and how to mount them with a docker volume -->

### Storage Rules

### Email templates

### Private key

## Custom User Schema

## Rate limiting

<!-- TODO MAX_REQUESTS, TIME_FRAME, healthz -->

## Environment Variables

### General

| Name                          | Default   | Description                                                                                                 |
| ----------------------------- | --------- | ----------------------------------------------------------------------------------------------------------- |
| `HASURA_ENDPOINT` (required)  |           | Url of the Hasura GraphQL engine endpoint used by the backend to access the database.                       |
| `HASURA_GRAPHQL_ADMIN_SECRET` |           | The secret set in the Hasura GraphQL Engine to allow admin access to the service. **Strongly recommended**. |
| `HOST`                        | localhost | Listening ip of the service                                                                                 |
| `PORT`                        | 3000      | Port of the service                                                                                         |
| `SERVER_URL`                  |           | Current server URL. Currently used only for creating links from email templates                             |
| `MAX_REQUESTS`                | 100       | Maximum requests per IP within the following `TIME_FRAME`.                                                  |
| `TIME_FRAME`                  | 900000    | Timeframe used to limit requests from the same IP, in milliseconds. Defaults to 15 minutes.                 |

### Authentication

| Name                         | Default                 | Description                                                                                                                                                                                  |
| ---------------------------- | ----------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `AUTH_LOCAL_USERS_ENABLE`    | true                    | Enable local users (email/pw) to register and login                                                                                                                                          |
| `ALLOWED_EMAIL_DOMAINS`      |                         | List of comma-separated email domain names that are allowed to register.                                                                                                                     |
| `AUTO_ACTIVATE_NEW_USERS`    | false                   | When set to true, automatically activate the users once registererd.                                                                                                                         |
| `COOKIE_SECRET`              |                         |                                                                                                                                                                                              |
| `DEFAULT_USER_ROLE`          | user                    |                                                                                                                                                                                              |
| `DEFAULT_ALLOWED_USER_ROLES` | user                    | Comma spearated list of allowed roles assigned to each user. Defaults to DEFAULT_USER_ROLE.                                                                                                  |
| `HIBP_ENABLE`                | false                   |                                                                                                                                                                                              |
| `JWT_ALGORITHM`              | RS256                   | Valid values: RS256, RS384, RS512, HS256, HS384, HS512                                                                                                                                       |
| `JWT_KEY`                    |                         | Encryption secret. Required when using a SHA (RS*) algorithm. When using a RSA algorithm (RS*), should contain a valid RSA PEM key, otherwise `JWT_KEY_FILE_PATH` will be used.              |
| `JWT_EXPIRES_IN`             | 15                      |                                                                                                                                                                                              |
| `JWT_KEY_FILE_PATH`          | custom/keys/private.pem | Path to the RSA PEM private key file when using a RSA (RS\*) algorithm and no `JWT_KEY` is set. When used, will create a random key if the file is not found.                                |
| `MIN_PASSWORD_LENGTH`        | 3                       | Minimum allowed password length.                                                                                                                                                             |
| `REDIRECT_URL_ERROR`         |                         |                                                                                                                                                                                              |
| `REDIRECT_URL_SUCCESS`       |                         |                                                                                                                                                                                              |
| `JWT_REFRESH_EXPIRES_IN`     | 43200                   |                                                                                                                                                                                              |
| `EMAILS_ENABLE`              | false                   | When set to true, emails are sent on certain steps, like after registration for account activation when autoactivation is deactivated, or for changing emails or passwords                   |
| `SMTP_HOST`                  |                         | SMTP server path to use for sending emails.                                                                                                                                                  |
| `SMTP_PASS`                  |                         | Password to authenticate on the SMTP server.                                                                                                                                                 |
| `SMTP_USER`                  |                         | Username to authenticate on the SMTP server.                                                                                                                                                 |
| `SMTP_PORT`                  | 587                     | SMTP server port.                                                                                                                                                                            |
| `SMTP_SECURE`                | false                   | Set to true when the SMTP uses SSL.                                                                                                                                                          |
| `REGISTRATION_CUSTOM_FIELDS` |                         | Fields that need to be passed on to the registration patload, and that correspond to columns of the `public.users`table.                                                                     |
| `JWT_CUSTOM_FIELDS`          |                         | List of comma-separated column names from the `public.users` tables that will be added to the `https://hasura.io/jwt/claims`JWT claims. Column names are kebab-cased and prefixed with `x-`. |
| `OTP_ISSUER`                 | HBP                     | One-Time Password issuer name used with Muti-factor authentication.                                                                                                                          |

### Storage

| Name                   | Default | Description |
| ---------------------- | ------- | ----------- |
| `S3_ENDPOINT`          |         |             |
| `S3_BUCKET`            |         |             |
| `S3_ACCESS_KEY_ID`     |         |             |
| `S3_SECRET_ACCESS_KEY` |         |             |
| `S3_SSL_ENABLED`       | true    |             |
