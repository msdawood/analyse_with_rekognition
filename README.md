# Image detection with Amazon Rekognition and AWS Lambda
This project demonstrates how to build a serverless application using **AWS Lambda**, **Amazon Rekognition**, and **Amazon S3** to automatically detect the presence of a "car" in an image uploaded to an S3 bucket. If a car is detected with at least **70% confidence**, the result is saved back to the S3 bucket in a separate folder in JSON format.

<img width="407" alt="Screenshot 2025-04-27 at 21 49 09" src="https://github.com/user-attachments/assets/e3128bc9-d705-47db-b3d6-66855634b38b" />

## Features
- Automatically triggers a Lambda function on new image upload to S3.
- Uses Amazon Rekognition to detect labels in the uploaded image.
- Searches for the label `"car"` with a confidence score of at least 70.
- Stores the detection result (success/failure + labels) back to the S3 bucket.
- Fully serverless and scalable.

## Tech Stack
- **AWS Lambda**
- **Amazon S3**
- **Amazon Rekognition**
- **AWS IAM**
- **Python 3.13**
- **Boto3 (AWS SDK for Python)**

## Setup

### Create Amazon S3 bucket
1. Create a S3 bucket with unique name (Ex: sda-image-rekognition-verification) in the AWS S3 Console
![image](https://github.com/user-attachments/assets/5d4f06da-d8d9-405f-8663-e70f0a2303bd)

2. Create two folders (input and output)
![image](https://github.com/user-attachments/assets/77c88ed4-f646-4d62-b978-1ee2d131a7dd)

### Create Lambda function
1. Click "Create function" in the AWS Lambda Console
   ![image](https://github.com/user-attachments/assets/e8c80bc5-6091-4c19-8613-4b04edfd2479)
2. Select "Author from scratch". Use name **RekogniseObject**. Select **Python 3.13** as Runtime.
3. Click "Create function"
![image](https://github.com/user-attachments/assets/897e7646-91ab-47c9-b255-5f5b112fe490)
 
4. Replace the boilerplaet code with the following code snippet and click "Deploy" button. You will see a success message at the top.
   ```
    import json
    import boto3

    label_to_detect = 'car'
    s3 = boto3.client('s3')

    def lambda_handler(event, context):

    # Get the object from the event
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']

    output_key = 'output/response.json'
    response = {'Status': 'Not Found', 'body': []}

    rekognition_client = boto3.client('rekognition')

    try:
        # Invoke rekognition detect_labels API to analyse the image stored in S3
        response_rekognition = rekognition_client.detect_labels(
            Image={
                'S3Object': {
                    'Bucket': bucket,
                    'Name': key
                }
            },
            MinConfidence=70
        )

        detected_labels = []

        if response_rekognition['Labels']:
            for label in response_rekognition['Labels']:
                detected_labels.append(label['Name'].lower())
            print(detected_labels)

            if label_to_detect in detected_labels:
                response['Status'] = f"Success! {label_to_detect} found"   
            else:
                response['Status'] = f"Failed! {label_to_detect} Not found"

            response['body'].append(response_rekognition['Labels'])
    except Exception as error:
        print(error)

    s3.put_object(
      Bucket=bucket,
      Key=output_key,
      Body=json.dumps(response, indent=4)
    )

    return response
   ```
5. Go back to the Lambda function and select Add trigger
   ![image](https://github.com/user-attachments/assets/75eed560-87ac-4ae1-a4b0-c1b609f755d0)

   Select S3 from the 'Select a source' drop down
   ![image](https://github.com/user-attachments/assets/4b8a0cb0-5ce1-48ca-887f-e65b22baf611)

   Select the S3 bucket created in the previous step. Provide `input/` in the prefix and `.jpg` in the suffix input boxes. Select "I acknowledge..." checkbox. Then select "Add" button to add the trigger.
   ![image](https://github.com/user-attachments/assets/88ff59e4-bf01-423f-a5a3-6664a70fb928)

   Once the trigger is added, it will be appeared in the "Configuration" tab under Triggers section.
   ![image](https://github.com/user-attachments/assets/61c19dd9-86ae-40c8-9964-de2332244ce0)

### Create IAM permissions for Lambda function
When the Lambda function is created a default execution role would have been attached to it with permissions for CloudWatch Logs. Edit the role to add permissions to access Amazon Rekognition with the input image to detect the labels as well as to Amazon S3 bucket to store the result json back in the S3 bucket.
The final policy document should look like below

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "logs:CreateLogGroup",
            "Resource": "arn:aws:logs:us-east-1:863518449712:*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": [
                "arn:aws:logs:us-east-1:863518449712:log-group:/aws/lambda/RekogniseObject:*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": [
                "arn:aws:s3:::sda-image-rekognition-verification/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": "rekognition:DetectLabels",
            "Resource": "*"
        }
    ]
}
```

### Test the application
1) Test with Car image
Upload an image containing car
![image](https://github.com/user-attachments/assets/a9024dfc-9b9b-4f29-9064-fa60819cdb4d)

Upload into the S3 bucket input folder.
![image](https://github.com/user-attachments/assets/738a8ff1-8eb4-49fb-b76b-54ef0b51a69e)

Once updloaded, S3 will trigger the lambda function which will read the image and invokes Amazon Rekognition detect_labels API. Based on the response, a JSON file will be added to output filder.
![image](https://github.com/user-attachments/assets/b8931f68-4d60-445f-8f2c-d28532ea9a5a)

2) Test with Tram image
Upload an image containing tram
![image](https://github.com/user-attachments/assets/c7c41050-c898-40af-9404-79ac0bae18c6)

Upload into the S3 bucket input folder.
![image](https://github.com/user-attachments/assets/738a8ff1-8eb4-49fb-b76b-54ef0b51a69e)

Once updloaded, S3 will trigger the lambda function which will read the image and invokes Amazon Rekognition detect_labels API. Based on the response, a JSON file will be added to output filder.
![image](https://github.com/user-attachments/assets/16244732-5f08-4acf-b76c-7c25665bac27)

## Cleanup
We can clean the resources we had created for this lab.
1) Go to the Lambda console. Select the RekogniseObject and select "Delete" from "Actions" menu.
2) Go to the S3 console. Select the S3 bucket sda-image-rekognition-verification and "Delete" from "Actions" menu. Please note the files and folders need to be deleted before deleting the S3 bucket.

