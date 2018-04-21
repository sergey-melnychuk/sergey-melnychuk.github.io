---
layout: post
title:  "Testing AWS client with LocalStack"
date:   2018-04-21 11:54:23 +0200
categories: aws localstack docker unit-testing
---
Using AWS SDK always leaves one question open for me - how to introduce test coverage for the integration code. On one side, it doesn't make any sense to test/mock the code outside of your control (unless you're owning AWS SDK code, of course, but I suggest, most probably, you don't). On the other side, leaving code uncovered doesn't really look good as well.

TL;DR [code][code-github]

One thing I knew for sure, is that I'm not the first one asking this question, and I indeed wasn't. No better option comes to mind then introduce integration test against 'real' 3rd-party system. Of course, there aren't more real AWS endpoint than the real AWS endpoint, but paying for _each_ run of the test doesn't seem like a wise option to me.

Right next after the real AWS in the list of real AWS endpoints, comes [LocalStack][localstack-home]. It allows you to spin up local endpoints that implement AWS services contracts. Someone finally made their own AWS endpoints and allows you to spin it up at your localhost for free! Installation and available services are very well [described][localstack-github], so I won't cover it here. If you messed up with your local python installation (as I did apparently), it might be useful to read [this][localstack-install].

At this point, when localstack is installed, the fun begins. I was most interested in mocking AWS S3 service, so the very basic way to check if localstack works, is to try it with AWS CLI.

Start localstack and wait until initialization is completed. Then it can be stopped with CRTL-C.

```
$ localstack start
Starting local dev environment. CTRL-C to quit.
Starting mock API Gateway (http port 4567)...
Starting mock DynamoDB (http port 4569)...
Starting mock SES (http port 4579)...
Starting mock Kinesis (http port 4568)...
Starting mock Redshift (http port 4577)...
Starting mock S3 (http port 4572)...
Starting mock CloudWatch (http port 4582)...
Starting mock CloudFormation (http port 4581)...
Starting mock SSM (http port 4583)...
Starting mock SQS (http port 4576)...
Starting local Elasticsearch (http port 4571)...
Starting mock SNS (http port 4575)...
Starting mock DynamoDB Streams service (http port 4570)...
Starting mock Firehose service (http port 4573)...
Starting mock Route53 (http port 4580)...
Starting mock ES service (http port 4578)...
Starting mock Lambda service (http port 4574)...
Ready.
# waiting for CRTL-C
```

Alternatively, localstack supports running with docker. Of course, you'll need docker daemon available at `unix:///var/run/docker.sock` in order to use this way. There's no magic though, under the hood the container is build from the image localstack/localstack.

```
$ localstack start --docker
Starting local dev environment. CTRL-C to quit.
docker run -it -e LOCALSTACK_HOSTNAME="localhost" -p 8080:8080 -p 443:443 -p 4567-4583:4567-4583 -p 4590-4593:4590-4593 -v "/private/var/folders/22/v0pf_r7x7tj6dyn5w3ysx58r0000gp/T/localstack:/tmp/localstack" -v "/var/run/docker.sock:/var/run/docker.sock" -e DOCKER_HOST="unix:///var/run/docker.sock" -e HOST_TMP_FOLDER="/private/var/folders/22/v0pf_r7x7tj6dyn5w3ysx58r0000gp/T/localstack" "localstack/localstack"
# ...
# output is very similar to the one above
```

I will need following command from the output above later:

```
docker run -it -e LOCALSTACK_HOSTNAME="localhost" \
-p 8080:8080 -p 443:443 -p 4567-4583:4567-4583 -p 4590-4593:4590-4593 \
-v "/private/var/folders/22/v0pf_r7x7tj6dyn5w3ysx58r0000gp/T/localstack:/tmp/localstack" \
-v "/var/run/docker.sock:/var/run/docker.sock" \
-e DOCKER_HOST="unix:///var/run/docker.sock" \
-e HOST_TMP_FOLDER="/private/var/folders/22/v0pf_r7x7tj6dyn5w3ysx58r0000gp/T/localstack" \
"localstack/localstack"
```

You can see the actual container running with docker command:

```
$ docker ps
CONTAINER ID        IMAGE                   COMMAND                  CREATED             STATUS              PORTS                                                                                                              NAMES
1ff66cd0daf4        localstack/localstack   "/usr/bin/supervisor…"   21 minutes ago      Up 21 minutes       0.0.0.0:443->443/tcp, 0.0.0.0:4567-4583->4567-4583/tcp, 0.0.0.0:4590-4593->4590-4593/tcp, 0.0.0.0:8080->8080/tcp   infallible_swartz
```

Once localstack is up, we can now try using it with AWS CLI.

```
$ aws --endpoint-url=http://localhost:4572 s3 ls s3://
$ # nothing to list
$ aws --endpoint-url=http://localhost:4572 s3api create-bucket --bucket test-bucket --region us-east-1
$ aws --endpoint-url=http://localhost:4572 s3 ls s3://
2006-02-03 17:45:09 test-bucket
$ aws --endpoint-url=http://localhost:4572 s3 ls s3://test-bucket/
$ echo 'test data' > test.data
$ aws --endpoint-url=http://localhost:4572 s3 cp test.data s3://test-bucket/
upload: ./test.data to s3://test-bucket/test.data
$ aws --endpoint-url=http://localhost:4572 s3 ls s3://test-bucket/
2018-04-18 12:53:23         10 test.data
$ aws --endpoint-url=http://localhost:4572 s3 cp s3://test-bucket/test.data ./test.check
download: s3://test-bucket/test.data to ./test.check
$ diff test.data test.check
$ # files are the same
```

Now I'm ready to write some Java code and cover it with tests against localstack. To spin up docker container from Java, I will use [docker-java][dockerjava-github], pretty straight-forward and well-documented docker API for Java. Docker provides native SDKs for Go and Python, but also exposes HTTP API (that is used by docker-java under the hood), you can read more about docker API [here][docker-api].

I will use Gradle to setup project, you can install it on Mac with `brew install gradle` and then check the installed version with `gradle --version`. At the moment when this post was written, I got gradle 4.7.

```bash
$ mkdir java-docker-localstack-demo
$ cd java-docker-localstack-demo/
$ gradle init --type java-application
...
$ tree
.
├── build.gradle
├── gradle
│   └── wrapper
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── gradlew
├── gradlew.bat
├── settings.gradle
└── src
    ├── main
    │   └── java
    │       └── App.java
    └── test
        └── java
            └── AppTest.java
```

Adding the docker-java dependency and required jersey lib.

```groovy
// AWS Java SDK S3 module library
compile 'com.amazonaws:aws-java-sdk-s3:1.11.313'

// Docker-Java library
testCompile 'com.github.docker-java:docker-java:3.0.14'

// Jersey libraries required by docker-java
testCompile 'org.glassfish.jersey.core:jersey-common:2.26'
testCompile 'org.glassfish.jersey.core:jersey-client:2.26'
testCompile 'org.glassfish.jersey.inject:jersey-hk2:2.26'
```

Now it's time to make new application useful. By useful here I mean to be able go get file from storage (S3) by given key, and to put some string content to storage under a given key. First let's declare the contract of a service, let it be FileService. I prefer to separate concerns early and provide interface as a contract first, at design phase.

```java
public interface FileService {
    /**
     * Get a content under given key
     * @param key key
     * @return
     */
    String get(String key) throws IOException;

    /**
     * Put a data as a content under given key
     * @param key key
     * @param data data
     */
    void put(String key, String data);

    /**
     * Create a bucket with given name
     * @param bucket bucket
     */
    void createBucket(String bucket);
}
```

One there's an interface, it's time to make some implementation of it. The provided implementation is rather straightforward and ignores AWS SDK runtime exceptions, which is fine considering main purpose of this post.

```java
public class FileServiceImpl implements FileService {
    private final String bucket;
    private final AmazonS3 amazonS3;

    public FileServiceImpl(String bucket, AmazonS3 amazonS3) {
        this.bucket = bucket;
        this.amazonS3 = amazonS3;
    }

    @Override
    public String get(String key) throws IOException {
        S3Object s3Object = amazonS3.getObject(bucket, key);
        try (S3ObjectInputStream objectInputStream = s3Object.getObjectContent()) {
            BufferedReader reader = new BufferedReader(new InputStreamReader(objectInputStream));
            StringBuilder sb = new StringBuilder();
            while (true) {
                String line = reader.readLine();
                if (line == null) break;
                sb.append(line);
                sb.append('\n');
            }
            return sb.toString();
        }
    }

    @Override
    public void put(String key, String data) {
        amazonS3.putObject(bucket, key, data);
    }

    @Override
    public void createBucket(String bucket) {
        amazonS3.createBucket(bucket);
    }
}
```

FileService requires AmazonS3 instance, and I don't want the users of FileService to even know about such implementation detail as S3 client running under the hood of my super-useful FileService! Nice thing to have would be some service, that can provide me an instance of FileService. "Factory" you're probably thinking right now, and you're right.

```java
package edu.blog.service;

import com.amazonaws.client.builder.AwsClientBuilder;
import com.amazonaws.services.s3.AmazonS3;
import com.amazonaws.services.s3.AmazonS3ClientBuilder;

public class FileServiceFactory {
    private static final String AWS_SIGNIN_REGION = "us-east2";

    public static FileService custom(String bucket, String awsServiceEndpoint) {
        return new FileServiceImpl(bucket, getS3(awsServiceEndpoint));
    }

    private static AmazonS3 getS3(String awsServiceEndpoint) {
        
        /*
        For standard client following line of code would be enough:
        `return AmazonS3ClientBuilder.defaultClient();`
         */

        AmazonS3ClientBuilder builder = AmazonS3ClientBuilder.standard();

        /*
        EndpointConfiguration has only constructor taking 2 arguments: service endpoint and sign-in region,
        thus both values must be provided. Using the 'us-east2' value as a sign-in region gets the job done.
         */

        builder.setEndpointConfiguration(new AwsClientBuilder.EndpointConfiguration(awsServiceEndpoint, AWS_SIGNIN_REGION));

        /*
        Enable path-style access in order to ensure service endpoint is not taken into account,
        (which is "${BUCKET_NAME}.localhost" for this client) as it is not valid DNS name.
         */

        builder.enablePathStyleAccess();

        return builder.build();
    }
}
```

Let's create some test-cases, one test-case to be precise. I believe this one is enough to check if integration is working properly and bucket is created, then file is first written and then read, and the content of a file read matches the content of a file written. Sounds simple and it really is.

```java
private static final String S3_SERVICE_ENDPOINT = "http://localhost:4572/";

@Test
public void testFileServiceAgainstLocalStack() throws IOException {
    FileService fileService = FileServiceFactory.custom(BUCKET, S3_SERVICE_ENDPOINT);
    fileService.createBucket(BUCKET);
    fileService.put(KEY, "{\"created\":true}\n");
    String content = fileService.get(KEY);
    assertEquals("{\"created\":true}\n", content);
}
```

I am no TDD purist, so I do not need to run this test to make sure it will fail. Trust me, it will. Unless someone is listening for TCP at 4572 and implements AWS S3 HTTP interface. But feel free to make sure it fails by `./gradlew clean test`.

Now it is time to make sure that someone is actually listening at port 4572 and this someone properly implements AWS S3 HTTP API. I will use localstack docker image and docker-java to start and stop the container. You will need docker daemon up and running to make this working.

Let's play a bit with localstack docker container. Specifically I am interested in container description got by `docker inspect ${CONTAINER_ID}`. I put `...` instead of major parts of output to highlight the most relevant parts.

```json
[
    {
        "Id": "602874b12142081e2942df877dfa483c7d271223702865ad8ebb9fe6292f1023",
        ...
        "HostConfig": {
            "Binds": [
                "/private/var/folders/22/v0pf_r7x7tj6dyn5w3ysx58r0000gp/T/localstack:/tmp/localstack",
                "/var/run/docker.sock:/var/run/docker.sock"
            ],
            "ContainerIDFile": "",
            "LogConfig": {
                "Type": "json-file",
                "Config": {}
            },
            "NetworkMode": "default",
            "PortBindings": {
                "443/tcp": [
                    {
                        "HostIp": "",
                        "HostPort": "443"
                    }
                ],
                ...
                "4572/tcp": [
                    {
                        "HostIp": "",
                        "HostPort": "4572"
                    }
                ],
                ...
                "8080/tcp": [
                    {
                        "HostIp": "",
                        "HostPort": "8080"
                    }
                ]
            },
            ...
        },
        ...
        "Mounts": [
            {
                "Type": "bind",
                "Source": "/private/var/folders/22/v0pf_r7x7tj6dyn5w3ysx58r0000gp/T/localstack",
                "Destination": "/tmp/localstack",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            },
            {
                "Type": "bind",
                "Source": "/var/run/docker.sock",
                "Destination": "/var/run/docker.sock",
                "Mode": "",
                "RW": true,
                "Propagation": "rprivate"
            }
        ],
        "Config": {
            ...
            "Env": [
                "HOST_TMP_FOLDER=/private/var/folders/22/v0pf_r7x7tj6dyn5w3ysx58r0000gp/T/localstack",
                "LOCALSTACK_HOSTNAME=localhost",
                "DOCKER_HOST=unix:///var/run/docker.sock",
                ...
            ],
            "Image": "localstack/localstack",
            "WorkingDir": "/opt/code/localstack",
            ...
        },
        "NetworkSettings": {
            ...
            "Ports": {
                "443/tcp": [
                    {
                        "HostIp": "0.0.0.0",
                        "HostPort": "443"
                    }
                ],
                ...
                "4572/tcp": [
                    {
                        "HostIp": "0.0.0.0",
                        "HostPort": "4572"
                    }
                ],
                ...
                "8080/tcp": [
                    {
                        "HostIp": "0.0.0.0",
                        "HostPort": "8080"
                    }
                ]
            },
            ...
        }
    }
]
```

Representing similar container configuration using docker-java API is shown below. Note usage utility functions `getTmpDir`. Delaying for a few seconds after container is started is necessary to make sure all localstack start-up and initialization routines are completed before usign the provided services.

```java
DockerClientConfig config = DefaultDockerClientConfig.createDefaultConfigBuilder()
        .withDockerHost("unix:///var/run/docker.sock")
        .build();

docker = DockerClientBuilder.getInstance(config).build();

ExposedPort tcp443 = ExposedPort.tcp(443);
ExposedPort tcp8080 = ExposedPort.tcp(8080);
ExposedPort tcp4572 = ExposedPort.tcp(4572);

Ports bindings = new Ports();
bindings.bind(tcp443, new Ports.Binding("0.0.0.0", "443/tcp"));
bindings.bind(tcp8080, new Ports.Binding("0.0.0.0", "8080/tcp"));
bindings.bind(tcp4572, new Ports.Binding("0.0.0.0", "4572/tcp"));

Volume volume = new Volume("/tmp/localstack");
String temp = getTmpDir("testing", "localstack");

CreateContainerResponse container = docker.createContainerCmd("localstack/localstack")
        .withName("testing-with-localstack")
        .withVolumes(volume)
        .withBinds(
                new Bind(temp, volume, AccessMode.DEFAULT, SELContext.DEFAULT),
                new Bind("/var/run/docker.sock", new Volume("/var/run/docker.sock")))
        .withEnv(
                "LOCALSTACK_HOSTNAME=localhost",
                "HOST_TMP_FOLDER=" + temp + "",
                "DOCKER_HOST=unix:///var/run/docker.sock")
        .withExposedPorts(
                tcp443,
                tcp8080,
                tcp4572)
        .withPortBindings(bindings)
        .exec();

containerId = container.getId();

docker.startContainerCmd(containerId).exec();
```

Once tests are finished, to stor and remove container these two line is enough.

```java
docker.stopContainerCmd(containerId).exec();
docker.removeContainerCmd(containerId).exec();
```

Now we are ready to start the container before testing and then stop/remove it afterwards.

The complete code runnable by gradle is available on [github][code-github].

[localstack-install]: https://stackoverflow.com/questions/31768128/pip-installation-usr-local-opt-python-bin-python2-7-bad-interpreter-no-such-f
[localstack-home]: https://localstack.cloud/
[localstack-github]: https://github.com/localstack/localstack
[dockerjava-github]: https://github.com/docker-java/docker-java
[docker-api]: https://docs.docker.com/develop/sdk/
[code-github]: https://github.com/sergey-melnychuk/java-docker-localstack-demo
