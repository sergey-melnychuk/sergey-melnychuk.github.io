---
layout: post
title:  "Testing AWS client with LocalStack"
date:   2018-04-01 14:45:00 +0200
categories: aws localstack docker unit-testing
---
Using AWS SDK always leaves one question open for me - how to introduce test coverage for the integration code. On one side, it doesn't make any sense to test/mock the code outside of your control (unless you're owning AWS SDK code, of course, but I suggest, most probably, you don't). On the other side, leaving code uncovered doesn't really look good as well.

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

```groovy
testCompile('com.github.docker-java:docker-java:3.0.14')
testCompile('org.glassfish.jersey.inject:jersey-hk2:2.26') // required for docker-java
```

```java
DockerClientConfig config = DefaultDockerClientConfig.createDefaultConfigBuilder()
        .withDockerHost("unix:///var/run/docker.sock")
        .build();

docker = DockerClientBuilder.getInstance(config).build();

Info info = docker.infoCmd().exec();
System.out.println("OS: " + info.getOperatingSystem());

ExposedPort tcp443 = ExposedPort.tcp(443);
ExposedPort tcp8080 = ExposedPort.tcp(8080);
ExposedPort tcp4572 = ExposedPort.tcp(4572);

Ports bindings = new Ports();
bindings.bind(tcp443, new Ports.Binding("0.0.0.0", "443/tcp"));
bindings.bind(tcp8080, new Ports.Binding("0.0.0.0", "8080/tcp"));
bindings.bind(tcp4572, new Ports.Binding("0.0.0.0", "4572/tcp"));

Volume volume = new Volume("/tmp/localstack");
String temp = makeTempDir("unitests", "localstack");

CreateContainerResponse container = docker.createContainerCmd("localstack/localstack")
        .withName("localstack-unit-testing")
        .withVolumes(volume)
        .withBinds(
                new Bind(temp, volume, AccessMode.DEFAULT, SELContext.DEFAULT),
                new Bind("/var/run/docker.sock", new Volume("/var/run/docker.sock"))
        )
        .withEnv(
                "LOCALSTACK_HOSTNAME=localhost",
                "HOST_TMP_FOLDER=" + temp + "",
                "DOCKER_HOST=unix:///var/run/docker.sock"
        )
        .withExposedPorts(
            tcp443,
            tcp8080,
            tcp4572
        )
        .withPortBindings(bindings)
        .exec();

containerId = container.getId();

docker.startContainerCmd(containerId).exec();
```

```java
    AmazonS3ClientBuilder builder = AmazonS3ClientBuilder.standard();
    AwsClientBuilder.EndpointConfiguration configuration =
            new AwsClientBuilder.EndpointConfiguration("http://localhost:4572", "us-east2");
    builder.setEndpointConfiguration(configuration);
    builder.withPathStyleAccessEnabled(true);
    s3 = builder.build();

```

[localstack-install]: https://stackoverflow.com/questions/31768128/pip-installation-usr-local-opt-python-bin-python2-7-bad-interpreter-no-such-f
[localstack-home]: https://localstack.cloud/
[localstack-github]: https://github.com/localstack/localstack
[dockerjava-github]: https://github.com/docker-java/docker-java
[docker-api]: https://docs.docker.com/develop/sdk/
