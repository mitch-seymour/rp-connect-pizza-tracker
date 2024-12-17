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
With our cache resource defined, we can now perform lookups of previously seen messages. This will help us determine if a customer's order status has changed. First, let's define some test cases that define the desired behavior (this approach of writing tests before code is called test-driven development). The input messages are defined using the `input_batch` configuration, and the expected outputs, which include a new field for tracking the previous order status (`previous_status`), are defined with the `output_batches` configuration.

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
            "order_status": "pending"
          }
      - content: |
          {
            "created_at": "2024-12-12T17:25:52.724229Z",
            "customer_address": "456 Oak Ave",
            "customer_id": 2,
            "customer_name": "Jane Smith",
            "details": "1 pepperoni pizza, 1 veggie pizza",
            "order_id": 2,
            "order_status": "pending"
          }
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
              "order_status": "pending",
              "previous_status": "none"
            }
        - json_equals: |
            {
              "customer_id": 2,
              "details": "1 pepperoni pizza, 1 veggie pizza",
              "order_id": 2,
              "order_status": "pending",
              "previous_status": "pending"
            }
        - json_equals: |
            {
              "customer_id": 2,
              "details": "1 pepperoni pizza, 1 veggie pizza",
              "order_id": 2,
              "order_status": "delivered",
              "previous_status": "pending"
            }
```

If you run the tests now:

```sh
rpk connect test /etc/redpanda/connect.yaml
```

You will see a failure since your pipeline isn't actually populating the new `previous_status` field yet.

```sh
Test '/etc/redpanda/connect.yaml' failed

Failures:

--- /etc/redpanda/connect.yaml ---

test record scrubbing [line 26]:
batch 0 message 0: json_equals: JSON content mismatch
{
    "customer_id": 2,
    "details": "1 pepperoni pizza, 1 veggie pizza",
    "order_id": 2,
    "order_status": "pending",
    "previous_status": null
}
```

This is expected. Now all you need to do is update the pipeline to hydrate the new field using our cache resource.

### Handling batches
The first change we will make to our pipeline is to add a `for_each` operator, which allows us to operate on batches of messages that may be coming from the database.

```yaml
pipeline:
  processors:
    - for_each:
      - bloblang: |
          root = {}
          root.customer_id = this.customer_id
          root.order_id = this.order_id
          root.order_status = this.order_status
          root.previous_status = this.previous_status
          root.details = this.details
```

### Using the branch operator
As you may recall, each step in the pipeline modifies the source message it processes. If you were to add another step immediately following the `bloblang` processor, that step would operate on the output of the `bloblang` transformation. However, to perform a lookup on the cache, you need to extract a single field from the payload and use that as the input for the lookup, without modifying the entire message. Using the `branch` operator, you can create a nested pipeline to handle the extracted data independently while keeping the main pipeline unaffected.

For example, to perform a cache lookup using the `order_id`, you can configure the `branch` operator as follows:

```yaml
pipeline:
  processors:
    - for_each:
      - bloblang: |
          ...

      # Retrieve the previous status from the cache
      - branch:
          processors:
            # Since cache misses are logged as errors, wrap the cache.get operation in a try/catch
            # to reduce error log noise
            - try:
              - cache:
                  # lookup the previous status for this order_id in our order_cache resource,
                  # which is an in-memory map. The in-memory map can be replaced with Redis
                  # or more durable storage in production
                  resource: order_cache
                  operator: get
                  key: '${! json("order_id") }'
            - catch:
              # If the order_id isn't in the cache, set the previous status to "none"
              - mapping: |
                  "pending"
          # This part of the branch operator allows us to map the results of this operation
          # back to the original source message
```

The code is heavily commented, so to understand the processor's configuration in more detail, please read through the comments before proceeding.

### Writing to the cache
Awesome! You're performing lookups on the cache now, but if you don't actually write the order statuses to the cache when they come in, the cache resource will always be empty. Therefore, after retrieving the previous status for the current order, you need to make sure to write the current status to the cache. To do this, you can use the `cache` processor.

Add the following processor to the `pipeline`, right after the `try` / `catch` processors.

```yaml
  # Update the cache with the latest order status
  - cache:
      resource: order_cache
      operator: set
      key: '${! json("order_id") }'
      value: '${! json("order_status") }'
```

### Notifying customers
There are different ways to notify customers, and full exploration of each option is beyond the scope of this tutorial. However, one approach is to use event-driven microservices, invoking an HTTP request to a dedicated notification service. For example, you could use the `http` processor to issue a `POST` request, as follows.

```yaml
      # Perform an action if the status has changed
      - branch:
          request_map: 'if this.status_changed { this }'
          processors:
            - http:
                url: "http://localhost:8787/api/status-change"
                method: POST
                headers:
                  Content-Type: application/json
                payload: |
                  {
                    "customer_id": this.customer_id,
                    "order_id": this.order_id,
                    "previous_status": this.previous_status,
                    "new_status": this.order_status
                  }
```

For the purpose of this tutorial, however, we will use the `log` operator to simply print a message each time a customer should be notified. To achieve this, add the following line to your `pipeline` config"

```yaml
  - branch:
      request_map: 'if this.previous_status != this.order_status { this }'
      processors:
        - log:
            level: INFO
            message: |
              Send customer update: ${! json("customer_id") }
              New Status: ${! json("order_status") }
```
The full file should now look like this:

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
    - for_each:
      - bloblang: |
          root = {}
          root.customer_id = this.customer_id
          root.order_id = this.order_id
          root.order_status = this.order_status
          root.previous_status = this.previous_status
          root.details = this.details

      # Retrieve the previous status from the cache
      - branch:
          processors:
            # Since cache misses are logged as errors, wrap the cache.get operation in a try/catch
            # to reduce error log noise
            - try:
              - cache:
                  # lookup the previous status for this order_id in our order_cache resource,
                  # which is an in-memory map. The in-memory map can be replaced with Redis
                  # or more durable storage in production
                  resource: order_cache
                  operator: get
                  key: '${! json("order_id") }'
            - catch:
              # If the order_id isn't in the cache, set the previous status to "none"
              - mapping: |
                  "none"
          # This part of the branch operator allows us to map the results of this operation
          # back to the original source message
          result_map: |
            root.previous_status = content().string()
    
      # Update the cache with the latest order status
      - cache:
          resource: order_cache
          operator: set
          key: '${! json("order_id") }'
          value: '${! json("order_status") }'

      # detect if status changed
      - bloblang: |
          root = this
          root.status_changed = if this.previous_status != this.order_status {
            true
          } else {
            false
          }

      # Perform an action if the status has changed
      - branch:
          request_map: 'if this.previous_status != this.order_status { this }'
          processors:
            - log:
                level: INFO
                message: |
                  Send customer update: ${! json("customer_id") }
                  New Status: ${! json("order_status") }

            # - http:
            #     url: "http://localhost:8787/api/status-change"
            #     method: POST
            #     headers:
            #       Content-Type: application/json
            #     payload: |
            #       {
            #         "customer_id": this.customer_id,
            #         "order_id": this.order_id,
            #         "previous_status": this.previous_status,
            #         "new_status": this.order_status
            #       }

output:
  label: "redpanda"
  kafka:
    addresses: [ 'redpanda-1:9092']
    topic: orders
    key: '${! json("customer_id") }'

cache_resources:
  - label: order_cache
    memory:
      default_ttl: 60s

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
            "order_status": "pending"
          }
      - content: |
          {
            "created_at": "2024-12-12T17:25:52.724229Z",
            "customer_address": "456 Oak Ave",
            "customer_id": 2,
            "customer_name": "Jane Smith",
            "details": "1 pepperoni pizza, 1 veggie pizza",
            "order_id": 2,
            "order_status": "pending"
          }
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
              "order_status": "pending",
              "previous_status": "none",
              "status_changed": true
            }
        - json_equals: |
            {
              "customer_id": 2,
              "details": "1 pepperoni pizza, 1 veggie pizza",
              "order_id": 2,
              "order_status": "pending",
              "previous_status": "pending",
              "status_changed": false
            }
        - json_equals: |
            {
              "customer_id": 2,
              "details": "1 pepperoni pizza, 1 veggie pizza",
              "order_id": 2,
              "order_status": "delivered",
              "previous_status": "pending",
              "status_changed": true
            }
```

If you run the tests again:

```sh
rpk connect test /etc/redpanda/connect.yaml
```

You should see that they succeed:

```sh
Test '/etc/redpanda/connect.yaml' succeeded
```

You can also re-run the pipeline with the following command:

```sh
rpk connect run /etc/redpanda/connect.yaml
```

You should see several logs like the following:

```sh
INFO Let customer 2 know their order is now: delivered
INFO Let customer 4 know their order is now: pending
INFO Let customer 3 know their order is now: preparing
INFO Let customer 1 know their order is now: pending
INFO Let customer 5 know their order is now: delivered
```

### Bonus
In this bonus section, we'll go a little bit deeper to introduce you to secrets and also the OpenAI processors. It will require an OpenAI API key, which you can create [here](https://platform.openai.com/api-keys). However, if you don't want to create an API key, feel free to skip this section.

Our goal in this section is to add some personalization to the system. Specifically, you'll use the OpenAI chat completion processor to create personalized notifications for your customers when their order status changes.

The simplest way to do this is to add the `openai_chat_completion` processor to your pipeline, like so:

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
      - label: openai
        branch:
          request_map: 'if this.previous_status != this.order_status && this.previous_order.status != "none" { this }'
          processors:
            - try:
              -   openai_chat_completion:
                    server_address: https://api.openai.com/v1
                    api_key: "${OPENAI_API_KEY}"
                    model: gpt-4o
                    system_prompt: |
                      Our customer just ordered some food and the order status just changed
                      from to ${! json(this.order_status)}. Please create a personalized message
                      to give them this update.
                      Bonus points if you can come up with something witty pertaining to their
                      order: ${! json(this.details)}.
            - catch:
              # If the order_id isn't in the cache, set the previous status to "none"
              - mapping: |
                  "Your order is now: " + this.order_status
          result_map: 'root.message = content().string()'

      # Perform an action if the status has changed
      - branch:
          request_map: 'if this.message != null { this }'
          processors:
            - log:
                level: INFO
                message: |
                  Send customer update: ${! json("customer_id") }
                  ${! json("message") }
```

If you look closely at the `api_key` configuration, you'll see a placeholder:

```yaml
api_key="${OPENAI_API_KEY}"
```

You could add your [OpenAI API key](https://platform.openai.com/api-keys) or other sensitive information directly to your configuration files, but that's not a secure practice. Instead, the syntax we're using (`${ENV_VAR_NAME}`) tells Redpanda Connect to inject that value from the environment. To run this pipeline now with the current environment variable, execute the following command:


```sh
docker exec -e OPENAI_API_KEY=YOUR_KEY -ti redpanda-1 \
  rpk connect run /etc/redpanda/connect.yaml
```

You'll see several customer notifications get logged to the screen. If you consume from the orders topic now:

```sh
rpk topic consume orders -f '%v' -n 1 -o -1 | jq '.'
```

You'll see personalized messages like the following:

```json
{
  "customer_id": 1,
  "details": "2 cheese pizzas, 1 garlic bread",
  "message": "Hey there, Pizza Enthusiast Extraordinaire!\n\nGreat news: your cheesy dreams are in the oven! Your order of 2 cheese pizzas and 1 garlic bread is now pending, which means the countdown to that delightful cheesy goodness and garlicky aroma has officially begun. Keep your taste buds on standby — your treat is on its way to being served!\n\nStay saucy! 🍕🧄",
  "order_id": 1,
  "order_status": "pending",
  "previous_status": "none"
}
```

That's amazing. The tests will need to be updated to add the new `message` field, but we'll leave that as an ecercise for the reader.

### Summary
We've barely scratched the surface of what you can do in Redpanda Connect, but the workflows for adding inputs, outputs, and processors, as well as running unit tests, adding cache resources, and independent branch operations to access the cache, will provide a good foundation for you to continue exploring Redpanda Connect.

Before you close this tutorial, we encourage to take a look at the [other processors](https://docs.redpanda.com/redpanda-connect/components/processors/about/) and pipeline building resources in the official documentation. Perhaps add a new operator to challenge your knowledge and get some self-guided experience. We also have an extended Bonus section coming up next, which contains an example of adding personalized status updates using an OpenAI operator.

