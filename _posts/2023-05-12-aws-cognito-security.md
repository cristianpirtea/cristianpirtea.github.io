---
layout: post
title: Improving security in AWS Cognito 
subtitle:
category: dev
tags: [cloud, aws]
author: Lucian Patian, Martin Birtel
author_email: patianl@haufe.com, martin.birtel@haufe-lexware.com
header-img: "images/aws-cognito-security/awsblogpost_bg8.png"
---

# A call for debate on how to better secure AWS Cognito

AWS Cognito is an identity management service for users who sign-up directly and for federated users who sign-in with external identity providers. It grants the ability to control access to web and mobile applications.

The user handling is being done via the [**User Pools**](http://docs.aws.amazon.com/cognito/latest/developerguide/getting-started-with-cognito-user-pools.html) while the identities and assign permissions for users are configured inside the [**Identity Pools**](http://docs.aws.amazon.com/cognito/latest/developerguide/getting-started-with-identity-pools.html)

Over time, we found out that some Cognito user pools were deployed in their default configuration, making them possible honeypots.

## Updating users via the public Cognito API:

We discovered that the [**app client configuration**](https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-settings-client-apps.html) attributes are writable by default. As a result, if a user obtains a token with the *aws.cognito.signin.user.admin* scope, they can modify a local user's attributes via the Cognito public endpoint (https://cognito-idp.REGION.amazonaws.com) using the *x-amz-target* header with the "CognitoIdentityProvider.UpdateUserAttributes" value. Moreover, a user can also delete its attributes or its account using the same approach.

Imagine that some developers are not aware of this fact and use custom attributes for tenant separation or RBAC. An attacker can breach the tenant separation, access foreign user data, etc. by modifying these attributes (e.g. by changing attribute "custom:tenant: companyA" to "custom:tenant: companyB").

### The generic solution 

* Remove the *aws.cognito.signin.user.admin* from the app client scopes. Keep in mind that this does not solve the issue for public clients. When authenticating directly against the Cognito public endpoint (initiateAuth) with a user and password flow (or others), you always get a token with the *aws.cognito.signin.user.admin* scope.
* Remove the write-access permission in the app client configuration. Consider that this breaks in the case of federation with an external IdP because when a user signs in, Cognito updates the mapped attributes with the latest information from the IdP, even if its current value already matches the latest information.
This happens automatically in Cognito's backend involving no public APIs. Due to this nature, your SAML logins won't be affected by blocking the API calls discussed below.

### The custom solution 

* Block undesired calls towards the Cognito public API using AWS WAF rules. Basically, you need to block API calls using the *x-amz-target* header containing the "AWSCognitoIdentityProviderService.API_ACTION" string. A list of all APIs can be found here: https://docs.aws.amazon.com/cognito/latest/developerguide/user-pools-API-operations.html#user-pool-apis-auth-unauth-token-auth 

You can read more about AWS WAF and Cognito here: https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-waf.html

You might be thinking how do you know which API calls should you allow and which should you block. You're in luck because we also faced the same issue. Our suggestion is to initially create a WAF rule in count mode which tracks all API calls made by your Cognito userpool towards the public endpoint, centralize the data and afterwards build a new WAF rule that blocks all API calls except the ones tracked by the first rule.

Below is an example of a WAF rule that counts all the *AWSCognitoIdentityProviderService* calls via the "x-amz-target" header:

```yaml

{
  "Name": "Cognito-counting-calls-to-public-api",
  "Priority": 0,
  "Statement": {
    "ByteMatchStatement": {
      "SearchString": "AWSCognitoIdentityProviderService",
      "FieldToMatch": {
        "SingleHeader": {
          "Name": "x-amz-target"
        }
      },
      "TextTransformations": [
        {
          "Priority": 0,
          "Type": "NONE"
        }
      ],
      "PositionalConstraint": "CONTAINS"
    }
  },
  "Action": {
    "Count": {}
  },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "Cognito-counting-calls-to-public-api"
  }
}

```  

The next step is to create a WAF Regex pattern set where you define the allowed calls tracked with the above rule. For this example we allowed only the *^AWSCognitoIdentityProviderService.InitiateAuth$* and *^AWSCognitoIdentityProviderService.GetUser$* patterns:

{:.center}
![]( /images/aws-cognito-security/waf_regex.png){:style="width:100%"} 

Further you create the WAF rule that blocks all API calls initiated by your userpool towards the public API, except the patterns defined in the regex:

```yaml

{
  "Name": "BLOCK-all-public-apis",
  "Priority": 1,
  "Statement": {
    "AndStatement": {
      "Statements": [
        {
          "NotStatement": {
            "Statement": {
              "RegexPatternSetReferenceStatement": {
                "ARN": "arn:aws:wafv2:REGION:ACCOUNT_NUMBER:regional/regexpatternset/REGEX_NAME/REGEX_ID",
                "FieldToMatch": {
                  "SingleHeader": {
                    "Name": "x-amz-target"
                  }
                },
                "TextTransformations": [
                  {
                    "Priority": 0,
                    "Type": "NONE"
                  }
                ]
              }
            }
          }
        },
        {
          "SizeConstraintStatement": {
            "FieldToMatch": {
              "SingleHeader": {
                "Name": "x-amz-target"
              }
            },
            "ComparisonOperator": "GT",
            "Size": 0,
            "TextTransformations": [
              {
                "Priority": 0,
                "Type": "NONE"
              }
            ]
          }
        }
      ]
    }
  },
  "Action": {
    "Block": {}
  },
  "VisibilityConfig": {
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "BLOCK-all-public-apis"
  }
}

```  

Besides the regex reference statement, we added an AND condition using the "Field to match" for the *x-amz-target* header
greater than 0. This is useful for scenarios where the *x-amz-target* header is missing, allowing the calls to bypass our
WAF block rule (example: SAML login where the Hosted UI is used).

## Account takeover via unverified email/phone

Most of the user pools are configured with multiple login options, including email, username or phone. By default, the user pool option "Keep original attribute value active when an update is pending" is turned on. Make sure it stays like this to be protected.

If the option "Keep original attribute value active when an update is pending" is not turned on and your application consuming a Cognito issued token does not check the *email_verified* attribute but uses it directly to load the data/identify of a user, it will be exposed to a possbile takeover.

An attacker can change the email attribute value of its own user to impersonate a victim's email address, then login to an application using an alternative login option like username. The application processing the Cognito issued token will see the victim's email address and use the unverified email attribute to load data/identify of the user.

### The solution 

* Make sure the *Keep original attribute value active when an update is pending* setting is on in all your Cognito userpools.
* Modify your applications to respect the *email_verified* and *phone_number_verified* claims.
* If possible, modify your applications not to rely on modifiable attributes like email, username, etc. [**Instead use the combination of immutable subject (sub) and issuer (iss) claims to identify a user**](https://openid.net/specs/openid-connect-core-1_0.html#ClaimStability). You can find extra details about the Cognito ID Token Payload [**here**](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-the-id-token.html)


### Have you faced any of the scenarios above? How did you mitigate the issues? What else caught your attention when it comes to securing Cognito? 
&nbsp;  