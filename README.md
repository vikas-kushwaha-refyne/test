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
