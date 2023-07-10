# Using AWS Lambda with Amazon DynamoDB streams

1. Create an execution role that give the Lambda function the permission to access AWS resources.
   - Trusted Entity: Lambda
   - Permissions: AWSLambdaDynamoDBExecutionRole
   `AWSLambdaDynamoDBExecutionRole` has the permissions that the function needs to read items from DynamoDB and write logs to CloudWatch Logs
2. Create a function and save it as `index.mjs`
3. Create a deployment package.

   ```bash
   zip function.zip index.mjs
   ```

4. Create a Lambda function with the create-function command.

    ```bash
    aws lambda create-function --function-name ProcessDynamoDBRecords \
    --zip-file fileb://function.zip --handler index.handler --runtime nodejs18.x \
    --role arn:aws:iam::504742720065:role/lambda-dynamodb-role
    ```

    > In AWS CLI (Command Line Interface), the "fileb://" prefix is used to specify that the file is in binary format. This is typically used when passing binary data, such as binary content or a binary file, as a parameter to a command.

    > In the case of the `create-function` command, the `--zip-file` option expects the binary content of the ZIP file containing the function code. By using the "fileb://" prefix, you are indicating that the ZIP file is being provided as binary data.

5. Test the Lambda function by invoking it manually using the `invoke` AWS Lambda CLI command and the DynamoDB event data in the `input.txt` file.

    Running the `invoke` command

    ```bash
    aws lambda invoke --function-name ProcessDynamoDBRecords \
    --cli-binary-format raw-in-base64-out \
    --payload file://input.txt outputfile.txt
    ```

    > The function returns the string message in the response body.

    Verify the output in the `outfile.txt`.

6. Create a DynamoDB table with a stream enabled

    DynamoDB table name: `lambda-dynamodb-stream`
    Primary key: `id` (String)
    DynamoDB Stream ARN: `arn:aws:dynamodb:us-east-1:504742720065:table/lambda-dynamodb-stream/stream/2023-07-10T06:52:46.139`

7. Add an event source in AWS Lambda

   Run the following AWS CLI `create-event-source-mapping` command. After the command runs, note down the `UUID`.

   ```bash
   aws lambda create-event-source-mapping --function-name ProcessDynamoDBRecords \
    --batch-size 100 --starting-position LATEST --event-source DynamoDB-stream-arn
   ```

   "UUID": "4eed71c2-fdbc-45e0-8a0b-f85d72145c28"

    This creates a mapping between the specified DynamoDB stream and the Lambda function. We can associate multiple DynamoDB streams with a single Lambda function or associate a single DynamoDB stream with multiple Lambda functions.

    List of event source mappings

    ```bash
    aws lambda list-event-source-mappings
    ```

    To narrow down the results while having a lot of event source mappings use the given `function-name` parameter.

    ```bash
    aws lambda list-event-source-mappings --function-name ProcessDynamoDBRecords
    ```

8. To test the setup

    As we perform the table updates, DynamoDB writes event records to the stream. As AWS Lambda polls the stream, it detect new records in the stream and invokes the lambda function on our behalf by passing events to the function.

    - In the DynamoDB console, add, update, and delete items to the table. DynamoDB writes records of these actions to the stream.

    - AWS Lambda polls the stream and when it detects updates to the stream, it invokes your Lambda function by passing in the event data it finds in the stream.

    - Your function runs and creates logs in Amazon CloudWatch. You can verify the logs reported in the Amazon CloudWatch console.

Reference Article - <https://docs.aws.amazon.com/lambda/latest/dg/with-ddb-example.html>
