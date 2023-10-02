# Cloudfront Auth

[Google Apps (G Suite)](https://developers.google.com/identity/protocols/OpenIDConnect), [Microsoft Azure AD](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-protocols-oauth-code), [GitHub](https://developer.github.com/apps/building-oauth-apps/authorization-options-for-oauth-apps/), [OKTA](https://www.okta.com/), [Auth0](https://auth0.com/), [Centrify](https://centrify.com) authentication for [CloudFront](https://aws.amazon.com/cloudfront/) using [Lambda@Edge](http://docs.aws.amazon.com/lambda/latest/dg/lambda-edge.html). The original use case for `cloudfront-auth` was to serve private S3 content over HTTPS without running a proxy server in EC2 to authenticate requests; but `cloudfront-auth` can be used authenticate requests of any Cloudfront origin configuration.

## Description

Upon successful authentication, a cookie (named `TOKEN`) with the value of a signed JWT is set and the user redirected back to the originally requested path. Upon each request, Lambda@Edge checks the JWT for validity (signature, expiration date, audience and matching hosted domain) and will redirect the user to configured provider's login when their session has timed out.

## Usage

If your CloudFront distribution is pointed at a S3 bucket, [configure origin access identity](http://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html#private-content-creating-oai-console) so S3 objects can be stored with private permissions. (Origin access identity requires the S3 ACL owner be the account owner. Use our [s3-object-owner-monitor](https://github.com/Widen/s3-object-owner-monitor) Lambda function if writing objects across multiple accounts.)

Enable SSL/HTTPS on your CloudFront distribution; AWS Certificate Manager can be used to provision a no-cost certificate.

Session duration is defined as the number of hours that the JWT is valid for. After session expiration, cloudfront-auth will redirect the user to the configured provider to re-authenticate. RSA keys are used to sign and validate the JWT.

## Lambda Deployment Packages

### Custom Packages

A custom package has all the configuration baked into it (including secrets). To build a custom package, execute:

```bash
./build.sh
```

The build script prompts you for the required configuration parameters, which are described in [Identity Provider Guides](#identity-provider-guides). If the files `id_rsa` and `id_rsa.pub` do not exist they will be automatically generated by the build. To disable all issued JWTs upload a new ZIP using the Lambda Console after deleting the `id_rsa` and `id_rsa.pub` files (a new key will be automatically generated).

### Generic Packages

A generic package retrieves its configuration at runtime from the [AWS Systems Manager](https://aws.amazon.com/systems-manager/) Parameter Store and [AWS Secrets Manager](https://aws.amazon.com/secrets-manager/). Currently, support for this type of package has been added for OKTA Native only. To disable all issued JWTs, rotate the key pair using the Secrets Manager console.

Generic packages are pre-built and available for download from the Releases page of this GitHub repository. [Terraform modules](./infra/terraform/README.md) are available for downloading and deploying generic packages and their configuration resources.

## Identity Provider Guides

### Github

1. Clone or download this repo
1. Navigate to your organization's [profile page](https://github.com/settings/profile), then choose OAuth Apps under Developer settings.
    1. Select **New OAuth App**
    1. For **Authorization callback URL** enter your Cloudfront hostname with your preferred path value for the authorization callback. Example: `https://my-cloudfront-site.example.com/_callback`
1. Execute `./build.sh` in the downloaded directory. NPM will run to download dependencies and a RSA key will be generated.
    1. Choose `Github` as the authorization method and enter the values for Client ID, Client Secret, Redirect URI, Session Duration and Organization
        * cloudfront-auth will check that users are a member of the entered Organization.
1. Upload the resulting `zip` file found in your distribution folder using the AWS Lambda console and jump to the [configuration step](#configure-lambda-and-cloudfront)

### Google

1. Clone or download this repo
1. Go to the **Credentials** tab of your [Google developers console](https://console.developers.google.com)
    1. Create a new Project
    1. Create an **OAuth Client ID** from the **Create credentials** menu
    1. Select **Web application** for the Application type
    1. Under **Authorized redirect URIs**, enter your Cloudfront hostname with your preferred path value for the authorization callback. Example: `https://my-cloudfront-site.example.com/_callback`
1. Execute `./build.sh` in the downloaded directory. NPM will run to download dependencies and a RSA key will be generated.
1. Choose `Google` as the authorization method and enter the values for Client ID, Client Secret, Redirect URI, Hosted Domain and Session Duration
1. Select the preferred authentication method
    1. Hosted Domain (verify email's domain matches that of the given hosted domain)
    1. JSON Email Lookup
        1. Enter your JSON Email Lookup URL (example below) that consists of a single JSON array of emails to search through
    1. Google Groups Lookup
        1. [Use Google Groups to authorize users](https://github.com/Widen/cloudfront-auth/wiki/Google-Groups-Setup)
1. Upload the resulting `zip` file found in your distribution folder using the AWS Lambda console and jump to the [configuration step](#configure-lambda-and-cloudfront)

### Microsoft Azure

1. Clone or download this repo
1. In your Azure portal, go to Azure Active Directory and select **App registrations**
    1. Create a new application registration with an application type of **Web app / api**
    1. Once created, go to your application `Settings -> Keys` and make a new key with your desired duration. Click save and copy the value. This will be your `client_secret`
    1. Above where you selected `Keys`, go to `Reply URLs` and enter your Cloudfront hostname with your preferred path value for the authorization callback. Example: `https://my-cloudfront-site.example.com/_callback`
1. Execute `./build.sh` in the downloaded directory. NPM will run to download dependencies and a RSA key will be generated.
1. Choose `Microsoft` as the authorization method and enter the values for [Tenant](https://docs.microsoft.com/en-us/azure/active-directory/develop/active-directory-howto-tenant), Client ID (**Application ID**), Client Secret (**previously created key**), Redirect URI and Session Duration
1. Select the preferred authentication method
    1. Azure AD Membership (default)
    1. JSON Username Lookup
        1. Enter your JSON Username Lookup URL (example below) that consists of a single JSON array of usernames to search through
1. Upload the resulting `zip` file found in your distribution folder using the AWS Lambda console and jump to the [configuration step](#configure-lambda-and-cloudfront)

### OKTA

1. Clone or download this repo
1. Sign in to OKTA with your administrator account and navigate to the `Applications` tab.
1. Add Application
    1. Select the `Web` application type
    1. Base URI: CloudFront distribution domain name (`https://{cf-endpoint}.cloudfront.net`)
    1. Login Redirect URI: CloudFront distribution domain name with callback path (`https://{cf-endpoint}.cloudfront.net/_callback`)
    1. Group Assignments: Optional
    1. Grant Type Allowed: Authorization Code
    1. Done
1. Gather the following information for Lambda configuration
    1. Client Id and Client Secret from the application created in our previous step (can be found at the bottom of the general tab)
    1. Base Url
        1. This is named the 'Org URL' and can be found in the top right of the Dashboard tab.
1. Execute `./build.sh` in the downloaded directory. NPM will run to download dependencies and a RSA key will be generated.
1. Choose `OKTA` as the authorization method and enter the values for Base URL (Org URL), Client ID, Client Secret, Redirect URI, and Session Duration
1. Upload the resulting `zip` file found in your distribution folder using the AWS Lambda console and jump to the [configuration step](#configure-lambda-and-cloudfront)

### Auth0

1. Clone or download this repo
1. Go to the **Dashboard** of your Auth0 admin page
    1. Click **New Application**
    1. Select **Regular Web App** and click **Create**.
    1. Now select an application type and follow the steps for 'Quick Start' or use your own app.
    1. Go to application **Settings** and enter required details. In **Allowed Callback URLs** enter your Cloudfront hostname with your preferred path value for the authorization callback. Example: `https://my-cloudfront-site.example.com/_callback`
1. Execute `./build.sh` in the downloaded directory. NPM will run to download dependencies and a RSA key will be generated.
1. Choose `AUTH0` as the authorization method and enter the values for Base URL (Auth0 Domain), Client ID, Client Secret, Redirect URI, and Session Duration
1. Upload the resulting `zip` file found in your distribution folder using the AWS Lambda console and jump to the [configuration step](#configure-lambda-and-cloudfront)

### Centrify

1. Clone or download this repo
1. Go to the **Dashboard** of your Centrify admin page
    1. Click **Web Apps** from the LHS.
    1. Click **Add Web App** and select the **Custom Tab**.
    1. Add an **OpenID Connect** webapp and click **Yes** to confirm.
1. Fill in naming and logo information and then switch to the **Trust** tab.
1. Enter service provider information. In **Authorized Redirect URIs** enter your Cloudfront hostname with your preferred path value for the authorization callback. Example: `https://my-cloudfront-site.example.com/_callback`
1. Execute `./build.sh` in the downloaded directory. NPM will run to download dependencies and a RSA key will be generated.
1. Choose `CENTRIFY` as the authorization method and enter the values for Base URL (Centrify Resource application URL), Client ID, Client Secret, Redirect URI, and Session Duration (which is available from the **Tokens** tab).
1. Upload the resulting `zip` file found in your distribution folder using the AWS Lambda console and jump to the [configuration step](#configure-lambda-and-cloudfront)

### OKTA Native

1. Clone or download this repo
1. Sign in to OKTA with your administrator account and navigate to the `Applications` tab.
1. Add Application
    1. Select the `Native` application type
    1. Base URI: CloudFront distribution domain name (`https://{cf-endpoint}.cloudfront.net`)
    1. Login Redirect URI: CloudFront distribution domain name with callback path (`https://{cf-endpoint}.cloudfront.net/_callback`)
    1. Group Assignments: Optional
    1. Grant Type Allowed: Authorization Code
    1. Done
1. Gather the following information for Lambda configuration
    1. Client Id from the application created in our previous step (can be found at the bottom of the general tab)
    1. Base Url
        1. This is named the 'Org URL' and can be found in the top right of the Dashboard tab.
1. To use the [generic package](#generic-packages), jump straight to [Terraform Modules for CloudFront Authentication](./infra/terraform/README.md).
1. Execute `./build.sh` in the downloaded directory. NPM will run to download dependencies and a RSA key will be generated.
1. Choose `OKTA Native` as the authorization method and enter the values for Base URL (Org URL), Client ID, PKCE Code Verifier Length, Redirect URI, and Session Duration
1. Upload the resulting `zip` file found in your distribution folder using the AWS Lambda console and jump to the [configuration step](#configure-lambda-and-cloudfront)

## Configure Lambda and CloudFront

See [Manual Deployment](https://github.com/Widen/cloudfront-auth/wiki/Manual-Deployment) __*or*__ [AWS SAM Deployment](https://github.com/Widen/cloudfront-auth/wiki/AWS-SAM-Deployment)

## Authorization Method Examples

* [Use Google Groups to authorize users](https://github.com/Widen/cloudfront-auth/wiki/Google-Groups-Setup)

* JSON array of email addresses

    ```json
    [ "foo@gmail.com", "bar@gmail.com" ]
    ```

## Testing

Detailed instructions on testing your function can be found [in the Wiki](https://github.com/Widen/cloudfront-auth/wiki/Debug-&-Test).

## Build Requirements

* [npm](https://www.npmjs.com/) ^7.20.0
* [node](https://nodejs.org/en/) ^16.0
* [openssl](https://www.openssl.org)

## Building Generic Packages

If you need to build a generic package yourself, execute:

```bash
./build.sh package
```

The supported values of `package` are:

* `okta_native` - builds a generic Lambda package for OKTA Native authentication
* `rotate_key_pair` - builds a Lambda package for rotating the RSA keys in AWS Secrets Manager

GitHub Actions automatically creates a new GitHub release when the repository owner pushes a tag that begins with `v`.

e.g.

```sh
git tag -a -m "Target AWS Lambda Node.js 14.x runtime" v3.0.0
git push origin v3.0.0
```

## Contributing

All contributions are welcome. Please create an issue in order open up communication with the community.

When implementing a new flow or using an already implemented flow, be sure to follow the same style used in `build.js`. The config.json file should have an object for each request made. For example, `openid.index.js` converts config.AUTH_REQUEST and config.TOKEN_REQUEST to querystrings for simplified requests (after adding dynamic variables such as state or nonce). For implementations that are not generic (most), endpoints are hardcoded in to the config (or discovery documents).

Be considerate of our [limitations](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/cloudfront-limits.html#limits-lambda-at-edge). The zipped function can be no more than 1MB in size and execution cannot take longer than 5 seconds, so we must pay close attention to the size of our dependencies and complexity of operations.
