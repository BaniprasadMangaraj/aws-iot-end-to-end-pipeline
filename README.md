
# 1 — Prep: assumptions & prerequisites

* AWS CLI configured (`aws configure`) for target `REGION` (example: `us-east-1`).
* You have console access to AWS IoT Core, Lambda, API Gateway, DynamoDB, IAM.
* Python 3.9+ and `pip` available for device and lambda packaging.
* Node / npm (optional for frontend).

Set environment variables (replace with your values):

```bash
export AWS_REGION=us-east-1
export ACCOUNT_ID=123456789012
```

---

# 2 — Create IoT Thing & Certificates (Console steps + CLI)

### Console (quick)

1. Open **AWS Console → IoT Core → Manage → Things → Create thing**

   * Choose **Create single thing**
   * Name: `MyDevice001`
   * Create (skip Device Shadow if not needed)
2. After thing created → **Security → Certificates** (or in the thing detail: **Certificates**)

   * Choose **Create certificate**
   * Click **Activate**
   * **Download**:

     * `certificate.pem.crt` (device cert)
     * `private.pem.key`
     * `AmazonRootCA1.pem` (Root CA)
     * optionally `public.pem.key` (not required)
   * Attach a policy (create below) and attach to this certificate.
3. Attach the certificate to your Thing (Manage → Things → select `MyDevice001` → Certificates → Attach).

### Minimal IoT Policy (Console or CLI)

Create a policy with least privilege for testing (restrict in prod). JSON example:

```json
{
  "Version":"2012-10-17",
  "Statement":[{
    "Effect":"Allow",
    "Action":[
      "iot:Connect",
      "iot:Publish",
      "iot:Subscribe",
      "iot:Receive"
    ],
    "Resource":[
      "*" 
    ]
  }]
}
```

Create via CLI:

```bash
cat > iot-policy.json <<'JSON'
{
  "Version":"2012-10-17",
  "Statement":[{
    "Effect":"Allow",
    "Action":[
      "iot:Connect",
      "iot:Publish",
      "iot:Subscribe",
      "iot:Receive"
    ],
    "Resource":["*"]
  }]
}
JSON

aws iot create-policy --policy-name IoTBasicPolicy --policy-document file://iot-policy.json
```

Then attach policy to certificate using the console or:

```bash
CERT_ARN="arn:aws:iot:$AWS_REGION:$ACCOUNT_ID:cert/your-cert-id"
aws iot attach-policy --policy-name IoTBasicPolicy --target "$CERT_ARN"
```

### Get your IoT MQTT endpoint

```bash
aws iot describe-endpoint --endpoint-type iot:Data-ATS --region $AWS_REGION
# response .endpointAddress e.g. a1b2c3d4e5f6g-ats.iot.us-east-1.amazonaws.com
```

Save that as `IOT_ENDPOINT`.

---

# 3 — Configure device to use certs & publish messages

Place the three files on the device: `certificate.pem.crt`, `private.pem.key`, `AmazonRootCA1.pem`.

Device example (Python) using AWS IoT SDK (`awscrt` / `awsiotsdk`):

Install:

```bash
pip install awscrt awsiotsdk
```

`device_publish.py`:

```python
import time, json
from awscrt import io, mqtt
from awsiot import mqtt_connection_builder

IOT_ENDPOINT = "a1b2c3d4e5f6g-ats.iot.us-east-1.amazonaws.com"  # from describe-endpoint
CLIENT_ID = "MyDevice001"
PATH_TO_CERT = "certificate.pem.crt"
PATH_TO_KEY = "private.pem.key"
PATH_TO_ROOT = "AmazonRootCA1.pem"
TOPIC = "sensor/data"

# Setup connection
event_loop_group = io.EventLoopGroup(1)
host_resolver = io.DefaultHostResolver(event_loop_group)
client_bootstrap = io.ClientBootstrap(event_loop_group, host_resolver)

mqtt_connection = mqtt_connection_builder.mtls_from_path(
    endpoint=IOT_ENDPOINT,
    cert_filepath=PATH_TO_CERT,
    pri_key_filepath=PATH_TO_KEY,
    client_bootstrap=client_bootstrap,
    ca_filepath=PATH_TO_ROOT,
    client_id=CLIENT_ID,
    clean_session=False,
    keep_alive_secs=30,
)

print("Connecting...")
connect_future = mqtt_connection.connect()
connect_future.result()
print("Connected!")

try:
    while True:
        payload = {
            "device_id": CLIENT_ID,
            "temperature": round(20 + (5 * (time.time() % 10)), 2),
            "humidity": round(40 + (3 * (time.time() % 7)), 2),
            "ts": int(time.time())
        }
        mqtt_connection.publish(topic=TOPIC, payload=json.dumps(payload), qos=mqtt.QoS.AT_LEAST_ONCE)
        print("Published:", payload)
        time.sleep(5)
finally:
    mqtt_connection.disconnect().result()
```

Run:

```bash
python device_publish.py
```

Verify in Console → IoT Core → Test → MQTT client → Subscribe to `sensor/data` to see messages.

---

# 4 — Create DynamoDB tables

Two tables:

* `IoTDeviceData` (store latest item per device)
* `WebsocketConnections` (store connection ids)

CLI:

```bash
aws dynamodb create-table \
  --table-name IoTDeviceData \
  --attribute-definitions AttributeName=device_id,AttributeType=S \
  --key-schema AttributeName=device_id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST --region $AWS_REGION

aws dynamodb create-table \
  --table-name WebsocketConnections \
  --attribute-definitions AttributeName=connectionId,AttributeType=S \
  --key-schema AttributeName=connectionId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST --region $AWS_REGION
```

---

# 5 — Create ProcessIoTData Lambda (invoked by IoT Rule)

This Lambda receives the IoT message (from the topic rule), stores it to DynamoDB, and broadcasts to connected WebSocket clients using API Gateway Management API.

`lambda/process_iot/app.py`:

```python
import os, json, boto3
from botocore.exceptions import ClientError

dynamodb = boto3.resource('dynamodb')
TABLE_NAME = os.environ.get('DDB_TABLE', 'IoTDeviceData')
CONN_TABLE = os.environ.get('CONNECTIONS_TABLE', 'WebsocketConnections')
WEBSOCKET_API_ID = os.environ.get('WEBSOCKET_API_ID')  # fill later
WEBSOCKET_STAGE = os.environ.get('WEBSOCKET_STAGE', 'prod')
REGION = os.environ.get('AWS_REGION', 'us-east-1')

def post_to_all_connections(payload):
    con_table = dynamodb.Table(CONN_TABLE)
    api_endpoint = f"https://{WEBSOCKET_API_ID}.execute-api.{REGION}.amazonaws.com/{WEBSOCKET_STAGE}"
    apigw = boto3.client('apigatewaymanagementapi', endpoint_url=api_endpoint)
    resp = con_table.scan(ProjectionExpression='connectionId')
    for item in resp.get('Items', []):
        conn_id = item['connectionId']
        try:
            apigw.post_to_connection(Data=json.dumps(payload).encode('utf-8'), ConnectionId=conn_id)
        except ClientError as e:
            # remove stale connections
            if e.response['Error']['Code'] in ('GoneException','410'):
                con_table.delete_item(Key={'connectionId': conn_id})
            else:
                print("Post error", e)

def lambda_handler(event, context):
    # IoT Rule may forward message object; sometimes the message will be in event['payload'] or event itself
    print("Raw event:", event)
    payload = event
    # If event looks like { "message": "<json-string>" } parse if needed
    if isinstance(event, dict) and 'payload' in event:
        try:
            payload = json.loads(event['payload'])
        except:
            payload = event['payload']
    # store latest
    table = dynamodb.Table(TABLE_NAME)
    device_id = payload.get('device_id', 'unknown')
    item = {
        'device_id': device_id,
        'ts': int(payload.get('ts', context.aws_request_id[:10])),
        'payload': payload
    }
    table.put_item(Item=item)
    # broadcast
    try:
        post_to_all_connections(payload)
    except Exception as e:
        print("Broadcast failed:", e)

    return {"statusCode":200, "body": json.dumps({'message':'processed'})}
```

`requirements.txt`:

```
boto3
```

**Lambda env variables** (set when creating function):

* `DDB_TABLE` = `IoTDeviceData`
* `CONNECTIONS_TABLE` = `WebsocketConnections`
* `WEBSOCKET_API_ID` = (fill after WebSocket API created)
* `WEBSOCKET_STAGE` = `prod`
* `AWS_REGION` = your region

**Role permissions** (Lambda execution role must include):

* `dynamodb:PutItem`, `dynamodb:Scan`, `dynamodb:DeleteItem` on both tables
* `execute-api:ManageConnections` on ARN: `arn:aws:execute-api:${REGION}:${ACCOUNT_ID}:${WEBSOCKET_API_ID}/*/@connections/*` (or wildcard when creating)

Example IAM policy snippet for the Lambda role (attach as inline policy):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect":"Allow",
      "Action":[
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:Scan",
        "dynamodb:DeleteItem"
      ],
      "Resource":[
        "arn:aws:dynamodb:REGION:ACCOUNT_ID:table/IoTDeviceData",
        "arn:aws:dynamodb:REGION:ACCOUNT_ID:table/WebsocketConnections"
      ]
    },
    {
      "Effect":"Allow",
      "Action":["execute-api:ManageConnections"],
      "Resource":["arn:aws:execute-api:REGION:ACCOUNT_ID:*/*/@connections/*"]
    }
  ]
}
```

(Replace REGION, ACCOUNT_ID and optionally WEBSOCKET_API_ID for tighter scope.)

---

# 6 — Create IoT Topic Rule to invoke Lambda

Create rule SQL: `SELECT * FROM 'sensor/data'`

CLI create rule payload file `iot-rule.json`:

```json
{
  "sql": "SELECT * FROM 'sensor/data'",
  "actions": [
    {
      "lambda": {
        "functionArn": "arn:aws:lambda:REGION:ACCOUNT_ID:function:ProcessIoTData"
      }
    }
  ],
  "ruleDisabled": false
}
```

Create:

```bash
aws iot create-topic-rule --rule-name SendToLambdaRule --topic-rule-payload file://iot-rule.json --region $AWS_REGION
```

Grant IoT permission to invoke Lambda:

```bash
aws lambda add-permission \
  --function-name ProcessIoTData \
  --statement-id iot-invoke \
  --action lambda:InvokeFunction \
  --principal iot.amazonaws.com \
  --source-arn "arn:aws:iot:$AWS_REGION:$ACCOUNT_ID:rule/SendToLambdaRule"
```

---

# 7 — Create API Gateway WebSocket API and connection handlers

### A) Create WebSocket API (Console recommended)

1. Console → **API Gateway → WebSocket APIs → Create**

   * Name: `IoTWebsocketAPI`
   * Route selection expression: `$request.body.action` (or use default)
2. Add routes:

   * `$connect` → Integration: Lambda `OnConnect`
   * `$disconnect` → Integration: Lambda `OnDisconnect`
   * `default` → optional
3. Deploy the API (create stage `prod`). Note the **API ID** (e.g., `abcd1234`) — this is your `WEBSOCKET_API_ID`.

### B) OnConnect Lambda (store connectionId)

`lambda/on_connect/app.py`:

```python
import os, boto3
dynamodb = boto3.resource('dynamodb')
CONN_TABLE = os.environ.get('CONNECTIONS_TABLE', 'WebsocketConnections')

def lambda_handler(event, context):
    connectionId = event['requestContext']['connectionId']
    table = dynamodb.Table(CONN_TABLE)
    table.put_item(Item={'connectionId': connectionId})
    return {'statusCode': 200}
```

### C) OnDisconnect Lambda (delete connectionId)

`lambda/on_disconnect/app.py`:

```python
import os, boto3
dynamodb = boto3.resource('dynamodb')
CONN_TABLE = os.environ.get('CONNECTIONS_TABLE', 'WebsocketConnections')

def lambda_handler(event, context):
    connectionId = event['requestContext']['connectionId']
    table = dynamodb.Table(CONN_TABLE)
    table.delete_item(Key={'connectionId': connectionId})
    return {'statusCode': 200}
```

**Permissions for these Lambdas**: allow `dynamodb:PutItem` or `dynamodb:DeleteItem` on `WebsocketConnections`.

After creating the WebSocket API, note the **URL**:

```
wss://{WEBSOCKET_API_ID}.execute-api.{REGION}.amazonaws.com/prod
```

Update `ProcessIoTData` lambda env var `WEBSOCKET_API_ID` to the ID.

Also ensure the `ProcessIoTData` role has `execute-api:ManageConnections`.

---

# 8 — Create HTTP API to fetch data (optional)

To let clients request historical/latest data, create a simple HTTP API in API Gateway that integrates with a `FetchIoTData` Lambda.

`lambda/fetch_iot/app.py`:

```python
import boto3, os, json
dynamodb = boto3.resource('dynamodb')
TABLE_NAME = os.environ.get('DDB_TABLE', 'IoTDeviceData')

def lambda_handler(event, context):
    params = event.get('queryStringParameters') or {}
    device_id = params.get('device_id')
    table = dynamodb.Table(TABLE_NAME)
    if device_id:
        resp = table.get_item(Key={'device_id': device_id})
        item = resp.get('Item')
        body = item or {}
    else:
        # Not ideal for production: scanning
        resp = table.scan(Limit=50)
        body = resp.get('Items', [])
    return {
        'statusCode': 200,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps(body)
    }
```

Deploy HTTP API via console or SAM linking to this Lambda.

---

# 9 — Client: WebSocket browser code (connect to WebSocket and receive pushes)

Use the WebSocket URL from step 7: `wss://{WEBSOCKET_API_ID}.execute-api.{REGION}.amazonaws.com/prod`

Simple JS snippet (works in browser console or in React):

```javascript
const wsUrl = "wss://abcd1234.execute-api.us-east-1.amazonaws.com/prod"; // replace
const ws = new WebSocket(wsUrl);

ws.onopen = () => {
  console.log("WebSocket connected");
  // Optionally send an action if your API expects messages
  // ws.send(JSON.stringify({ action: 'subscribe', device_id: 'MyDevice001'}));
};

ws.onmessage = (event) => {
  try {
    const data = JSON.parse(event.data);
    console.log("Received push:", data);
    // update UI
  } catch(e) {
    console.log("Message:", event.data);
  }
};

ws.onclose = () => console.log("WebSocket closed");
ws.onerror = (err) => console.log("WebSocket error", err);
```

If you deployed the HTTP API `FetchIoTData`, you can poll:

```javascript
fetch('https://abcd1234.execute-api.us-east-1.amazonaws.com/prod/iotdata?device_id=MyDevice001')
  .then(r=>r.json()).then(console.log)
```

---

# 10 — Testing flow summary (end-to-end)

1. Start device script → publishes to `sensor/data`.
2. IoT Core receives and triggers **IoT Rule** (`SendToLambdaRule`) that invokes `ProcessIoTData` Lambda.
3. `ProcessIoTData` stores message into `IoTDeviceData` and **posts to all connectionIds** using `apigatewaymanagementapi` to the WebSocket API.
4. Clients connected to WebSocket (`OnConnect` stored their `connectionId`) receive the pushed message instantly.
5. Clients can also query `FetchIoTData` HTTP API to retrieve stored/latest data.

---

# 11 — Important CLI / config commands checklist (quick)

```bash
# 1. Get IoT endpoint:
aws iot describe-endpoint --endpoint-type iot:Data-ATS --region $AWS_REGION

# 2. Create IoT Policy (if not created):
aws iot create-policy --policy-name IoTBasicPolicy --policy-document file://iot-policy.json

# 3. Create DynamoDB tables (run from step 4)

# 4. Create Lambdas (zip packaging and upload or use SAM)

# 5. Create WebSocket API and deploy (Console easier). Note WEBSOCKET_API_ID and stage.

# 6. Set env var WEBSOCKET_API_ID on ProcessIoTData lambda

# 7. Create IoT Topic Rule (see step 6)

# 8. Add permission: allow IoT to invoke Lambda (see step 6 add-permission)

# 9. Start device script (device_publish.py)
```

---

# 12 — Troubleshooting tips

* If Lambda logs show `ClientError` when posting to connection, check `WEBSOCKET_API_ID`, `WEBSOCKET_STAGE` and that the `apigatewaymanagementapi` endpoint URL matches `https://{api-id}.execute-api.{region}.amazonaws.com/{stage}`.
* If clients don’t receive messages: ensure `WebsocketConnections` table contains connection IDs (check OnConnect logs).
* If IoT Rule fails to invoke Lambda: check `aws lambda add-permission` was executed, and IoT Rule is enabled.
* For high-throughput, use SQS/Kinesis between IoT Rule and Lambda to buffer.
* Use CloudWatch Logs for every Lambda to inspect payload and errors.

---

# 13 — Security & production notes

* Do NOT use wildcard `"Resource": "*"` in production policies — restrict to ARNs.
* Rotate certificates if devices get compromised.
* Use Cognito or IAM-based auth for browser/mobile clients rather than unsecured WebSocket endpoints.
* Use throttling/monitoring to avoid Lambda spikes.


