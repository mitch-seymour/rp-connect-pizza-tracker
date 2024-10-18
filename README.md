# RP Connect Pizza Tracker

You just bought a pizza restaurant, nice!

Now you want to implement a pizza tracker. You're a Redpanda customer so you have access to state of the art streaming tech. Let's get started.

## Setup

First, we need to create a topic to hold all of the order events. The events will look something like this.

```json
{
  "order_id": "",
  "customer_name": "",
  "customer_address": "",
  "details": "",
  "created_at": "",
}
```

The big question that most beginners have trouble with is deciding on a partition count. You're still growing your customer base so you don't have a good grasp of your volume, but planning for the future and using the powers of 2 rule, we'll select a reasonable number of 16 partitions.

There are several CLIs in the Kafka ecosystem, but `rpk` is 🔥, so we'll use that. Run the following command to create your topic.

```
rpk topic create orders -p4
```

## Capturing orders
When orders come in, they are written directly to the company's main transactional database: Postgresql.

We want to get this data in Redpanda so we can build monitors for order updates and notify our customers. We have a few options, including:

- Producing the data directly from our backend application that processes orders
- Using Redpanda Connect to listen for changes in Postgresql and automatically produce the data to Redpanda

Let's go with the second option. To do so, we need

