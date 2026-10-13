In AWS, **state machine** usually refers to an **AWS Step Functions state machine**.

AWS Step Functions lets you coordinate multiple steps of a workflow. Each step is represented by a **state**, and Step Functions controls:

- Which state runs next
- What data is passed between states
- Error handling and retries
- Conditional branching
- Parallel execution
- Waiting and polling
- Workflow execution history

A Step Functions state machine is defined using **Amazon States Language**, usually in JSON or YAML.

## Example use case: Order approval workflow

Suppose an order should follow this process:

1. Receive an order
2. Check the order amount
3. If the amount is less than or equal to `$1,000`, approve automatically
4. Otherwise, send it for manual approval
5. Notify the customerv

The workflow can be represented as:

```text
OrderReceived
      |
      v
CheckOrderAmount
   /          \
<= 1000       > 1000
  |             |
  v             v
AutoApprove   ManualReview
   \           /
    v         v
 NotifyCustomer
```

## Step Functions definition

The following is an AWS Step Functions state machine definition using Amazon States Language:

```json
{
  "Comment": "Order approval workflow",
  "StartAt": "CheckOrderAmount",
  "States": {
    "CheckOrderAmount": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.order.amount",
          "NumericLessThanEquals": 1000,
          "Next": "AutoApprove"
        }
      ],
      "Default": "ManualReview"
    },
    "AutoApprove": {
      "Type": "Pass",
      "Result": {
        "approvalStatus": "APPROVED",
        "approvalType": "AUTOMATIC"
      },
      "ResultPath": "$.approval",
      "Next": "NotifyCustomer"
    },
    "ManualReview": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:manual-review-function",
      "ResultPath": "$.approval",
      "Retry": [
        {
          "ErrorEquals": [
            "Lambda.ServiceException",
            "Lambda.AWSLambdaException",
            "Lambda.SdkClientException"
          ],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2
        }
      ],
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "ResultPath": "$.error",
          "Next": "ManualReviewFailed"
        }
      ],
      "Next": "NotifyCustomer"
    },
    "NotifyCustomer": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:notify-customer-function",
      "Parameters": {
        "orderId.$": "$.order.orderId",
        "customerEmail.$": "$.order.customerEmail",
        "approval.$": "$.approval"
      },
      "End": true
    },
    "ManualReviewFailed": {
      "Type": "Fail",
      "Error": "ManualReviewFailed",
      "Cause": "The order could not be manually reviewed"
    }
  }
}
```

Replace these values with your AWS details:

```text
REGION
ACCOUNT_ID
manual-review-function
notify-customer-function
```

## Input to the state machine

You can start an execution with this JSON:

```json
{
  "order": {
    "orderId": "ORD-1001",
    "amount": 750,
    "customerEmail": "customer@example.com"
  }
}
```

Because the amount is `750`, the execution follows this path:

```text
CheckOrderAmount
        |
        v
AutoApprove
        |
        v
NotifyCustomer
```

For an order with an amount of `2500`:

```json
{
  "order": {
    "orderId": "ORD-1002",
    "amount": 2500,
    "customerEmail": "customer@example.com"
  }
}
```

The path becomes:

```text
CheckOrderAmount
        |
        v
ManualReview
        |
        v
NotifyCustomer
```

## Important Step Functions state types

### `Pass`

Used to pass data or add a fixed result.

```json
{
  "PrepareData": {
    "Type": "Pass",
    "Result": {
      "message": "Data prepared"
    },
    "ResultPath": "$.result",
    "End": true
  }
}
```

### `Task`

Used to call an AWS service, such as Lambda, Glue, ECS, SNS, or DynamoDB.

```json
{
  "RunLambda": {
    "Type": "Task",
    "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:my-function",
    "End": true
  }
}
```

### `Choice`

Used for conditional logic.

```json
{
  "CheckAmount": {
    "Type": "Choice",
    "Choices": [
      {
        "Variable": "$.amount",
        "NumericGreaterThan": 1000,
        "Next": "NeedsApproval"
      }
    ],
    "Default": "Approved"
  }
}
```

### `Wait`

Pauses the workflow.

```json
{
  "WaitForTenSeconds": {
    "Type": "Wait",
    "Seconds": 10,
    "Next": "ContinueWorkflow"
  }
}
```

### `Parallel`

Runs multiple branches at the same time.

```json
{
  "RunParallelTasks": {
    "Type": "Parallel",
    "Branches": [
      {
        "StartAt": "SendEmail",
        "States": {
          "SendEmail": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:send-email",
            "End": true
          }
        }
      },v
      {
        "StartAt": "UpdateDatabase",
        "States": {
          "UpdateDatabase": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:update-database",
            "End": true
          }
        }
      }
    ],
    "End": true
  }
}
```

## Input and output flow

By default, the output of one state becomes the input of the next state.

For example, the initial input is:

```json
{
  "orderId": "ORD-1001",
  "amount": 750
}
```

A `Pass` state can add data using `ResultPath`:

```json
{
  "ApproveOrder": {
    "Type": "Pass",
    "Result": {
      "status": "APPROVED"
    },
    "ResultPath": "$.approval",
    "End": true
  }
}
```

The resulting data becomes:

```json
{
  "orderId": "ORD-1001",
  "amount": 750,
  "approval": {
    "status": "APPROVED"
  }
}
```

## JSONPath in Step Functions

AWS Step Functions uses JSONPath to select data.

| Expression | Meaning |
|---|---|
| `$.order.orderId` | Reads the order ID from the workflow input |
| `$.order.amount` | Reads the order amount |
| `$.approval.status` | Reads the approval status |
| `$` | The entire current state input |
| `$.items[0]` | The first element of an items array |

The `.$` suffix means that the value is a JSONPath expression:

```json
{
  "orderId.$": "$.order.orderId",
  "amount.$": "$.order.amount"
}
```

Without `.$`, the value is treated as a literal string:

```json
{
  "orderId": "$.order.orderId"
}
```

## Step Functions context object

Step Functions also provides execution metadata through the **context object**. You access it with `$$`.

Examples:

```text
$$.Execution.Id
$$.Execution.Name
$$.Execution.StartTime
$$.State.Name
$$.State.RetryCount
$$.StateMachine.Id
```

For example:

```json
{
  "RecordExecution": {
    "Type": "Pass",
    "Parameters": {
      "executionId.$": "$$.Execution.Id",
      "executionName.$": "$$.Execution.Name",
      "currentState.$": "$$.State.Name"
    },
    "ResultPath": "$.executionInfo",
    "End": true
  }
}
```

This can produce:

```json
{
  "executionInfo": {
    "executionId": "arn:aws:states:region:account:execution:OrderWorkflow:abc123",
    "executionName": "order-execution-001",
    "currentState": "RecordExecution"
  }
}
```

The context object is useful for:

- Logging execution IDs
- Correlating workflow activity
- Creating unique record IDs
- Tracking retries
- Auditing state transitions
- Passing execution metadata to Lambda

## Lambda used by the `ManualReview` state

Example Python Lambda:

```python
def lambda_handler(event, context):
    order = event["order"]

    amount = order["amount"]

    if amount <= 5000:
        status = "APPROVED"
    else:
        status = "REJECTED"

    return {
        "approvalStatus": status,
        "approvalType": "MANUAL",
        "orderId": order["orderId"]
    }
```

The Lambda receives the current Step Functions state input:

```json
{
  "order": {
    "orderId": "ORD-1002",
    "amount": 2500,
    "customerEmail": "customer@example.com"
  }
}
```

Its output is stored at:

```text
$.approval
```

Therefore, the workflow data becomes:

```json
{
  "order": {
    "orderId": "ORD-1002",
    "amount": 2500,
    "customerEmail": "customer@example.com"
  },
  "approval": {
    "approvalStatus": "APPROVED",
    "approvalType": "MANUAL",
    "orderId": "ORD-1002"
  }
}
```

## How to implement it in AWS

1. Create the Lambda functions:
   - `manual-review-function`
   - `notify-customer-function`

2. Open **AWS Step Functions** in the AWS Console.

3. Choose **Create state machine**.

4. Select:
   - **Write workflow in code**
   - JSON or YAML definition

5. Paste the state machine definition.

6. Configure an IAM execution role with permission to invoke the Lambda functions.
v
7. Create the state machine.

8. Choose **Start execution**.

9. Paste the input JSON.

10. Review the execution graph and event history.

The main concept is:

> In AWS Step Functions, a state machine is a serverless workflow definition where each state performs an action, evaluates a condition, waits, or handles an error before moving to the next state.
