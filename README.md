# pact-sample
A sample project showing how to use Pact to implement consumer-driven tests in Groovy.

## the idea

With the increasing popularity of microservices, it can be quite hard to track the impact of a change done to a given service over consumers of this service. Consumers of this service can be other microservices or user interfaces.

A quite common setting is to have a RESTful Web API acting as a backend component to single-page app, responsible for acting as an UI to the end user. In such a setting, breaking changes can be introduced to the backend without the people responsible for the frontend being aware of it, causing disturbances to the experience provided to user.

One way of overcoming this issue is to use [Consumer-Driven Contracts](https://martinfowler.com/articles/consumerDrivenContracts.html). Such technique proposes that the consumer of the information defines a contract between itself and the producer of the information, and both parties should conform to that contract at all times.

This project is to demonstrate in a very simple and concise manner how to implement Consumer-Driven Contracts between two parties.

You'll notice there are two sub-folders here:

* **consumer**: a very simple command-line interface that accepts only one command: `status`. When invoked, this interface will connect to a backend component via HTTP, retrieve information about its availability and display it to the user. Two pieces of information should be displayed to the user: the backend status (it should be `OK` at all times, hopefully) and the date when that information was provided. Pretty basic, very very simple, our focus is **not** in showing off CLI skills, but rather to show how Consumer-Driven Contracts work on the client side.

* **producer**: a very basic Spring-Boot service, with just one endpoint (`/status`) that accepts `GET` calls. When a `GET /status` request is issued against this service, a JSON response should be sent back (e.g.: `{"status":"OK","currentDateTime":"2017-06-27T13:54:29.214"}`). Again, very basic, very simple and minimalistic, the focus is not to come up with a super fancy service, but rather to demonstrate how a backend component can comply with a contract defined by its consumers.

## the implementation

This project uses [Pact-JVM](https://github.com/DiUS/pact-jvm), a spin-off from [Pact](https://github.com/realestate-com-au/pact) to implement Consumer-Driven Contracts.

The underlying programming language chosen for both consumer and provider is Groovy.

## implementing the idea

### references

Before diving into code, I'll paste here the sources of information I've used to implement this example:
* https://github.com/DiUS/pact-jvm/tree/master/pact-jvm-consumer-groovy
* http://www.chuanchuanlaw.com/pact-how-to-write-consumer-test/
* http://www.chuanchuanlaw.com/pact-how-to-write-provider-test/
* http://dius.com.au/2016/02/03/microservices-pact/
* https://github.com/mstine/microservices-pact
* https://github.com/DiUS/pact-workshop-jvm

### step 0: creating the project

Well, the technique is called "**Consumer-Driven** Contracts" for a reason. So I guess it makes sense to start by the consuming part of the project :)

So, to create the project, I just used Gradle to create a basic project: `gradle init --type groovy-library`. A small project containing a `Library` class, with its respective test was created.

After removing some of the code generated by Gradle and adding the dependencies to use Pact-JVM with Groovy, `build.gradle` looks like this:

```
apply plugin: 'groovy'

repositories {
    jcenter()
}

dependencies {
    compile 'org.codehaus.groovy:groovy-all:2.4.11'

    testCompile 'org.spockframework:spock-core:1.0-groovy-2.4'
    testCompile 'org.codehaus.groovy.modules.http-builder:http-builder:0.7'
    testCompile 'au.com.dius:pact-jvm-consumer-groovy_2.11:3.5.0'
    testCompile 'au.com.dius:pact-jvm-consumer-junit_2.11:2.1.13'
}
```

In case you're asking what's `http-builder` is doing there, it's where Groovy's `RESTClient` sits, and this will be quite handy to implement the remote calls needed for the test.

### step 1: implementing the contract test

As previously stated, the consumer component...

* relies on a `/status` endpoint made available by the producer;
* expects such endpoint to accept `GET` calls and return a JSON object containing two attributes: `status` and `currentDateTime`;

Such expectations should then be clearly stated in the contract to be held between consuming and producing parties.

`StatusEndpointPact.groovy` below depicts how this contract is proposed by the consumer.

```
import org.junit.Test
import au.com.dius.pact.consumer.PactVerificationResult
import au.com.dius.pact.consumer.groovy.PactBuilder
import groovyx.net.http.RESTClient

class StatusEndpointPact {

    @Test
    void "pact for /status"() {
        def statusEndpointPact = new PactBuilder()

        statusEndpointPact {
            serviceConsumer "Status CLI" 	          // Define the service consumer by name
            hasPactWith "Status Endpoint"           // Define the service provider that the consumer has a pact with
            port 1234                               // The port number for the service. It is optional, leave it out to use a random one

            given('status endpoint is up')
            uponReceiving('a status enquiry')
            withAttributes(method: 'get', path: '/status')
            willRespondWith(
                    status: 200,
                    headers: ['Content-Type': 'application/json'],
                    body: '{"status":"OK","currentDateTime":"2017-06-27T13:54:29.214"}'
            )
        }

        // Execute the run method to have the mock server run.
        // It takes a closure to execute your requests and returns a PactVerificationResult.
        PactVerificationResult result = statusEndpointPact.runTest {
            def client = new RESTClient('http://localhost:1234/')
            def response = client.get(path: '/status')

            assert response.status == 200
            assert response.contentType == 'application/json'
            assert response.data.status == 'OK'
        }

        assert result == PactVerificationResult.Ok.INSTANCE  // This means it is all good
    }
}
```
