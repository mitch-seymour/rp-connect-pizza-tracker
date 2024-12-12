# RP Connect Pizza Tracker

> Work in progress! Started this to answer a question brought up in training and am currently iterating :)

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

We're using the `customer_id` as the record key (see the last line) since we want all events for a given customer to be ordered. To deploy these changes, run the following command:

```sh
rpk connect run /etc/redpanda/connect.yaml
```

Then, verify that the data is now in Redpanda. You can do that from the CLI:

```sh
rpk topic consume orders
```

or from [Redpanda Console](http://localhost:8080/topics/orders).

You should see data flowing through Redpanda now. As they say in the pizza industry, "Now we're cooking with gas"!

You now know the basics of moving data between various input and output destinations using Redpanda Connect. In other words, the important "Extract" and "Load" steps in your traditional ETL or ELT pipelines.

In the next section, we'll explore one of Redpanda Connect's biggest strengths: its processing layer.

## Transforming data with Redpanda Connect

Now that data is flowing through Redpanda, you may have noticed that some sensitive customer information is included in each payload. Namely, the customer's name and address:

```json
{
  "created_at": "2024-12-12T17:25:52.724229Z",
  "customer_address": "456 Oak Ave",
  "customer_id": 2,
  "customer_name": "Jane Smith",
  "details": "1 pepperoni pizza, 1 veggie pizza",
  "order_id": 2,
  "order_status": "delivered"
}
```

Our order tracking system doesn't actually need this information, so let's add some lightweight stream processing to remove this sensitive information.

Just like pizza, Redpanda Connect is a system of many layers. Sandwiched between input and output configurations, we often use `pipelines` to perform one or more data processing steps. At a high-level, it looks like this:

```yaml
input:
  ...

pipeline:
  processors:
    ...

output:
  ...
```

Within each `pipeline`, we have one or more `processors`, which define what we can actually do with the data. There are nearly [100 processors](https://docs.redpanda.com/redpanda-connect/components/processors/about/), covering a wide range of use cases, from filtering and transforming data, to enriching it with external sources, aggregating records, or even creating [advanced workflows](https://docs.redpanda.com/redpanda-connect/components/processors/workflow/). You are only limited by your imagination.

While covering every processor is not possible in just a short tutorial, we will explore a few powerful options. Perhaps the most flexible and powerful processor is `bloblang`, a dynamic and expressive language for filtering, transforming, and mapping data. With `bloblang`, you can manipulate fields, enrich messages, and even create complex conditional logic, making it an essential tool for customizing pipelines to fit your specific needs.

The core features can be found in [the Redpanda Connect documentation](https://docs.redpanda.com/redpanda-connect/guides/bloblang/about/), but the goal of the `bloblang` processor is to simply map an input document into a new output document. This is perfect for our current use case of cleansing out input record of sensitive information.

The bloblang syntax for creating a new `orders` document with the sensitive information removed is shown here:

```yaml
pipeline:
  processors:
    - bloblang: |
        root = {}
        root.customer_id = this.customer_id
        root.order_id = this.order_id
        root.order_status = this.order_status
        root.details = this.details
```

As you can see, we start by creating an empty document `{}` called `root`. When then transfer the non-sensitive fields (`customer_id`, `order_id`, `order_status`, `details`) from the current document (`this`) to the new document (`root`). Go ahead and replace `connect.yaml` with the following code to see this in action.

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
        root.order_status = this.order_status
        root.details = this.details

output:
  label: "redpanda"
  kafka:
    addresses: [ 'redpanda-1:9092']
    topic: orders
    key: '${! json("customer_id") }'
```

Then, run the pipeline again with the following command:

```sh
rpk connect run /etc/redpanda/connect.yaml
```

Finally, verify that the order data has now been scrubbed:

```
rpk topic consume orders -f '%v' -n 1 -o -1 | jq '.'
```

The output will show records that are cleaner and more privacy-friendly.

```json
{
  "customer_id": 3,
  "details": "3 meat lovers pizzas",
  "order_id": 3,
  "order_status": "preparing"
}
```

As we've discussed in other Redpanda University courses, this type of operation which operates on a single record at a time is called a **stateless operation**. These are among the simplest and most common processing tasks that you'll see in the wild. Later in this tutorial, we'll take a look at some more complicated **stateful operations**, like aggregating records. However, before we do that, let's take a quick look at testing so that we can build confidently.

## Testing
As you build Redpanda Connect pipelines, it's a good idea to add tests to prevent accidental regressions or bugs in your code. Luckily, unit tests can be written using a simple, declarative syntax, right alongside your YAML definitions. The following example demonstrates a simple test case for the record-scrubbing pipeline we just created:

```yaml
tests:
  - name: test record scrubbing
    environment: {}
    input_batch:
      - content: |
          {
            "created_at": "2024-12-12T17:25:52.724229Z",
            "customer_address": "456 Oak Ave",
            "customer_id": 2,
            "customer_name": "Jane Smith",
            "details": "1 pepperoni pizza, 1 veggie pizza",
            "order_id": 2,
            "order_status": "delivered"
          }
    output_batches:
      -
        - json_equals: |
            {
              "customer_id": 2,
              "details": "1 pepperoni pizza, 1 veggie pizza",
              "order_id": 2,
              "order_status": "delivered"
            }
```

By specifying one or more input records, and then asserting the contents of the output record, we can ensure our code performs the task we expect it to. Go ahead and update your `connect.yaml` with this code, so that the full file looks like this:

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
        root.order_status = this.order_status
        root.details = this.details

output:
  label: "redpanda"
  kafka:
    addresses: [ 'redpanda-1:9092']
    topic: orders
    key: '${! json("customer_id") }'

tests:
  - name: test record scrubbing
    environment: {}
    input_batch:
      - content: |
          {
            "created_at": "2024-12-12T17:25:52.724229Z",
            "customer_address": "456 Oak Ave",
            "customer_id": 2,
            "customer_name": "Jane Smith",
            "details": "1 pepperoni pizza, 1 veggie pizza",
            "order_id": 2,
            "order_status": "delivered"
          }
    output_batches:
      -
        - json_equals: |
            {
              "customer_id": 2,
              "details": "1 pepperoni pizza, 1 veggie pizza",
              "order_id": 2,
              "order_status": "delivered"
            }
```

To run the test, you can use `rpk`. For example, run the following command:

```sh
rpk connect test /etc/redpanda/connect.yaml
```

You'll see the following output printed to the screen:

```
Test '/etc/redpanda/connect.yaml' succeeded
```

For larger pipelines, you may want to break out your tests into a separate file. This is also supported, and you can find more information in [the official documentation](https://docs.redpanda.com/redpanda-connect/configuration/unit_testing/#writing-a-test).

## More Operators
It may seem like we're hopping around a bit, but the four concepts you've learned so far (inputs, outputs, pipelines, and testing) will allow you to explore the rest of Redpanda Connect's features with ease. The testing piece is especially important as it will allow us to confirm our expectations and assumptions in a quicker iteration loop. This will be useful in the next section as we introduce a more complex feature: notifying customers when their order status changes.

### Stateful Resources
A key feature of our pizza tracker is its ability to proactively notify customers as their orders progress through various stages. No more anxiously refreshing a browser or hovering over a screen wondering when their extra cheese pizza will show up at the door.

While this may seem like a straightforward feature, implementing it requires leveraging some of the more advanced capabilities of Redpanda Connect.

To meet the requirements, our pizza tracking pipeline needs to accomplish the following:

- For each incoming record, determine if the status of the given order has changed.
- If the status remains the same, take no action to avoid sending unnecessary notifications.
- If the status has changed, trigger a notification to the customer. (For now, this will be simulated by logging a message.)

To achieve this, the pipeline must maintain some awareness of previously-seen events. This "awareness" or memory of previously-seen events will make our application **stateful**.

A simple key-value store or cache is well-suited for this purpose. Each time a status update is received, the pipeline will check the cache for the previous status associated with the order ID. If a change is detected, it will log the notification. Additionally, the cache will be updated with the latest status to ensure future records are processed correctly.

Luckily for us, Redpanda Connect includes a `cache` processor and resource to facilitate this. To define a cache resource, you can add the following line to your `connect.yaml` file:

```yaml
cache_resources:
  - label: order_cache
    memory:
      default_ttl: 1800s
```

Here, we are choosing an in-memory cache with an 1800s (30 minute) timeout. However, more durable options like Redis, Memcached, and Dynamodb are [also available](https://docs.redpanda.com/redpanda-connect/components/caches/about/). Next, we'll update our pipeline to retrieve and store order statuses within this cache resource.

### Using Caches





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

