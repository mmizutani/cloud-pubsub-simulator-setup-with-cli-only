# cloud-pubsub-simulator-setup-with-cli-only

This is a short guide to setup the Google Cloud Pub/Sub simulator container using only the gcloud CLI.

## Background

Google's [official documentation for the Pub/Sub emulator]([text](https://cloud.google.com/pubsub/docs/emulator#using_the_emulator)) guides you to git clone a repository and run a bunch of Python wrapper scripts to setup the Pub/Sub emulator.

```python
$ git clone https://github.com/googleapis/python-pubsub
$ cd python-pubsub/samples/snippets
$ virtualenv env
$ source env/bin/activate
$ python -m pip install -r requirements.txt
$ python publisher.py create my-topic
$ python subscriber.py create my-topic my-subscription
```

This involves installing Python and the Google Cloud Python SDKs. If you are going to setup the Pub/Sub emulator in a CI/CD pipeline, you will have to build a custom container image with Python, the Python SDKs and the custom Python scripts installed or mount a volume with such additional tools. Tedious! 😫

Is there any smarter/simpler way to setup the containerized Pub/Sub emulator? Yes, it is.

This short guide demonstrates that, unlike the cumbersome procedure above, we can actually setup the Pub/Sub emulator with the desired topics and subscriptions already setup using only the gcloud CLI (bash scripts) without ever installing any programming language code or SDK.

## Steps

1. Copy the contents of the following Docker Compose config file to your project (or mix it in into your existing Docker Compose file). You can also use similar syntax in the [`services` section](https://docs.github.com/en/actions/use-cases-and-examples/using-containerized-services/about-service-containers) of a GitHub Actions Workflow.

    ```yaml
    # compose.yml
    # This sample docker compose file starts a Cloud Pub/Sub emulator and automatically creates a topic and subscription of desired names upon startup.
    services:
      pubsub:
        image: google/cloud-sdk:emulators
        command:
          - /bin/bash
          - -ceu
          - |
            set -m
            gcloud beta emulators pubsub start --project=$$CLOUDSDK_CORE_PROJECT --host-port=0.0.0.0:$$PUBSUB_EMULATOR_HOST_PORT --quiet &
            while ! echo > /dev/tcp/localhost/$$PUBSUB_EMULATOR_HOST_PORT; do sleep 1; done
            gcloud pubsub topics create $$TOPIC
            gcloud pubsub subscriptions create $$SUBSCRIPTION --topic=$$TOPIC
            fg %1
        ports:
          # Point your Cloud Pub/Sub SDK client to this port by exporting the environment variable
          # `export PUBSUB_EMULATOR_HOST=localhost:8085` outside of the docker compose environment
          # or by setting the `PUBSUB_EMULATOR_HOST=pubsub:8085` environment variable in the service
          # that uses the Cloud Pub/Sub SDK client in this docker compose environment.
          - 8085:8085
        environment:
          # Equivalent to `gcloud config configurations activate default`
          CLOUDSDK_ACTIVE_CONFIG_NAME: default
          # Equivalent to `gcloud config set auth/disable_credentials true`
          CLOUDSDK_AUTH_DISABLE_CREDENTIALS: true
          # Equivalent to `gcloud config set account emulator@example.com`
          CLOUDSDK_CORE_ACCOUNT: emulator@example.com
          # Equivalent to `gcloud config set project test`
          CLOUDSDK_CORE_PROJECT: test
          # Equivalent to `gcloud config set api_endpoint_overrides/pubsub http://localhost:8085/`
          # Must be an absolute URI that begins with https:// and ends with a trailing /.
          CLOUDSDK_API_ENDPOINT_OVERRIDES_PUBSUB: http://localhost:8085/
          PUBSUB_EMULATOR_HOST_PORT: "8085"
          PUBSUB_EMULATOR_HOST: localhost:8085
          # Change these values `TOPIC` and `SUBSCRIPTION` to match your desired topic and subscription names
          TOPIC: test
          SUBSCRIPTION: test
    ```

2. Change the container environment variables `TOPIC` and `SUBSCRIPION` to the names appropriate for your testing with the Cloud Pub/Sub emulator.

3. Launch the Cloud Pub/Sub emulator container:

    ```bash
    $ docker compose up

    [+] Running 1/0
    ✔ Container cloud-pubsub-simulator-setup-with-cli-only-pubsub-1  Created                                                                                                                                                                                                                                             0.0s
    Attaching to pubsub-1
    pubsub-1  | /bin/bash: connect: Connection refused
    pubsub-1  | /bin/bash: line 3: /dev/tcp/localhost/8085: Connection refused
    pubsub-1  | Executing: /google-cloud-sdk/platform/pubsub-emulator/bin/cloud-pubsub-emulator --host=0.0.0.0 --port=8085
    pubsub-1  | /bin/bash: connect: Connection refused
    pubsub-1  | /bin/bash: line 3: /dev/tcp/localhost/8085: Connection refused
    pubsub-1  | [pubsub] This is the Google Pub/Sub fake.
    pubsub-1  | [pubsub] Implementation may be incomplete or differ from the real system.
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:18 AM com.google.cloud.pubsub.testing.v1.Main main
    pubsub-1  | [pubsub] INFO: IAM integration is disabled. IAM policy methods and ACL checks are not supported
    pubsub-1  | [pubsub] SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder".
    pubsub-1  | [pubsub] SLF4J: Defaulting to no-operation (NOP) logger implementation
    pubsub-1  | [pubsub] SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.
    pubsub-1  | /bin/bash: connect: Connection refused
    pubsub-1  | /bin/bash: line 3: /dev/tcp/localhost/8085: Connection refused
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:18 AM com.google.cloud.pubsub.testing.v1.Main main
    pubsub-1  | [pubsub] INFO: Server started, listening on 8085
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:19 AM io.gapi.emulators.netty.HttpVersionRoutingHandler channelRead
    pubsub-1  | [pubsub] INFO: Detected non-HTTP/2 connection.
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:20 AM io.gapi.emulators.netty.HttpVersionRoutingHandler channelRead
    pubsub-1  | [pubsub] INFO: Detected non-HTTP/2 connection.
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:20 AM io.gapi.emulators.grpc.HttpAdapter$StubMethodHandler handle
    pubsub-1  | [pubsub] INFO: Forwarding key: Host
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:20 AM io.gapi.emulators.grpc.HttpAdapter$StubMethodHandler handle
    pubsub-1  | [pubsub] INFO: Forwarding key: user-agent
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:20 AM io.gapi.emulators.grpc.HttpAdapter$StubMethodHandler handle
    pubsub-1  | [pubsub] INFO: Forwarding key: accept-encoding
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:20 AM io.gapi.emulators.grpc.HttpAdapter$StubMethodHandler handle
    pubsub-1  | [pubsub] INFO: Forwarding key: accept
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:20 AM io.gapi.emulators.grpc.HttpAdapter$StubMethodHandler handle
    pubsub-1  | [pubsub] INFO: Not forwarding key: Connection
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:20 AM io.gapi.emulators.grpc.HttpAdapter$StubMethodHandler handle
    pubsub-1  | [pubsub] INFO: Not forwarding key: Content-Length
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:20 AM io.gapi.emulators.grpc.HttpAdapter$StubMethodHandler handle
    pubsub-1  | [pubsub] INFO: Forwarding key: content-type
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:20 AM io.gapi.emulators.netty.HttpVersionRoutingHandler channelRead
    pubsub-1  | [pubsub] INFO: Detected HTTP/2 connection.
    pubsub-1  | Created topic [projects/test/topics/test].
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:21 AM io.gapi.emulators.netty.HttpVersionRoutingHandler channelRead
    pubsub-1  | [pubsub] INFO: Detected non-HTTP/2 connection.
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:21 AM io.gapi.emulators.grpc.HttpAdapter$StubMethodHandler handle
    pubsub-1  | [pubsub] INFO: Forwarding key: Host
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:21 AM io.gapi.emulators.grpc.HttpAdapter$StubMethodHandler handle
    pubsub-1  | [pubsub] INFO: Forwarding key: user-agent
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:21 AM io.gapi.emulators.grpc.HttpAdapter$StubMethodHandler handle
    pubsub-1  | [pubsub] INFO: Forwarding key: accept-encoding
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:21 AM io.gapi.emulators.grpc.HttpAdapter$StubMethodHandler handle
    pubsub-1  | [pubsub] INFO: Forwarding key: accept
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:21 AM io.gapi.emulators.grpc.HttpAdapter$StubMethodHandler handle
    pubsub-1  | [pubsub] INFO: Not forwarding key: Connection
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:21 AM io.gapi.emulators.grpc.HttpAdapter$StubMethodHandler handle
    pubsub-1  | [pubsub] INFO: Not forwarding key: Content-Length
    pubsub-1  | [pubsub] Sep 23, 2024 3:25:21 AM io.gapi.emulators.grpc.HttpAdapter$StubMethodHandler handle
    pubsub-1  | [pubsub] INFO: Forwarding key: content-type
    pubsub-1  | Created subscription [projects/test/subscriptions/test].
    pubsub-1  | gcloud beta emulators pubsub start --project=$CLOUDSDK_CORE_PROJECT --host-port=0.0.0.0:$PUBSUB_EMULATOR_HOST_PORT --quiet
    ```

    The Pub/Sub topic and subscription of the names specified via the environment variables are automatically created upon the container startup.

    > Created topic [projects/test/topics/test].
    > Created subscription [projects/test/subscriptions/test].

4. On the machine that runs your application, set the `PUBSUB_EMULATOR_HOST` environment variable to the host and port of the Cloud Pub/Sub emulator container:

    ```bash
    export PUBSUB_EMULATOR_HOST=localhost:8085
    ```

    If the application that uses the Pub/Sub emulator is running within the same docker compose project, set the environment variable `PUBSUB_EMULATOR_HOST=pubsub:8085` to the container:

    ```yaml
    services:
      app:
        ...
        environment:
          PUBSUB_EMULATOR_HOST: pubsub:8085
    ```

    ```yaml
    services:
      app:
        ...
        environment:
          - PUBSUB_EMULATOR_HOST=pubsub:8085
    ```

5. Publish messages to the containerized Pub/Sub topic or subscribe messages in the containerized Pub/Sub subscription in your application.

6. (Optional) You can control the topics and subscriptions in the emulator (e.g., publish messages) in an ad-hoc manner using the gcloud CLI embedded within the official emulator container image:

    ```bash
    $ docker compose exec pubsub \
        gcloud pubsub topics publish test --message '{"key":"value"}'
    messageIds:
    - '1'
    ```

The only thing we need is just the official Pub/Sub emulator container image: `google/cloud-sdk:emulators`. There is no need to setup Python convenience scripts or build custom container images that wrap the official emulator container.
You can also edit the `services.pubsub.command` section of the docker compose file to change or add `gcloud pubsub topics/subscriptions` sub commands to setup additional topics/subscriptions and specify options such as message filters, message ordering, and ack deadlines.
Flexible and concise! 😄

## How this works

The trick above works thanks to the following:

- `gcloud` CLI is builtin in the official Pub/Sub emulator container image.
  This is partly because the Cloud Pub/Sub Emulator is installed as a component of the Google Cloud CLI.
- The configurations of the Google Cloud CLI can be overridden not only via the `gcloud config set` commands but also via the the environment variables in the special patterns `CLOUDSDK_*` (`CLOUDSDK_<SECTION_NAME>_<PROPERTY_NAME>`). There is one-to-one correspondence between the configuration items the gcloud CLI and the environment variable keys.
  This functionality is explained in [Managing gcloud CLI properties | Google Cloud CLI Documentation](https://cloud.google.com/sdk/docs/properties#setting_properties_using_environment_variables).
- The API endpoints that the Google Cloud CLI communicates with can be overridden by changing the configurations via either the commands `gcloud config set api_endpoint_overrides/<SERVICE> <URL>` or the environment variables `CLOUDSDK_API_ENDPOINT_OVERRIDES_<SERVICE>`.
  This functionality is explained in [gcloud topic endpoint-override | Google Cloud CLI Documentation](https://cloud.google.com/sdk/gcloud/reference/topic/endpoint-override).
  In the case of Cloud Pub/Sub, the name of the override environment variable and the format of the URL are detailed in [List of locational endpoints
 | Pub/Sub APIs overview | Pub/Sub Documentation | Google Cloud](https://cloud.google.com/pubsub/docs/reference/service_apis_overview#set_a_locational_endpoint_override).
