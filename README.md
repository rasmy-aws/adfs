# A simple SAML Identity Provider (IdP) provisioner

Based on [cfn-identity-provider](https://github.com/cevoaustralia/cfn-identity-provider)

This CloudFormation template creates a SAML identity provider in Amazon Web Services' (AWS) Identity and Access
Management (IAM) configuration.

In order to use it, you'll need:

* an AWS account
* rights within that AWS account to create, update, and delete:
  * CloudFormation stacks
  * IAM Roles and Policies
  * Lambda functions
  * Identity Providers
* a SAML Identity Provider (IdP)
* the Federation metadata (an XML document) from the Identity Provider (how to get this differs for every IdP)

## Preparing metadata

1. Download metadata from your IdP
2. Make it all be on one line:
   `tr -d '\n' < FederationMetadata.xml > FederationMetadataSingleLine.xml`
3. Copy the content of the XML into a YAML file with the following format:

    ```yaml
    Metadata: '<?xml version="1.0" encoding="UTF-8"?><md:EntityDescrip...'
    ```

   Important to remember to put the XML payload in single quotes `'`.
   Then save the file as `FederationMetadata.yaml`.
4. Upload the YAML file to an S3-bucket and specify the bucket path in the `params.json`. Make sure to update the bucket policy to allow accounts in your organization to read the file.

    ```json
    ...
    {
      "Version": "2012-10-17",
      "Statement": [
        {
            "Sid": "AllowGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::federation-metadata-bucket/*",
            "Condition": {
              "StringEquals": {
                "aws:PrincipalOrgID": [ "o-yyyyyyyyyy" ]
              }
            }
          }
        ]
      }
    ...
    ```

## Configuration

You can set the name of your identity provider via the `SamlProviderName` parameter to the stack; this can be
configured in the `params.json`. It defaults to `adf-saml` which is a protected prefix in the iam-baseline.

You also need to update the trust policy of the IAM roles in the iam-baseline to trust the SAML provider in each account. The `ProviderCreator` custom resource returns the ARN of the SAML provider as its physical resource ID.

You can simply `Ref` the custom resource to use the SAML provider ARN.

For example the PowerUserRole would look as follows:

```yaml
...

  PowerUserRole:
    Condition: PowerUserRequired
    Type: "AWS::IAM::Role"
    Properties:
      RoleName: adf-poweruser-role
      AssumeRolePolicyDocument:
        Version: "2012-10-17"
        Statement:
          - Effect: Allow
            Principal:
                Federated: !Join ['/', [ 'arn:aws:iam:${AWS::AccountId}:saml-provider', !Ref SAMLProvider]]
            Action: sts:AssumeRoleWithSAML
            Condition:
              StringEquals:
                "SAML:aud": "https://signin.aws.amazon.com/saml"
      ManagedPolicyArns:
        - !Ref PowerUserPolicy

...
```

## Updating the stack

The stack can be updated, though the only changes you can make to the Identity Provider is to change the SAML
Metadata document, in case you need to update the trust relationship.