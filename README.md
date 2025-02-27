Using the ARN of the **SSM Parameter** works pretty well even in **Cross-Accounts** where the SSM Parameter was shared via AWS RAM instead of using the name as this will throw an error. 

`aws ssm get-parameter --name "myparam"`

An error occurred (ValidationException) when calling the GetParameter operation: Parameter name: can't be prefixed with "ssm" (case-insensitive). If formed as a path, it can consist of sub-paths divided by slash symbol; each sub-path can be formed as a mix of letters, numbers and the following 3 symbols .-_

Instead use:

   ` $ aws ssm get-parameter --name arn:aws:service:us-east-1:0000000000:parameter/myparam`

Output:
```
{
        "Parameter": {
            "Name": "/myparam",
            "Type": "String",
            "Value": "ami-06ad392018375648b",
            "Version": 3,
            "LastModifiedDate": "2025-02-26T02:29:26.198000+00:00",
            "ARN": "arn:aws:service:us-east-1:0000000000:parameter/myparam",
            "DataType": "aws:ec2:image"
        }
}
```


**NOTE:** If the parameter has a forward slash already do not add it

 `$ aws ssm get-parameter --name arn:aws:service:us-east-1:0000000000:parameter${myparam}`

If you are trying to access it in the **Cross-Account through the CLI or Lambda Function**:

   ```
    import boto3
    import os
    
    ssm = boto3.client('ssm')
    
    def lambda_handler(event, context):
        parameter_name = os.environ['PARAMETER_NAME']  # Read from environment variable
        
        try:
            response = ssm.get_parameter(
                Name=parameter_name,
                WithDecryption=True  # If it's a SecureString parameter
            )
            return {
                'statusCode': 200,
                'parameter_value': response['Parameter']['Value']
            }
        except Exception as e:
            return {
                'statusCode': 400,
                'error': str(e)
            }
```

Also make sure to have these in the SOURCE ACCOUNT(in RAM)

```
ssm:GetParameter
ssm:GetParameters
ssm:DescribeParameters
```

and the DESTINATION ACCOUNT(in the Lambda Role) create this inline policy

```
{
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "ssm:GetParameter",
                    "ssm:GetParameters",
                    "ssm:DescribeParameters"
                ],
                "Resource": "arn:aws:ssm:REGION:ACCOUNT_A_ID:parameter/myparam"
            }
        ]
}
```
Or Attach the **AmazonSSMFullAccess** to test, then restrict it later for security(So that the Role wont be too open to what is not needed by the system)


