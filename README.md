# AWS-Lambda-Power-Tuning-with-Cognito
AWS Lambda Power Tuning with Cognito Integration A secure, serverless architecture demonstrating how to optimize AWS Lambda performance using the AWS Lambda Power Tuning state machine, while securing access with Amazon Cognito authentication and API Gateway.

## 🏗️ Architecture Overview

🔒 **Authentication & Authorization Flow**

1) User Login (via Cognito):
The user authenticates using Amazon Cognito User Pool through a login page (hosted UI or custom frontend).

2) Authorization Code Retrieval:
Once authenticated, Cognito returns an authorization code to the client application (served via CloudFront).

3) Token Exchange:
The client exchanges the authorization code for access and ID tokens, which are used to authenticate API requests.

4) Secure API Access:
The client sends a GET request to the Amazon API Gateway, including the Cognito-issued JWT token in the Authorization header.

5) Invoke Lambda:
The API Gateway validates the token with Cognito and invokes the backend AWS Lambda function.

6) Data Query:
The Lambda function securely queries Amazon DynamoDB to fetch or process application data.

7) Response Delivery:
The Lambda response is sent back through the API Gateway to the authenticated client.

<img width="1024" height="1536" alt="Diagram" src="https://github.com/user-attachments/assets/095bc3f3-5b40-4df4-8e21-b86696337c9c" />

## 🔧 AWS Services Used
| Service | Purpose |
|----------|----------|
| **AWS Lambda** | Runs test invocations with different memory settings |
| **AWS Step Functions** | Coordinates Lambda runs for power tuning |
| **Amazon Cognito** | Provides secure user authentication |
| **Amazon API Gateway** | Acts as a secured entry point to invoke the state machine |


---

## ⚙️ Setup Instructions

🚀 1. **Create a DynamoDB Table**

Open the AWS Management Console → DynamoDB.

Click Create table.

Set:

Table name: UserData

Primary key: userId (String)

Keep defaults and create the table.

⚙️ 2. **Create a Lambda Function**

Go to AWS Lambda → Create function.

Choose Author from scratch.

Name: CognitoUserHandler

Runtime: Python 3.12 (or Node.js 18.x if you prefer)

Add permissions:

Attach policy: AmazonDynamoDBFullAccess.

(Optional) Create a dedicated IAM Role for security best practice.

Example Lambda Code (Python)
import json
import boto3
from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('UserData')

def lambda_handler(event, context):
    try:
        user_id = event['requestContext']['authorizer']['claims']['sub']
        action = event.get('queryStringParameters', {}).get('action', 'get')

        if action == 'get':
            response = table.get_item(Key={'userId': user_id})
            return {'statusCode': 200, 'body': json.dumps(response.get('Item', {}))}
        elif action == 'add':
            data = json.loads(event['body'])
            table.put_item(Item={'userId': user_id, **data})
            return {'statusCode': 201, 'body': json.dumps({'message': 'Item added'})}
        else:
            return {'statusCode': 400, 'body': 'Invalid action'}

    except Exception as e:
        return {'statusCode': 500, 'body': json.dumps({'error': str(e)})}

🔐 3. **Create a Cognito User Pool**

Go to Amazon Cognito → Manage User Pools → Create user pool.

Choose:

Name: ServerlessAppUserPool

Sign-in option: Email.

Under App Clients, create a client app (uncheck “Generate client secret”).

Under App Integration, note:

App client ID

Cognito domain name (e.g., yourapp.auth.us-east-1.amazoncognito.com)

![User pool](https://github.com/user-attachments/assets/4eefd053-1be9-4fb3-aee4-2f633b714a40)

<img width="1188" height="303" alt="image" src="https://github.com/user-attachments/assets/15db52f8-af15-4684-967b-f8c5a5eabab5" />

<img width="1522" height="467" alt="image" src="https://github.com/user-attachments/assets/292be539-0593-41b5-ab89-e426f95f7c18" />


🌐 4. **Create an API Gateway (HTTP or REST API)**

Go to API Gateway → Create API → Choose HTTP API.

Create a new route:

Method: POST

Path: /userdata

Attach your Lambda as the Integration Target.

🔒 5. **Add Cognito Authorizer to API Gateway**

In API Gateway, go to Authorizers → Create.

Choose Cognito and select your User Pool.

<img width="1522" height="508" alt="image" src="https://github.com/user-attachments/assets/582153e0-4796-4f53-8839-dccbc9c05eae" />


In your route settings:

Under Authorization, select your new Cognito Authorizer.

Now, only authenticated users can call your API.

<img width="1522" height="665" alt="image" src="https://github.com/user-attachments/assets/6080258a-5758-4a32-b526-919c0b58bcf7" />


🧩 6. **Deploy and Test Using Postman**

**Step 1** — Get Cognito Tokens

Use Cognito’s Hosted UI or directly call the /oauth2/token endpoint:

POST

https://<your_cognito_domain>.auth.<region>.amazoncognito.com/oauth2/token


Headers:

Content-Type: application/x-www-form-urlencoded


Body (x-www-form-urlencoded):

grant_type=password
client_id=<your_app_client_id>
username=<user_email>
password=<user_password>


You’ll get a response with:

{
  "access_token": "<JWT_ACCESS_TOKEN>",
  "id_token": "<JWT_ID_TOKEN>",
  "refresh_token": "<REFRESH_TOKEN>"
}

**Step 2** — Call API Gateway

POST

https://<api_gateway_id>.execute-api.<region>.amazonaws.com/userdata


Headers:

Authorization: Bearer <JWT_ACCESS_TOKEN>
Content-Type: application/json


Body (JSON):

{
  "name": "John Doe",
  "email": "john@example.com"
}


✅ Response:

{
  "message": "Item added"
}

🧱 7. **Architecture Summary**

Flow:

User → Cognito (Authenticate) → API Gateway (Authorize) → Lambda (Process) → DynamoDB (Store)


Cognito issues JWT tokens.

API Gateway validates token via Cognito Authorizer.

Lambda executes secure business logic.

DynamoDB stores user data.

⚡8. **Validation & Performance Analysis — Final Statement** 

After executing the AWS Lambda Power Tuning state machine across multiple memory configurations, results indicated the optimal balance between execution time, cost efficiency, and throughput.

Below are my findings : 

⚙️ 1️⃣ AWS Lambda Power Tuning
Using the AWS Lambda Power Tuning Step Function, I benchmarked execution times and costs across memory configurations from 128 MB to 1024 MB.

📊 Key Results:
a) Best performance: 1024 MB (≈400 ms average time)
b) Best cost: 1024 MB : surprisingly, higher memory was cheaper overall due to faster execution.
c) Worst performance: 128 MB (~1600 ms)
d) Sweet spot: Increasing memory gave more CPU, improving both speed and efficiency.

![Power Tuning](https://github.com/user-attachments/assets/0f43f7a5-1f45-4ba0-935b-a8fd12cf8f16)


💬 Insight: Lambda pricing isn’t linear — allocating more memory can reduce total cost for short-running compute-heavy functions.

🧪 2️⃣ Load Testing with Postman
Next, I simulated user traffic through Postman’s load testing feature, hitting the same API Gateway endpoint.

📈 Results from 556 requests:
⚡ Avg response time: 390 ms
💪 P90 latency: 428 ms
💰 P95 latency: 492 ms
🧊 P99 latency spike: 2.8 s (cold starts)
✅ Error rate: 0%

![Postman tuning](https://github.com/user-attachments/assets/d00dfd3e-3d6e-4b49-bb49-0677af78076f)


Observation: The API handled consistent load with great stability and no failed requests. Occasional P99 spikes highlight the typical cold-start behavior of Lambda — which can be further reduced with provisioned concurrency or warming strategies.

🔍 What I Learned:

a) Tuning Lambda memory has a direct and measurable impact on both performance and cost.

b) Combining AWS Lambda Power Tuning + Postman Load Testing gives a full performance picture — from backend execution to real-world API latency.

c) Serverless testing and observability are as important as deployment itself.
