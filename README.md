# End-to-End Testing Workflow

## Testing Phases

### Phase 1: Unit Testing (Lambda Function Alone)

**Duration:** 30-60 minutes

#### Test 1.1: Empty Queue Handling
```bash
# 1. Purge SQS queue
aws sqs purge-queue --queue-url YOUR_QUEUE_URL

# 2. Invoke Lambda manually
aws lambda invoke \
  --function-name web_engage_service_lambda \
  --payload '{"batch_size": 1000, "time_limit_seconds": 30, "chunk_size": 100, "delay_between_chunks": 0.5, "max_concurrent_uploads": 2, "sqs_queue_url": "YOUR_QUEUE_URL", "webengage_api_key": "test-key", "webengage_api_url": "https://api.webengage.com/v1/users/bulk"}' \
  response.json

# 3. Check response
cat response.json
```

**Expected:** 
```json
{
  "statusCode": 200,
  "body": "{\"message\": \"No records to process\", \"records_processed\": 0}"
}
```

**✅ Pass Criteria:** Lambda handles empty queue gracefully

---

#### Test 1.2: Small Batch (100 records)
```bash
# 1. Generate 100 test records
python sqs_test_data_generator.py  # NUM_TEST_RECORDS = 100

# 2. Invoke Lambda
aws lambda invoke \
  --function-name web_engage_service_lambda \
  --payload file://test_events/basic_small_batch.json \
  response.json

# 3. Verify CloudWatch Logs
aws logs tail /aws/lambda/web_engage_service_lambda --follow
```

**Expected CloudWatch Logs:**
```
Starting WebEngage chunked upload job
Collected 100 records in 5.23s
Successfully transformed 100/100 records in 0.05s
Splitting 100 records into 10 chunks of 10 records each
✓ Successfully uploaded chunk 1/10
✓ Successfully uploaded chunk 2/10
...
Upload Summary: 10/10 chunks succeeded
Estimated records uploaded: 100/100
Successfully deleted 100 messages from SQS
```

**✅ Pass Criteria:** 
- All 100 records processed
- All chunks uploaded successfully
- Messages deleted from SQS

---

#### Test 1.3: Medium Batch (1,000 records)
```bash
# 1. Generate 1,000 records
# Modify: NUM_TEST_RECORDS = 1000
python sqs_test_data_generator.py

# 2. Use production-like settings
aws lambda invoke \
  --function-name web_engage_service_lambda \
  --payload file://test_events/medium_batch.json \
  response.json
```

**Create `test_events/medium_batch.json`:**
```json
{
  "batch_size": 2000,
  "time_limit_seconds": 60,
  "chunk_size": 100,
  "delay_between_chunks": 0.5,
  "max_concurrent_uploads": 5,
  "sqs_queue_url": "YOUR_QUEUE_URL",
  "webengage_api_key": "YOUR_KEY",
  "webengage_api_url": "https://api.webengage.com/v1/users/bulk"
}
```

**✅ Pass Criteria:**
- Execution time < 90 seconds
- All 1,000 records processed
- No errors in logs

---

#### Test 1.4: Large Batch (10,000 records)
```bash
# 1. Generate 10,000 records
# NUM_TEST_RECORDS = 10000
python sqs_test_data_generator.py

# 2. Test with production settings
aws lambda invoke \
  --function-name web_engage_service_lambda \
  --payload file://test_events/large_batch.json \
  response.json
```

**✅ Pass Criteria:**
- Execution completes (no timeout)
- All records processed
- Consistent chunk upload rate
- Memory usage stable

---

### Phase 2: Integration Testing (With WebEngage API)

**Duration:** 1-2 hours

#### Test 2.1: WebEngage API Connectivity
```bash
# Test actual WebEngage API call
curl -X POST https://api.webengage.com/v1/users/bulk \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "users": [
      {
        "userId": "test_user_1",
        "email": "test1@example.com",
        "firstName": "Test",
        "lastName": "User"
      }
    ]
  }'
```

**✅ Pass Criteria:**
- Status 200 or 201
- Valid response from WebEngage

---

#### Test 2.2: Rate Limit Testing
```bash
# Generate 5,000 records
python sqs_test_data_generator.py  # NUM_TEST_RECORDS = 5000

# Use aggressive settings to test rate limiting
{
  "batch_size": 10000,
  "time_limit_seconds": 120,
  "chunk_size": 100,
  "delay_between_chunks": 0.1,  # Very fast!
  "max_concurrent_uploads": 10,
  "sqs_queue_url": "YOUR_QUEUE_URL",
  "webengage_api_key": "YOUR_KEY",
  "webengage_api_url": "https://api.webengage.com/v1/users/bulk"
}
```

**Watch for:**
- Rate limit errors (429 status codes)
- Adjust `delay_between_chunks` until no rate limits

**✅ Pass Criteria:**
- Find optimal delay that avoids rate limits
- Document: "WebEngage accepts X requests/minute"

---

#### Test 2.3: Failure Recovery
```bash
# Test with intentionally bad API key
{
  "webengage_api_key": "INVALID_KEY",
  ...other settings...
}
```

**Expected:**
- Chunks fail with 401/403 errors
- Messages remain in SQS (not deleted)
- Clear error logs

**✅ Pass Criteria:**
- Lambda doesn't crash
- SQS messages preserved
- Error logged clearly

---

### Phase 3: EventBridge Integration Testing

**Duration:** 2-4 hours (due to waiting for triggers)

#### Test 3.1: Manual EventBridge Trigger
```bash
# Create EventBridge rule (disabled initially)
aws events put-rule \
  --name webengage-test-trigger \
  --schedule-expression "rate(5 minutes)" \
  --state DISABLED

# Add Lambda target
aws events put-targets \
  --rule webengage-test-trigger \
  --targets "Id"="1","Arn"="YOUR_LAMBDA_ARN","Input"='{"batch_size": 1000, ...}'

# Enable rule
aws events enable-rule --name webengage-test-trigger

# Wait 5 minutes and check CloudWatch Logs
```

**✅ Pass Criteria:**
- Lambda triggered automatically every 5 minutes
- Logs show EventBridge as invocation source

---

#### Test 3.2: Concurrent Invocation Test
```bash
# Verify Reserved Concurrency = 1
aws lambda get-function-concurrency \
  --function-name web_engage_service_lambda

# Manually trigger Lambda while EventBridge trigger is running
aws lambda invoke \
  --function-name web_engage_service_lambda \
  --payload file://test_event.json \
  response.json
```

**Expected:**
- Second invocation waits or is throttled
- Only 1 Lambda instance runs at a time

**✅ Pass Criteria:**
- Reserved concurrency prevents concurrent executions
- No race conditions observed

---

### Phase 4: Production Smoke Test

**Duration:** 24 hours monitoring

#### Test 4.1: Production Configuration
```json
{
  "batch_size": 50000,
  "time_limit_seconds": 120,
  "chunk_size": 1000,
  "delay_between_chunks": 1.0,
  "max_concurrent_uploads": 5,
  "sqs_queue_url": "PROD_QUEUE_URL",
  "webengage_api_key": "PROD_API_KEY",
  "webengage_api_url": "https://api.webengage.com/v1/users/bulk"
}
```

#### Test 4.2: Monitoring Checklist

**Hour 1-2:**
- [ ] Lambda executes every 2 minutes
- [ ] No errors in CloudWatch Logs
- [ ] SQS depth decreasing if messages present
- [ ] WebEngage receiving data

**Hour 2-6:**
- [ ] Consistent execution times
- [ ] No memory issues
- [ ] No rate limit errors
- [ ] Cost tracking (Lambda invocations)

**Hour 6-24:**
- [ ] Long-term stability
- [ ] Handle varying SQS loads
- [ ] No unexpected failures
- [ ] CloudWatch metrics healthy

---

## Complete Testing Checklist

### Pre-Testing Setup
- [ ] Lambda function deployed
- [ ] Reserved Concurrency = 1
- [ ] IAM permissions configured
- [ ] SQS queue created
- [ ] CloudWatch Logs enabled
- [ ] Test data generator ready
- [ ] WebEngage API credentials valid

### Unit Tests
- [ ] Empty queue handling
- [ ] Small batch (100 records)
- [ ] Medium batch (1,000 records)
- [ ] Large batch (10,000 records)
- [ ] Transformation correctness
- [ ] Error handling

### Integration Tests
- [ ] WebEngage API connectivity
- [ ] Rate limit compliance
- [ ] Chunk upload parallelism
- [ ] SQS message deletion
- [ ] Failure recovery
- [ ] Partial success handling

### EventBridge Tests
- [ ] Rule creation
- [ ] Scheduled triggering
- [ ] Manual triggering
- [ ] Concurrent execution prevention
- [ ] Parameter passing

### Production Tests
- [ ] Smoke test (24 hours)
- [ ] Performance monitoring
- [ ] Cost tracking
- [ ] Error alerting
- [ ] Rollback plan ready

---

## Troubleshooting Guide

### Issue: Lambda Times Out
**Symptoms:** Execution stops after X minutes
**Check:**
```bash
aws lambda get-function-configuration \
  --function-name web_engage_service_lambda \
  --query 'Timeout'
```
**Fix:** Increase timeout to 600 seconds

---

### Issue: No Messages Collected from SQS
**Symptoms:** "Collected 0 records"
**Check:**
```bash
aws sqs get-queue-attributes \
  --queue-url YOUR_QUEUE_URL \
  --attribute-names ApproximateNumberOfMessages
```
**Possible Causes:**
- Queue is empty
- Wrong queue URL
- IAM permissions missing
- Visibility timeout too long

---

### Issue: Rate Limit Errors (429)
**Symptoms:** Chunk uploads failing with "429 Too Many Requests"
**Fix:** Increase `delay_between_chunks`:
```json
{
  "delay_between_chunks": 2.0  // From 1.0 to 2.0
}
```

---

### Issue: Messages Not Deleted from SQS
**Symptoms:** Same messages processed repeatedly
**Check CloudWatch Logs for:**
- Upload success/failure
- Deletion operation logs
**Fix:** Verify IAM permissions include `sqs:DeleteMessage`

---

### Issue: Multiple Lambda Instances Running
**Symptoms:** Concurrent WebEngage uploads
**Check:**
```bash
aws lambda get-function-concurrency \
  --function-name web_engage_service_lambda
```
**Fix:** Set Reserved Concurrency = 1

---

## Performance Benchmarks

### Expected Execution Times

| Records | Polling | Transform | Upload | Total |
|---------|---------|-----------|--------|-------|
| 100 | 5s | 0.01s | 5s | ~10s |
| 1,000 | 15s | 0.05s | 10s | ~25s |
| 10,000 | 60s | 0.5s | 30s | ~90s |
| 50,000 | 120s | 2.5s | 60s | ~180s |

### Expected Costs (per execution)

| Records | Lambda Cost | API Calls | Total Cost |
|---------|-------------|-----------|------------|
| 100 | $0.0001 | 1 | ~$0.0001 |
| 1,000 | $0.0003 | 10 | ~$0.0003 |
| 10,000 | $0.001 | 100 | ~$0.001 |
| 50,000 | $0.003 | 50 | ~$0.003 |

---

## Success Criteria Summary

**System is production-ready when:**
- ✅ All unit tests pass
- ✅ Integration tests show no rate limiting
- ✅ EventBridge triggers reliably
- ✅ 24-hour smoke test completes without issues
- ✅ Monitoring and alerts configured
- ✅ Documentation complete
- ✅ Rollback plan tested



TEST CASES - 
```json
{
  "_comment": "TEST 1: Minimal Test - Empty Queue",
  "_description": "Tests Lambda behavior when SQS has no messages",
  "_expected_result": "Returns 200 with 'No records to process'",
  "_duration": "~5-10 seconds",
  
  "batch_size": 1000,
  "time_limit_seconds": 30,
  "chunk_size": 100,
  "delay_between_chunks": 0.5,
  "max_concurrent_uploads": 2,
  "sqs_queue_url": "https://sqs.us-east-1.amazonaws.com/123456789012/user-updates-queue",
  "webengage_api_key": "your-api-key-here",
  "webengage_api_url": "https://api.webengage.com/v1/users/bulk",
  "sqs_max_messages_per_poll": 10,
  "sqs_wait_time_seconds": 5
}

---

{
  "_comment": "TEST 2: Small Batch Test",
  "_description": "Tests with ~100 records in SQS (populate first!)",
  "_expected_result": "Processes 100 records, uploads in 10 chunks",
  "_duration": "~15-30 seconds",
  "_prerequisites": "Run: python sqs_test_data_generator.py with NUM_TEST_RECORDS=100",
  
  "batch_size": 200,
  "time_limit_seconds": 30,
  "chunk_size": 10,
  "delay_between_chunks": 0.5,
  "max_concurrent_uploads": 2,
  "sqs_queue_url": "https://sqs.us-east-1.amazonaws.com/123456789012/user-updates-queue",
  "webengage_api_key": "your-api-key-here",
  "webengage_api_url": "https://api.webengage.com/v1/users/bulk",
  "sqs_max_messages_per_poll": 10,
  "sqs_wait_time_seconds": 5
}

---

{
  "_comment": "TEST 3: Medium Batch Test",
  "_description": "Tests with ~1,000 records",
  "_expected_result": "Processes 1,000 records, uploads in 10 chunks of 100",
  "_duration": "~45-60 seconds",
  "_prerequisites": "Run: python sqs_test_data_generator.py with NUM_TEST_RECORDS=1000",
  
  "batch_size": 2000,
  "time_limit_seconds": 60,
  "chunk_size": 100,
  "delay_between_chunks": 0.5,
  "max_concurrent_uploads": 5,
  "sqs_queue_url": "https://sqs.us-east-1.amazonaws.com/123456789012/user-updates-queue",
  "webengage_api_key": "your-api-key-here",
  "webengage_api_url": "https://api.webengage.com/v1/users/bulk",
  "sqs_max_messages_per_poll": 10,
  "sqs_wait_time_seconds": 10
}

---

{
  "_comment": "TEST 4: Production Simulation",
  "_description": "Tests with realistic production settings (10K records)",
  "_expected_result": "Processes up to 10,000 records, uploads in chunks of 1,000",
  "_duration": "~120-180 seconds",
  "_prerequisites": "Run: python sqs_test_data_generator.py with NUM_TEST_RECORDS=10000",
  
  "batch_size": 50000,
  "time_limit_seconds": 120,
  "chunk_size": 1000,
  "delay_between_chunks": 1.0,
  "max_concurrent_uploads": 5,
  "sqs_queue_url": "https://sqs.us-east-1.amazonaws.com/123456789012/user-updates-queue",
  "webengage_api_key": "your-api-key-here",
  "webengage_api_url": "https://api.webengage.com/v1/users/bulk",
  "sqs_max_messages_per_poll": 10,
  "sqs_wait_time_seconds": 20
}

---

{
  "_comment": "TEST 5: Fast Rate Test",
  "_description": "Tests faster upload rate (might hit rate limits)",
  "_expected_result": "Shows if rate limiting is properly handled",
  "_duration": "~60-90 seconds",
  "_note": "Watch for 429 errors in logs - adjust delay if needed",
  
  "batch_size": 5000,
  "time_limit_seconds": 60,
  "chunk_size": 500,
  "delay_between_chunks": 0.2,
  "max_concurrent_uploads": 10,
  "sqs_queue_url": "https://sqs.us-east-1.amazonaws.com/123456789012/user-updates-queue",
  "webengage_api_key": "your-api-key-here",
  "webengage_api_url": "https://api.webengage.com/v1/users/bulk",
  "sqs_max_messages_per_poll": 10,
  "sqs_wait_time_seconds": 10
}

---

{
  "_comment": "TEST 6: Conservative Production Settings",
  "_description": "Very safe settings for first production run",
  "_expected_result": "Slow but reliable processing",
  "_duration": "~180-240 seconds",
  "_use_case": "Initial production deployment",
  
  "batch_size": 10000,
  "time_limit_seconds": 90,
  "chunk_size": 500,
  "delay_between_chunks": 2.0,
  "max_concurrent_uploads": 3,
  "sqs_queue_url": "https://sqs.us-east-1.amazonaws.com/123456789012/user-updates-queue",
  "webengage_api_key": "your-api-key-here",
  "webengage_api_url": "https://api.webengage.com/v1/users/bulk",
  "sqs_max_messages_per_poll": 10,
  "sqs_wait_time_seconds": 20
}

---

{
  "_comment": "TEST 7: Error Handling Test - Invalid API Key",
  "_description": "Tests error handling with bad credentials",
  "_expected_result": "Graceful failure, messages remain in SQS, clear error logs",
  "_duration": "~30 seconds",
  
  "batch_size": 500,
  "time_limit_seconds": 30,
  "chunk_size": 100,
  "delay_between_chunks": 0.5,
  "max_concurrent_uploads": 2,
  "sqs_queue_url": "https://sqs.us-east-1.amazonaws.com/123456789012/user-updates-queue",
  "webengage_api_key": "INVALID_KEY_FOR_TESTING",
  "webengage_api_url": "https://api.webengage.com/v1/users/bulk",
  "sqs_max_messages_per_poll": 10,
  "sqs_wait_time_seconds": 5
}

---

{
  "_comment": "TEST 8: Aggressive Settings - Stress Test",
  "_description": "Pushes limits to see maximum throughput",
  "_expected_result": "High throughput or rate limit errors",
  "_duration": "~60-90 seconds",
  "_warning": "May trigger WebEngage rate limits!",
  
  "batch_size": 20000,
  "time_limit_seconds": 60,
  "chunk_size": 1000,
  "delay_between_chunks": 0.1,
  "max_concurrent_uploads": 15,
  "sqs_queue_url": "https://sqs.us-east-1.amazonaws.com/123456789012/user-updates-queue",
  "webengage_api_key": "your-api-key-here",
  "webengage_api_url": "https://api.webengage.com/v1/users/bulk",
  "sqs_max_messages_per_poll": 10,
  "sqs_wait_time_seconds": 5
}

---

{
  "_comment": "TEST 9: Minimal Configuration - Uses All Defaults",
  "_description": "Only required fields, rest use Lambda defaults",
  "_expected_result": "Uses DEFAULT values from code",
  "_note": "Good for testing default behavior",
  
  "sqs_queue_url": "https://sqs.us-east-1.amazonaws.com/123456789012/user-updates-queue",
  "webengage_api_key": "your-api-key-here",
  "webengage_api_url": "https://api.webengage.com/v1/users/bulk"
}

---

{
  "_comment": "TEST 10: DRY RUN - Mock WebEngage URL",
  "_description": "Tests everything except actual WebEngage upload",
  "_expected_result": "Processes data but uploads fail (expected)",
  "_use_case": "Test polling and transformation without hitting WebEngage",
  "_note": "Use a mock endpoint like requestbin.com or httpbin.org",
  
  "batch_size": 1000,
  "time_limit_seconds": 60,
  "chunk_size": 100,
  "delay_between_chunks": 0.1,
  "max_concurrent_uploads": 3,
  "sqs_queue_url": "https://sqs.us-east-1.amazonaws.com/123456789012/user-updates-queue",
  "webengage_api_key": "mock-key",
  "webengage_api_url": "https://httpbin.org/post",
  "sqs_max_messages_per_poll": 10,
  "sqs_wait_time_seconds": 5
}
```


# Test with dummy data

# Complete Guide: Pushing Data to SQS

## Method 1: Python Script (RECOMMENDED)

### Step 1: Install Dependencies
```bash
pip install boto3
```

### Step 2: Configure AWS Credentials

**Option A: AWS CLI Configuration**
```bash
aws configure
# Enter:
# AWS Access Key ID: YOUR_ACCESS_KEY
# AWS Secret Access Key: YOUR_SECRET_KEY
# Default region name: us-east-1
# Default output format: json
```

**Option B: Environment Variables**
```bash
export AWS_ACCESS_KEY_ID="your-access-key"
export AWS_SECRET_ACCESS_KEY="your-secret-key"
export AWS_DEFAULT_REGION="us-east-1"
```

**Option C: IAM Role (if running on EC2/Lambda)**
- No configuration needed
- EC2 instance role automatically provides credentials

### Step 3: Update Configuration in Script

Edit `sqs_test_data_generator.py`:
```python
# MODIFY THESE VALUES
SQS_QUEUE_URL = "https://sqs.us-east-1.amazonaws.com/YOUR_ACCOUNT/YOUR_QUEUE"
AWS_REGION = "us-east-1"
NUM_TEST_RECORDS = 100  # Change this number
```

### Step 4: Run the Script
```bash
python sqs_test_data_generator.py
```

**Output:**
```
======================================================================
SQS Test Data Generator & Pusher
======================================================================

======================================================================
Sample Message Preview:
======================================================================
{
  "user_id": "user_000001",
  "email": "james.smith@example.com",
  ...
}
======================================================================

Checking current queue state...
======================================================================
Current Queue Status:
======================================================================
Visible messages (ready to process): 0
In-flight messages (being processed): 0
Delayed messages: 0
Total messages: 0
======================================================================

Generating and sending 100 user records...
======================================================================
Starting SQS Population
======================================================================
Queue URL: https://sqs.us-east-1.amazonaws.com/...
Records to generate: 100
Mode: Batch
======================================================================

Progress: 10.0% | Batch 1/10 | Sent: 10/10
Progress: 20.0% | Batch 2/10 | Sent: 10/10
...
Progress: 100.0% | Batch 10/10 | Sent: 10/10

======================================================================
Summary:
======================================================================
Total messages sent successfully: 100
Failed messages: 0
Success rate: 100.00%
Time elapsed: 2.45 seconds
Rate: 40.8 messages/second
======================================================================

✅ Ready for Lambda testing!
```

---

## Method 2: AWS CLI

### Send Single Message

**Basic:**
```bash
aws sqs send-message \
  --queue-url "https://sqs.us-east-1.amazonaws.com/123456789012/user-updates-queue" \
  --message-body '{"user_id":"user_001","email":"test@example.com","first_name":"John","last_name":"Doe"}'
```

**With Proper JSON:**
```bash
aws sqs send-message \
  --queue-url "https://sqs.us-east-1.amazonaws.com/123456789012/user-updates-queue" \
  --message-body '{
    "user_id": "user_001",
    "email": "john.doe@example.com",
    "first_name": "John",
    "last_name": "Doe",
    "phone": "+1-555-0101",
    "attributes": {
      "age": 30,
      "city": "New York",
      "plan": "premium"
    },
    "timestamp": 1705750200000
  }'
```

**From File:**
```bash
# Create message.json file first
echo '{
  "user_id": "user_001",
  "email": "john.doe@example.com",
  "first_name": "John",
  "last_name": "Doe"
}' > message.json

# Send from file
aws sqs send-message \
  --queue-url "YOUR_QUEUE_URL" \
  --message-body file://message.json
```

### Send Batch Messages (Up to 10)

**Create batch.json:**
```json
[
  {
    "Id": "1",
    "MessageBody": "{\"user_id\":\"user_001\",\"email\":\"user1@example.com\",\"first_name\":\"John\",\"last_name\":\"Doe\"}"
  },
  {
    "Id": "2",
    "MessageBody": "{\"user_id\":\"user_002\",\"email\":\"user2@example.com\",\"first_name\":\"Jane\",\"last_name\":\"Smith\"}"
  },
  {
    "Id": "3",
    "MessageBody": "{\"user_id\":\"user_003\",\"email\":\"user3@example.com\",\"first_name\":\"Bob\",\"last_name\":\"Johnson\"}"
  }
]
```

**Send batch:**
```bash
aws sqs send-message-batch \
  --queue-url "YOUR_QUEUE_URL" \
  --entries file://batch.json
```

### Send Multiple Messages in Loop

**Bash Script:**
```bash
#!/bin/bash

QUEUE_URL="https://sqs.us-east-1.amazonaws.com/123456789012/user-updates-queue"

for i in {1..100}; do
  aws sqs send-message \
    --queue-url "$QUEUE_URL" \
    --message-body "{\"user_id\":\"user_$i\",\"email\":\"user$i@example.com\",\"first_name\":\"User\",\"last_name\":\"$i\"}"
  
  echo "Sent message $i/100"
  sleep 0.1
done

echo "All messages sent!"
```

---

## Method 3: AWS Console (Manual)

### Step-by-Step:

1. **Go to SQS Console**
   - Navigate to: https://console.aws.amazon.com/sqs/

2. **Select Your Queue**
   - Click on your queue name: `user-updates-queue`

3. **Send Message**
   - Click **"Send and receive messages"** button (top right)

4. **Enter Message Body**
   - In the "Message body" text area, paste:
   ```json
   {
     "user_id": "user_001",
     "email": "john.doe@example.com",
     "first_name": "John",
     "last_name": "Doe",
     "phone": "+1-555-0101",
     "attributes": {
       "age": 30,
       "city": "New York",
       "plan": "premium"
     },
     "timestamp": 1705750200000
   }
   ```

5. **Click "Send message"**

6. **Repeat** for multiple messages (tedious but works for testing)

7. **Verify Messages**
   - Click "Poll for messages" button
   - You should see your messages appear

---

## Method 4: Node.js Script

**install_and_run.sh:**
```bash
npm install @aws-sdk/client-sqs
node push_to_sqs.js
```

**push_to_sqs.js:**
```javascript
const { SQSClient, SendMessageCommand, SendMessageBatchCommand } = require("@aws-sdk/client-sqs");

const client = new SQSClient({ region: "us-east-1" });
const QUEUE_URL = "https://sqs.us-east-1.amazonaws.com/123456789012/user-updates-queue";

// Generate dummy user
function generateUser(index) {
  return {
    user_id: `user_${String(index).padStart(6, '0')}`,
    email: `user${index}@example.com`,
    first_name: `User${index}`,
    last_name: `Test`,
    phone: `+1-555-${String(Math.floor(Math.random() * 10000)).padStart(4, '0')}`,
    attributes: {
      age: Math.floor(Math.random() * 50) + 18,
      city: "New York",
      plan: ["free", "basic", "premium"][Math.floor(Math.random() * 3)]
    },
    timestamp: Date.now()
  };
}

// Send single message
async function sendMessage(user) {
  const command = new SendMessageCommand({
    QueueUrl: QUEUE_URL,
    MessageBody: JSON.stringify(user)
  });
  
  try {
    const response = await client.send(command);
    return { success: true, messageId: response.MessageId };
  } catch (error) {
    return { success: false, error: error.message };
  }
}

// Send batch messages
async function sendBatch(users) {
  const entries = users.map((user, index) => ({
    Id: String(index),
    MessageBody: JSON.stringify(user)
  }));
  
  const command = new SendMessageBatchCommand({
    QueueUrl: QUEUE_URL,
    Entries: entries
  });
  
  try {
    const response = await client.send(command);
    return {
      successful: response.Successful.length,
      failed: response.Failed.length
    };
  } catch (error) {
    console.error("Batch send error:", error);
    return { successful: 0, failed: entries.length };
  }
}

// Main function
async function main() {
  const NUM_RECORDS = 100;
  const BATCH_SIZE = 10;
  
  console.log(`Sending ${NUM_RECORDS} messages to SQS...`);
  
  let totalSuccessful = 0;
  let totalFailed = 0;
  
  for (let i = 0; i < NUM_RECORDS; i += BATCH_SIZE) {
    const batchUsers = [];
    for (let j = i; j < Math.min(i + BATCH_SIZE, NUM_RECORDS); j++) {
      batchUsers.push(generateUser(j));
    }
    
    const result = await sendBatch(batchUsers);
    totalSuccessful += result.successful;
    totalFailed += result.failed;
    
    console.log(`Progress: ${Math.min(i + BATCH_SIZE, NUM_RECORDS)}/${NUM_RECORDS} | Sent: ${result.successful}/${batchUsers.length}`);
    
    // Small delay
    await new Promise(resolve => setTimeout(resolve, 100));
  }
  
  console.log("\n=================================");
  console.log(`Total successful: ${totalSuccessful}`);
  console.log(`Total failed: ${totalFailed}`);
  console.log(`Success rate: ${(totalSuccessful/NUM_RECORDS*100).toFixed(2)}%`);
  console.log("=================================\n");
}

main();
```

---

## Method 5: Quick Test with curl (HTTP API)

**Not recommended for production, but works for quick tests**

```bash
# Note: This requires AWS Signature V4, so it's complex
# Better to use AWS CLI or SDK

# Install awscurl helper
pip install awscurl

# Send message
awscurl --service sqs \
  --region us-east-1 \
  -X POST \
  "https://sqs.us-east-1.amazonaws.com/123456789012/user-updates-queue" \
  -d "Action=SendMessage&MessageBody=$(echo '{"user_id":"user_001","email":"test@example.com"}' | jq -sRr @uri)"
```

---

## Quick Reference: Get Your Queue URL

### Method 1: AWS Console
1. Go to SQS Console
2. Click on your queue
3. Copy the URL shown at the top

### Method 2: AWS CLI
```bash
aws sqs list-queues

# Or get specific queue
aws sqs get-queue-url --queue-name user-updates-queue
```

**Output:**
```json
{
    "QueueUrl": "https://sqs.us-east-1.amazonaws.com/123456789012/user-updates-queue"
}
```

### Method 3: boto3 (Python)
```python
import boto3

sqs = boto3.client('sqs', region_name='us-east-1')
response = sqs.get_queue_url(QueueName='user-updates-queue')
print(response['QueueUrl'])
```

---

## Verification: Check Messages in Queue

### AWS CLI:
```bash
# Get queue attributes
aws sqs get-queue-attributes \
  --queue-url "YOUR_QUEUE_URL" \
  --attribute-names All

# Peek at messages (doesn't remove them)
aws sqs receive-message \
  --queue-url "YOUR_QUEUE_URL" \
  --max-number-of-messages 10 \
  --visibility-timeout 0
```

### Python:
```python
import boto3
import json

sqs = boto3.client('sqs')
queue_url = "YOUR_QUEUE_URL"

# Get queue depth
response = sqs.get_queue_attributes(
    QueueUrl=queue_url,
    AttributeNames=['ApproximateNumberOfMessages']
)
print(f"Messages in queue: {response['Attributes']['ApproximateNumberOfMessages']}")

# Receive and print first message
response = sqs.receive_message(
    QueueUrl=queue_url,
    MaxNumberOfMessages=1
)

if 'Messages' in response:
    message = response['Messages'][0]
    print("Sample message:")
    print(json.dumps(json.loads(message['Body']), indent=2))
```

### AWS Console:
1. Go to your queue
2. Click "Send and receive messages"
3. Click "Poll for messages"
4. View messages (they remain in queue)

---

## Troubleshooting

### Error: "The security token included in the request is invalid"
**Fix:** Configure AWS credentials properly
```bash
aws configure
# Or set environment variables
```

### Error: "Access Denied"
**Fix:** Add SQS permissions to your IAM user/role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:SendMessageBatch",
        "sqs:GetQueueUrl",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:*:*:user-updates-queue"
    }
  ]
}
```

### Error: "Queue does not exist"
**Fix:** Create the queue first
```bash
aws sqs create-queue --queue-name user-updates-queue
```

### Error: "Rate exceeded"
**Fix:** Add delays between sends
```python
import time
time.sleep(0.1)  # 100ms delay
```

---

## Performance Tips

### Batch Sending is Faster
- Single messages: ~5-10 messages/second
- Batch (10 at a time): ~50-100 messages/second
- **Always use batch when sending multiple messages**

### Async Sending is Even Faster
```python
import asyncio
import aioboto3

async def send_messages_async(num_messages):
    session = aioboto3.Session()
    async with session.client('sqs') as sqs:
        tasks = []
        for i in range(num_messages):
            task = sqs.send_message(
                QueueUrl=QUEUE_URL,
                MessageBody=json.dumps(generate_user(i))
            )
            tasks.append(task)
        
        await asyncio.gather(*tasks)

# Run
asyncio.run(send_messages_async(100))
```

---

## Summary: Best Method for Each Scenario

| Scenario | Best Method | Why |
|----------|-------------|-----|
| **Quick 1-10 messages** | AWS Console | Visual, easy, no code |
| **Testing (10-1000 messages)** | Python Script | Fast, flexible, reusable |
| **Production (1000+ messages)** | Python Script (batch) | Efficient, reliable |
| **One-time test** | AWS CLI | Quick, no dependencies |
| **CI/CD pipeline** | AWS CLI or SDK | Scriptable, automated |
| **From another app** | SDK (boto3/aws-sdk) | Native integration |

**Recommendation: Use the Python script** - it's the most versatile and efficient!
