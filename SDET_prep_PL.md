# Przygotowanie do Rekrutacji SDET — Poziom MID

---

## Spis treści
1. [Testowanie w Pythonie](#1-testowanie-w-pythonie)
2. [Testowanie AWS / Cloud](#2-testowanie-aws--cloud)
3. [Testowanie ROS2 / Robotyka](#3-testowanie-ros2--robotyka)
4. [Testowanie Embedded](#4-testowanie-embedded)
5. [Ogólne Koncepty SDET](#5-ogólne-koncepty-sdet)
6. [Testowanie API](#6-testowanie-api)
7. [Selenium](#7-selenium)
8. [Git](#8-git)
9. [Python OOP — Pytania Zaawansowane](#9-python-oop--pytania-zaawansowane)

---

## 1. Testowanie w Pythonie

### 1.1 Poziomy testów wg ISTQB

**Pytanie:** Wymień i opisz poziomy testów wg ISTQB. Kiedy stosować każdy z nich?

**Odpowiedź:**

| Poziom | Cel | Kto testuje |
|--------|-----|-------------|
| Jednostkowe (Unit) | Pojedyncza funkcja/klasa w izolacji | Developer / SDET |
| Integracyjne | Komunikacja między komponentami | SDET |
| Systemowe | Cały system end-to-end | SDET / QA |
| Akceptacyjne (UAT) | Wymagania biznesowe spełnione | Klient / PO |

**Pytania drążące:**
- Jak odróżnić test integracyjny od systemowego? → Integracyjny sprawdza interfejsy między modułami, systemowy sprawdza cały przepływ danych przez system.
- Dlaczego testy jednostkowe nie wystarczą? → Komponenty mogą działać osobno, ale błędnie komunikować się ze sobą.

```python
# Test jednostkowy — izolacja z mockiem
def test_calculate_discount_unit():
    service = DiscountService()
    assert service.calculate(100, 0.10) == 90.0

# Test integracyjny — prawdziwe warstwy, mock zewnętrzny
def test_order_integration(db_session, mock_payment_gateway):
    repo = OrderRepository(db_session)
    svc = OrderService(repo, mock_payment_gateway)
    order = svc.place_order(user_id=1, items=[{"id": "A", "qty": 2}])
    assert order.status == "confirmed"
    assert repo.find(order.id) is not None
```

---

### 1.2 Typy testów

**Pytanie:** Jakie typy testów wyróżnia ISTQB?

**Odpowiedź:**

- **Funkcjonalne** — czy system robi to, co powinien (happy path, edge cases)
- **Niefunkcjonalne** — wydajność, bezpieczeństwo, użyteczność, niezawodność
- **Biało-skrzynkowe** — testy ze znajomością kodu źródłowego (pokrycie instrukcji, decyzji)
- **Związane ze zmianami** — retesty (sprawdzenie czy bug jest naprawiony) + regresja (czy nic nie zostało zepsute)

**Pytanie drążące:** Czym różni się retest od testu regresyjnego?  
→ Retest sprawdza konkretny naprawiony defekt. Regresja sprawdza, czy naprawa nie zepsuła innych funkcji.

---

### 1.3 Techniki testowania czarno-skrzynkowego

**Pytanie:** Opisz techniki testowania czarno-skrzynkowego z przykładami.

**Odpowiedź:**

**1. Klasy równoważności (Equivalence Partitioning)**  
Dzielimy dane wejściowe na grupy, gdzie każda wartość z grupy powinna zachowywać się tak samo.

```python
# Walidacja wieku: <0 (niepoprawne), 0-17 (niepełnoletni), 18-120 (dorosły), >120 (niepoprawne)
@pytest.mark.parametrize("age,expected", [
    (-1, "invalid"),    # klasa: poniżej zera
    (10, "minor"),      # klasa: niepełnoletni
    (25, "adult"),      # klasa: dorosły
    (150, "invalid"),   # klasa: powyżej max
])
def test_age_validation(age, expected):
    assert classify_age(age) == expected
```

**2. Analiza wartości brzegowych (Boundary Value Analysis)**  
Testujemy wartości na granicach klas — tam najczęściej występują błędy.

```python
@pytest.mark.parametrize("age,expected", [
    (0, "minor"),    # dolna granica klasy minor
    (17, "minor"),   # górna granica klasy minor
    (18, "adult"),   # dolna granica klasy adult
    (120, "adult"),  # górna granica klasy adult
])
def test_age_boundaries(age, expected):
    assert classify_age(age) == expected
```

**3. Tablica decyzyjna (Decision Table Testing)**  
Stosujemy, gdy wyjście zależy od kombinacji wielu warunków.

| Zalogowany | Premium | Rabat |
|-----------|---------|-------|
| Tak | Tak | 20% |
| Tak | Nie | 5% |
| Nie | - | 0% |

**4. Przejścia między stanami (State Transition Testing)**  
Modelujemy system jako maszynę stanów i testujemy przejścia.

```python
# Stany: nowy → potwierdzony → wysłany → dostarczony
def test_order_state_transitions(order_service):
    order = order_service.create()
    assert order.status == "new"
    order_service.confirm(order.id)
    assert order.status == "confirmed"
    order_service.ship(order.id)
    assert order.status == "shipped"
```

---

### 1.4 Piramida testów

**Pytanie:** Co to jest piramida testów i dlaczego ma taką formę?

**Odpowiedź:**

```
        /\
       /E2E\          ← Mało, wolne, drogie, kruche
      /------\
     /Integracja\     ← Średnio, sprawdzają kontrakt między warstwami
    /------------\
   / Unit         \   ← Dużo, szybkie, tanie, izolowane
  /______________  \
```

- **Unit** — izolacja z mockami, milisekundy
- **Integracja** — prawdziwa baza / serwisy, sekundy
- **E2E** — przeglądarka/UI, minuty

**Gotcha:** Odwrócona piramida (lod z E2E) = duże koszty utrzymania, długi feedback loop, trudne debugowanie.

**Pytanie drążące:** Gdzie w piramidzie mieszczą się testy API?  
→ Na poziomie integracyjnym — są szybsze niż E2E (brak UI), ale testują prawdziwy kontrakt HTTP.

---

### 1.5 pytest — podstawy i zaawansowane funkcje

**Pytanie:** Co to są fixtures w pytest i jak działają zakresy?

**Odpowiedź:**

Fixture to funkcja dostarczająca zależności (dane, obiekty, połączenia) do testów. Pytest wstrzykuje je automatycznie przez nazwę parametru.

```python
import pytest

@pytest.fixture(scope="session")  # raz na całą sesję testową
def db_connection():
    conn = create_db_connection()
    yield conn
    conn.close()

@pytest.fixture(scope="function")  # domyślnie — raz na test
def user(db_connection):
    u = db_connection.insert_user(name="Alice")
    yield u
    db_connection.delete_user(u.id)

def test_user_name(user):
    assert user.name == "Alice"
```

**Zakresy fixture (od najszerszego do najwęższego):**  
`session` → `package` → `module` → `class` → `function`

**Pytanie drążące:** Kiedy `scope="session"` może spowodować problemy?  
→ Gdy testy modyfikują wspólny stan (np. baza danych). Rozwiązanie: transakcja z rollbackiem lub `scope="function"` z czyszczeniem.

---

### 1.6 Mockowanie

**Pytanie:** Czym różni się mock od stub i fake?

**Odpowiedź:**

| Pojęcie | Opis | Weryfikuje wywołania? |
|---------|------|----------------------|
| **Stub** | Zwraca predefiniowane dane | Nie |
| **Mock** | Stub + weryfikuje czy i jak był wywołany | Tak |
| **Fake** | Uproszczona implementacja (np. in-memory DB) | Nie |
| **Spy** | Prawdziwy obiekt z nagrywaniem wywołań | Opcjonalnie |

```python
from unittest.mock import patch, MagicMock

def test_send_email_calls_smtp(user_service):
    with patch("myapp.services.smtplib.SMTP") as mock_smtp:
        instance = mock_smtp.return_value.__enter__.return_value
        user_service.register("alice@example.com")
        instance.sendmail.assert_called_once()

def test_get_user_returns_stub_data():
    mock_repo = MagicMock()
    mock_repo.find.return_value = User(id=1, name="Alice")
    service = UserService(mock_repo)
    result = service.get(1)
    assert result.name == "Alice"
```

**Gotcha:** Patchowanie w złym miejscu — patchuj tam, gdzie obiekt jest *używany*, nie tam, gdzie jest *zdefiniowany*.

```python
# BŁĄD: patchuje oryginalne miejsce definicji
patch("os.path.exists")

# POPRAWNIE: patchuje w module, który go importuje
patch("mymodule.os.path.exists")
```

---

### 1.7 Pokrycie kodu

**Pytanie:** Co to jest pokrycie kodu i jakie są jego rodzaje?

**Odpowiedź:**

```bash
pytest --cov=src --cov-report=html --cov-fail-under=80
```

| Rodzaj | Co mierzy |
|--------|-----------|
| Pokrycie instrukcji (Statement) | Czy każda linia kodu była wykonana |
| Pokrycie gałęzi (Branch) | Czy każda gałąź if/else była przetestowana |
| MC/DC | Każdy warunek niezależnie wpływa na wynik (lotnictwo, embedded) |

**Gotcha:** 100% pokrycie nie gwarantuje braku błędów. Można pokryć kod bez żadnych asercji.

```python
def divide(a, b):
    return a / b

def test_divide():
    divide(10, 2)  # 100% pokrycia, ale brak assert — test nic nie sprawdza!
```

---

## 2. Testowanie AWS / Cloud

### 2.1 Strategie mockowania AWS

**Pytanie:** Jak testować kod korzystający z AWS bez dostępu do prawdziwej chmury?

**Odpowiedź:**

| Narzędzie | Poziom | Opis |
|-----------|--------|------|
| **moto** | Jednostkowy/Integracyjny | Mock AWS w pamięci, bardzo szybki |
| **LocalStack** | Integracyjny/E2E | Pełna emulacja AWS lokalnie (Docker) |
| **AWS SAM Local** | E2E | Lokalne uruchamianie Lambda/API Gateway |
| **CDK Assertions** | Jednostkowy (IaC) | Testowanie szablonów CloudFormation |

---

### 2.2 Testowanie S3 z moto

**Pytanie:** Jak napisać test dla kodu operującego na S3?

**Odpowiedź:**

```python
import boto3
import pytest
from moto import mock_aws

@pytest.fixture(autouse=True)
def aws_credentials(monkeypatch):
    monkeypatch.setenv("AWS_ACCESS_KEY_ID", "testing")
    monkeypatch.setenv("AWS_SECRET_ACCESS_KEY", "testing")
    monkeypatch.setenv("AWS_DEFAULT_REGION", "us-east-1")

@pytest.fixture
def s3_client():
    with mock_aws():
        client = boto3.client("s3", region_name="us-east-1")
        client.create_bucket(Bucket="test-bucket")
        yield client

def test_upload_file(s3_client):
    s3_client.put_object(Bucket="test-bucket", Key="hello.txt", Body=b"Hello World")
    response = s3_client.get_object(Bucket="test-bucket", Key="hello.txt")
    assert response["Body"].read() == b"Hello World"

def test_list_objects(s3_client):
    s3_client.put_object(Bucket="test-bucket", Key="a.txt", Body=b"A")
    s3_client.put_object(Bucket="test-bucket", Key="b.txt", Body=b"B")
    response = s3_client.list_objects_v2(Bucket="test-bucket")
    assert response["KeyCount"] == 2
```

**Gotcha:** Zawsze ustaw fałszywe zmienne środowiskowe. Bez nich moto może próbować połączyć się z prawdziwym AWS.

---

### 2.3 Testowanie Lambda

**Pytanie:** Jak testować funkcje AWS Lambda?

**Odpowiedź:**

Testuj logikę Lambda w trzech warstwach:

```python
# Warstwa 1: logika biznesowa — czyste testy jednostkowe
from myapp.lambda_handler import process_record

def test_process_record_valid():
    record = {"body": '{"user_id": 1, "amount": 100}'}
    result = process_record(record)
    assert result["status"] == "processed"

# Warstwa 2: handler z mockowanymi zależnościami
from unittest.mock import patch, MagicMock
from myapp.lambda_handler import handler

def test_handler_success():
    event = {"Records": [{"body": '{"user_id": 1, "amount": 50}'}]}
    with patch("myapp.lambda_handler.dynamodb") as mock_db:
        mock_db.Table.return_value.put_item.return_value = {}
        result = handler(event, context={})
    assert result["statusCode"] == 200

# Warstwa 3: integracja z moto
@mock_aws
def test_handler_integration():
    ddb = boto3.resource("dynamodb", region_name="us-east-1")
    ddb.create_table(
        TableName="orders",
        KeySchema=[{"AttributeName": "id", "KeyType": "HASH"}],
        AttributeDefinitions=[{"AttributeName": "id", "AttributeType": "S"}],
        BillingMode="PAY_PER_REQUEST",
    )
    event = {"Records": [{"body": '{"user_id": 1, "amount": 50}'}]}
    result = handler(event, {})
    assert result["statusCode"] == 200
    items = ddb.Table("orders").scan()["Items"]
    assert len(items) == 1
```

---

### 2.4 Testowanie SQS i SNS

**Pytanie:** Jak testować przepływ SQS → Lambda?

**Odpowiedź:**

```python
import json
import boto3
from moto import mock_aws

@mock_aws
def test_sqs_message_processing():
    sqs = boto3.client("sqs", region_name="us-east-1")
    queue_url = sqs.create_queue(QueueName="test-queue")["QueueUrl"]

    # Wyślij wiadomość
    sqs.send_message(
        QueueUrl=queue_url,
        MessageBody=json.dumps({"order_id": "abc-123", "total": 250.00})
    )

    # Odbierz i sprawdź
    messages = sqs.receive_message(QueueUrl=queue_url, MaxNumberOfMessages=1)["Messages"]
    assert len(messages) == 1
    body = json.loads(messages[0]["Body"])
    assert body["order_id"] == "abc-123"

@mock_aws
def test_sns_fanout():
    sns = boto3.client("sns", region_name="us-east-1")
    sqs = boto3.client("sqs", region_name="us-east-1")

    topic_arn = sns.create_topic(Name="order-events")["TopicArn"]
    queue_url = sqs.create_queue(QueueName="subscriber")["QueueUrl"]
    queue_arn = sqs.get_queue_attributes(
        QueueUrl=queue_url, AttributeNames=["QueueArn"]
    )["Attributes"]["QueueArn"]

    sns.subscribe(TopicArn=topic_arn, Protocol="sqs", Endpoint=queue_arn)
    sns.publish(TopicArn=topic_arn, Message="test event")

    messages = sqs.receive_message(QueueUrl=queue_url)["Messages"]
    assert len(messages) == 1
```

---

### 2.5 Testowanie DynamoDB

**Pytanie:** Jak testować operacje na DynamoDB?

**Odpowiedź:**

```python
import boto3
import pytest
from moto import mock_aws

@pytest.fixture
def orders_table():
    with mock_aws():
        ddb = boto3.resource("dynamodb", region_name="us-east-1")
        table = ddb.create_table(
            TableName="orders",
            KeySchema=[
                {"AttributeName": "pk", "KeyType": "HASH"},
                {"AttributeName": "sk", "KeyType": "RANGE"},
            ],
            AttributeDefinitions=[
                {"AttributeName": "pk", "AttributeType": "S"},
                {"AttributeName": "sk", "AttributeType": "S"},
            ],
            BillingMode="PAY_PER_REQUEST",
        )
        yield table

def test_put_and_get_item(orders_table):
    orders_table.put_item(Item={"pk": "USER#1", "sk": "ORDER#001", "total": 150})
    response = orders_table.get_item(Key={"pk": "USER#1", "sk": "ORDER#001"})
    assert response["Item"]["total"] == 150

def test_query_by_pk(orders_table):
    from boto3.dynamodb.conditions import Key
    orders_table.put_item(Item={"pk": "USER#1", "sk": "ORDER#001", "total": 100})
    orders_table.put_item(Item={"pk": "USER#1", "sk": "ORDER#002", "total": 200})
    result = orders_table.query(KeyConditionExpression=Key("pk").eq("USER#1"))
    assert result["Count"] == 2
```

---

### 2.6 Testowanie infrastruktury jako kodu (CDK Assertions)

**Pytanie:** Jak testować szablony CDK/CloudFormation bez deploymentu?

**Odpowiedź:**

```python
import aws_cdk as cdk
from aws_cdk.assertions import Template, Match
from myapp.cdk_stack import MyLambdaStack

def test_lambda_created_with_correct_runtime():
    app = cdk.App()
    stack = MyLambdaStack(app, "TestStack")
    template = Template.from_stack(stack)

    template.has_resource_properties("AWS::Lambda::Function", {
        "Runtime": "python3.12",
        "Timeout": 30,
    })

def test_s3_bucket_has_versioning():
    app = cdk.App()
    stack = MyLambdaStack(app, "TestStack")
    template = Template.from_stack(stack)

    template.has_resource_properties("AWS::S3::Bucket", {
        "VersioningConfiguration": {"Status": "Enabled"}
    })

def test_lambda_count():
    app = cdk.App()
    stack = MyLambdaStack(app, "TestStack")
    template = Template.from_stack(stack)
    template.resource_count_is("AWS::Lambda::Function", 2)
```

---

## 3. Testowanie ROS2 / Robotyka

### 3.1 Architektura ROS2

**Pytanie:** Opisz kluczowe koncepty ROS2 i jak wpływają na strategię testowania.

**Odpowiedź:**

| Koncepcja | Opis | Wpływ na testy |
|-----------|------|----------------|
| **Node** | Proces wykonujący obliczenia | Jednostka testowania |
| **Topic** | Kanał komunikacji pub/sub | Sprawdzamy wiadomości |
| **Service** | Komunikacja request/response | Sprawdzamy odpowiedź |
| **Action** | Długotrwałe zadania z feedbackiem | Sprawdzamy feedback + wynik |
| **Parameter** | Konfiguracja noda | Sprawdzamy zachowanie przy różnych wartościach |

```
Publisher Node  →  /topic  →  Subscriber Node
Client Node     ↔  /service  ↔  Server Node
Action Client   ↔  /action   ↔  Action Server (+ cel + feedback + wynik)
```

---

### 3.2 Testy jednostkowe noda ROS2

**Pytanie:** Jak testować node ROS2 w izolacji?

**Odpowiedź:**

```python
import pytest
import rclpy
from rclpy.node import Node
from std_msgs.msg import Float64

@pytest.fixture(scope="session")
def rclpy_init():
    rclpy.init()
    yield
    rclpy.shutdown()

@pytest.fixture
def temperature_node(rclpy_init):
    node = TemperatureMonitorNode()
    yield node
    node.destroy_node()

def test_node_name(temperature_node):
    assert temperature_node.get_name() == "temperature_monitor"

def test_default_parameter(temperature_node):
    threshold = temperature_node.get_parameter("threshold").value
    assert threshold == 80.0

def test_alarm_triggered_above_threshold(temperature_node):
    temperature_node.process_temperature(85.0)
    assert temperature_node.alarm_active is True

def test_no_alarm_below_threshold(temperature_node):
    temperature_node.process_temperature(75.0)
    assert temperature_node.alarm_active is False
```

---

### 3.3 Testowanie komunikacji Pub/Sub

**Pytanie:** Jak testować przepływ wiadomości między nodami?

**Odpowiedź:**

```python
import time
import rclpy
from rclpy.node import Node
from std_msgs.msg import String

class MessageCollector(Node):
    def __init__(self, topic: str, msg_type):
        super().__init__("test_collector")
        self.received = []
        self.create_subscription(msg_type, topic, self.received.append, 10)

    def wait_for(self, count: int, timeout: float = 2.0) -> bool:
        deadline = time.time() + timeout
        while len(self.received) < count and time.time() < deadline:
            rclpy.spin_once(self, timeout_sec=0.1)
        return len(self.received) >= count

def test_publisher_sends_messages(rclpy_init):
    publisher_node = SensorPublisher()
    collector = MessageCollector("/sensor_data", String)

    # Wyzwól publikację
    publisher_node.publish_reading(42.5)

    assert collector.wait_for(1), "Brak wiadomości w czasie"
    assert "42.5" in collector.received[0].data

    publisher_node.destroy_node()
    collector.destroy_node()
```

---

### 3.4 Testowanie serwisów ROS2

**Pytanie:** Jak testować serwisy ROS2 (request/response)?

**Odpowiedź:**

```python
import pytest
from rclpy.node import Node
from example_interfaces.srv import AddTwoInts

def test_add_two_ints_service(rclpy_init):
    service_node = AddTwoIntsServer()
    client_node = Node("test_client")
    client = client_node.create_client(AddTwoInts, "add_two_ints")

    assert client.wait_for_service(timeout_sec=2.0), "Serwis niedostępny"

    request = AddTwoInts.Request()
    request.a = 7
    request.b = 3

    future = client.call_async(request)
    rclpy.spin_until_future_complete(client_node, future, timeout_sec=2.0)

    assert future.result().sum == 10

    client_node.destroy_node()
    service_node.destroy_node()
```

---

### 3.5 Launch Testing (testy wielonodowe)

**Pytanie:** Jak testować scenariusze angażujące wiele nodów?

**Odpowiedź:**

```python
import launch
import launch_ros.actions
import launch_testing
import launch_testing.actions
import pytest

@pytest.fixture
def launch_description():
    sensor_node = launch_ros.actions.Node(
        package="my_robot",
        executable="sensor_node",
        name="sensor",
    )
    processor_node = launch_ros.actions.Node(
        package="my_robot",
        executable="data_processor",
        name="processor",
    )
    return launch.LaunchDescription([
        sensor_node,
        processor_node,
        launch_testing.actions.ReadyToTest(),
    ])

def test_pipeline_end_to_end(launch_service, proc_output, launch_description):
    collector = MessageCollector("/processed_data", Float64)
    # Poczekaj na przetworzone dane z całego pipeline'u
    assert collector.wait_for(5, timeout=5.0)
    values = [msg.data for msg in collector.received]
    assert all(0 <= v <= 100 for v in values)
```

---

### 3.6 Mockowanie sprzętu w ROS2

**Pytanie:** Jak izolować testy od prawdziwego sprzętu robota?

**Odpowiedź:**

```python
# Zamiast prawdziwego węzła kamery — node symulujący
class MockCameraNode(Node):
    def __init__(self, images: list):
        super().__init__("mock_camera")
        self._images = iter(images)
        self._pub = self.create_publisher(Image, "/camera/image_raw", 10)
        self.create_timer(0.1, self._publish_next)

    def _publish_next(self):
        try:
            self._pub.publish(next(self._images))
        except StopIteration:
            pass

def test_object_detection_with_mock_camera(rclpy_init):
    test_images = [create_test_image_with_object("car"), create_blank_image()]
    camera = MockCameraNode(test_images)
    detector = ObjectDetectionNode()
    results_collector = MessageCollector("/detections", Detection)

    assert results_collector.wait_for(1, timeout=3.0)
    assert results_collector.received[0].class_name == "car"

    camera.destroy_node()
    detector.destroy_node()
```

---

### 3.7 Testowanie z plikami bag

**Pytanie:** Do czego służą pliki bag w ROS2 i jak ich używać w testach?

**Odpowiedź:**

Pliki bag to nagrania wiadomości ROS2 — idealne do odtwarzania rzeczywistych scenariuszy.

```python
# Nagraj dane ze sprzętu
# ros2 bag record /sensor_data /camera/image_raw -o test_scenario_1

# Odtwórz w teście
import subprocess
import time

def test_with_bag_file(rclpy_init, tmp_path):
    collector = MessageCollector("/processed_output", String)
    processor = DataProcessorNode()

    bag_process = subprocess.Popen([
        "ros2", "bag", "play", "test_data/scenario_1.bag"
    ])

    time.sleep(3.0)  # Czas na odtworzenie
    bag_process.terminate()

    assert len(collector.received) > 0
    assert all(msg.data != "error" for msg in collector.received)

    processor.destroy_node()
```

---

## 4. Testowanie Embedded

### 4.1 Hierarchia testów MIL/SIL/HIL

**Pytanie:** Wyjaśnij model V i hierarchię MIL/SIL/HIL.

**Odpowiedź:**

```
Wymagania          ←→   Testy akceptacyjne
  Architektura     ←→   Testy integracyjne  
    Projekt        ←→   Testy systemowe
      Implementacja ←→  Testy jednostkowe

MIL (Model-in-the-Loop):  Algorytm w Simulink — brak kodu
SIL (Software-in-the-Loop): Skompilowany kod na PC — szybkie, CI-friendly
HIL (Hardware-in-the-Loop): Kod na prawdziwym MCU + symulowane I/O
```

| Poziom | Szybkość | Koszt | Wykrywa |
|--------|----------|-------|---------|
| MIL | Najszybszy | Najniższy | Błędy algorytmu |
| SIL | Szybki | Niski | Błędy kodu, integracji warstw |
| HIL | Wolny | Wysoki | Problemy z timingiem, sprzętem |

---

### 4.2 Abstrakcja HAL (Hardware Abstraction Layer)

**Pytanie:** Jak projektować kod embedded z myślą o testowalności?

**Odpowiedź:**

Kluczem jest HAL — warstwa abstrakcji, którą możemy podmienić na mock podczas testów SIL.

```python
from abc import ABC, abstractmethod

# Interfejs HAL — kontrakt sprzętowy
class GpioHal(ABC):
    @abstractmethod
    def digital_write(self, pin: int, value: bool) -> None: ...
    
    @abstractmethod
    def digital_read(self, pin: int) -> bool: ...

# Produkcyjna implementacja — prawdziwy sprzęt
class RaspberryPiGpio(GpioHal):
    def digital_write(self, pin: int, value: bool) -> None:
        import RPi.GPIO as GPIO
        GPIO.output(pin, GPIO.HIGH if value else GPIO.LOW)
    
    def digital_read(self, pin: int) -> bool:
        import RPi.GPIO as GPIO
        return GPIO.input(pin) == GPIO.HIGH

# Mock do testów SIL
class MockGpio(GpioHal):
    def __init__(self):
        self._pins: dict[int, bool] = {}
        self.write_calls: list[tuple[int, bool]] = []
    
    def digital_write(self, pin: int, value: bool) -> None:
        self._pins[pin] = value
        self.write_calls.append((pin, value))
    
    def digital_read(self, pin: int) -> bool:
        return self._pins.get(pin, False)

# Logika aplikacji — zależy od abstrakcji, nie implementacji
class LedController:
    LED_PIN = 5
    
    def __init__(self, gpio: GpioHal):
        self._gpio = gpio
    
    def turn_on(self) -> None:
        self._gpio.digital_write(self.LED_PIN, True)
    
    def is_on(self) -> bool:
        return self._gpio.digital_read(self.LED_PIN)

# Test SIL — bez prawdziwego sprzętu
def test_led_turns_on():
    mock_gpio = MockGpio()
    controller = LedController(mock_gpio)
    controller.turn_on()
    assert controller.is_on() is True
    assert mock_gpio.write_calls == [(5, True)]
```

---

### 4.3 Testowanie protokołów szeregowych (UART/SPI/I2C)

**Pytanie:** Jak testować komunikację przez protokoły szeregowe?

**Odpowiedź:**

```python
from unittest.mock import patch, MagicMock
import serial

class UartSensor:
    COMMAND_READ = b"\x01\x02\x03"
    
    def __init__(self, port: str, baud: int = 115200):
        self._serial = serial.Serial(port, baud, timeout=1.0)
    
    def read_temperature(self) -> float:
        self._serial.write(self.COMMAND_READ)
        response = self._serial.read(4)
        if len(response) != 4:
            raise TimeoutError("Brak odpowiedzi czujnika")
        return int.from_bytes(response[:2], "big") / 100.0

def test_read_temperature_parses_response():
    with patch("serial.Serial") as mock_serial_cls:
        mock_port = MagicMock()
        # 2500 = 25.00°C
        mock_port.read.return_value = b"\x09\xC4\x00\x00"
        mock_serial_cls.return_value = mock_port
        
        sensor = UartSensor("/dev/ttyUSB0")
        temp = sensor.read_temperature()
        
        assert temp == pytest.approx(25.0, abs=0.1)
        mock_port.write.assert_called_once_with(UartSensor.COMMAND_READ)

def test_timeout_raises_error():
    with patch("serial.Serial") as mock_serial_cls:
        mock_port = MagicMock()
        mock_port.read.return_value = b""  # timeout — brak danych
        mock_serial_cls.return_value = mock_port
        
        sensor = UartSensor("/dev/ttyUSB0")
        with pytest.raises(TimeoutError):
            sensor.read_temperature()
```

---

### 4.4 Framework HIL z pyserial

**Pytanie:** Jak zbudować prosty framework HIL do testowania firmware?

**Odpowiedź:**

```python
import serial
import time
import struct
from dataclasses import dataclass

@dataclass
class FirmwareCommand:
    CMD_PING = 0x01
    CMD_SET_OUTPUT = 0x02
    CMD_READ_ADC = 0x03

class HilFramework:
    def __init__(self, port: str, baud: int = 115200):
        self._ser = serial.Serial(port, baud, timeout=2.0)
        time.sleep(0.5)  # Czas na reset MCU po otwarciu portu
    
    def ping(self) -> bool:
        self._ser.write(bytes([FirmwareCommand.CMD_PING]))
        response = self._ser.read(1)
        return response == b"\xAA"
    
    def set_output(self, channel: int, value: float) -> None:
        payload = struct.pack(">Bf", channel, value)
        self._ser.write(bytes([FirmwareCommand.CMD_SET_OUTPUT]) + payload)
    
    def read_adc(self, channel: int) -> float:
        self._ser.write(bytes([FirmwareCommand.CMD_READ_ADC, channel]))
        data = self._ser.read(4)
        return struct.unpack(">f", data)[0]
    
    def close(self):
        self._ser.close()

# Testy HIL — wymagają podłączonego MCU
@pytest.mark.hil
class TestFirmwareHil:
    @pytest.fixture
    def hil(self):
        framework = HilFramework("/dev/ttyUSB0")
        assert framework.ping(), "Firmware nie odpowiada"
        yield framework
        framework.close()
    
    def test_ping_responds(self, hil):
        assert hil.ping() is True
    
    def test_adc_reads_in_range(self, hil):
        voltage = hil.read_adc(channel=0)
        assert 0.0 <= voltage <= 3.3
```

---

### 4.5 CAN Bus

**Pytanie:** Jak testować komunikację CAN Bus?

**Odpowiedź:**

```python
import can

# Produkcyjna implementacja
class CanInterface:
    def __init__(self, channel: str = "can0"):
        self._bus = can.interface.Bus(channel=channel, bustype="socketcan")
    
    def send(self, arbitration_id: int, data: bytes) -> None:
        msg = can.Message(arbitration_id=arbitration_id, data=data, is_extended_id=False)
        self._bus.send(msg)
    
    def receive(self, timeout: float = 1.0) -> can.Message | None:
        return self._bus.recv(timeout=timeout)

# Mock do testów jednostkowych
class MockCanBus:
    def __init__(self):
        self.sent_messages: list[can.Message] = []
        self._incoming: list[can.Message] = []
    
    def send(self, msg: can.Message) -> None:
        self.sent_messages.append(msg)
    
    def recv(self, timeout: float = 1.0) -> can.Message | None:
        return self._incoming.pop(0) if self._incoming else None
    
    def inject(self, arbitration_id: int, data: bytes) -> None:
        self._incoming.append(
            can.Message(arbitration_id=arbitration_id, data=data)
        )

def test_engine_speed_message():
    mock_bus = MockCanBus()
    engine_ctrl = EngineController(mock_bus)
    engine_ctrl.set_rpm(3000)
    
    assert len(mock_bus.sent_messages) == 1
    msg = mock_bus.sent_messages[0]
    assert msg.arbitration_id == 0x0C0
    rpm_from_msg = int.from_bytes(msg.data[:2], "big") * 0.25
    assert rpm_from_msg == pytest.approx(3000, abs=1)
```

---

### 4.6 Pokrycie kodu MC/DC

**Pytanie:** Co to jest pokrycie MC/DC i dlaczego jest wymagane w systemach krytycznych?

**Odpowiedź:**

**MC/DC (Modified Condition/Decision Coverage)** — każdy warunek w decyzji musi niezależnie wpływać na jej wynik.

Wymagane przez: DO-178C (lotnictwo), ISO 26262 ASIL-D (automotive).

```python
# Warunek: arm if altitude > 100 AND speed > 50 AND safety_switch is True
def should_arm_system(altitude: float, speed: float, safety_switch: bool) -> bool:
    return altitude > 100 and speed > 50 and safety_switch

# Dla MC/DC: każdy warunek musi determinować wynik niezależnie
# Tabela testów MC/DC:
# alt>100  spd>50  safety  wynik   który warunek testujemy
#   T        T       T       T     |
#   F        T       T       F     | alt jest decydujący
#   T        F       T       F     | speed jest decydujący
#   T        T       F       F     | safety jest decydujący

@pytest.mark.parametrize("altitude,speed,safety,expected,comment", [
    (150, 60, True,  True,  "Wszystkie warunki spełnione"),
    (50,  60, True,  False, "Altitude decyduje — False"),
    (150, 30, True,  False, "Speed decyduje — False"),
    (150, 60, False, False, "Safety decyduje — False"),
])
def test_arm_system_mc_dc(altitude, speed, safety, expected, comment):
    assert should_arm_system(altitude, speed, safety) == expected, comment
```

---

## 5. Ogólne Koncepty SDET

### 5.1 Richardson Maturity Model

**Pytanie:** Co to jest Richardson Maturity Model? Opisz poziomy.

**Odpowiedź:**

Model oceny dojrzałości API REST (0-3), gdzie 3 oznacza w pełni RESTful API.

| Poziom | Nazwa | Opis |
|--------|-------|------|
| 0 | The Swamp of POX | Jeden endpoint, wszystko przez POST |
| 1 | Zasoby | Osobne URI dla każdego zasobu |
| 2 | Verby HTTP | Właściwe HTTP methods (GET/POST/PUT/DELETE) |
| 3 | Hypermedia (HATEOAS) | Odpowiedź zawiera linki do możliwych akcji |

```json
// Poziom 3 — HATEOAS: klient nie musi znać URL-i z góry
{
  "order_id": "123",
  "status": "confirmed",
  "_links": {
    "self": {"href": "/orders/123"},
    "cancel": {"href": "/orders/123/cancel", "method": "DELETE"},
    "payment": {"href": "/orders/123/payment", "method": "POST"}
  }
}
```

---

### 5.2 TAE v2 i Architektura Systemu Testów

**Pytanie:** Co to jest TAE v2 i gTAA?

**Odpowiedź:**

**TAE (Test Automation Engineer)** — sylwetka kompetencyjna wg ISTQB.  
**gTAA (Generic Test Automation Architecture)** — referencyjna architektura systemu automatyzacji.

Warstwy gTAA:

```
┌─────────────────────────────────────┐
│  Warstwa Definicji Testów           │  ← Skrypty, dane, logi
├─────────────────────────────────────┤
│  Warstwa Wykonawcza                 │  ← pytest, runner, scheduler
├─────────────────────────────────────┤
│  Warstwa Adaptacji                  │  ← Page Objects, API klienty, mocki
├─────────────────────────────────────┤
│  System Under Test (SUT)            │  ← Testowana aplikacja
└─────────────────────────────────────┘
```

**Gotcha:** Dobrze zbudowana architektura pozwala na wymianę dowolnej warstwy bez zmian w pozostałych.

---

### 5.3 REST — zasady i ograniczenia

**Pytanie:** Co odróżnia REST od "zwykłego" HTTP API?

**Odpowiedź:**

REST (Representational State Transfer) to styl architektoniczny z 6 ograniczeniami:

1. **Klient-Serwer** — separacja odpowiedzialności
2. **Bezstanowość** — każde żądanie zawiera wszystkie potrzebne informacje
3. **Buforowalność** — odpowiedzi mogą być cachowane
4. **Jednolity interfejs** — spójne URI, standardowe metody HTTP
5. **System warstwowy** — klient nie wie, ile warstw jest między nim a serwerem
6. **Kod na żądanie** *(opcjonalne)* — serwer może przesłać wykonywalny kod

**Pytanie drążące:** Co to znaczy, że REST jest "bezstanowy"?  
→ Serwer nie przechowuje stanu sesji klienta. Każde żądanie musi zawierać token autoryzacji, parametry itp.

---

### 5.4 Wzorce projektowe

**Pytanie:** Opisz wzorce projektowe używane w automatyzacji testów.

**Odpowiedź:**

**Page Object Model (POM)**

```python
class LoginPage:
    URL = "/login"
    
    def __init__(self, driver):
        self._driver = driver
    
    def navigate(self):
        self._driver.get(self.URL)
        return self
    
    def login(self, username: str, password: str) -> "DashboardPage":
        self._driver.find_element(By.ID, "username").send_keys(username)
        self._driver.find_element(By.ID, "password").send_keys(password)
        self._driver.find_element(By.ID, "submit").click()
        return DashboardPage(self._driver)

def test_successful_login(driver):
    dashboard = LoginPage(driver).navigate().login("admin", "secret")
    assert dashboard.welcome_message() == "Witaj, admin"
```

**Factory**

```python
from abc import ABC, abstractmethod

class TestDriver(ABC):
    @abstractmethod
    def get(self, url: str): ...

class DriverFactory:
    @staticmethod
    def create(browser: str) -> TestDriver:
        drivers = {
            "chrome": ChromeDriver,
            "firefox": FirefoxDriver,
            "headless": HeadlessChromeDriver,
        }
        if browser not in drivers:
            raise ValueError(f"Nieznana przeglądarka: {browser}")
        return drivers[browser]()
```

**Builder**

```python
class UserBuilder:
    def __init__(self):
        self._data = {"role": "user", "active": True}
    
    def with_role(self, role: str) -> "UserBuilder":
        self._data["role"] = role
        return self
    
    def inactive(self) -> "UserBuilder":
        self._data["active"] = False
        return self
    
    def with_email(self, email: str) -> "UserBuilder":
        self._data["email"] = email
        return self
    
    def build(self) -> dict:
        return self._data.copy()

# Czytelne tworzenie danych testowych
admin_user = UserBuilder().with_role("admin").with_email("admin@test.com").build()
inactive_user = UserBuilder().inactive().build()
```

**Singleton (obiekt drivera)**

```python
class DriverManager:
    _instance = None
    
    @classmethod
    def get_instance(cls):
        if cls._instance is None:
            cls._instance = cls._create_driver()
        return cls._instance
    
    @classmethod
    def _create_driver(cls):
        options = webdriver.ChromeOptions()
        return webdriver.Chrome(options=options)
    
    @classmethod
    def quit(cls):
        if cls._instance:
            cls._instance.quit()
            cls._instance = None
```

---

### 5.5 SOLID — Zasady Projektowania

**Pytanie:** Wyjaśnij zasady SOLID z przykładami w Pythonie.

**Odpowiedź:**

**S — Single Responsibility Principle**  
Klasa powinna mieć tylko jeden powód do zmiany.

```python
# ZŁAMANIE SRP
class ReportGenerator:
    def generate(self, data): ...
    def save_to_db(self, report): ...   # inna odpowiedzialność
    def send_email(self, report): ...   # jeszcze inna!

# POPRAWNIE
class ReportGenerator:
    def generate(self, data) -> Report: ...

class ReportRepository:
    def save(self, report: Report) -> None: ...

class ReportNotifier:
    def notify(self, report: Report) -> None: ...
```

**O — Open/Closed Principle**  
Otwarte na rozszerzanie, zamknięte na modyfikacje.

```python
from abc import ABC, abstractmethod

class Discount(ABC):
    @abstractmethod
    def apply(self, price: float) -> float: ...

class PercentageDiscount(Discount):
    def __init__(self, percent: float):
        self._percent = percent
    
    def apply(self, price: float) -> float:
        return price * (1 - self._percent / 100)

class FixedDiscount(Discount):
    def __init__(self, amount: float):
        self._amount = amount
    
    def apply(self, price: float) -> float:
        return max(0, price - self._amount)

# Nowy typ rabatu → nowa klasa, nie modyfikacja istniejących
```

**L — Liskov Substitution Principle**  
Podtypy muszą być zastępowalne przez typy bazowe.

```python
class Bird:
    def move(self) -> str:
        return "leci"

class Penguin(Bird):
    def move(self) -> str:
        return "pływa"  # OK — to jest ruch, tylko inny rodzaj

# BŁĄD LSP:
class FlyingBird(Bird):
    def fly(self) -> str:
        return "leci"

class Ostrich(FlyingBird):  # Struś nie lata! Złamanie LSP
    def fly(self):
        raise NotImplementedError("Strusie nie latają")
```

**I — Interface Segregation Principle**  
Klasy nie powinny być zmuszane do implementacji metod, których nie używają.

```python
# ZŁE: jeden duży interfejs
class Worker(ABC):
    @abstractmethod
    def work(self): ...
    @abstractmethod
    def eat(self): ...
    @abstractmethod
    def sleep(self): ...

# DOBRZE: małe, specyficzne interfejsy
class Workable(ABC):
    @abstractmethod
    def work(self): ...

class Eatable(ABC):
    @abstractmethod
    def eat(self): ...

class Robot(Workable):  # Robot pracuje, nie je
    def work(self):
        return "Robot pracuje"

class Human(Workable, Eatable):
    def work(self): return "Człowiek pracuje"
    def eat(self): return "Człowiek je"
```

**D — Dependency Inversion Principle**  
Moduły wysokiego poziomu nie powinny zależeć od niskopoziomowych.

```python
# ZŁE: bezpośrednia zależność od konkretnej implementacji
class OrderService:
    def __init__(self):
        self._db = MySQLDatabase()  # konkretna klasa!

# DOBRZE: zależność od abstrakcji
class Database(ABC):
    @abstractmethod
    def save(self, entity): ...

class OrderService:
    def __init__(self, db: Database):  # wstrzyknięcie abstrakcji
        self._db = db

# Użycie
service = OrderService(PostgresDatabase())
# lub
service = OrderService(InMemoryDatabase())  # do testów
```

---

### 5.6 Zasady OOP

**Pytanie:** Opisz cztery filary OOP z przykładami w Pythonie.

**Odpowiedź:**

**Enkapsulacja** — ukrywamy stan i udostępniamy kontrolowany dostęp.

```python
class BankAccount:
    def __init__(self, balance: float):
        self.__balance = balance  # prywatny atrybut
    
    @property
    def balance(self) -> float:
        return self.__balance
    
    def deposit(self, amount: float) -> None:
        if amount <= 0:
            raise ValueError("Kwota musi być dodatnia")
        self.__balance += amount
```

**Abstrakcja** — ukrywamy złożoność, ujawniamy tylko to, co istotne.

```python
from abc import ABC, abstractmethod

class PaymentProcessor(ABC):
    @abstractmethod
    def charge(self, amount: float, card_token: str) -> str: ...
    # Klient nie wie, jak działa Stripe/Braintree pod spodem

class StripeProcessor(PaymentProcessor):
    def charge(self, amount, card_token):
        # Złożona logika Stripe API ukryta tutaj
        return stripe.charge(amount, card_token)
```

**Dziedziczenie** — klasa potomna dziedziczy atrybuty i metody klasy bazowej.

```python
class Animal:
    def __init__(self, name: str):
        self.name = name
    
    def speak(self) -> str:
        return f"{self.name} wydaje dźwięk"

class Dog(Animal):
    def speak(self) -> str:
        return f"{self.name} szczeka: Hau!"

class Cat(Animal):
    def speak(self) -> str:
        return f"{self.name} miauczy: Miau!"
```

**Polimorfizm** — różne typy reagują na ten sam interfejs w różny sposób.

```python
animals: list[Animal] = [Dog("Rex"), Cat("Whiskers"), Animal("Stworzenie")]
for animal in animals:
    print(animal.speak())  # każdy woła inną implementację speak()
```

---

### 5.7 Co to jest framework testowy?

**Pytanie:** Czym jest framework testowy? Jakie są jego dwie "twarze"?

**Odpowiedź:**

**Technologia (narzędzie):** Selenium, Playwright, Cypress, pytest, Robot Framework — narzędzia do wykonywania testów.

**Implementacja (architektura):** Zestaw zasad, konwencji i infrastruktury określający:
- Jak zorganizowane są testy (struktura katalogów)
- Jak zarządzane są dane testowe
- Jak uruchamiane są testy (CI/CD, konfiguracja)
- Jak raportowane są wyniki
- Jak współdzielone są zasoby (fixtures, konfiguracja)

**Dobry framework:**
- Jest niezależny od narzędzia (można wymienić Selenium na Playwright bez przepisywania wszystkiego)
- Redukuje powielanie kodu
- Ułatwia utrzymanie przy zmieniających się wymaganiach

---

### 5.8 Code Review — na co zwracać uwagę?

**Pytanie:** Na co zwracasz uwagę podczas code review testów?

**Odpowiedź:**

1. **Konwencje nazewnicze** — czy nazwa testu mówi co testuje i czego oczekuje?
2. **Reużywalność** — czy ten kod można wyekstrahować do fixture/helpera?
3. **Asercje** — czy test ma asercje? Czy są precyzyjne?
4. **Izolacja** — czy testy nie zależą od kolejności wykonania?
5. **Czytelność** — czy test jest zrozumiały bez znajomości całej bazy kodu?
6. **Odpowiedzialność** — czy jeden test sprawdza jedną rzecz?
7. **Kruche selektory** — w UI testach: czy używa `data-testid` zamiast XPath?
8. **Hardkodowane wartości** — czy dane testowe są wyjaśnione?

```python
# ZŁE — niejasna nazwa, nieczytelny XPath, wiele asercji
def test_1():
    el = driver.find_element(By.XPATH, "//div[3]/span[2]/a")
    el.click()
    assert "success" in driver.current_url
    assert driver.find_element(By.ID, "msg").text != ""

# DOBRZE — opisowa nazwa, data-testid, jedna odpowiedzialność
def test_login_redirects_to_dashboard_on_valid_credentials(login_page):
    dashboard = login_page.login("valid_user", "valid_pass")
    assert dashboard.is_displayed()
```

---

### 5.9 Zasady projektowania — KISS, DRY, YAGNI

**Pytanie:** Co oznaczają KISS, DRY i YAGNI? Jak stosować je w kodzie testów?

---

**KISS — Keep It Simple, Stupid (Trzymaj to prosto)**

> Preferuj najprostsze rozwiązanie, które działa. Złożoność to dług techniczny.

```python
# ZŁE — przeładowana fabryka asercji
def assert_response(response, *, status=200, key=None, value=None, schema=None):
    assert response.status_code == status
    if key and value:
        assert response.json()[key] == value
    if schema:
        validate(response.json(), schema)

# DOBRZE — bezpośrednie, czytelne, oczywiste
def test_create_user():
    r = requests.post("/users", json={"name": "Alice"})
    assert r.status_code == 201
    assert r.json()["name"] == "Alice"
```

**Pytanie drążące:** Kiedy KISS wchodzi w konflikt z DRY?  
→ Gdy wyekstrahowanie helpera usuwa powielenie, ale utrudnia zrozumienie testu jako autonomicznego dokumentu. Testy to dokumentacja — czytelność jest ważniejsza niż skrótowość.

---

**DRY — Don't Repeat Yourself (Nie powtarzaj się)**

> Każda wiedza powinna mieć jedno, autorytatywne miejsce w kodzie.

```python
# ZŁE — ten sam setup kopiowany do każdego testu
def test_admin_moze_usunac():
    token = requests.post("/auth", json={"user": "admin", "pass": "s3cr3t"}).json()["token"]
    r = requests.delete("/users/1", headers={"Authorization": f"Bearer {token}"})
    assert r.status_code == 204

def test_admin_moze_listowac():
    token = requests.post("/auth", json={"user": "admin", "pass": "s3cr3t"}).json()["token"]
    r = requests.get("/users", headers={"Authorization": f"Bearer {token}"})
    assert r.status_code == 200

# DOBRZE — setup wyekstrahowany do fixture
@pytest.fixture
def admin_client():
    token = requests.post("/auth", json={"user": "admin", "pass": "s3cr3t"}).json()["token"]
    session = requests.Session()
    session.headers["Authorization"] = f"Bearer {token}"
    return session

def test_admin_moze_usunac(admin_client):
    assert admin_client.delete("/users/1").status_code == 204

def test_admin_moze_listowac(admin_client):
    assert admin_client.get("/users").status_code == 200
```

**Gotcha:** DRY dotyczy *wiedzy*, nie tylko tekstu. Dwa testy, które wyglądają podobnie, ale weryfikują różne zachowania, NIE powinny być łączone — ich podobieństwo jest przypadkowe, nie semantyczne.

---

**YAGNI — You Aren't Gonna Need It (Nie będziesz tego potrzebował)**

> Nie implementuj czegoś, dopóki naprawdę tego nie potrzebujesz.

```python
# ZŁE — "elastyczna" infrastruktura dla hipotetycznych przyszłych testów
class BaseApiTest:
    def __init__(self, base_url, timeout=30, retry=3, auth_type="bearer",
                 ssl_verify=True, proxy=None, custom_headers=None):
        ...  # 80 linii "elastyczności" o którą nikt nie prosił

# DOBRZE — zacznij od tego, czego potrzebujesz teraz
@pytest.fixture
def api(base_url):
    session = requests.Session()
    session.base_url = base_url
    return session
```

**Pytanie drążące:** Jak YAGNI odnosi się do zasady Open/Closed z SOLID?  
→ OCP mówi "projektuj pod rozszerzenie". YAGNI mówi "nie dodawaj punktów rozszerzenia spekulatywnie". Rozwiązanie: dodaj punkty rozszerzenia *gdy pojawi się drugi przypadek użycia*, nie wcześniej.

| Zasada | Objaw naruszenia |
|--------|-----------------|
| KISS | "Nikt nie rozumie tego testu bez 30 minut kontekstu" |
| DRY | Copy-paste setupu w 10 plikach testowych |
| YAGNI | Nieużywane parametry, martwe gałęzie, przedwczesne abstrakcje |

---

## 6. Testowanie API

### 6.1 Metody HTTP

**Pytanie:** Opisz metody HTTP. Które są bezpieczne i idempotentne?

**Odpowiedź:**

| Metoda | Opis | Bezpieczna | Idempotentna |
|--------|------|-----------|--------------|
| GET | Pobierz zasób | Tak | Tak |
| POST | Utwórz zasób | Nie | Nie |
| PUT | Zastąp zasób w całości | Nie | Tak |
| PATCH | Częściowo zaktualizuj | Nie | Nie* |
| DELETE | Usuń zasób | Nie | Tak |
| HEAD | Jak GET, ale bez body | Tak | Tak |
| OPTIONS | Sprawdź dozwolone metody | Tak | Tak |

*PATCH może być idempotentny zależnie od implementacji.

**Bezpieczna** = nie zmienia stanu serwera.  
**Idempotentna** = wielokrotne wywołanie daje ten sam efekt co jedno.

---

### 6.2 Kody statusu HTTP

**Pytanie:** Jakie znasz kody statusu HTTP? Kiedy użyć 400 vs 422?

**Odpowiedź:**

| Zakres | Znaczenie | Przykłady |
|--------|-----------|-----------|
| 2xx | Sukces | 200 OK, 201 Created, 204 No Content |
| 3xx | Przekierowanie | 301 Moved, 304 Not Modified |
| 4xx | Błąd klienta | 400, 401, 403, 404, 422, 429 |
| 5xx | Błąd serwera | 500 Internal Error, 503 Unavailable |

**400 Bad Request** — żądanie jest niepoprawne składniowo (np. niepoprawny JSON).  
**422 Unprocessable Entity** — składnia OK, ale dane są semantycznie niepoprawne (np. brakujące wymagane pole).

**401 vs 403:**
- **401 Unauthorized** — brak uwierzytelnienia (nie wiesz kim jesteś)
- **403 Forbidden** — uwierzytelniony, ale brak uprawnień (wiem kim jesteś, ale nie możesz)

```python
import requests

def test_api_returns_201_on_create():
    response = requests.post("/api/users", json={"name": "Alice", "email": "alice@test.com"})
    assert response.status_code == 201
    assert "id" in response.json()

def test_api_returns_422_on_missing_email():
    response = requests.post("/api/users", json={"name": "Alice"})  # brak email
    assert response.status_code == 422
    errors = response.json()["errors"]
    assert any(e["field"] == "email" for e in errors)

def test_api_returns_401_without_token():
    response = requests.get("/api/admin/users")  # brak tokenu
    assert response.status_code == 401

def test_api_returns_403_with_insufficient_role():
    headers = {"Authorization": "Bearer user_token_not_admin"}
    response = requests.get("/api/admin/users", headers=headers)
    assert response.status_code == 403
```

---

### 6.3 PUT vs PATCH

**Pytanie:** Jaka jest różnica między PUT a PATCH?

**Odpowiedź:**

**PUT** — zastępuje cały zasób. Pola pominięte w body są usuwane/resetowane.  
**PATCH** — częściowo aktualizuje zasób. Zmieniane są tylko podane pola.

```python
# PUT — zastępuje całość
# PUT /users/1 {"name": "Bob"}
# Wynik: {id: 1, name: "Bob", email: null}  ← email usunięty!

# PATCH — aktualizuje częściowo
# PATCH /users/1 {"name": "Bob"}
# Wynik: {id: 1, name: "Bob", email: "alice@test.com"}  ← email zachowany

def test_put_replaces_entire_resource(api_client, existing_user):
    response = api_client.put(f"/users/{existing_user.id}", json={"name": "Bob"})
    assert response.status_code == 200
    data = response.json()
    assert data["name"] == "Bob"
    assert data.get("email") is None  # nadpisane

def test_patch_updates_only_given_fields(api_client, existing_user):
    original_email = existing_user.email
    response = api_client.patch(f"/users/{existing_user.id}", json={"name": "Bob"})
    assert response.status_code == 200
    data = response.json()
    assert data["name"] == "Bob"
    assert data["email"] == original_email  # zachowane
```

---

### 6.4 PathParam vs QueryParam

**Pytanie:** Kiedy używać path parameters, a kiedy query parameters?

**Odpowiedź:**

**Path Parameter** — identyfikuje konkretny zasób (część ścieżki URI).  
**Query Parameter** — filtruje, sortuje lub konfiguruje sposób zwracania zasobów.

```
GET /users/123          ← PathParam: konkretny użytkownik
GET /users?role=admin   ← QueryParam: filtrowanie użytkowników

GET /orders/456/items   ← PathParam: pozycje konkretnego zamówienia
GET /orders?status=paid&sort=date&limit=20  ← QueryParam: filtrowanie + sortowanie
```

```python
import requests

def test_get_user_by_id_path_param():
    response = requests.get("/users/123")
    assert response.status_code == 200
    assert response.json()["id"] == 123

def test_filter_users_by_role_query_param():
    response = requests.get("/users", params={"role": "admin"})
    assert response.status_code == 200
    users = response.json()
    assert all(u["role"] == "admin" for u in users)

def test_pagination_query_params():
    response = requests.get("/products", params={"page": 2, "limit": 10})
    data = response.json()
    assert len(data["items"]) <= 10
    assert data["page"] == 2
```

---

### 6.5 Uwierzytelnianie

**Pytanie:** Opisz typy uwierzytelniania w API.

**Odpowiedź:**

**Basic Auth** — login i hasło zakodowane Base64. Proste, ale niezabezpieczone bez HTTPS.

```python
import requests
from requests.auth import HTTPBasicAuth

response = requests.get("/api/data", auth=HTTPBasicAuth("user", "pass"))
# lub krócej:
response = requests.get("/api/data", auth=("user", "pass"))
```

**JWT (JSON Web Token)** — trójczęściowy token (header.payload.signature). Bezstanowy.

```
eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoxfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
     Header (alg)        Payload (dane)           Signature (podpis)
```

```python
def test_jwt_protected_endpoint():
    # Zaloguj się i pobierz token
    login_resp = requests.post("/auth/login", json={"user": "alice", "pass": "secret"})
    token = login_resp.json()["access_token"]
    
    # Użyj tokenu w kolejnym żądaniu
    headers = {"Authorization": f"Bearer {token}"}
    response = requests.get("/api/protected", headers=headers)
    assert response.status_code == 200
```

**OAuth 2.0** — delegowanie uprawnień. Użytkownik autoryzuje aplikację przez zewnętrzny serwis (Google, Facebook).

---

## 7. Selenium

### 7.1 Lokatory (selektory)

**Pytanie:** Jakie znasz lokatory w Selenium i jak ustalić priorytety ich użycia?

**Odpowiedź:**

```
Priorytet (od najlepszego):
1. ID              → najbardziej stabilny i szybki
2. Name            → dla pól formularzy
3. CSS Selector    → szybki, czytelny
4. data-testid     → dedykowany do testów, bardzo stabilny
5. Link Text       → dla linków
6. XPath           → ostateczność — kruchy, wolny
```

```python
from selenium.webdriver.common.by import By

# NAJLEPIEJ: ID lub data-testid
driver.find_element(By.ID, "submit-button")
driver.find_element(By.CSS_SELECTOR, "[data-testid='submit-button']")

# DOBRZE: CSS Selector
driver.find_element(By.CSS_SELECTOR, ".login-form input[type='email']")

# UNIKAJ: kruchy XPath zależny od struktury DOM
driver.find_element(By.XPATH, "//div[3]/form/div[1]/input")  # ZŁE

# LEPSZY XPath jeśli musisz:
driver.find_element(By.XPATH, "//input[@placeholder='Adres email']")
```

**Gotcha:** `data-testid` wymaga współpracy z developerami — muszą dodać atrybuty do elementów. Warto to ustalić jako standard w projekcie.

---

### 7.2 Waity (oczekiwanie na elementy)

**Pytanie:** Opisz rodzaje waitów w Selenium. Kiedy stosować każdy?

**Odpowiedź:**

**Implicit Wait** — globalne oczekiwanie na pojawienie się elementu przy każdym `find_element`.

```python
driver.implicitly_wait(10)  # sekundy — ustawione raz, działa dla wszystkich
```

**Explicit Wait** — oczekiwanie na konkretny warunek dla konkretnego elementu.

```python
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

wait = WebDriverWait(driver, timeout=10)

# Czekaj aż element będzie klikalny
button = wait.until(EC.element_to_be_clickable((By.ID, "submit")))
button.click()

# Czekaj aż tekst pojawi się w elemencie
wait.until(EC.text_to_be_present_in_element((By.ID, "status"), "Sukces"))

# Czekaj aż URL się zmieni
wait.until(EC.url_contains("/dashboard"))
```

**Fluent Wait** — jak explicit, ale z konfigurowalnymi interwałami i ignorowanymi wyjątkami.

```python
from selenium.webdriver.support.ui import WebDriverWait
from selenium.common.exceptions import NoSuchElementException

wait = WebDriverWait(
    driver,
    timeout=15,
    poll_frequency=0.5,  # sprawdzaj co 0.5 sekundy
    ignored_exceptions=[NoSuchElementException]
)
element = wait.until(EC.visibility_of_element_located((By.ID, "loader")))
```

**Zasada:** Nigdy nie używaj `time.sleep()` w testach UI — jest kruche i spowalnia testy. Zamiast tego zawsze używaj explicit wait.

---

## 8. Git

### 8.1 Podstawowe operacje

**Pytanie:** Opisz podstawowe polecenia Git używane codziennie.

**Odpowiedź:**

```bash
git clone <url>          # pobierz repozytorium
git pull                 # pobierz zmiany ze zdalnego repo (fetch + merge)
git fetch                # pobierz zmiany, ale nie merguj
git add <plik>           # dodaj plik do staging area
git commit -m "msg"      # zapisz zmiany lokalnie
git push                 # wyślij commity do zdalnego repo
git status               # sprawdź stan working directory
git log --oneline        # historia commitów
git diff                 # różnice między working directory a staging
```

---

### 8.2 Git Fetch

**Pytanie:** Jaka jest różnica między `git fetch` a `git pull`? Kiedy używać `fetch`?

**Odpowiedź:**

```
git fetch  =  pobierz zmiany ze zdalnego repo → NIE merguj automatycznie
git pull   =  git fetch + git merge (lub rebase, zależnie od konfiguracji)
```

```bash
# Pobierz zmiany ze wszystkich gałęzi bez mergowania
git fetch origin

# Sprawdź co zostało pobrane (remote-tracking branch)
git log origin/main --oneline --graph

# Porównaj swoją gałąź z tym co jest na zdalnym
git diff main origin/main

# Dopiero teraz merguj świadomie
git merge origin/main
# lub
git rebase origin/main
```

**Kiedy używać `fetch` zamiast `pull`:**
- Chcesz *zobaczyć* zmiany zanim je zintegrujesz
- Pracujesz na gałęzi, która nie powinna być automatycznie aktualizowana
- Chcesz zaktualizować remote-tracking branches bez wpływu na working directory
- CI/CD: pobierz informacje o tagach/branchach bez zmiany lokalnego kodu

```bash
# Typowy workflow z fetch
git fetch origin                     # pobierz wszystko
git log HEAD..origin/main --oneline  # co mnie omija?
git diff HEAD origin/main            # jakie zmiany?
git rebase origin/main               # świadoma integracja
```

**Gotcha:** `git pull --rebase` jest bezpieczniejszą alternatywą dla `git pull` na feature branchach — unika merge commitów i zachowuje czystą historię.

---

### 8.3 Merge vs Rebase

**Pytanie:** Jaka jest różnica między git merge a git rebase?

**Odpowiedź:**

**Merge** — łączy historię dwóch gałęzi, tworząc nowy merge commit. Zachowuje pełną historię.

```
main:   A --- B --- C ------- M (merge commit)
                    \       /
feature:             D --- E
```

**Rebase** — "przepisuje" commity feature branch na wierzchołek main. Tworzy liniową historię.

```
main:   A --- B --- C
                     \
feature:              D' --- E'  (nowe commity, przepisane)
```

```bash
# Merge (zachowuje historię)
git checkout main
git merge feature/login

# Rebase (liniowa historia)
git checkout feature/login
git rebase main
```

**Kiedy stosować:**
- **Merge** → publiczne gałęzie, gdy historia jest ważna
- **Rebase** → lokalne gałęzie feature przed mergem do main — czystsza historia

**Gotcha:** Nigdy nie rebasuj commitów, które już zostały wypchnięte na publiczną gałąź!

---

### 8.3 Cherry-pick

**Pytanie:** Do czego służy `git cherry-pick`?

**Odpowiedź:**

Cherry-pick kopiuje konkretne commity z jednej gałęzi na inną bez mergowania całej gałęzi.

```bash
git log --oneline feature/hotfix
# abc1234 Fix critical bug in payment
# def5678 WIP: experimental change

# Przenieś tylko hotfix do main, bez WIP zmian
git checkout main
git cherry-pick abc1234
```

**Kiedy używać:**
- Hotfix do wielu wersji (prod, staging)
- Przypadkowo zacommitowane zmiany na złej gałęzi
- Wybór konkretnych zmian z gałęzi eksperymentalnej

---

### 8.4 Git Stash

**Pytanie:** Do czego służy `git stash`?

**Odpowiedź:**

Stash tymczasowo odkłada niezacommitowane zmiany, żeby móc przełączyć gałąź.

```bash
git stash                     # odłóż zmiany
git stash list                # lista odłożonych zmian
git stash pop                 # przywróć najnowsze (i usuń ze stash)
git stash apply stash@{1}     # przywróć konkretne (zostaje w stash)
git stash drop stash@{0}      # usuń ze stash
git stash push -m "WIP login" # nadaj nazwę
```

---

### 8.5 Git Reset

**Pytanie:** Czym różni się `git reset --soft`, `--mixed` i `--hard`?

**Odpowiedź:**

| Opcja | Commit | Staging | Working Dir |
|-------|--------|---------|-------------|
| `--soft` | Cofnięty | Zachowany | Zachowany |
| `--mixed` (domyślny) | Cofnięty | Wyczyszczony | Zachowany |
| `--hard` | Cofnięty | Wyczyszczony | **Wyczyszczony** |

```bash
# Cofnij ostatni commit, zachowaj zmiany w staging
git reset --soft HEAD~1

# Cofnij ostatni commit, zmiany w working directory
git reset HEAD~1

# UWAGA: nieodwracalne — wymaż wszystko
git reset --hard HEAD~1
```

**Gotcha:** `--hard` usuwa niezacommitowane zmiany bezpowrotnie. Używaj ostrożnie!

---

## 9. Python OOP — Pytania Zaawansowane

### 9.1 Overriding vs Overloading w Pythonie

**Pytanie:** Czym różni się overriding od overloading? Jak overloading działa w Pythonie?

**Odpowiedź:**

**Overriding** — podklasa nadpisuje metodę klasy bazowej (ta sama sygnatura, inna implementacja).

```python
class Shape:
    def area(self) -> float:
        return 0.0

class Circle(Shape):
    def __init__(self, radius: float):
        self.radius = radius
    
    def area(self) -> float:  # override
        return 3.14159 * self.radius ** 2
```

**Overloading** — w Javie: wiele metod o tej samej nazwie, różne parametry. W Pythonie nie istnieje natywnie — symulujemy przez domyślne argumenty lub `functools.singledispatch`.

```python
# Python: domyślne argumenty zamiast overloadingu
def greet(name: str, greeting: str = "Cześć") -> str:
    return f"{greeting}, {name}!"

greet("Alice")           # "Cześć, Alice!"
greet("Alice", "Hej")    # "Hej, Alice!"

# Lub singledispatch dla różnych typów
from functools import singledispatch

@singledispatch
def process(value):
    raise NotImplementedError(f"Nieznany typ: {type(value)}")

@process.register(int)
def _(value: int):
    return f"Liczba całkowita: {value}"

@process.register(str)
def _(value: str):
    return f"Napis: {value.upper()}"
```

---

### 9.2 ABC vs Protocol (duck typing)

**Pytanie:** Kiedy używać ABC, a kiedy Protocol?

**Odpowiedź:**

```python
from abc import ABC, abstractmethod
from typing import Protocol

# ABC — nominalne dziedziczenie (explicit "is-a")
class Serializable(ABC):
    @abstractmethod
    def to_json(self) -> str: ...

class User(Serializable):
    def to_json(self) -> str:
        return '{"name": "Alice"}'

# Protocol — strukturalne podtypowanie (duck typing)
class Serializable(Protocol):
    def to_json(self) -> str: ...

class User:  # NIE dziedziczy z Serializable!
    def to_json(self) -> str:
        return '{"name": "Alice"}'

def save(obj: Serializable) -> None:  # działa z Protocol bez dziedziczenia
    print(obj.to_json())

save(User())  # OK — User ma metodę to_json()
```

**Kiedy ABC:** Chcesz wymusić dziedziczenie i chronić przed pominięciem implementacji.  
**Kiedy Protocol:** Chcesz testować kaczki (duck typing) bez wymagania konkretnej hierachii.

---

### 9.3 Lambda i domknięcia (closures)

**Pytanie:** Co to jest lambda? Jaki gotcha czai się w domknięciach Pythona?

**Odpowiedź:**

```python
# Lambda — anonimowa funkcja
double = lambda x: x * 2
sorted_users = sorted(users, key=lambda u: u.name)

# Closure — funkcja zapamiętuje zmienne ze swojego zakresu
def make_multiplier(n):
    def multiply(x):
        return x * n  # n jest zapamiętane w domknięciu
    return multiply

times3 = make_multiplier(3)
print(times3(5))  # 15

# GOTCHA: domknięcia przechwytują zmienną, nie jej wartość!
functions = []
for i in range(3):
    functions.append(lambda x: x + i)  # wszystkie dzielą tę samą i!

print([f(0) for f in functions])  # [2, 2, 2] — nie [0, 1, 2]!

# POPRAWKA: przechwytuj wartość przez domyślny argument
functions = []
for i in range(3):
    functions.append(lambda x, i=i: x + i)  # i=i "zamraża" wartość

print([f(0) for f in functions])  # [0, 1, 2] — poprawnie
```

---

### 9.4 Modyfikatory dostępu

**Pytanie:** Jak Python implementuje modyfikatory dostępu?

**Odpowiedź:**

Python nie ma prawdziwych modyfikatorów dostępu — stosuje konwencje:

```python
class BankAccount:
    def __init__(self):
        self.balance = 0           # publiczny — dostępny wszędzie
        self._internal = 0         # chroniony — konwencja "nie dotykaj z zewnątrz"
        self.__secret = 0          # prywatny — name mangling: _BankAccount__secret

    def get_secret(self):
        return self.__secret  # dostępne wewnątrz klasy

account = BankAccount()
print(account.balance)       # OK
print(account._internal)     # Działa, ale łamie konwencję
print(account.__secret)      # AttributeError — nie istnieje jako __secret
print(account._BankAccount__secret)  # 0 — ale to bardzo zły pomysł!
```

**Gotcha:** Name mangling (`__name`) to ochrona przed przypadkowym nadpisaniem w podklasach, nie prawdziwe ukrycie.

---

### 9.5 Kolekcje Pythona

**Pytanie:** Jakie kolekcje oferuje Python? Kiedy użyć której?

**Odpowiedź:**

| Kolekcja | Mutowalna | Duplikaty | Porządek | Kiedy używać |
|----------|-----------|-----------|----------|--------------|
| `list` | Tak | Tak | Tak | Domyślna sekwencja |
| `tuple` | Nie | Tak | Tak | Niezmienne dane, klucze słownika |
| `set` | Tak | Nie | Nie | Unikalność, operacje na zbiorach |
| `frozenset` | Nie | Nie | Nie | Niezmienne zbiory |
| `dict` | Tak | Klucze: Nie | Tak (3.7+) | Mapowanie klucz-wartość |

```python
from collections import defaultdict, Counter, deque, OrderedDict

# defaultdict — automatycznie tworzy brakujące klucze
word_count = defaultdict(int)
for word in "ala ma kota ala":
    word_count[word] += 1
# {"ala": 2, "ma": 1, "kota": 1}

# Counter — liczenie elementów
c = Counter("abracadabra")
c.most_common(2)  # [("a", 5), ("b", 2)]

# deque — wydajne wstawianie/usuwanie z obu stron O(1)
dq = deque([1, 2, 3])
dq.appendleft(0)  # O(1) — lista byłaby O(n)
```

---

### 9.6 Porównywanie obiektów: `==` vs `is`

**Pytanie:** Jaka jest różnica między `==` a `is` w Pythonie?

**Odpowiedź:**

- `==` — porównuje **wartości** (woła `__eq__`)
- `is` — porównuje **tożsamość** (czy to ten sam obiekt w pamięci)

```python
a = [1, 2, 3]
b = [1, 2, 3]
c = a

print(a == b)   # True  — te same wartości
print(a is b)   # False — różne obiekty w pamięci
print(a is c)   # True  — c to alias a

# GOTCHA: int caching (-5 do 256)
x = 256
y = 256
print(x is y)   # True — Python cache'uje małe liczby całkowite

x = 257
y = 257
print(x is y)   # False! — nowe obiekty poza zakresem cache'u

# ZAWSZE porównuj Singleton przez is
result = some_function()
if result is None:   # POPRAWNIE
    ...
if result == None:   # DZIAŁA, ale niepytoniczne (może woła __eq__)
    ...
```

---

### 9.7 Typy proste vs obiekty (int vs Integer — analogia Javy)

**Pytanie:** Jak Python traktuje typy — odpowiednik int vs Integer z Javy?

**Odpowiedź:**

W Pythonie **wszystko jest obiektem** — nie ma odpowiednika prymitywów Javy. Jednak istnieje ważna różnica między typami mutowalnymi i niemutowalnymi.

```python
# int, float, str, bool, tuple — NIEZMIENNE (immutable)
x = 5
y = x
x += 1
print(y)  # 5 — y nie zmieniło się, int jest niemutowalny

# list, dict, set — ZMIENNE (mutable)
a = [1, 2, 3]
b = a          # b to ALIAS do tego samego obiektu!
a.append(4)
print(b)       # [1, 2, 3, 4] — b "widzi" zmianę!

# Kopiowanie
import copy
b = a.copy()         # płytka kopia — OK dla list bez zagnieżdżenia
b = copy.deepcopy(a) # głęboka kopia — bezpieczna dla złożonych struktur

# Typ Optional (odpowiednik Integer — może być null)
from typing import Optional

def find_user(user_id: int) -> Optional[dict]:
    # Zwraca None jeśli nie znaleziono
    return None
```

---

### 9.8 Abstrakcyjne klasy bazowe (ABC)

**Pytanie:** Jak działają klasy abstrakcyjne w Pythonie?

**Odpowiedź:**

```python
from abc import ABC, abstractmethod

class PaymentGateway(ABC):
    @abstractmethod
    def charge(self, amount: float, token: str) -> str:
        """Zwraca transaction_id."""
        ...
    
    @abstractmethod
    def refund(self, transaction_id: str) -> bool: ...
    
    def validate_amount(self, amount: float) -> None:
        """Konkretna metoda — dostępna w podklasach."""
        if amount <= 0:
            raise ValueError("Kwota musi być dodatnia")

class StripeGateway(PaymentGateway):
    def charge(self, amount: float, token: str) -> str:
        self.validate_amount(amount)
        # Implementacja Stripe
        return "txn_123"
    
    def refund(self, transaction_id: str) -> bool:
        return True

# Nie można instancjonować klasy abstrakcyjnej
gateway = PaymentGateway()  # TypeError!

# Można instancjonować tylko pełną implementację
gateway = StripeGateway()   # OK
```

---

### 9.9 Dekoratory

**Pytanie:** Co to są dekoratory w Pythonie? Podaj przykłady przydatne w testach.

**Odpowiedź:**

Dekorator to funkcja, która opakowuje inną funkcję, dodając zachowanie.

```python
import functools
import time

# Własny dekorator — mierzenie czasu
def timer(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} zajął {elapsed:.3f}s")
        return result
    return wrapper

@timer
def slow_operation():
    time.sleep(1)

# Dekoratory pytest
@pytest.mark.parametrize("x,y,expected", [
    (2, 3, 5),
    (0, 0, 0),
    (-1, 1, 0),
])
def test_add(x, y, expected):
    assert x + y == expected

@pytest.mark.skip(reason="Funkcja niezaimplementowana")
def test_not_ready():
    assert future_feature() == "done"

@pytest.mark.xfail(reason="Znany bug #123")
def test_known_bug():
    assert broken_function() == "correct"

# Dekorator retry dla niestabilnych testów API
def retry(max_attempts: int = 3, delay: float = 1.0):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except AssertionError as e:
                    last_exception = e
                    time.sleep(delay)
            raise last_exception
        return wrapper
    return decorator

@retry(max_attempts=3, delay=2.0)
def test_flaky_external_api():
    response = requests.get("/sometimes-slow-endpoint")
    assert response.status_code == 200
```

---

### 9.10 Generatory i yield

**Pytanie:** Co to są generatory i kiedy ich użyć w kontekście testów?

**Odpowiedź:**

```python
# Generator — zwraca wartości po jednej, leniwy (lazy)
def fibonacci():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

gen = fibonacci()
first_ten = [next(gen) for _ in range(10)]
# [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# W testach: generowanie danych testowych bez ładowania wszystkich do pamięci
def generate_test_records(count: int):
    for i in range(count):
        yield {
            "id": i,
            "name": f"User_{i}",
            "email": f"user{i}@test.com"
        }

def test_bulk_import(db_session):
    records = list(generate_test_records(10_000))
    result = db_session.bulk_insert(records)
    assert result.inserted_count == 10_000

# Fixture używające yield — zarządzanie cyklem życia
@pytest.fixture
def temp_file(tmp_path):
    file = tmp_path / "test_data.csv"
    file.write_text("id,name\n1,Alice\n2,Bob")
    yield file  # "setup" kończy się tutaj
    # "teardown" — wykonuje się po teście
    if file.exists():
        file.unlink()
```

