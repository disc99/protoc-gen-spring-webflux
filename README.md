# protoc-gen-spring-webflux
gRPC to JSON handler generator protoc plugin for Spring WebFlux inspired by [grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway).

The protoc-gen-spring-webflux is a plugin of the Google protocol buffers compiler
[protoc](https://github.com/protocolbuffers/protobuf).
It reads protobuf service definitions and generates Spring WebFlux HTTP server classes which
translates RESTful HTTP API into gRPC. This server is generated according to the [`google.api.http`](https://github.com/googleapis/googleapis/blob/master/google/api/http.proto#L46) annotations in your service definitions.

Uses an external HTTP server separately from the gRPC server to convert Http requests to gRPC calls.
![image](https://user-images.githubusercontent.com/5003722/71714336-cbd1b100-2e50-11ea-84ea-791e6f8b3aee.png)

## Usage

1. Define your [gRPC](https://grpc.io/docs/) service using protocol buffers.

```protoc:example.proto
syntax = "proto3";

package example.demo;

// messages...

service EchoService {

    rpc GetEcho(EchoRequest) returns (EchoResponse) {
    }

    rpc CreateEcho(CreateEchoRequest) returns (CreateEchoResponse) {
    }
}
```
2. Add a [`google.api.http`](https://github.com/googleapis/googleapis/blob/master/google/api/http.proto#L46) annotation to your .proto file for HTTP API.

```diff
syntax = "proto3";

package example.demo;
+import "google/api/annotations.proto";

// messages...

service EchoService {

    rpc GetEcho(EchoRequest) returns (EchoResponse) {
+        option (google.api.http) = {
+            get: "/echo/{echo}"
+        };
    }

    rpc CreateEcho(CreateEchoRequest) returns (CreateEchoResponse) {
+        option (google.api.http) = {
+              post: "/echo"
+              body: "*"
+        };
    }
}
```

3. Generate routing handler class using `protoc-gen-spring-webflux`

3-a. Use protoc plugin on Linux or OSX

* When building in a Linux or OSX environment, build with the protoc plugin.

```diff
# build.gradle

protobuf {
    protoc {
        // ...
    }

    plugins {
        // ...
+       webflux {
+           artifact = 'io.github.protobuf-x:protoc-gen-spring-webflux:${PROTOC_GEN_SPRING_WEBFLUX_VERSION}'
+       }
    }

    generateProtoTasks {
        all()*.plugins {
            // ...
+           webflux {}
        }
    }
}
```

3-b. Use protoc command

* Since the protobuf plugin for windows is not supported, please execute the protoc command if you need to build on windows.

```bash
protoc -I. \
    --spring-webflux_out=. \
     example.proto
```

Download the latest binaries from Maven Central Repository.

[![Maven Central](https://maven-badges.herokuapp.com/maven-central/io.github.protobuf-x/protoc-gen-spring-webflux/badge.svg)](https://maven-badges.herokuapp.com/maven-central/io.github.protobuf-x/protoc-gen-spring-webflux/)

4. Write an routing of the Spring WebFlux `Handler` server.

```java:HandlerServerConfg.java
@Configuration
class HandlerServerConfg {
    @Bean
    ExampleHandlers.EchoServiceHandler exampleHandlers() {
        ManagedChannel channel = ManagedChannelBuilder.forAddress(/*...*/)
                .usePlaintext()
                .build();
        EchoServiceGrpc.EchoServiceStub stub = EchoServiceGrpc.newStub(channel);

        // ExampleHandlers is a class generated by protoc-gen-spring-webflux
        return EchoServiceRest.newGrpcProxyBuilder()
                .setStub(stub)
                .setIncludeHeaders(Collections.singletonList("Authorization"))
                .build();
    }

    @Bean
    RouterFunction<ServerResponse> routing(ExampleHandlers.EchoServiceHandler handler) {
        return RouterFunctions.route()
                // Use the handleAll method to route everything to the generated Handler.
                .add(handler.allRoutes())
                // Handler can be routed individually by using the generated method.
                .GET("/echo/{id}", handler::getEcho)
                .POST("/echo", handler::createEcho)
                .build();
    }
}
```

## Missing Features Shortlist
* Streams not supported.
* Custom patterns not supported.
* Variables not supported.
* Not supporting * and ** in path.


## License
(The MIT License)

Copyright (c) 2020 @disc99
