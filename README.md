# Python 3 Simple NATS (simple-nats)

A simple NATS client for Python 3 compatibility (no asyncio nonsense).

This package is not currently available in PyPI, but the package is just a single
file, so you can download it and use it as is.

# Usage

    import nats

## Create NATS client object

    nc = nats.NatsClient()

Create a client object with a given client name

    nc = nats.NatsClient(client_name='my_consumer')

## Connect to a NATS server

Use `<server>:<port>` format. Currently clusters are not
supported.
You can also specify: "nats://<server>:<port>".
There is no default port.
You will need to specifiy that explicitly.
Additionally, there is not IPv6 support as of yet, so use IPv4.

    nc.connect('localhost:4222')

You can get server information as a dictionary after connecting with

    nc.server_info

## Subscribe to a topic

You can subscribe to a topic with

    def callback(msg):
        print(msg)

    sid = nc.subscribe('my.topic', queue_group='my_group', cb=callback)

The `queue_group` option is optional. It may also be passed by index rather than as a
named arg.
`cb` is not optional. If it is not specified, the method returns, failing silently.
TODO in the future is to add error responses.

`nc.subscribe` returns a `sid` to identify the subscription for that client.
This is necessary to unsubscribe from the topic later on that client.
If no `sid` is returned, you can assume that the `subscribe` call failed.

The `msg` parameter passed to the callback will be structured like so

    {'op': 'MSG',
     'subject': 'my.topic',
     'sid': 1,
     'reply_to': None,
     'payload': b'{"foo": "bar"}'}

## Unsubscribe from a topic

Using the `sid` acquired earlier, you can unsubscribe from a topic with

    nc.unsubscribe(sid)

This method does not return any information about success or failure.
You can also set it to unsubscribe in the future after $n$ messages have been received
with

    max_messages = 20
    nc.unsubscribe(sid, max_messages)

This will automatically unsubscribe from the topic after 20 messages have been received by
the client.

## Publish a message

The publish call expects the payload to be a bytes object, so raw string values should be
encoded with `my_string.encode('utf8')`. A message can be sent with:

    import json

    nc.publish('my.topic', json.dumps({'foo': 'bar'}).encode('utf8'))

## Execute a remote procedure call

One nice feature of NATS is the ability to execute RPCs and receive an asynchronous response.
This is a feature not available in systems like Kafka, despite Kafka being superior in many other
respects.

An RPC can be executed like so

    import msgpack

    def process_answer(answer):
        payload = msgpack.loads(answer['payload'])
        print("Got answer! {}".format(payload)

    payload = msgpack.dumps({'foo': 'bar'})
    nc.request('my.topic', payload, cb=process_answer)

In this case `my.topic` is the topic where the RPC workers are listening for function calls.
The NATS client will auto-generate a reply-to inbox to receive the result from the workers and
set an auto-unsubscribe after one answer is received.
In the future, the ability to accept arbitrary $n$ answers will be added as an optional parameter to this library.

As before, payloads must be bytes-like objects

The message received on clients subscribed to `my.topic` will look something like this

    {'op': 'MSG',
     'subject': 'my.topic',
     'sid': 1,
     'reply_to': 'requests.ba788a5d-ebc5-4e60-9dea-c791125c36d3.1',
     'payload': b'\x81\xa3foo\xa3bar'}

This client generates a reply-to inbox ID in the format `requests.<client name>.<inbox ID>.
The inbox ID auto-increments in a thread-safe manner. The only consideration is that the client name
should be unique on the cluster.
If one is not specified when the client is created, a client name will be randomly generated using `uuid.uuid4()`.

## Disconnecting

NATS does not have a disconnect command.
The protocol is schema-less and semi-stateless.
