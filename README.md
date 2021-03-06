# Combining static IAM roles with application logic to create application-defined, dynamic, bespoke access to AWS resources at scale

When you develop applications using Amazon Cognito you can grant end users direct access to AWS resources using temporary credentials based on their group membership. This is a great mechanism to reduce server load, simplify applications, maintain security and deliver solutions quickly. However there are scenarios when a customer may not know the permissions a user is entitled to before hand. A SaaS company for example would like to delegate the security control for resources to their customers.

In this blog post we will describe a solution that generates bespoke IAM policies either at login time or at the point of accessing a resource within a serverless web application.  The access control is templated from normal IAM policies, but it ultimately application-defined. This allows far greater flexibility around controlling which users have access to which AWS resources. This example shows how a FinTech start up can delegate the control of trading book updates to the traders using their platform. Trading book updates are published to an IoT topic.

## Deploying the solution

### Pre-requisites

* node

### IoT Thing

Run the commands below from the root folder.

* Set up certificate. Make note of the Certificate ID.

  ```bash
  cd scripts
  npm install
  node configureCert.js
  ```

* Download the root CA certificate

  ```bash
  cd ../iot
  ./download-root-cert.sh
  ```

### Backend

* The backend is deployed using CloudFormation templates. Navigate to the cloudformation folder. Copy env_vars.sh.sample as env_vars.sh. Update the S3 bucket name used to store the CloudFormation templates, the certificate ID from the step above and optionally the region. Run the script deploy.sh.
* Update the trust policy of the Default IoT role to allow Lambda to use it with AWS STS. It is not possible to set this up via Cloudformation because it results in a circular dependency. This is what the policy would look like.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com",
        "AWS": "arn:aws:sts::<ACCOUNT_ID>:assumed-role/serverless-auth-PolicyGeneratorLambdaExecRole-<xxxxx>/<LAMBDA_NAME>"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

### Reference data

Use scripts below to set up Cognito users (traders) and example topic (book) subscriptions. The environment variable AWS_REGION must be set.

* Set up users. This creates user ids for the traders in Amazon Cognito.
  `while read -r LINE; do node createUsers.js $LINE; done < users.txt`

* Set up subscriptions. This maps traders to specific trade books in a DynamoDB table.
  `while read -r LINE; do node subscribeTopics.js $LINE; done < subscriptions.txt`

### Testing the Solution

* Set the AWS_REGION environment variable to the region you are using.

  ```bash
  export AWS_REGION=<your region>
  ```

* Set the IOT_ENDPOINT environment variable to your IoT ATS endpoint necause we are using the root CA issued  by Amazon Trusted Services (ATS) endpoint. You can find the endpoint using the command below.

  ```bash
  aws iot describe-endpoint --endpoint-type iot:Data-ATS
  export IOT_ENDPOINT=<endpointAddress from command above>
  ```

* Subscribe to messages for a specific customer using the command below from the `scripts` folder.
  `node fetchUpdates.js fund1_trader1 Fund1_U5ser_1@ fund1/book1`.
* Now open another termnal and publish messages using the command below from the `scripts` folder.
  `node publishIot.js`.

  ```bash
  Attempting to connect...
  connect
  Publishing test messages to books
  finished writing messages... press ctrl-c to quit...
  ```

* If you go back to the first terminal running the fetch updates script, you should see something like this.
You should see an output like this.

  ```bash
  API Endpoint: <your API gateway endpoint>
  User pool ID: <Cognito iser pool id>
  Client ID: <Cognito user pool client>
  connected to iot
  Subscribed to fund1/book1
  message fund1/book1 {"test_data":"1/1"}
  press ctrl-c to quit...
  ```

* You can refer to the file `subscriptions.txt` to verify that fund1_trader1 has permissions to both fund1/book1 and fund1/book2. You should only see messages published to the book trader is subscribed to. This means if you try `node fetchUpdates.js fund1_trader1 Fund1_U5ser_1@ fund1/book2`, you should still be able to receive the message `{"test_data":"1/2"}`. You can test other combinations of traders and books.
