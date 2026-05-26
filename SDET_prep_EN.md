# MID SDET — Interview Preparation (English)

---

## Table of Contents

- [1. Python Testing](#1-python-testing)
  - [1.1 ISTQB — Test Levels, Types & Techniques](#11-istqb--test-levels-types--techniques)
  - [1.2 Test Pyramid](#12-test-pyramid)
- [2. AWS / Cloud Testing](#2-aws--cloud-testing)
  - [2.1 AWS Testing Overview](#21-aws-testing-overview)
  - [2.2 Mocking AWS with moto](#22-mocking-aws-with-moto)
  - [2.3 Testing AWS Lambda](#23-testing-aws-lambda)
  - [2.4 Testing SQS & SNS](#24-testing-sqs--sns)
  - [2.5 Testing DynamoDB](#25-testing-dynamodb)
  - [2.6 LocalStack vs moto](#26-localstack-vs-moto)
  - [2.7 IaC Testing (CDK / Terraform)](#27-iac-testing-cdk--terraform)
- [3. ROS2 / Robotics Testing](#3-ros2--robotics-testing)
  - [3.1 ROS2 Architecture](#31-ros2-architecture)
  - [3.2 pytest + rclpy Setup](#32-pytest--rclpy-setup)
  - [3.3 Testing Publisher / Subscriber](#33-testing-publisher--subscriber)
  - [3.4 Testing Services](#34-testing-services)
  - [3.5 Launch Testing](#35-launch-testing)
  - [3.6 Hardware Mocking](#36-hardware-mocking)
  - [3.7 Bag Files in Tests](#37-bag-files-in-tests)
- [4. Embedded Testing](#4-embedded-testing)
  - [4.1 MIL / SIL / HIL](#41-mil--sil--hil)
  - [4.2 HAL Mocking](#42-hal-mocking)
  - [4.3 Protocol Testing (UART / SPI / I2C)](#43-protocol-testing-uart--spi--i2c)
  - [4.4 Code Coverage for Embedded](#44-code-coverage-for-embedded)
  - [4.5 HIL Framework with pyserial](#45-hil-framework-with-pyserial)
  - [4.6 CAN Bus Testing](#46-can-bus-testing)
- [5. General SDET Concepts](#5-general-sdet-concepts)
  - [5.1 Richardson Maturity Model](#51-richardson-maturity-model)
  - [5.2 TAE v2 + TAS v1](#52-tae-v2--tas-v1)
  - [5.3 REST vs RESTful](#53-rest-vs-restful)
  - [5.4 Design Patterns](#54-design-patterns)
  - [5.5 SOLID Principles](#55-solid-principles)
  - [5.6 OOP Principles](#56-oop-principles)
  - [5.7 Framework Definition](#57-framework-definition)
  - [5.8 Code Review](#58-code-review)
- [6. API Testing](#6-api-testing)
  - [6.1 HTTP Methods](#61-http-methods)
  - [6.2 HTTP Status Codes](#62-http-status-codes)
  - [6.3 Authentication](#63-authentication)
  - [6.4 PUT vs PATCH](#64-put-vs-patch)
  - [6.5 PathParam vs QueryParam](#65-pathparam-vs-queryparam)
- [7. Selenium](#7-selenium)
  - [7.1 Locators](#71-locators)
  - [7.2 Waits](#72-waits)
- [8. Git](#8-git)
- [9. Python OOP — Deep Dive](#9-python-oop--deep-dive)

---

## 1. Python Testing

---

### 1.1 ISTQB — Test Levels, Types & Techniques

---

**Q: What are the four ISTQB test levels?**

| Level | Goal | Who writes | Example |
|-------|------|-----------|---------|
| **Unit** | Test single function/class in isolation | Developer / SDET | `pytest` — calculator without DB |
| **Integration** | Test cooperation of 2+ modules | SDET | API + DB — does the endpoint actually save data |
| **System** | Test full system as black-box | QA / SDET | E2E checkout flow on staging |
| **Acceptance (UAT)** | Validate business requirements are met | Client / PO | Acceptance tests before go-live |

```
           ▲ cost & time
           │
Acceptance │  ████ (few, slow)
  System   │  ██████████
Integration│  ████████████████
    Unit   │  ████████████████████████ (many, fast)
           └──────────────────────────────► isolation
```

---

**Q: What are the ISTQB test types?**

| Type | Description | Example |
|------|-------------|---------|
| **Functional** | Does the system do what it should | Login, add-to-cart |
| **Non-functional** | How well it does it (performance, security, usability) | Load test 1000 req/s, OWASP DAST |
| **White-box** | Tester knows the internal code | Statement/branch coverage, unit tests |
| **Regression** | New changes don't break existing features | Full automated suite on every PR |
| **Retests** | A specific bug is now fixed | Retest after fix, close ticket |

---

**Q: Describe the black-box testing techniques.**

**1. Equivalence Partitioning**

Divide inputs into groups where each element behaves identically. Test one representative per group.

```
Age field (18–65):
  Invalid < 18  → test: 17
  Valid 18–65   → test: 40
  Invalid > 65  → test: 66
```

```python
@pytest.mark.parametrize("age, expected", [
    (17, "invalid"),
    (40, "valid"),
    (66, "invalid"),
])
def test_age_validation(age, expected):
    assert validate_age(age) == expected
```

**2. Boundary Value Analysis**

Test values at the edges of equivalence classes — that's where bugs hide.

```python
@pytest.mark.parametrize("age, expected", [
    (17, "invalid"),  # just below min
    (18, "valid"),    # min boundary
    (19, "valid"),    # just above min
    (64, "valid"),    # just below max
    (65, "valid"),    # max boundary
    (66, "invalid"),  # just above max
])
def test_age_boundaries(age, expected):
    assert validate_age(age) == expected
```

**3. Decision Table Testing**

For combinations of Boolean conditions → actions.

```
| User logged in | Is admin | Action              |
|----------------|----------|---------------------|
| No             | -        | Redirect to login   |
| Yes            | No       | Show user dashboard |
| Yes            | Yes      | Show admin panel    |
```

**4. State Transition Testing**

Model the system as a state machine, test every valid and invalid transition.

```python
def test_order_state_transitions(order_service):
    order = order_service.create()
    assert order.status == "PENDING"
    order_service.pay(order.id)
    assert order.status == "PAID"
    order_service.ship(order.id)
    assert order.status == "SHIPPED"
    with pytest.raises(InvalidStateError):
        order_service.pay(order.id)   # invalid: can't pay a shipped order
```

---

**Q: Statement Coverage vs Decision Coverage — what's the difference?**

**Statement Coverage** — was every line of code executed at least once?

**Decision Coverage (Branch Coverage)** — was every branch (true AND false) of every decision point exercised?

```python
def process(x):
    if x > 0:
        return "positive"
    return "non-positive"

# Statement 100%: test x=1  (both lines run)
# Decision  100%: test x=1 AND x=-1  (both branches)
```

**Gotcha:** 100% statement ≠ 100% decision. Always aim for branch coverage in CI.

```python
pytest --cov=src --cov-branch --cov-report=term-missing --cov-fail-under=90
```

**MC/DC (Modified Condition/Decision Coverage)** — required for safety-critical systems (DO-178C, ISO 26262). Each individual condition must independently affect the decision outcome.

---

### 1.2 Test Pyramid

---

**Q: What is the test pyramid and what are its layers?**

```
          /\
         /  \
        / E2E\          ← few (5–10%), slow, expensive, flaky
       /______\
      /        \
     /Integration\      ← medium (20–30%), test API/DB/services
    /____________\
   /              \
  /   Unit Tests   \    ← many (60–70%), fast, cheap, deterministic
 /_________________ \
```

| Layer | Goal | Python tools | Speed |
|-------|------|-------------|-------|
| **Unit** | Single function/class, mock everything external | `pytest`, `unittest.mock` | ms |
| **Integration** | Real DB/API interactions between modules | `pytest`, `requests`, `testcontainers` | s |
| **E2E** | Full user flow through UI | `playwright`, `selenium` | min |

---

**Q: What is the "Ice Cream Cone" anti-pattern?**

Inverted pyramid — many slow E2E tests, few unit tests.

Consequences:
- CI takes 45 min instead of 5 min
- Flaky tests block every deploy
- Hard to pinpoint root cause of failures

**Fix:** Replace E2E tests with contract tests (Pact) + API-level integration tests.

---

**Q: Additional recruitment questions:**

- *Why are unit tests faster than E2E?*
  → No network, no browser, no DB — pure in-memory function calls. 1000 unit tests < 5s; 1000 E2E could take hours.

- *When would you skip unit tests and write integration tests instead?*
  → When the logic IS the integration — complex SQL query, message broker consumption. Mocking the DB tests the mock, not the query.

- *What is the "test trophy" (Kent C. Dodds)?*
  → Trophy emphasizes integration tests most (widest band), with static analysis at the base. Best confidence-to-cost ratio for modern web apps.

---

## 2. AWS / Cloud Testing

---

### 2.1 AWS Testing Overview

---

**Q: What do we test in AWS and what tools do we use?**

Two layers:
- **Application code** using AWS services (Lambda, S3, SQS, DynamoDB)
- **Infrastructure** (CloudFormation, CDK, Terraform) — correct resources, permissions, policies

```
Python AWS testing tools:
┌────────────────────────────────────────────────────┐
│  moto       — mock boto3 calls in-process (fast)   │
│  LocalStack — Docker: full AWS emulator locally    │
│  boto3      — official AWS SDK for Python          │
│  aws-cdk    — Infrastructure as Code (Python DSL)  │
│  checkov    — static analysis for IaC security     │
└────────────────────────────────────────────────────┘
```

| Approach | When | Speed | Fidelity |
|----------|------|-------|---------|
| `moto` | Unit tests, fast CI | ★★★★★ | ★★★ |
| `LocalStack` | Integration tests, local dev | ★★★★ | ★★★★ |
| AWS Sandbox | Pre-production E2E | ★★ | ★★★★★ |

---

### 2.2 Mocking AWS with moto

---

**Q: How do you use moto to mock AWS services in Python tests?**

`moto` intercepts all `boto3` calls and simulates AWS service responses in-memory — zero real AWS connection needed.

```python
import boto3
import pytest
from moto import mock_aws

@pytest.fixture
def aws_credentials(monkeypatch):
    monkeypatch.setenv("AWS_ACCESS_KEY_ID",     "testing")
    monkeypatch.setenv("AWS_SECRET_ACCESS_KEY", "testing")
    monkeypatch.setenv("AWS_DEFAULT_REGION",    "us-east-1")

@pytest.fixture
def s3_client(aws_credentials):
    with mock_aws():
        client = boto3.client("s3", region_name="us-east-1")
        client.create_bucket(Bucket="test-bucket")
        yield client

def test_upload_and_retrieve(s3_client):
    s3_client.put_object(Bucket="test-bucket", Key="result.json", Body=b'{"ok": true}')
    body = s3_client.get_object(Bucket="test-bucket", Key="result.json")["Body"].read()
    assert body == b'{"ok": true}'

def test_nonexistent_key_raises(s3_client):
    from botocore.exceptions import ClientError
    with pytest.raises(ClientError) as exc:
        s3_client.get_object(Bucket="test-bucket", Key="ghost.txt")
    assert exc.value.response["Error"]["Code"] == "NoSuchKey"
```

---

### 2.3 Testing AWS Lambda

---

**Q: How do you test Lambda functions in Python — from unit to integration?**

Lambda handler is just a Python function — import and call it directly.

```python
# lambda_function.py
import json, boto3, os

def handler(event, context):
    s3 = boto3.client("s3")
    bucket = event["Records"][0]["s3"]["bucket"]["name"]
    key    = event["Records"][0]["s3"]["object"]["key"]
    obj    = s3.get_object(Bucket=bucket, Key=key)
    data   = json.loads(obj["Body"].read())
    transformed = {k: str(v).upper() for k, v in data.items()}
    s3.put_object(
        Bucket=os.environ["OUTPUT_BUCKET"],
        Key=f"processed/{key}",
        Body=json.dumps(transformed)
    )
    return {"statusCode": 200, "processed": len(transformed)}
```

```python
# test_lambda_function.py
import json, pytest
from moto import mock_aws
import boto3
from lambda_function import handler

class FakeLambdaContext:
    function_name = "test-fn"
    aws_request_id = "test-id"

@pytest.fixture(autouse=True)
def env(monkeypatch):
    monkeypatch.setenv("AWS_ACCESS_KEY_ID", "testing")
    monkeypatch.setenv("AWS_SECRET_ACCESS_KEY", "testing")
    monkeypatch.setenv("AWS_DEFAULT_REGION", "us-east-1")
    monkeypatch.setenv("OUTPUT_BUCKET", "out-bucket")

@pytest.fixture
def s3(env):
    with mock_aws():
        c = boto3.client("s3", region_name="us-east-1")
        c.create_bucket(Bucket="in-bucket")
        c.create_bucket(Bucket="out-bucket")
        yield c

def s3_event(bucket, key):
    return {"Records": [{"s3": {"bucket": {"name": bucket}, "object": {"key": key}}}]}

def test_transforms_to_uppercase(s3):
    s3.put_object(Bucket="in-bucket", Key="d.json", Body=json.dumps({"name": "alice"}))
    handler(s3_event("in-bucket", "d.json"), FakeLambdaContext())
    out = json.loads(s3.get_object(Bucket="out-bucket", Key="processed/d.json")["Body"].read())
    assert out == {"name": "ALICE"}

def test_missing_file_raises(s3):
    from botocore.exceptions import ClientError
    with pytest.raises(ClientError):
        handler(s3_event("in-bucket", "missing.json"), FakeLambdaContext())
```

---

### 2.4 Testing SQS & SNS

---

**Q: How do you test SQS queue processing and SNS notifications?**

```python
import json, pytest, boto3
from moto import mock_aws

@pytest.fixture
def aws_setup(monkeypatch):
    monkeypatch.setenv("AWS_ACCESS_KEY_ID", "testing")
    monkeypatch.setenv("AWS_SECRET_ACCESS_KEY", "testing")
    monkeypatch.setenv("AWS_DEFAULT_REGION", "us-east-1")
    with mock_aws():
        sqs = boto3.client("sqs", region_name="us-east-1")
        sns = boto3.client("sns", region_name="us-east-1")
        queue_url = sqs.create_queue(QueueName="orders")["QueueUrl"]
        topic_arn = sns.create_topic(Name="notifications")["TopicArn"]
        yield {"sqs": sqs, "sns": sns, "queue_url": queue_url, "topic_arn": topic_arn}

def test_message_processed_and_deleted(aws_setup):
    sqs, url = aws_setup["sqs"], aws_setup["queue_url"]
    sqs.send_message(QueueUrl=url, MessageBody=json.dumps({"id": "001"}))

    msgs = sqs.receive_message(QueueUrl=url, MaxNumberOfMessages=1).get("Messages", [])
    assert len(msgs) == 1
    assert json.loads(msgs[0]["Body"])["id"] == "001"

    sqs.delete_message(QueueUrl=url, ReceiptHandle=msgs[0]["ReceiptHandle"])
    remaining = sqs.receive_message(QueueUrl=url).get("Messages", [])
    assert remaining == []

def test_empty_queue_returns_nothing(aws_setup):
    msgs = aws_setup["sqs"].receive_message(
        QueueUrl=aws_setup["queue_url"]
    ).get("Messages", [])
    assert msgs == []
```

---

### 2.5 Testing DynamoDB

---

**Q: How do you test CRUD operations on DynamoDB?**

```python
import pytest, boto3
from moto import mock_aws
from decimal import Decimal
from boto3.dynamodb.conditions import Key

@pytest.fixture
def table(monkeypatch):
    monkeypatch.setenv("AWS_ACCESS_KEY_ID", "testing")
    monkeypatch.setenv("AWS_SECRET_ACCESS_KEY", "testing")
    monkeypatch.setenv("AWS_DEFAULT_REGION", "us-east-1")
    with mock_aws():
        db = boto3.resource("dynamodb", region_name="us-east-1")
        t = db.create_table(
            TableName="results",
            KeySchema=[
                {"AttributeName": "run_id",    "KeyType": "HASH"},
                {"AttributeName": "test_name", "KeyType": "RANGE"},
            ],
            AttributeDefinitions=[
                {"AttributeName": "run_id",    "AttributeType": "S"},
                {"AttributeName": "test_name", "AttributeType": "S"},
            ],
            BillingMode="PAY_PER_REQUEST"
        )
        yield t

def test_put_and_get(table):
    table.put_item(Item={"run_id": "r1", "test_name": "login", "status": "passed", "duration": Decimal("1.5")})
    item = table.get_item(Key={"run_id": "r1", "test_name": "login"}).get("Item")
    assert item["status"] == "passed"
    assert item["duration"] == Decimal("1.5")

def test_query_by_run(table):
    for t in ["a", "b", "c"]:
        table.put_item(Item={"run_id": "r2", "test_name": t, "status": "passed"})
    assert table.query(KeyConditionExpression=Key("run_id").eq("r2"))["Count"] == 3

def test_missing_item_returns_none(table):
    assert table.get_item(Key={"run_id": "x", "test_name": "y"}).get("Item") is None
```

---

### 2.6 LocalStack vs moto

---

**Q: What's the difference between LocalStack and moto — when to use each?**

| | moto | LocalStack |
|--|------|-----------|
| How it works | In-process mock of boto3 | Docker containers emulating AWS HTTP APIs |
| Start time | Instant | ~10-30s |
| Speed | ★★★★★ | ★★★ |
| Fidelity | ★★★ | ★★★★ |
| Cross-service flows | Limited | Full (S3 triggers Lambda etc.) |
| Docker required | No | Yes |
| Best for | Unit / fast CI | Integration / local dev |

```python
# LocalStack with testcontainers
from testcontainers.localstack import LocalStackContainer
import boto3, pytest

@pytest.fixture(scope="session")
def s3_ls():
    with LocalStackContainer(image="localstack/localstack:3") as ls:
        client = boto3.client("s3", endpoint_url=ls.get_url(),
                              aws_access_key_id="test",
                              aws_secret_access_key="test",
                              region_name="us-east-1")
        client.create_bucket(Bucket="integration-bucket")
        yield client

def test_full_round_trip(s3_ls):
    s3_ls.put_object(Bucket="integration-bucket", Key="hello.txt", Body=b"world")
    assert s3_ls.get_object(Bucket="integration-bucket", Key="hello.txt")["Body"].read() == b"world"
```

---

### 2.7 IaC Testing (CDK / Terraform)

---

**Q: How do you test Infrastructure as Code?**

Three levels — mirrors the test pyramid:

1. **Static analysis** — `checkov`, `cfn-lint`. No deploy needed. Catches: missing encryption, open security groups, insecure IAM policies.
2. **CDK unit tests** — synthesize stack to CloudFormation template and assert on it. Fast, no deploy.
3. **Integration** — deploy to sandbox AWS account, verify real behaviour.

```python
import aws_cdk as cdk
from aws_cdk import aws_s3 as s3, aws_lambda as lambda_, Duration
from aws_cdk.assertions import Template, Match
from constructs import Construct
import pytest

class ReportingStack(cdk.Stack):
    def __init__(self, scope: Construct, id: str, **kwargs):
        super().__init__(scope, id, **kwargs)
        bucket = s3.Bucket(self, "Reports", versioned=True,
                           encryption=s3.BucketEncryption.S3_MANAGED)
        fn = lambda_.Function(self, "Processor",
                              runtime=lambda_.Runtime.PYTHON_3_12,
                              code=lambda_.Code.from_asset("src"),
                              handler="handler.main",
                              timeout=Duration.seconds(30))
        bucket.grant_read_write(fn)

@pytest.fixture(scope="module")
def template():
    app = cdk.App()
    return Template.from_stack(ReportingStack(app, "Test"))

def test_bucket_versioning_enabled(template):
    template.has_resource_properties("AWS::S3::Bucket",
        {"VersioningConfiguration": {"Status": "Enabled"}})

def test_lambda_runtime(template):
    template.has_resource_properties("AWS::Lambda::Function", {"Runtime": "python3.12"})

def test_lambda_timeout(template):
    template.has_resource_properties("AWS::Lambda::Function", {"Timeout": 30})

def test_one_lambda_only(template):
    template.resource_count_is("AWS::Lambda::Function", 1)
```

---

## 3. ROS2 / Robotics Testing

---

### 3.1 ROS2 Architecture

---

**Q: Describe ROS2 communication mechanisms from a tester's perspective.**

| Mechanism | Pattern | Use case | Blocking? |
|-----------|---------|----------|---------|
| **Topic** | Pub/Sub async | Continuous data (sensor, camera, odometry) | No |
| **Service** | Request/Response sync | One-shot operations (reset, config) | Yes |
| **Action** | Goal/Feedback/Result | Long-running tasks (navigation, grasping) | Optional |
| **Parameter** | Get/Set | Runtime node configuration | Yes |

```
┌──────────────┐  Topic /scan     ┌──────────────┐
│  LiDAR Node  │ ───────────────► │ Safety Node  │
└──────────────┘  (LaserScan)     └──────┬───────┘
                                         │  Service /emergency_stop
                                         ▼
                                  ┌──────────────┐
                                  │ Motor Driver │
                                  └──────────────┘
```

---

### 3.2 pytest + rclpy Setup

---

**Q: How do you set up a ROS2 test environment with pytest?**

```python
# conftest.py
import pytest, rclpy
from rclpy.executors import SingleThreadedExecutor

@pytest.fixture(scope="session", autouse=True)
def rclpy_session():
    rclpy.init()
    yield
    rclpy.shutdown()

@pytest.fixture
def executor():
    exec_ = SingleThreadedExecutor()
    yield exec_
    exec_.shutdown()

@pytest.fixture
def ros_node(executor):
    node = rclpy.create_node("test_node")
    executor.add_node(node)
    yield node
    node.destroy_node()
```

---

### 3.3 Testing Publisher / Subscriber

---

**Q: How do you test ROS2 topic communication (Pub/Sub)?**

```python
import time, rclpy, pytest
from rclpy.node import Node
from geometry_msgs.msg import Twist

class VelocityPublisher(Node):
    def __init__(self):
        super().__init__("vel_pub")
        self.pub = self.create_publisher(Twist, "/cmd_vel", 10)

    def publish(self, linear_x: float, angular_z: float):
        msg = Twist()
        msg.linear.x = linear_x
        msg.angular.z = angular_z
        self.pub.publish(msg)

class MessageCollector(Node):
    def __init__(self, topic, msg_type):
        super().__init__("collector")
        self.received = []
        self.create_subscription(msg_type, topic, self.received.append, 10)

    def wait_for(self, count: int, timeout: float = 2.0) -> bool:
        deadline = time.time() + timeout
        while len(self.received) < count and time.time() < deadline:
            rclpy.spin_once(self, timeout_sec=0.1)
        return len(self.received) >= count

def test_velocity_published(rclpy_session):
    pub = VelocityPublisher()
    col = MessageCollector("/cmd_vel", Twist)
    pub.publish(1.0, 0.5)
    assert col.wait_for(1), "No message received"
    assert col.received[0].linear.x == pytest.approx(1.0)
    pub.destroy_node()
    col.destroy_node()
```

---

### 3.4 Testing Services

---

**Q: How do you test ROS2 services (request/response)?**

```python
from example_interfaces.srv import AddTwoInts
import rclpy, pytest

class AddService(rclpy.node.Node):
    def __init__(self):
        super().__init__("add_service")
        self.create_service(AddTwoInts, "add_two_ints",
                            lambda req, res: setattr(res, "sum", req.a + req.b) or res)

def call(node, client, a, b):
    req = AddTwoInts.Request()
    req.a, req.b = a, b
    fut = client.call_async(req)
    rclpy.spin_until_future_complete(node, fut, timeout_sec=5.0)
    assert fut.done(), "Service timed out"
    return fut.result().sum

def test_add(rclpy_session):
    srv  = AddService()
    node = rclpy.create_node("test_client")
    cli  = node.create_client(AddTwoInts, "add_two_ints")
    assert cli.wait_for_service(timeout_sec=5.0)
    assert call(node, cli, 3, 5) == 8
    assert call(node, cli, -1, 1) == 0
    srv.destroy_node()
    node.destroy_node()
```

---

### 3.5 Launch Testing

---

**Q: What is launch testing in ROS2?**

`launch_testing` starts a full multi-node system for integration tests — the E2E level of robotic testing.

```python
import pytest, launch, launch_ros.actions, launch_testing, launch_testing.actions

@pytest.mark.launch_test
def generate_test_description():
    node = launch_ros.actions.Node(package="my_robot", executable="controller", name="ctrl")
    return (
        launch.LaunchDescription([node, launch_testing.actions.ReadyToTest()]),
        {"ctrl": node}
    )

class TestSystem(launch_testing.TestCase):
    def test_controller_starts(self, proc_output):
        proc_output.assertWaitFor("controller ready", timeout=10)

@launch_testing.post_shutdown_test()
class TestShutdown(launch_testing.TestCase):
    def test_clean_exit(self, proc_output):
        launch_testing.asserts.assertExitCodes(proc_output)
```

---

### 3.6 Hardware Mocking

---

**Q: How do you test ROS2 nodes without physical hardware?**

Publish synthetic sensor data from a mock node instead of real hardware:

```python
from rclpy.node import Node
from sensor_msgs.msg import LaserScan
import math

class MockLidar(Node):
    def __init__(self, pattern: str = "clear"):
        super().__init__("mock_lidar")
        self.pub = self.create_publisher(LaserScan, "/scan", 10)
        self.pattern = pattern
        self.create_timer(0.1, self._publish)

    def _publish(self):
        msg = LaserScan()
        msg.angle_min, msg.angle_max = -math.pi, math.pi
        msg.angle_increment = math.pi / 180
        msg.range_min, msg.range_max = 0.1, 10.0
        if self.pattern == "clear":
            msg.ranges = [5.0] * 360
        elif self.pattern == "obstacle_front":
            msg.ranges = [5.0] * 360
            msg.ranges[0] = 0.3        # obstacle 0.3m ahead
        self.pub.publish(msg)
```

---

### 3.7 Bag Files in Tests

---

**Q: What are ROS2 bag files and how do you use them in tests?**

A bag file (`rosbag2`) is a recording of all ROS2 messages with timestamps. Play it back to test algorithms against real robot data without the physical robot.

```bash
ros2 bag record -o test_run_01 /scan /imu/data /cmd_vel /odom
ros2 bag play test_run_01 --rate 2.0
```

```python
import subprocess, time, rclpy
from nav_msgs.msg import Odometry

class OdomCollector(rclpy.node.Node):
    def __init__(self):
        super().__init__("odom_col")
        self.positions = []
        self.create_subscription(Odometry, "/odom",
            lambda m: self.positions.append((m.pose.pose.position.x, m.pose.pose.position.y)), 10)

def test_robot_moves_in_bag(rclpy_session, bag_path):
    proc = subprocess.Popen(["ros2", "bag", "play", bag_path, "--rate", "2.0"])
    col  = OdomCollector()
    deadline = time.time() + 10.0
    while time.time() < deadline:
        rclpy.spin_once(col, timeout_sec=0.1)
    proc.terminate()
    assert len(col.positions) > 10
    start, end = col.positions[0], col.positions[-1]
    dist = ((end[0]-start[0])**2 + (end[1]-start[1])**2) ** 0.5
    assert dist > 0.1, "Robot did not move"
    col.destroy_node()
```

---

## 4. Embedded Testing

---

### 4.1 MIL / SIL / HIL

---

**Q: What do MIL, SIL, HIL mean and how do they form a testing strategy?**

| Level | Full name | Runs on | What's mocked | Speed |
|-------|-----------|---------|--------------|-------|
| **MIL** | Model-in-the-Loop | PC / Simulink | Everything (HW + OS) | ★★★★★ |
| **SIL** | Software-in-the-Loop | PC | Hardware (HAL) | ★★★★ |
| **HIL** | Hardware-in-the-Loop | Real MCU/CPU | External environment | ★★ |

```
V-Model:
Requirements ─────────────────────► Acceptance (HIL)
  Architecture ─────────────────► Integration (SIL)
    Design ─────────────────────► Component (SIL)
      Code ───────────────────► Unit tests (MIL)
```

---

### 4.2 HAL Mocking

---

**Q: What is HAL and how do you mock it in Python (SIL)?**

HAL (Hardware Abstraction Layer) separates application logic from specific hardware. In SIL tests, replace the real HAL with a mock — run embedded code on a PC.

```python
# hal_interface.py
from abc import ABC, abstractmethod
from typing import Optional

class GpioHal(ABC):
    @abstractmethod
    def digital_write(self, pin: int, value: bool) -> None: ...
    @abstractmethod
    def digital_read(self, pin: int) -> bool: ...
    @abstractmethod
    def set_pin_mode(self, pin: int, mode: str) -> None: ...

class AdcHal(ABC):
    @abstractmethod
    def read_channel(self, channel: int) -> int: ...

# app.py
class TemperatureSensor:
    ADC_MAX = 4095
    VREF    = 3.3

    def __init__(self, adc: AdcHal, channel: int):
        self.adc, self.channel = adc, channel

    def read_celsius(self) -> float:
        raw     = self.adc.read_channel(self.channel)
        voltage = (raw / self.ADC_MAX) * self.VREF
        return voltage / 0.01   # LM35: 10mV/°C

class StatusLed:
    def __init__(self, gpio: GpioHal, pin: int):
        self.gpio, self.pin = gpio, pin
        self.gpio.set_pin_mode(pin, "OUTPUT")

    def turn_on(self):  self.gpio.digital_write(self.pin, True)
    def turn_off(self): self.gpio.digital_write(self.pin, False)
    def is_on(self) -> bool: return self.gpio.digital_read(self.pin)
```

```python
# test_embedded.py
import pytest
from unittest.mock import MagicMock
from hal_interface import GpioHal, AdcHal
from app import TemperatureSensor, StatusLed

@pytest.fixture
def mock_gpio():
    g = MagicMock(spec=GpioHal)
    g.digital_read.return_value = False
    return g

@pytest.fixture
def mock_adc():
    return MagicMock(spec=AdcHal)

def test_room_temperature(mock_adc):
    mock_adc.read_channel.return_value = 310   # ≈ 25°C
    assert 24.0 < TemperatureSensor(mock_adc, 0).read_celsius() < 26.0

def test_reads_correct_channel(mock_adc):
    mock_adc.read_channel.return_value = 0
    TemperatureSensor(mock_adc, channel=2).read_celsius()
    mock_adc.read_channel.assert_called_once_with(2)

def test_led_init_sets_output_mode(mock_gpio):
    StatusLed(mock_gpio, pin=17)
    mock_gpio.set_pin_mode.assert_called_once_with(17, "OUTPUT")

def test_led_turn_on(mock_gpio):
    led = StatusLed(mock_gpio, pin=17)
    led.turn_on()
    mock_gpio.digital_write.assert_called_with(17, True)
```

---

### 4.3 Protocol Testing (UART / SPI / I2C)

---

**Q: How do you test embedded communication protocols without hardware?**

Mock the serial/hardware interface and verify correct byte sequences.

```python
import struct, pytest
from unittest.mock import MagicMock
from hal_interface import UartHal

class Mpu6050Driver:
    ADDR             = 0x68
    REG_ACCEL_XOUT_H = 0x3B
    REG_PWR_MGMT_1   = 0x6B

    def __init__(self, uart: UartHal):
        self.uart = uart

    def wake_up(self) -> bool:
        cmd = bytes([self.ADDR, self.REG_PWR_MGMT_1, 0x00])
        return self.uart.write(cmd) == len(cmd)

    def read_acceleration(self) -> tuple[float, float, float]:
        self.uart.write(bytes([self.ADDR | 0x80, self.REG_ACCEL_XOUT_H, 6]))
        data = self.uart.read(6, timeout_ms=100)
        if data is None or len(data) < 6:
            raise IOError("Failed to read acceleration data")
        x, y, z = struct.unpack(">hhh", data)
        scale = 9.81 / 16384.0
        return x * scale, y * scale, z * scale

@pytest.fixture
def mock_uart():
    u = MagicMock(spec=UartHal)
    u.write.return_value = 3
    return u

def test_wake_up_correct_bytes(mock_uart):
    assert Mpu6050Driver(mock_uart).wake_up() is True
    mock_uart.write.assert_called_once_with(bytes([0x68, 0x6B, 0x00]))

def test_acceleration_parsed_correctly(mock_uart):
    mock_uart.read.return_value = struct.pack(">hhh", 8192, 0, 16384)
    ax, ay, az = Mpu6050Driver(mock_uart).read_acceleration()
    assert ax == pytest.approx(0.5 * 9.81, rel=0.01)
    assert ay == pytest.approx(0.0, abs=0.01)
    assert az == pytest.approx(1.0 * 9.81, rel=0.01)

def test_timeout_raises(mock_uart):
    mock_uart.read.return_value = None
    with pytest.raises(IOError, match="Failed to read"):
        Mpu6050Driver(mock_uart).read_acceleration()
```

---

### 4.4 Code Coverage for Embedded

---

**Q: How do you measure code coverage for embedded systems?**

For Python on embedded (MicroPython, RPi): standard `coverage.py`.
For C/C++ on MCU: `gcov`/`lcov` or commercial tools (VectorCAST, LDRA).
Safety-critical: MC/DC coverage required by DO-178C and ISO 26262.

```python
@pytest.mark.parametrize("obstacle, velocity, estop, expected", [
    # Each condition independently affects the outcome (MC/DC)
    (5.0,  0.0, True,  False),   # estop dominates
    (5.0,  0.0, False, True),    # estop=False → True (independent effect)
    (0.2,  0.5, False, False),   # close + fast → False
    (0.5,  0.5, False, True),    # distance independently matters
    (0.2,  0.05, False, True),   # velocity independently matters
])
def test_safe_to_move_mcdc(obstacle, velocity, estop, expected):
    assert check_safe_to_move(obstacle, velocity, estop) == expected
```

```bash
pytest --cov=src --cov-branch --cov-fail-under=95 --cov-report=html
```

---

### 4.5 HIL Framework with pyserial

---

**Q: How do you build a HIL test framework in Python?**

```python
import serial, time, pytest

class SerialHil:
    def __init__(self, port: str, baud: int = 115200):
        self.ser = serial.Serial(port, baud, timeout=1.0)
        time.sleep(0.1)

    def cmd(self, c: str) -> str:
        self.ser.write(f"{c}\n".encode())
        return self.ser.readline().decode().strip()

    def read_sensor(self, sid: int) -> float:
        return float(self.cmd(f"READ {sid}").split("=")[1])

    def set_output(self, oid: int, val: float) -> bool:
        return self.cmd(f"SET {oid} {val}") == "OK"

    def close(self): self.ser.close()

@pytest.fixture(scope="session")
def dut(request):
    port = request.config.getoption("--hil-port", default="/dev/ttyUSB0")
    hil = SerialHil(port)
    yield hil
    hil.close()

@pytest.mark.hil
def test_temperature_in_range(dut):
    assert 15.0 < dut.read_sensor(1) < 40.0

@pytest.mark.hil
@pytest.mark.parametrize("duty, expected_v", [(0.0, 0.0), (0.5, 1.65), (1.0, 3.3)])
def test_pwm_voltage(dut, duty, expected_v):
    dut.set_output(1, duty)
    time.sleep(0.1)
    assert abs(dut.read_sensor(2) - expected_v) < 0.1
```

```ini
# pytest.ini
[pytest]
markers =
    hil: requires physical hardware — use --hil-port=/dev/ttyUSB0
    sil: software-in-the-loop, runs on PC only
```

---

### 4.6 CAN Bus Testing

---

**Q: How do you test CAN Bus communication?**

```bash
sudo modprobe vcan && sudo ip link add dev vcan0 type vcan && sudo ip link set up vcan0
```

```python
import can, cantools, pytest, threading, time

@pytest.fixture(scope="session")
def bus():
    b = can.interface.Bus(channel="vcan0", bustype="socketcan")
    yield b
    b.shutdown()

class CanCollector:
    def __init__(self, bus, timeout=2.0):
        self.bus, self.messages, self._run, self.timeout = bus, [], False, timeout

    def __enter__(self):
        self._run = True
        self._t = threading.Thread(target=self._recv, daemon=True)
        self._t.start()
        return self

    def _recv(self):
        d = time.time() + self.timeout
        while self._run and time.time() < d:
            m = self.bus.recv(0.1)
            if m: self.messages.append(m)

    def __exit__(self, *_):
        self._run = False
        self._t.join(1.0)

def test_rpm_message_decoded(bus):
    db  = cantools.database.load_file("vehicle.dbc")
    msg = db.get_message_by_name("EngineStatus")
    data = msg.encode({"RPM": 2500})
    with CanCollector(bus) as col:
        bus.send(can.Message(arbitration_id=msg.frame_id, data=data))
    assert len(col.messages) > 0
    decoded = db.decode_message(col.messages[0].arbitration_id, col.messages[0].data)
    assert decoded["RPM"] == pytest.approx(2500, abs=10)
```

---

## 5. General SDET Concepts

---

### 5.1 Richardson Maturity Model

---

**Q: What is the Richardson Maturity Model and what are its levels?**

RMM grades how "truly RESTful" an API is. Described by Leonard Richardson, popularized by Martin Fowler.

| Level | Name | Description |
|-------|------|-------------|
| 0 | Swamp of POX | Single URI, single verb (usually POST). RPC over HTTP. SOAP lives here. |
| 1 | Resources | Multiple URIs (one per resource), still only POST. |
| 2 | HTTP Verbs | Correct GET/POST/PUT/PATCH/DELETE + meaningful HTTP status codes. |
| 3 | HATEOAS | Responses contain links telling the client what actions are available next. |

```json
// Level 3 — HATEOAS response
{
  "id": 42, "name": "Alice",
  "_links": {
    "self":   { "href": "/users/42" },
    "orders": { "href": "/users/42/orders" },
    "delete": { "href": "/users/42", "method": "DELETE" }
  }
}
```

**Gotcha:** Most production APIs reach Level 2 and call themselves "REST". True Level 3 is rare.

---

**Q: Why does RMM matter for SDETs?**

- **Audit API contracts** — if the API claims to be REST, verify it uses correct verbs and status codes.
- **Write better tests** — at Level 2 you can assert on HTTP status codes meaningfully. At Level 0 everything returns `200 OK` even on error.
- **Navigate HATEOAS APIs** in tests by following `_links` instead of hardcoding URLs.

```python
# Level 0 test smell — useless assertion
response = requests.post("/api", json={"action": "deleteUser", "id": 999})
assert response.status_code == 200   # always true at Level 0

# Level 2 — HTTP semantics carry meaning
response = requests.delete("/users/999")
assert response.status_code == 404
```

---

### 5.2 TAE v2 + TAS v1

---

**Q: What is ISTQB TAE v2 and what is gTAA?**

**TAE** = Certified Tester — Test Automation Engineer (ISTQB CT-TAE v2.0)

**gTAA** = generic Test Automation Architecture — central framework of TAE v2:

```
┌──────────────────────────────────────────┐
│         Test Generation Layer            │  ← BDD specs, Gherkin, design
├──────────────────────────────────────────┤
│         Test Definition Layer            │  ← test scripts, fixtures, data
├──────────────────────────────────────────┤
│         Test Execution Layer             │  ← runner, CI, parallelization
├──────────────────────────────────────────┤
│         Test Adaptation Layer            │  ← adapters to SUT (API, UI, DB)
└──────────────────────────────────────────┘
                    │
                    ▼
            SUT (System Under Test)
```

```python
# Adaptation layer example
class UserApiClient:
    def __init__(self, base_url: str, token: str):
        self.session = requests.Session()
        self.session.headers["Authorization"] = f"Bearer {token}"
        self.base_url = base_url

    def get_user(self, user_id: int) -> dict:
        r = self.session.get(f"{self.base_url}/users/{user_id}")
        r.raise_for_status()
        return r.json()

# Definition layer example
@pytest.fixture(scope="session")
def api_client(config):
    return UserApiClient(config.base_url, config.token)

@pytest.fixture
def existing_user(api_client):
    user = api_client.create_user({"name": "Test", "email": "t@t.com"})
    yield user
    api_client.delete_user(user["id"])
```

---

### 5.3 REST vs RESTful

---

**Q: What is the difference between REST and RESTful?**

| | REST | RESTful |
|--|------|---------|
| What it is | An architectural *style* (Roy Fielding, 2000) | An API that *conforms* to REST constraints |
| Nature | Theory / set of 6 constraints | Practice / implementation |

**The 6 REST constraints:**

1. **Client–Server** — UI and data separated; client doesn't care how server stores data.
2. **Stateless** — each request contains all info to process it; server stores no session state.
3. **Cacheable** — responses declare themselves cacheable or not (`Cache-Control` header).
4. **Uniform Interface** — URI identifies resources; HATEOAS; self-descriptive messages.
5. **Layered System** — client can't tell if it talks to server directly or through CDN/gateway.
6. **Code on Demand** *(optional)* — server can send executable code (JavaScript).

**Gotcha:** Most APIs violate Uniform Interface (no HATEOAS). Pure REST is an ideal; Level 2 RMM is the industry standard.

---

### 5.4 Design Patterns

---

**Q: Which design patterns matter most for SDETs?**

#### Page Object Model (POM)

```python
class LoginPage:
    URL = "/login"
    _username = (By.ID, "username")
    _password = (By.ID, "password")
    _submit   = (By.CSS_SELECTOR, "button[type='submit']")

    def __init__(self, driver):
        self.driver = driver
        self.wait = WebDriverWait(driver, 10)

    def open(self): self.driver.get(self.URL); return self

    def login(self, u, p):
        self.driver.find_element(*self._username).send_keys(u)
        self.driver.find_element(*self._password).send_keys(p)
        self.driver.find_element(*self._submit).click()

    @property
    def error(self): return self.wait.until(EC.visibility_of_element_located((By.CLASS_NAME, "error"))).text
```

#### Factory Pattern

```python
import factory
from faker import Faker
fake = Faker()

class UserFactory(factory.DictFactory):
    name  = factory.LazyFunction(fake.name)
    email = factory.LazyFunction(fake.email)
    role  = "viewer"

class AdminFactory(UserFactory):
    role = "admin"
```

#### Builder Pattern

```python
class SearchRequestBuilder:
    def __init__(self): self._q, self._page, self._filters = "", 1, {}

    def with_query(self, q):         self._q = q;       return self
    def with_page(self, p):          self._page = p;    return self
    def with_filter(self, k, v):     self._filters[k] = v; return self

    def build(self) -> dict:
        return {"query": self._q, "page": self._page, "filters": self._filters}

def test_search(api_client):
    payload = SearchRequestBuilder().with_query("laptop").with_filter("category", "electronics").build()
    assert api_client.post("/search", json=payload).status_code == 200
```

#### Strategy Pattern

```python
class AuthStrategy(ABC):
    @abstractmethod
    def apply(self, session: requests.Session) -> None: ...

class BearerTokenAuth(AuthStrategy):
    def __init__(self, token): self.token = token
    def apply(self, s): s.headers["Authorization"] = f"Bearer {self.token}"

class ApiClient:
    def __init__(self, base_url, auth: AuthStrategy):
        self.session = requests.Session()
        auth.apply(self.session)
```

#### Facade Pattern

```python
class OrderFacade:
    def __init__(self, api): self.api = api

    def place_order(self, product_id, qty) -> dict:
        user    = self.api.create_user(UserFactory())
        cart    = self.api.create_cart(user["id"])
        self.api.add_to_cart(cart["id"], product_id, qty)
        payment = self.api.create_payment(cart["id"])
        return self.api.checkout(cart["id"], payment["id"])
```

| Situation | Pattern |
|-----------|---------|
| UI tests with many locators | Page Object Model |
| Varied test data | Factory |
| Complex request construction | Builder |
| Different auth / environments | Strategy |
| Multi-step setup hides intent | Facade |

---

### 5.5 SOLID Principles

---

**Q: What is SOLID and why does it matter for SDETs?**

**S — Single Responsibility Principle**
```python
# Bad: one class does too much
class UserTestHelper:
    def create_via_api(self, data): ...
    def save_to_csv(self, user): ...
    def assert_valid(self, user): ...

# Good: each class has one responsibility
class UserApiClient:  def create(self, data): ...
class UserCsvExporter: def save(self, user, path): ...
class UserAssertions:  def assert_valid(self, user): ...
```

**O — Open/Closed Principle**
```python
class ReportFormatter(ABC):
    @abstractmethod
    def format(self, data: dict) -> str: ...

class HtmlFormatter(ReportFormatter):
    def format(self, data): return f"<html>{data}</html>"

class JsonFormatter(ReportFormatter):
    def format(self, data): return json.dumps(data)

# Add new format = new class, no existing code modified
```

**L — Liskov Substitution Principle**
```python
# Subclass must honour the parent's contract
class BrokenClient(ReadOnlyApiClient):
    def get(self, url):
        raise NotImplementedError("offline")  # breaks LSP — caller can't substitute
```

**I — Interface Segregation Principle**
```python
# Many small ABCs > one fat ABC
class BrowserEnv(ABC):
    @abstractmethod
    def start_browser(self): ...

class ApiEnv(ABC):
    @abstractmethod
    def start_api(self): ...

class ApiTestEnv(ApiEnv):           # implements only what it needs
    def start_api(self): ...
```

**D — Dependency Inversion Principle**
```python
class OrderService:
    def __init__(self, db: Database):   # depend on abstraction
        self.db = db

# In test — inject mock
service = OrderService(db=InMemoryDatabase())
```

---

### 5.6 OOP Principles

---

**Q: Describe the four pillars of OOP with Python examples.**

**1. Encapsulation**
```python
class ApiSession:
    def __init__(self, token):
        self._token = token             # protected by convention
        self.__session = requests.Session()  # name-mangled

    def get(self, url):
        self.__session.headers["Authorization"] = f"Bearer {self._token}"
        return self.__session.get(url).json()
```

**2. Abstraction**
```python
class TestClient(ABC):
    @abstractmethod
    def get(self, endpoint: str) -> dict: ...
    @abstractmethod
    def post(self, endpoint: str, payload: dict) -> dict: ...
```

**3. Inheritance**
```python
class BasePage:
    def __init__(self, driver): self.driver = driver

class LoginPage(BasePage):
    def open(self): self.driver.get("/login"); return self
```

**4. Polymorphism**
```python
class Notifier(ABC):
    @abstractmethod
    def send(self, msg: str) -> None: ...

class SlackNotifier(Notifier):
    def send(self, msg): slack.post(msg)

class MockNotifier(Notifier):
    def __init__(self): self.sent = []
    def send(self, msg): self.sent.append(msg)

def notify_all(notifiers: list[Notifier], msg: str):
    for n in notifiers: n.send(msg)  # polymorphic call
```

---

### 5.7 Framework Definition

---

**Q: What is a framework — two faces in SDET context?**

| Face | Meaning | Examples |
|------|---------|---------|
| **Tech** | The tool/library that runs tests | `pytest`, `Selenium`, `Playwright`, `Robot Framework` |
| **Implementation** | Conventions, structure, rules for writing tests | Folder structure, naming, data management, reporting |

**Framework vs Library:**

```python
# Framework (pytest) — IoC: it calls YOUR code
def test_login(): assert True       # pytest discovers and calls this

# Library (requests) — YOU call IT
response = requests.get("https://api.example.com/users")
```

---

### 5.8 Code Review

---

**Q: What do you look for in a code review of test code?**

1. **Naming tells a story:** `test_login_with_expired_password_prompts_reset()` not `test_login2()`
2. **No shared mutable state between tests** — each test must be fully isolated
3. **Fixtures over repeated setup** — extract 50-line setup blocks to fixtures
4. **Informative assertion messages:**
   ```python
   assert r.json()["status"] == "active", \
       f"Expected active, got {r.json()['status']!r}. Response: {r.json()}"
   ```
5. **Parametrize instead of copy-paste:**
   ```python
   @pytest.mark.parametrize("role,endpoint,ok", [
       ("admin",  "/admin", True),
       ("viewer", "/admin", False),
   ])
   def test_access(role, endpoint, ok, token_factory):
       r = client_for(role).get(endpoint)
       assert (r.status_code == 200) == ok
   ```
6. **`@skip` always has a ticket and date:** `@pytest.mark.skip(reason="PROJ-123: fix by 2025-07-01")`
7. **Linter must pass:** `ruff`, `black`, `mypy`

---

### 5.9 Software Design Principles — KISS, DRY, YAGNI

---

**Q: What are KISS, DRY, and YAGNI? How do they apply to test code?**

**KISS — Keep It Simple, Stupid**

> Prefer the simplest solution that works. Complexity is a liability.

```python
# BAD — over-engineered assertion factory
def assert_response(response, *, status=200, key=None, value=None, schema=None):
    assert response.status_code == status
    if key and value:
        assert response.json()[key] == value
    if schema:
        validate(response.json(), schema)

# GOOD — direct, readable, obvious
def test_create_user():
    r = requests.post("/users", json={"name": "Alice"})
    assert r.status_code == 201
    assert r.json()["name"] == "Alice"
```

**Drill-down:** Where does KISS conflict with DRY?  
→ When extracting a helper to remove duplication makes the test harder to understand on its own — KISS wins. Tests are documentation; clarity beats brevity.

---

**DRY — Don't Repeat Yourself**

> Every piece of knowledge should have a single, authoritative representation.

```python
# BAD — same setup repeated in every test
def test_admin_can_delete():
    token = requests.post("/auth", json={"user": "admin", "pass": "s3cr3t"}).json()["token"]
    r = requests.delete("/users/1", headers={"Authorization": f"Bearer {token}"})
    assert r.status_code == 204

def test_admin_can_list():
    token = requests.post("/auth", json={"user": "admin", "pass": "s3cr3t"}).json()["token"]
    r = requests.get("/users", headers={"Authorization": f"Bearer {token}"})
    assert r.status_code == 200

# GOOD — extract repeated setup into a fixture
@pytest.fixture
def admin_client():
    token = requests.post("/auth", json={"user": "admin", "pass": "s3cr3t"}).json()["token"]
    session = requests.Session()
    session.headers["Authorization"] = f"Bearer {token}"
    return session

def test_admin_can_delete(admin_client):
    assert admin_client.delete("/users/1").status_code == 204

def test_admin_can_list(admin_client):
    assert admin_client.get("/users").status_code == 200
```

**Gotcha:** DRY applies to *knowledge*, not just text. Two tests that look similar but verify different behaviors should NOT be merged — their similarity is accidental, not semantic.

---

**YAGNI — You Aren't Gonna Need It**

> Don't implement something until you actually need it.

```python
# BAD — adding "flexible" infrastructure for hypothetical future tests
class BaseApiTest:
    def __init__(self, base_url, timeout=30, retry=3, auth_type="bearer",
                 ssl_verify=True, proxy=None, custom_headers=None):
        ...  # 80 lines of "flexibility" nobody asked for

# GOOD — start with what you need now
@pytest.fixture
def api(base_url):
    session = requests.Session()
    session.base_url = base_url
    return session
```

**Drill-down:** How does YAGNI interact with SOLID's Open/Closed Principle?  
→ OCP says "design for extension." YAGNI says "don't add extension points speculatively." Resolution: add extension points *when the second use case arrives*, not before.

| Principle | Violation symptom |
|-----------|------------------|
| KISS | "No one can read this test without 30 min context" |
| DRY | Copy-paste of setup code across 10 test files |
| YAGNI | Unused parameters, dead branches, premature abstractions |

---

## 6. API Testing

---

### 6.1 HTTP Methods

---

**Q: Describe HTTP methods and their semantics.**

| Method | Idempotent | Safe | Use |
|--------|-----------|------|-----|
| **GET** | ✓ | ✓ | Retrieve resource |
| **POST** | ✗ | ✗ | Create new resource |
| **PUT** | ✓ | ✗ | Full replacement |
| **PATCH** | ✗ | ✗ | Partial update |
| **DELETE** | ✓ | ✗ | Remove resource |
| **HEAD** | ✓ | ✓ | Headers only (check existence) |
| **OPTIONS** | ✓ | ✓ | What methods does endpoint support? |

**Safe** = does not change server state.
**Idempotent** = calling N times = same result as calling once.

```python
def test_get_is_idempotent(api_client):
    r1, r2 = api_client.get("/users/1"), api_client.get("/users/1")
    assert r1.json() == r2.json()

def test_delete_idempotent(api_client, existing_user):
    uid = existing_user["id"]
    assert api_client.delete(f"/users/{uid}").status_code == 204
    assert api_client.delete(f"/users/{uid}").status_code in (204, 404)

def test_post_not_idempotent(api_client):
    r1 = api_client.post("/users", json={"name": "Alice"})
    r2 = api_client.post("/users", json={"name": "Alice"})
    assert r1.json()["id"] != r2.json()["id"]
```

---

### 6.2 HTTP Status Codes

---

**Q: What are the HTTP status code groups and the most important codes?**

| Range | Category | Key codes |
|-------|----------|-----------|
| **2xx** | Success | 200 OK, 201 Created, 204 No Content |
| **3xx** | Redirection | 301 Permanent, 302 Temporary, 304 Not Modified |
| **4xx** | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 405 Method Not Allowed, 409 Conflict, 422 Unprocessable Entity, 429 Too Many Requests |
| **5xx** | Server Error | 500 Internal Server Error, 502 Bad Gateway, 503 Unavailable |

**Commonly confused pairs:**

| Pair | Difference |
|------|-----------|
| **401 vs 403** | 401 = not authenticated (bad/missing token). 403 = authenticated but not authorized. |
| **400 vs 422** | 400 = malformed request format. 422 = format OK, business validation failed. |
| **301 vs 302** | 301 = permanent redirect (update bookmarks). 302 = temporary. |

```python
def test_unauthenticated_returns_401(no_auth_client):
    assert no_auth_client.get("/users/1").status_code == 401

def test_unauthorized_returns_403(viewer_client):
    assert viewer_client.delete("/admin/settings").status_code == 403

def test_validation_error_returns_422(api_client):
    r = api_client.post("/users", json={"name": ""})
    assert r.status_code == 422
    assert any(e["field"] == "name" for e in r.json()["detail"])
```

---

### 6.3 Authentication

---

**Q: Describe the main API authentication types and how to test them.**

**Basic Authentication**
```python
from requests.auth import HTTPBasicAuth

def test_basic_auth_success():
    r = requests.get(URL, auth=HTTPBasicAuth("user", "pass"))
    assert r.status_code == 200

def test_basic_auth_wrong_password():
    r = requests.get(URL, auth=HTTPBasicAuth("user", "wrong"))
    assert r.status_code == 401
```

**OAuth 2.0 (Client Credentials)**
```python
@pytest.fixture(scope="session")
def oauth_token():
    r = requests.post("https://auth.example.com/oauth/token", data={
        "grant_type": "client_credentials",
        "client_id": os.environ["CLIENT_ID"],
        "client_secret": os.environ["CLIENT_SECRET"],
    })
    return r.json()["access_token"]

def test_oauth_endpoint(oauth_token):
    r = requests.get(URL, headers={"Authorization": f"Bearer {oauth_token}"})
    assert r.status_code == 200
```

**JWT**
```python
import jwt, time

def test_jwt_claims(api_client):
    token = api_client.post("/auth/login", json={"user": "alice", "pass": "s"}).json()["token"]
    payload = jwt.decode(token, options={"verify_signature": False})
    assert payload["sub"] == "alice"
    assert payload["exp"] > time.time()

def test_expired_jwt_rejected():
    expired = jwt.encode({"sub": "alice", "exp": int(time.time()) - 3600}, "secret", "HS256")
    r = requests.get(URL, headers={"Authorization": f"Bearer {expired}"})
    assert r.status_code == 401
```

**JWT structure:** `Header.Payload.Signature` — Base64URL encoded, NOT encrypted. Anyone can decode the payload — never store sensitive data in it.

---

### 6.4 PUT vs PATCH

---

**Q: What is the difference between PUT and PATCH?**

| | PUT | PATCH |
|--|-----|-------|
| Operation | Full resource replacement | Partial update |
| Idempotent | ✓ | ✗ (usually) |
| Missing fields | Set to null/default | Left unchanged |

```python
# Existing: {"id":1, "name":"Alice", "email":"a@co.com", "role":"admin"}

def test_put_nulls_missing_fields(api_client):
    r = api_client.put("/users/1", json={"name": "Alice Updated"})
    assert r.json()["email"] is None       # not sent → nulled

def test_patch_preserves_missing_fields(api_client):
    r = api_client.patch("/users/1", json={"name": "Alice Updated"})
    assert r.json()["email"] == "a@co.com" # unchanged
    assert r.json()["role"] == "admin"     # unchanged
```

---

### 6.5 PathParam vs QueryParam

---

**Q: What is the difference between path parameters and query parameters?**

| | Path Parameter | Query Parameter |
|--|---------------|-----------------|
| Location | In the path: `/users/{id}` | After `?`: `/users?role=admin` |
| Required | Yes | Usually not |
| Purpose | Identify a specific resource | Filter, sort, paginate |

```python
# PathParam — identifies specific resource
def test_path_param(api_client):
    r = api_client.get("/users/42")
    assert r.json()["id"] == 42

# QueryParam — filters a collection
def test_query_param_filter(api_client):
    r = api_client.get("/users", params={"role": "admin"})
    assert all(u["role"] == "admin" for u in r.json()["items"])

def test_query_param_pagination(api_client):
    r = api_client.get("/users", params={"page": 2, "pageSize": 10})
    assert len(r.json()["items"]) <= 10
    assert r.json()["page"] == 2
```

---

## 7. Selenium

---

### 7.1 Locators

---

**Q: What locators does Selenium offer and which to prefer?**

| Locator | Priority | Notes |
|---------|----------|-------|
| `By.ID` | ★★★★★ | Always prefer — unique by definition |
| `By.CSS_SELECTOR` | ★★★★ | Flexible, readable, fast |
| `By.NAME` | ★★★★ | Good for form fields |
| `By.XPATH` | ★★★ | Only when CSS can't do it (e.g., parent traversal, text search) |
| `By.CLASS_NAME` | ★★ | Classes change frequently |
| `By.TAG_NAME` | ★ | Too generic |
| `By.LINK_TEXT` | ★★ | Only for `<a>` tags |

**Best practice — `data-testid`:**
```html
<button data-testid="login-submit">Login</button>
```
```python
driver.find_element(By.CSS_SELECTOR, "[data-testid='login-submit']")
# Decoupled from CSS class changes — survives UI redesigns
```

**XPath — when justified:**
```python
# Navigate UP the DOM (CSS can't do this)
driver.find_element(By.XPATH, "//label[text()='Password']/following-sibling::input")

# Search by text
driver.find_element(By.XPATH, "//button[contains(text(),'Submit')]")
```

**Avoid:** Absolute XPaths like `/html/body/div[3]/div/form/div[2]/button` — breaks on any structural change.

---

### 7.2 Waits

---

**Q: What are the three wait types in Selenium?**

**1. Implicit Wait** — global timeout applied to every `find_element`. Set once.

```python
driver.implicitly_wait(10)
element = driver.find_element(By.ID, "username")  # waits up to 10s
```

**Gotcha:** Never mix Implicit + Explicit Wait — produces unpredictable compound timeouts.

**2. Explicit Wait** — wait for a specific condition on a specific element.

```python
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

wait = WebDriverWait(driver, timeout=10)

wait.until(EC.element_to_be_clickable((By.ID, "submit"))).click()
wait.until(EC.invisibility_of_element_located((By.ID, "spinner")))
wait.until(EC.url_contains("/dashboard"))
wait.until(EC.text_to_be_present_in_element((By.CLASS_NAME, "status"), "Success"))
```

| `expected_conditions` | Use when |
|----------------------|---------|
| `visibility_of_element_located` | Element must be visible (not just in DOM) |
| `element_to_be_clickable` | Visible + enabled |
| `presence_of_element_located` | In DOM (may be hidden) |
| `invisibility_of_element_located` | Wait for loader/modal to disappear |
| `staleness_of` | Wait for page refresh to complete |

**3. Fluent Wait** — Explicit Wait with custom poll interval and ignored exceptions.

```python
from selenium.common.exceptions import NoSuchElementException, StaleElementReferenceException

wait = WebDriverWait(driver, timeout=30, poll_frequency=0.5,
                     ignored_exceptions=[NoSuchElementException, StaleElementReferenceException])
element = wait.until(lambda d: d.find_element(By.ID, "dynamic-content"))
```

---

## 8. Git

---

**Q: Describe key Git commands and when to use them.**

| Command | What it does |
|---------|-------------|
| `git pull` | Fetch + merge from remote (= fetch + merge) |
| `git push` | Send local commits to remote |
| `git commit` | Save staged changes to local repo |
| `git merge` | Combine two branches, creates merge commit |
| `git rebase` | Move commits onto tip of another branch — linear history |
| `git cherry-pick` | Copy a specific commit (by SHA) to current branch |

---

**Q: What is the difference between merge and rebase?**

```
Initial state:
  main: A - B - C
  feat:     D - E

git merge feat → main:
  A - B - C - M    (M = merge commit, history preserved)
              ↑
          D - E

git rebase main (on feat), then fast-forward:
  A - B - C - D' - E'   (linear history, SHAs rewritten)
```

**When merge:** Shared/public branches (main, develop) — preserve history.
**When rebase:** Local feature branches before PR — clean linear history.

**Never rebase a branch others have pulled** — rewrites SHA history, breaks teammates' repos.

---

**Q: What is git cherry-pick?**

Copies one specific commit to the current branch without merging the whole history.

```bash
git log --oneline
# abc1234 fix: critical auth bypass

git checkout hotfix/v2.1
git cherry-pick abc1234         # copy that fix here
git cherry-pick -x abc1234      # adds "(cherry picked from commit abc1234)" to message
git cherry-pick --no-commit abc1234  # stage only, don't auto-commit
```

**Use cases:**
1. Backport a bugfix to an older release branch
2. Hotfix done on a feature branch → port to main without the whole feature

---

**Q: What is the difference between `git fetch` and `git pull`?**

```
git fetch  =  download remote changes → do NOT auto-merge
git pull   =  git fetch + git merge (or rebase, depending on config)
```

```bash
# Download changes from all branches without touching working directory
git fetch origin

# Inspect what was downloaded (remote-tracking branch)
git log origin/main --oneline --graph

# Compare your branch against the remote
git diff main origin/main

# Now integrate consciously
git merge origin/main   # or: git rebase origin/main
```

**When to use `fetch` instead of `pull`:**
- You want to *review* remote changes before integrating
- CI/CD: update tag/branch info without affecting local code
- You want to update remote-tracking refs without touching the working tree

```bash
# Typical fetch-first workflow
git fetch origin                     # get everything
git log HEAD..origin/main --oneline  # what am I missing?
git diff HEAD origin/main            # what changed?
git rebase origin/main               # deliberate integration
```

**Gotcha:** `git pull --rebase` is safer than plain `git pull` on feature branches — avoids spurious merge commits and keeps history linear.

---

**Q: Additional Git questions:**

- *What does `git stash` do?*
  ```bash
  git stash        # shelve WIP changes, clean working directory
  git stash pop    # restore last stash
  git stash list   # show all stashes
  ```

- *How do you undo the last commit without losing changes?*
  ```bash
  git reset --soft HEAD~1    # undo commit, keep changes staged
  git reset --mixed HEAD~1   # undo commit + unstage, keep files
  git reset --hard HEAD~1    # DESTRUCTIVE — undo commit + delete changes
  ```

---

## 9. Python OOP — Deep Dive

---

**Q: Overriding vs Overloading in Python**

**Overriding** — subclass redefines a method from the parent class (same signature, different implementation).

```python
class BaseReporter:
    def report(self, results): return f"Total: {len(results)}"

class HtmlReporter(BaseReporter):
    def report(self, results):      # overrides parent
        return "<table>" + "".join(f"<tr><td>{r}</td></tr>" for r in results) + "</table>"

class VerboseReporter(BaseReporter):
    def report(self, results):
        return super().report(results) + f"\nDetails: {results}"  # call parent
```

**Overloading** — not native in Python. Use default args or `singledispatch`:

```python
from functools import singledispatch

@singledispatch
def process(data): raise NotImplementedError

@process.register(str)
def _(data: str): return data.upper()

@process.register(list)
def _(data: list): return [x * 2 for x in data]
```

---

**Q: Abstract Class vs Protocol in Python**

**ABC** — requires explicit inheritance:

```python
from abc import ABC, abstractmethod

class Database(ABC):
    @abstractmethod
    def save(self, record: dict) -> None: ...
    @abstractmethod
    def find(self, id: int) -> dict: ...

    def exists(self, id: int) -> bool:   # concrete method
        return self.find(id) is not None

# Database() → TypeError: Can't instantiate abstract class
```

**Protocol** — structural typing, no inheritance needed (Python 3.8+):

```python
from typing import Protocol

class Saveable(Protocol):
    def save(self, record: dict) -> None: ...
    def find(self, id: int) -> dict: ...

class RedisDB:                # does NOT inherit Saveable
    def save(self, record): ...
    def find(self, id): ...

def store(db: Saveable, data):   # type checker accepts RedisDB here
    db.save(data)
```

| | ABC | Protocol |
|--|-----|---------|
| Requires inheritance | Yes | No |
| Runtime `isinstance` | Yes | Optional (`@runtime_checkable`) |
| Philosophy | Nominal typing | Structural / duck typing |

---

**Q: Lambda — what is it and when to use it?**

```python
square  = lambda x: x ** 2
evens   = list(filter(lambda x: x % 2 == 0, range(10)))
users.sort(key=lambda u: u["age"])

@pytest.mark.parametrize("fn, expected", [
    (lambda x: x.upper(), "HELLO"),
    (lambda x: x[::-1],   "olleh"),
])
def test_transforms(fn, expected):
    assert fn("hello") == expected
```

**Gotcha — late binding closure:**
```python
funcs = [lambda: i for i in range(5)]
funcs[0]()   # → 4, not 0!

funcs = [lambda i=i: i for i in range(5)]   # fix: capture by default arg
funcs[0]()   # → 0 ✓
```

---

**Q: Access modifiers in Python**

| Convention | Example | Meaning |
|-----------|---------|---------|
| No prefix | `self.name` | Public |
| Single `_` | `self._token` | Protected by convention — don't use from outside |
| Double `__` | `self.__secret` | Name-mangled — hard (but not impossible) to access externally |

```python
class ApiClient:
    def __init__(self):
        self.base_url = "https://api.example.com"  # public
        self._session = requests.Session()          # "protected"
        self.__api_key = "secret"                  # mangled to _ApiClient__api_key

client = ApiClient()
client._ApiClient__api_key   # still accessible, Python never truly hides
```

---

**Q: Key Python collections — when to use which?**

| Collection | Ordered | Mutable | Duplicates | Best for |
|-----------|---------|---------|-----------|---------|
| `list` | ✓ | ✓ | ✓ | General sequence, stack |
| `tuple` | ✓ | ✗ | ✓ | Immutable data, multiple return values |
| `set` | ✗ | ✓ | ✗ | Unique elements, set operations |
| `dict` | ✓ (3.7+) | ✓ | Keys: ✗ | Key→value mapping |
| `deque` | ✓ | ✓ | ✓ | Fast append/pop from both ends |

```python
from collections import Counter, defaultdict, deque

Counter(["pass","fail","pass"])      # Counter({'pass': 2, 'fail': 1})
defaultdict(list)["new"].append(1)   # no KeyError
queue = deque([1,2,3]); queue.appendleft(0)  # O(1)
```

---

**Q: `==` vs `is` in Python**

| Operator | Checks | |
|----------|--------|--|
| `==` | Value equality (calls `__eq__`) | `[1,2] == [1,2]` → `True` |
| `is` | Identity (same object in memory) | `[1,2] is [1,2]` → `False` |

```python
a = [1, 2]; b = [1, 2]; c = a
a == b  # True  — same values
a is b  # False — different objects
a is c  # True  — same object

# CORRECT: use 'is' only for None checks
if result is None: ...

# NEVER: use == for None — works but non-idiomatic
if result == None: ...
```

**Custom `__eq__`:**
```python
class TestResult:
    def __init__(self, name, status):
        self.name, self.status = name, status

    def __eq__(self, other):
        if not isinstance(other, TestResult): return NotImplemented
        return self.name == other.name and self.status == other.status

    def __hash__(self):
        return hash((self.name, self.status))  # always implement if __eq__ defined
```

