# RP Connect Pizza Tracker

You just bought a pizza restaurant, nice! You tell people it's because you always wanted to feed the hungry, but really, you just like eating obscene amounts of BBQ chicken pizza when no one's looking.

We won't judge you, but we do want to help you implement an order tracker so you can keep up with the big chains. You're a Redpanda customer so you have access to state of the art streaming tech to build real-time systems. Let's get started.

# Design
Our system will provide real-time tracking for pizza orders, allowing customers to monitor their order status and track the delivery in real-time. The system follows an event-driven architecture using Redpanda for handling order status updates and real-time delivery tracking.

## Key Features
- Order Status Updates: As the order progresses (e.g., "Received", "Preparing", "Out for Delivery", "Delivered"), events are published to a Kafka topic, and customers are notified in real-time.
- Real-Time GPS Tracking: Drivers update their GPS location periodically, which is streamed to the customer app, enabling live delivery tracking.
- Asynchronous Notifications: A notification service listens for order updates and triggers SMS or push notifications to keep customers informed throughout the process.

By decoupling services with Redpanda, the system scales efficiently and ensures reliable event processing for real-time tracking and notifications.

## Setup
First, start Redpanda.

## Setup

First, we need to create a topic to hold all of the order events. The events will look something like this.

```json
{
  "order_id": "",
  "customer_id": "",
  "customer_name": "",
  "customer_address": "",
  "details": "",
  "order_status": "",
  "created_at": "",
}
```

The big question that most beginners have trouble with is deciding on a partition count. You're still growing your customer base so you don't have a good grasp of your volume, but planning for the future and using the powers of 2 rule, we'll select a reasonable number of 4 partitions. Yeah we could overthink it, but we're in the pizza business so lets roll with it.
 
There are several CLIs in the Kafka ecosystem, but `rpk` is 🔥, so we'll use that. Run the following command to create your topic.

```
rpk topic create orders -p 4
```

## Capturing orders
When orders come in, they are written directly to the company's main transactional database: Postgresql. In fact, you can see some orders are already in the database:

```sh
docker exec -ti postgres psql -c "select * from orders"
```

We want to get this data in Redpanda so we can build monitors for order updates and notify our customers. We have a few options, including:

- Producing the data directly from our backend application that processes orders
- Using Redpanda Connect to pull records directly Postgresql and automatically produce the data to Redpanda

Let's go with the second option.

First, create a file called `connect.yaml`. In this file, we'll use YAML to tell Redpanda where to pull data from, how to transform it, and where to send it to.

Start by adding a single input, which tells Connect to read data from our `orders` table.

```yaml
input:
  sql_select:
    driver: postgres
    dsn: postgres://root:secret@postgres:5432/root?sslmode=disable
    table: orders
    columns: [ '*' ]
```

See if it works by running the `rpk connect` command.

```sh
rpk connect run /etc/redpanda/connect.yaml
```

You should see some orders get printed to the screen. This is a good start, but we want the orders data in Redpanda. You can see that the `orders` topic is empty by running this command:

```sh
rpk topic consume orders
```

If you're more of a UI person, you could also hop on over to Redpanda Console and look at the topic. It will be empty.
[http://localhost:8080/topics/orders](http://localhost:8080/topics/orders).

Let's hydrate the topic now by adding a Kafka output. Open the `connect.yaml` file again and run the following code, which tells Redpanda Connect to send the data to the `orders` topic in our Redpanda cluster.

```yaml
input:
  sql_select:
    driver: postgres
    dsn: postgres://root:secret@postgres:5432/root?sslmode=disable
    table: orders
    columns: [ '*' ]

# add the output
output:
  label: "redpanda"
  kafka:
    addresses: [ 'redpanda-1:9092']
    topic: orders
    key: '${! json("customer_id") }'
```

We're using the `customer_id` as the record key (see the last line) since we want all events for a given customer to be ordered.

Now we're cooking with gas! Run the pipeline again:

```sh
rpk connect run /etc/redpanda/connect.yaml
```

And verify that the data is now in Redpanda. You can do that from the CLI:

```sh
rpk topic consume orders
```

or from [Redpanda Console](http://localhost:8080/topics/orders).

Looking good! But ah geez, David from the legal team says we shouldn't be copying sensitive data like the customer's address and name to our order tracking system.

Let's add a pipeline to clean that up.

```yaml
input:
  sql_select:
    driver: postgres
    dsn: postgres://root:secret@postgres:5432/root?sslmode=disable
    table: orders
    columns: [ '*' ]

# add the pipeline
pipeline:
  processors:
    - bloblang: |
        root = {}
        root.customer_id = this.customer_id
        root.order_id = this.order_id
        root.details = this.details

output:
  label: "redpanda"
  kafka:
    addresses: [ 'redpanda-1:9092']
    topic: orders
    key: '${! json("customer_id") }'
```

Run the pipeline again:

```sh
rpk connect run /etc/redpanda/connect.yaml
```

And then verify:

```
rpk topic consume orders -f '%v' -n 1 -o -1 | jq '.'
```

The records should look better and more privacy-friendly.

```json
{
  "customer_id": 3,
  "details": "3 meat lovers pizzas",
  "order_id": 3
}
```

We now have order data flowing through Redpanda thanks to Redpanda Connect, and can finish implementing the order tracking system. Using Redpanda Connect's cache resources, we can turn this stateless application into a stateful one, storing the most recent order status for each customer and triggering a notification when the order is on its way.

We'll leave that as an exercise for the reader, but a good way to achieve that would be to use Redpanda Connect's cache resource to track these changes.

You can also add some personalization to the system by using the OpenAI chat completion processor to create personalized notifications when the message is on its way. Here's an example (update the API key [your own](https://platform.openai.com/api-keys).

```yaml
input:
  sql_select:
    driver: postgres
    dsn: postgres://root:secret@postgres:5432/root?sslmode=disable
    table: orders
    columns: [ '*' ]

# add the pipeline
pipeline:
  processors:
    - branch:
        processors:
          - openai_chat_completion:
              server_address: https://api.openai.com/v1
              api_key: "TODO"
              model: gpt-4o
              system_prompt: |
                Our customer just ordered some food and now it's on the way!
                Please create a personalized message to let them know their food
                from Wild Slice will be there shortly. Bonus points if you can come
                up with something witty pertaining to their order: ${! json(this.details)}.
        result_map: 'root.message = content().string()'
    - bloblang: |
        root = {}
        root.customer_id = this.customer_id
        root.order_id = this.order_id
        root.details = this.details
        root.message = this.message

output:
  label: "redpanda"
  kafka:
    addresses: [ 'redpanda-1:9092']
    topic: orders
    key: '${! json("customer_id") }'
```

Check the Redpanda topic again and you see the following:

```sh
rpk topic consume orders -f '%v' -n 1 -o -1 | jq '.'
```

Example response:

```json
{
  "customer_id": 5,
  "details": "4 cheese pizzas, 2 sodas",
  "message": "Hey Chris!\n\nGreat news! Your order from Wild Slice has already arrived, so there's no need to wait any longer for that cheesy goodness. Your 4 cheese pizzas are waiting to turn your evening into a melty masterpiece, accompanied by your favorite fizzy sidekicks—2 refreshing sodas. It's time to grab a slice and embrace the ultimate cheese party!\n\nThanks for letting us sprinkle some joy into your day!\n\nStay Cheesy,  \nThe Wild Slice Team 🍕🧀",
  "order_id": 5
}
```

