# Migrating to Clever Cloud RabbitMQ

## Summary

Today we have our own RabbitMQ hosted on AWS (t2.medium). In the past, the main issues we had was due to RabbitMQ. Actually, the engine was requesting too much the RabbitMQ server and this leads to memory overflow, RabbitMQ server was stucked and we were losing messages. (Bad for customers, it should never happen anymore).
Today, we've fixed few things to have a more stable RabbitMQ environment, so it works ok on our medium server. To be more confident as the usage load is going to increase and to focus on Zenaton feature first, we would like to give the queues management to [Clever Cloud](clever-cloud.com) who do it well (cluster with replicated node).

## Problem

For making this possible we need to migrate the existing RabbitMQ to the new Clever Cloud infrastructure.
- We must not loose any messages
- We should minimize the downtime duration
- Small delay is ok
- Users should not notice anything

Reminder: As Clever Cloud is a PaaS, we do not have access to the server by SSH. (Heroku like)

## Proposal

- The agent is already reconnecting automatically when he loses the connection, so if we restart the engine and change the targeted RabbitMQ url, it should be fine.
- Migrate all the RabbitMQ administration with a mix command (Users & Password and access rights)
- Implement a temporary services that will consumed all the incoming messages from the old RabbitMQ and re-inject it in the new one.
- Change the RabbitMQ url in the engine and restart it.
