# nameko-grpc

This is a prototype implementation of a GRPC server and client for use in [nameko](https://nameko.io) microservices.

All four of the request-response patterns are implemented and tested:

1. unary-unary
2. unary-stream
3. stream-unary
4. stream-stream

Asynchronous calls are also supported for every pattern.

Python 3.4+ is supported.

## Tests

Most tests are run against every permutation of GRPC server/client to Nameko server/client. This roughly demonstrates equivalence between the two implementations.

Additionally, we run the interop tests from the offical GRPC repo, which are used to verify compatibility between language implementations. The Nameko GRPC implementation supports every feature that the official Python GRPC implementation does.


## Style

The example protobufs in this repo use `snake_case` for method names as per the Nameko conventions rather than `CamelCase` as per GRPC. This is not mandatory -- decorated method names simply match to the methods defined in the protobufs.


## Example

### Server

Example Nameko service that can respond to GRPC requests:

``` python
from example_pb2 import ExampleReply
from example_pb2_grpc import exampleStub


grpc = Grpc.implementing(exampleStub)


class ExampleService:
    name = "example"

    @grpc
    def unary_unary(self, request, context):
        message = request.value * (request.multiplier or 1)
        return ExampleReply(message=message)

    @grpc
    def unary_stream(self, request, context):
        message = request.value * (request.multiplier or 1)
        yield ExampleReply(message=message, seqno=1)
        yield ExampleReply(message=message, seqno=2)

    @grpc
    def stream_unary(self, request, context):
        messages = []
        for req in request:
            message = req.value * (req.multiplier or 1)
            messages.append(message)

        return ExampleReply(message=",".join(messages))

    @grpc
    def stream_stream(self, request, context):
        for index, req in enumerate(request):
            message = req.value * (req.multiplier or 1)
            yield ExampleReply(message=message, seqno=index + 1)

```

### Client

Example Nameko service that can make GRPC requests:

``` python
from example_pb2 import ExampleReply
from example_pb2_grpc import exampleStub

from nameko.rpc import rpc


class ClientService:
    name = "client"

    example_grpc = GrpcProxy("//127.0.0.1", exampleStub)

    @rpc
    def method(self):
        responses = self.example_grpc.unary_stream(ExampleRequest(value="A"))
        for response in responses:
            print(response.message)

```

Example standalone client, can be used with or without Eventlet:

``` python
from example_pb2 import ExampleReply
from example_pb2_grpc import exampleStub

from nameko_grpc.client import Client

with Client("//127.0.0.1", exampleStub) as client:
    responses = client.unary_stream(ExampleRequest(value="A"))
    for response in responses:
        print(response.message)

```
