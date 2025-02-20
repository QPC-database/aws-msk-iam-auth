## Amazon MSK Library for AWS Identity and Access Management 

## License

This project is licensed under the Apache-2.0 License.

## Introduction
The Amazon MSK Library for AWS Identity and Access Management enables developers to use 
AWS Identity and Access Management (IAM) to connect to their Amazon Managed Streaming for Apache Kafka (Amazon MSK) clusters. 
It allows JVM based Apache Kafka clients to use AWS IAM for authentication and authorization against 
Amazon MSK clusters that have AWS IAM enabled as an authentication mechanism.

This library provides a new Simple Authentication and Security Layer (SASL) mechanism called `AWS_MSK_IAM`. This new 
SASL mechanism can be used by Kafka clients to authenticate against Amazon MSK clusters using AWS IAM.

* [Amazon Managed Streaming for Apache Kafka][MSK]
* [AWS Identity and Access Management][IAM]
* [AWS IAM authentication and authorization for MSK ][MSK_IAM]

## Building from source
After you've downloaded the code from GitHub, you can build it using Gradle. Use this command:
 
 `gradle clean build`
 
The generated jar files can be found at: `build/libs/`.

An uber jar containing the library and all its relocated dependencies except the kafka client and `slf4j-api` can
 also be built. Use this command: 

`gradle clean shadowJar` 

The generated uber jar file can also be found at: `build/libs/`. At runtime, the uber jar expects to find the kafka
 client library and the `sl4j-api` library on the classpath. 

## Using the Amazon MSK Library for IAM Authentication
The recommended way to use this library is to consume it from maven central while building a Kafka client application.

  ``` xml
  <dependency>
      <groupId>software.amazon.msk</groupId>
      <artifactId>aws-msk-iam-auth</artifactId>
      <version>1.1.0</version>
  </dependency>
  ```
If you want to use it with a pre-existing Kafka client, you could build the uber jar and place it in the Kafka client's
classpath.

## Configuring a Kafka client to use AWS IAM
You can configure a Kafka client to use AWS IAM for authentication by adding the following properties to the client's 
configuration. 

```properties
# Sets up TLS for encryption and SASL for authN.
security.protocol = SASL_SSL

# Identifies the SASL mechanism to use.
sasl.mechanism = AWS_MSK_IAM

# Binds SASL client implementation.
sasl.jaas.config = software.amazon.msk.auth.iam.IAMLoginModule required;

# Encapsulates constructing a SigV4 signature based on extracted credentials.
# The SASL client bound by "sasl.jaas.config" invokes this class.
sasl.client.callback.handler.class = software.amazon.msk.auth.iam.IAMClientCallbackHandler
```
This configuration finds IAM credentials using the [AWS Default Credentials Provider Chain][DefaultCreds]. To summarize,
the Default Credential Provider Chain looks for credentials in this order:

1. Environment variables: AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY. 
1. Java system properties: aws.accessKeyId and aws.secretKey. 
1. Web Identity Token credentials from the environment or container.
1. The default credential profiles file– typically located at ~/.aws/credentials (location can vary per platform), and shared by many of the AWS SDKs and by the AWS CLI.  
You can create a credentials file by using the aws configure command provided by the AWS CLI, or you can create it by editing the file with a text editor. For information about the credentials file format, see [AWS Credentials File Format][CredsFile].
1. It can be used to load credentials from credential profiles other than the default one by setting the environment variable  
AWS_PROFILE to the name of the alternate credential profile. Profiles can be used to load credentials from other sources such as AWS IAM Roles. See [AWS Credentials File Format][CredsFile] for more details.
1. Amazon ECS container credentials– loaded from the Amazon ECS if the environment variable AWS_CONTAINER_CREDENTIALS_RELATIVE_URI is set. 
1. Instance profile credentials: used on EC2 instances, and delivered through the Amazon EC2 metadata service.

### Specifying an alternate credential profile for a client

If the client wants to specify a particular credential profile as part of the client configuration rather than through 
the environment variable AWS_PROFILE, they can pass in the name of the profile as a client configuration property:
```properties
# Binds SASL client implementation. Uses the specified profile name to look for credentials.
sasl.jaas.config = software.amazon.msk.auth.iam.IAMLoginModule required awsProfileName="<Credential Profile Name>";
```
#### Specifying a role based credential profile for a client

Some clients may want to assume a role and use the role's temporary credentials to communicate with a MSK
 cluster. One way to do that is to create a credential profile for that role following the rules for 
 [Using an IAM role in the CLI][RoleProfileCLI].
 They can then pass in the name of the credential profile as described [above](#specifying-an-alternate-credential-profile-for-a-client).
 
As an example, let's say a Kafka client is running on an Ec2 instance and the Kafka client wants to use an IAM role
 called `msk_client_role`. The Ec2 instance profile has permissions to assume the `msk_client_role` IAM role although
  `msk_client_role` is not attached to the instance profile.

In such a case, we create a credential profile called `msk_client` that assumes the role `msk_client_role`. 
The credential profile looks like:

```
[msk_client]
role_arn = arn:aws:iam::123456789012:role/msk_client_role
credential_source = Ec2InstanceMetadata
```
The credential profile name `msk_client` is passed in as a client configuration property:
```properties
sasl.jaas.config = software.amazon.msk.auth.iam.IAMLoginModule required awsProfileName="msk_client";
```

Many more examples of configuring credential profiles with IAM roles can be found in [Using an IAM role in the CLI][RoleProfileCLI]. 

### Specifying an AWS IAM Role for a client
The library supports another way to configure a client to assume an IAM role and use the role's temporary credentials.
The IAM role's ARN and optionally the session name for the client can be passed in as client configuration property:

```properties
sasl.jaas.config=software.amazon.msk.auth.iam.IAMLoginModule required awsRoleArn="arn:aws:iam::123456789012:role/msk_client_role" awsRoleSessionName="prodoucer";
```
In this case, the `awsRoleArn` specifies the ARN for the IAM role the client should use and `awsRoleSessionName
` specifies the session name that this particular client should use while assuming the IAM role. If the same IAM
 Role is used in multiple contexts, the session names can be used to differentiate between the different contexts.
The `awsRoleSessionName` is optional.
 
The Default Credential Provider Chain must contain the permissions necessary to assume the client role.
For example, if the client is an EC2 instance, its instance profile should have permission to assume the
 `msk_client_role`.

## Details
This library introduces a new SASL mechanism called `AWS_MSK_IAM`. The `IAMLoginModule` is used to register the
 `IAMSaslClientProvider` as a `Provider` for the `AWS_MSK_IAM` mechanism. The `IAMSaslClientProvider` is used to
 generate a new `IAMSaslClient` every time a new connection to a Kafka broker is opened or an existing connection
  is re-authenticated.  

The `IAMSaslClient` is used to perform the actual SASL authentication for a client. It evaluates challenges and creates
 responses that can be used to authenticate a Kafka client to a Kafka broker configured with AWS IAM authentication
 . Its initial response contains an authentication payload that includes a signature generated using the client's
  credentials. The `IAMClientCallbackHandler` is used to extract the client credentials that are then used for
   generating the signature.
 
 The authentication payload and the signature are generated by the `AWS4SignedPayloadGenerator` class based on the
  parameters specified in `AuthenticationRequestParams`. The authentication payload consists of a JSON object:
  
```json
{
    "version" : "2020_10_22",
    "host" : "<broker address>",
    "user-agent": "<user agent string from the client>",
    "action": "kafka-cluster:Connect",
    "x-amz-algorithm" : "<algorithm>",
    "x-amz-credential" : "<clientAWSAccessKeyID>/<date in yyyyMMdd format>/<region>/kafka-cluster/aws4_request",
    "x-amz-date" : "<timestamp in yyyyMMdd'T'HHmmss'Z' format>",
    "x-amz-security-token" : "<clientAWSSessionToken if any>",
    "x-amz-signedheaders" : "host",
    "x-amz-expires" : "<expiration in seconds>",
    "x-amz-signature" : "<AWS SigV4 signature computed by the client>"
}
``` 
## Generating the authentication payload
Please note that all the keys in the authentication payload are in lowercase.

The values of the following keys in the authentication payload are fixed for a client:
*  `"version"` currently always has the value `"2020_10_22"`
* `"user-agent"` is a string passed in by the client library to describe the client. The simplest user agent is `"<name
 of client library>"`. However, more details can be added to the user agent as well `"<name of client library
 /<version of client library>/<os and version>/<version of language>"`

The values of the remaining keys will be generated as part of calculating the signature of the authentication payload
. The signature is calculated by following the rules for generating [presigned urls][PreSigned]. Although, this library
  uses the `AWS4Signer` from the [AWS SDK for Java][AwsSDK] to generate the signature we outline the steps followed
 in calculating the signature and generating the authentication payload from it.
  
 The inputs to this calculation are 
 1. The AWS Credentials that will be used to sign the authentication payload. This has 3 parts: the client `AWSAccessKeyId`,
  the client `AWSSecretyKeyId` and the optional client `SessionToken`.
 1. The hostname of the Kafka broker to which the client wishes to connect.
 1. The AWS region in which the Kafka broker exists.
 1. The timestamp when the connection is being established.
 
The steps in the calculation are:
1. Generate a canonical request based on the inputs.
1. Generate a string to sign based on the canonical request.
1. Calculate the signature based on the string to sign and the inputs.
1. Put it all together in the authentication payload.
 
### Generate a Canonical Request
We start by generating a canonical request with an empty payload based on the inputs.
The canonical request in this case has the following form
```
"GET\n"+
"/\n"+
<CanonicalQueryString>+"\n"+
<CanonicalHeaders>+"\n"+
<SignedHeaders>+"\n"+
<HashedPayload>
```

#### Canonical Query String
`<CanonicalQueryString>` specifies the authentication parameters as URI-encoded query string parameters. We URI-encode
 query parameter names and values  individually. We also sort the parameters in the canonical query string
  alphabetically by key name. The sorting occurs after encoding. 
The `<CanonicalQueryString>` can be calculated by:
```
UriEncode("Action")+"="+UriEncode("kafka-cluster:Connect")+"&"+
UriEncode("X-Amz-Algorithm")+"="+UriEncode("AWS4-HMAC-SHA256") + "&" +
UriEncode("X-Amz-Credential")+"="+UriEncode("<clientAWSAccessKeyID>/<timestamp in yyyyMMdd format>/<AWS region>/kafka-cluster/aws4_request") + "&" +
UriEncode("X-Amz-Date")+"="+UriEncode("<timestamp in yyyyMMdd'T'HHmmss'Z' format>") + "&" +
UriEncode("X-Amz-Expires")+"="+UriEncode("900") + "&" +
UriEncode("X-Amz-Security-Token")+"="+UriEncode("<client Session Token>") + "&" +
UriEncode("X-Amz-SignedHeaders")+"="+UriEncode("host")
```
The exact definition of URIEncode from generating [presigned urls][PreSigned] maybe found [later](#UriEncode).

The query string parameters are in order:
* `"Action"`: Always has the value `"kafka-cluster:Connect"` 
* `"X-Amz-Algorithm"`: Describes the algorithm used to calculate the signature. Currently it is `"AWS4-HMAC-SHA256"`
* `"X-Amz-Credential"`: Contains the access key ID, timestamp in `yyyyMMdd` format, the scope of the credential and
 the constant string `aws4_request`. The scope is defined as the AWS region of the Kafka broker and the name of the 
 service ("kafka-cluster" in this case). For example if the broker is in `us-west-2` region, the scope is `us-west-2/kafka-cluster`.
  This scope will be used again later to calculate the signature in [String To Sign](#string-to-sign) and must 
  match the one  used to calculate the signature. 
* `"X-Amz-Date"`: The date and time format must follow the ISO 8601 standard, and must be formatted with the
 "yyyyMMddTHHmmssZ" format. The date and time must be in UTC.
* `"X-Amz-Expires"` :  Provides the time period, in seconds, for which the generated presigned URL is valid. We
 recommend 900 seconds.
* `"X-Amz-Security-Token"`: The session token if it is specified as part of the AWS Credential. Otherwise this query
 parameter is skipped.
* `"X-Amz-Signedheaders"` is currently always set to `host`.

#### Canonical Header
`<CanonicalHeaders>` is a list of request headers with their values.  Header names must be in lowercase. 
Individual header name and value pairs are separated by the newline character (`"\n"`). In this case there is just one
 header. So `<CanonicalHeaders>` is calculated by:
 ```
"host"+":"+"<broker host name>"+"\n"
```

#### Signed Headers
`<SignedHeaders>` is an alphabetically sorted, semicolon-separated list of lowercase request header names. In this
 case there is just one header. So `<SignedHeaders>` is calculated by:
 ```
"host"
```
#### Hashed Payload
Since the payload is empty the `<HashedPayload>` is calculated as
```
Hex(SHA256Hash(""))
```
where 
* `Hex` is a function to do lowercase base 16 encoding.
* `SHA256Hash` is a Secure Hash Algorithm (SHA) cryptographic hash function.

### String To Sign
From the canonical string, we derive the string that will be used to sign the authentication payload.
The String to Sign is calculated as:
```
"AWS4-HMAC-SHA256" + "\n" +
"<timestamp in yyyyMMdd format>" + "\n" +
<Scope> + "\n" +
Hex(SHA256Hash(<CanonicalRequest>))
```
where 
* `Hex` is a function to do lowercase base 16 encoding.
* `SHA256Hash` is a Secure Hash Algorithm (SHA) cryptographic hash function.

The `<Scope>` is defined as the AWS region of the Kafka broker and the name of the service ("kafka-cluster" in this
 case). For example if the broker is in `us-west-2` region, the scope is `"us-west-2/kafka-cluster"`. It must be the same scope
as was defined for the `"X-Amz-Credential"` query parameter while generating the [canonical query string](#canonical-query-string).

### Calculate Signature
The signature is calculated by:
```
DateKey              = HMAC-SHA256("AWS4"+"<client AWSSecretAccessKey>", "<timestanp in YYYYMMDD>")
DateRegionKey        = HMAC-SHA256(<DateKey>, "<aws-region>")
DateRegionServiceKey = HMAC-SHA256(<DateRegionKey>, "kafka-cluster")
SigningKey           = HMAC-SHA256(<DateRegionServiceKey>, "aws4_request")
Signature            = Hex(HMAC-SHA256(<SigningKey>, <StringToSign>))
```
where
* `Hex` is a function to do lowercase base 16 encoding.
* `HMAC-SHA256` is a function that computes HMAC by using the SHA256 algorithm with the signing key provided. 

The `<Signature>` is the final signature.

## Putting it all together
As mentioned earlier, the authentication payload is a json object with certain keys. All the keys are in lower case.
The following keys in the authentication payload json are constant:
* `"version"` 
* `"user-agent"` 

All the query parameters calculated earlier are added to the authentication payload json. The keys are lower case
 strings and the values are the ones calculated [earlier](#canonical-query-string) but the values are not uri encoded:
* `"action"`
* `"x-amz-algorithm"`
* `"x-amz-credential"`
* `"x-amz-date"`
* `"x-amz-expires"`
* `"x-amz-security-token"`
* `"x-amz-signedheaders"`

The host header is added to the authentication payload json
* `"host"` and its value is set to the hostname of the broker being connected.

The `<Signature>` calculated in the [previous step](#calculate-signature) is added to the authentication payload json as
* `"x-amz-signature"` and its value is set to`<Signature>` 

This finally yields the authentication payload that looks like
```json
{
    "version" : "2020_10_22",
    "host" : "<broker address>",
    "user-agent": "<user agent string from the client>",
    "action": "kafka-cluster:Connect",
    "x-amz-algorithm" : "<algorithm>",
    "x-amz-credential" : "<clientAWSAccessKeyID>/<date in yyyyMMdd format>/<region>/kafka-cluster/aws4_request",
    "x-amz-date" : "<timestamp in yyyyMMdd'T'HHmmss'Z' format>",
    "x-amz-security-token" : "<clientAWSSessionToken if any>",
    "x-amz-signedheaders" : "host",
    "x-amz-expires" : "<expiration in seconds>",
    "x-amz-signature" : "<AWS SigV4 signature computed by the client>"
}
``` 
## Message Exchange with Kafka Broker

This authentication payload is sent as the first message from the client to the Kafka broker. The kafka broker then
 responds with a challenge. We expect a non-empty response from the broker if authentication using AWS IAM succeeded.
The authentication response is a json object that may be logged:

```json
{
  "version" : "2020_10_22",
  "request-id" : "<broker generated request id>"
}
```
The `request-id` which is generated by the broker can be useful for debugging issues with AWS IAM authentication
 between the client and the broker.



 
### UriEncode
Snipped from the detailed rules for generating [presigned urls][PreSigned].
URI encode every byte. UriEncode() must enforce the following rules:

* URI encode every byte except the unreserved characters: 'A'-'Z', 'a'-'z', '0'-'9', '-', '.', '_', and '~'.
* The space character is a reserved character and must be encoded as "%20" (and not as "+").
* Each URI encoded byte is formed by a '%' and the two-digit hexadecimal value of the byte.
* Letters in the hexadecimal value must be uppercase, for example "%1A".
* Encode the forward slash character, '/', everywhere except in the object key name. For example, if the object key
 name is photos/Jan/sample.jpg, the forward slash in the key name is not encoded.

The following is an example UriEncode() function in Java.

```java
public class UriEncode {
public static String UriEncode(CharSequence input, boolean encodeSlash) {
          StringBuilder result = new StringBuilder();
          for (int i = 0; i < input.length(); i++) {
              char ch = input.charAt(i);
              if ((ch >= 'A' && ch <= 'Z') || (ch >= 'a' && ch <= 'z') || (ch >= '0' && ch <= '9') || ch == '_' || ch == '-' || ch == '~' || ch == '.') {
                  result.append(ch);
              } else if (ch == '/') {
                  result.append(encodeSlash ? "%2F" : ch);
              } else {
                  result.append(toHexUTF8(ch));
              }
          }
          return result.toString();
      } 
}
```
   
## Release Notes
### Release 1.1.0
* Add support for credential profiles based on AWS Single Sign-On (SSO).
* Add support for clients using IAM Roles without using credential profiles.
* Bug fix for credential profiles with IAM Roles. 
### Release 1.0.0
* First version of the Amazon MSK Library for IAM Authentication


See [CONTRIBUTING](CONTRIBUTING.md#security-issue-notifications) for more information.


[MSK]: https://aws.amazon.com/msk/
[IAM]: https://aws.amazon.com/iam/
[MSK_IAM]: https://docs.aws.amazon.com/msk/latest/developerguide/iam-access-control.html
[DefaultCreds]: https://docs.aws.amazon.com/sdk-for-java/v1/developer-guide/credentials.html
[CredsFile]: https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html
[PreSigned]: https://docs.aws.amazon.com/AmazonS3/latest/API/sigv4-query-string-auth.html
[AwsSDK]: https://github.com/aws/aws-sdk-java
[RoleProfileCLI]: https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-role.html
