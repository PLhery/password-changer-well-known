# Password Changer .well-known [DRAFT]

The goal of this respository is to provide documentation for using password changer .well-known and be listed as compatible on Dashlane's automatic password changer feature.

## Introduction

Password changer provides the ability to automatically update passwords by calling a compatible service's API endpoint.

In order to be compatible, a service needs:

-   have a file placed within the `.well-known` folder at the web root of the service describing the API endpoint to call
-   an API endpoint that will receive the password change request

## Implementing the .well-known on your server

The following JSON file should be placed at the root of your website: `/.well-known/password-changer`.

```js
{
    "version": "1.0",
    "endpoints": [
        // Multiple endpoints (if you need QA)
        {
            "auth": "Form",
            "url": "https://api.example.com/1.0/password_changer" // Must be https
        },
        {
            "auth": "Form",
            "url": "https://api.example.com/1.0/password_changer_qa", // Must be https
            "allowList": ["hashOfEmail1", "hashOfEmail1"] // SHA256 hash list of authorized emails
        }
    ]
}
```

The `auth` parameter defines how Password Changer will authenticate to the service API. Please refer to the next section for the available auth options.

The `url` parameter defines the address of the API endpoint where password changer will send requests.

The `allowList` parameter defines a list of authorized emails that will be allowed to use a specific endpoint.
For instance, if you need to do some testing of a new API endpoint, you can set the hash (SHA256) of your Dashlane's account login in this list.

## API endpoint: Authentication

### Form Auth

Password Changer will make an HTTPS POST request to the specified API endpoint with the following structure:

```bash
curl --location --request POST 'https://api.example.com/api/changePassword' \
--header 'Content-Type: application/x-www-form-urlencoded' \
--data-urlencode 'password=oldpassword' \
--data-urlencode 'login=user@mail.com' \
--data-urlencode 'newPassword=azerty12'

```

The 3 expected body fields are:

-   `username` (string)
-   `password` (string)
-   `newPassword` (string)

### Others Auth

For now we only support `Form` Auth but in the future we will support `Basic` Auth and `Digest` Auth.

## API endpoint: Replying to password change requests

In order to update the password in the user's vault or handle a potential error, password changer expect a JSON formated object with the following specifications.

All reply should be JSON object with a status key like this:

```json
{
    "status": "STATUS_CODE"
}
```

### Success

If the change is successful, you should reply a **200 response** with `OK` status.

```json
{
    "status": "OK"
}
```

### Errors

In case of an error, you should reply a **401 response** and we support the following error codes:

| **Error code**                                       | **Description**                                                                                                |
| ---------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| LOGIN.PASSWORD_INCORRECT                             | Incorrect user supplied password.                                                                              |
| LOGIN.NOT_FOUND                                      | Login supplied by user not present in site authentication database.                                            |
| LOGIN.GENERIC_FAILURE                                | Incorrect credentials, login or password.                                                                      |
| LOGIN.ACCOUNT_LOCKED                                 | The user account is locked.                                                                                    |
| SECURITY_REQUIREMENT.TOO_SHORT                       | The new password is too short.                                                                                 |
| SECURITY_REQUIREMENT.TOO_LONG                        | The new password is too long.                                                                                  |
| SECURITY_REQUIREMENT.CAN_NOT_REUSE_PREVIOUS_PASSWORD | The new password must be different from the old password.                                                      |
| SECURITY_REQUIREMENT.NO_SEQUENTIAL_CHARS             | The password can't contain a sequence of the same character.                                                   |
| USER.PROFILE_INCOMPLETE                              | The user must complete their profile before being able to change their password.                               |
| USER.ACCOUNT_NOT_VERIFIED                            | The user account is not verified. Typically a link sent by email during the registration has not been clicked. |
| USER.NEEDS_TO_ACCEPT_TOS                             | The user must accept Term Of Service.                                                                          |
| NEED_USER_ACTION                                     | There's something that prevent us to change user password (unpaid bill...).                                    |
| WEBSITE_UNAVAILABLE                                  | The site is in maintenance, not accessible.                                                                    |
| SECURITY_REQUIREMENT.NOT_STRONG_ENOUGH               | New password rejected by site password policies.                                                               |
| ABORTED                                              | Password change has been interrupted.                                                                          |
| VERIFICATION.METHOD_VERIFICATION_FAIL                | (NA)                                                                                                           |
| VERIFICATION.WRONG_CODE                              | Bad user challenge code.                                                                                       |
| VERIFICATION.TIMEOUT                                 | The user challenge resolution is taking too long.                                                              |
| VERIFICATION.UNKNOWN_VERIFICATION_ERROR              | Something bad happened while solving a user challenge.                                                         |
| UNKNOWN_ERROR                                        | Something bad happened.                                                                                        |

For instance, if the password is incorrect, here is the response to send:

```json
{
    "status": "LOGIN.PASSWORD_INCORRECT"
}
```

### Captcha and 2FA

You should reply a **400 response** with the `NEED_VERIFICATION` status.

We will then ask the user to answer solve the challenge, and will call again your endpoint.
Two parameters, `verificationResponse` and optionally `verificationResponseKey`, will be sent alongside the other parameters.

Dashlane only supports two main methods of verification for now:

1. `2faVerification`:
   Reply should be the following :

    ```js
    {
        "status": "NEED_VERIFICATION",
        "verificationType": "2FA",
        "2faVerification": {
            // This hint will be only shown when the user clicks on the "more info" button
            "hintText": "Enter the code we sent to your number ending in 99",
            "type": "SMS", // Optional. Can be SMS, EMAIL, APP, OTHER.
            "inputType": "DIGITS", // Optional. Can be DIGITS, LETTERS or ANY
            "inputLength": 4, // Optional
            "responseKey": "jkf3kf32ewfji32ijgger" // Optional. We can send this key back with the user response
        }
    }
    ```

2. `reCaptchaVerification`:

    ```js
    {
        "status": "NEED_VERIFICATION",
        "verificationType": "RECAPTCHA_V2",
        "reCaptchaVerification": {
            "sitekey": "6Le-wvkSAAAAAPBMRTvw0Q4Muexq9bi0DJwx_mJ-", // Sitekey, provided by recaptcha
            "domain": "dashlane-recaptcha.com", // Optional. The domain must be whitelisted in your recaptcha interface
            "responseKey": "jkf3kf32ewfji32ijgger" // Optional. We can send this key back with the user response
        }
    }
    ```

In the response, the parameter `verificationResponse` will be the [response token](https://developers.google.com/recaptcha/docs/verify), usually called `g-recaptcha-response`.

This object `reCaptchaVerification` can also be in the well-known manifest, in case you always need a verified captcha.

## Ready to join?

Once you have implemented this you can get in touch with us by email and we'll add your website to the list : [dev-relationship (at) dashlane.com](mailto:dev-relationship@dashlane.com).

If you have any question on this process use the same email or open an issue on the repository.

## License

This document and associated materials are provided by Dashlane under [CC BY 4.0 License](https://creativecommons.org/licenses/by/4.0/).