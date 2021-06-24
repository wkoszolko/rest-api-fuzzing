# Rest Api Fuzzing
The main goal of this repository is to provide examples of how to use REST API Fuzzing tools for 
automatically testing cloud services through their REST APIs and finding security and reliability bugs in these services.

## Importance of Rest API Fuzzing

Maintaining high-quality APIs is expensive. It's difficult to write unit/integration 
tests that cover all edge-cases corner-cases and find all security vulnerabilities. 
These APIs are consantly evolving, imply an increased attack surface and thus increasing demand for testing to keep up.

Also, working with legacy systems without proper test raise in an increased need to provide coverage in order to reduce the cost of development.

## Types of bugs 

Rest API Fuzzers are able to find the following classes of bugs:

1. API Specification
2. Input validation
3. Data leaks
3. Resource leaks
4. Security vulnerabilities
5. Regression

## Available tools

Below you can find examples of REST API Fuzzers. 
Each tool is capable of automatically testing your APIs based on Swagger/OpenApi specification.

| Tool | Description  |
|-------|----|
| [ZAP](https://www.zaproxy.org/) | Primary, ZAP is web app scanner. ZAP can test REST and SOAP APIs. Developed by OWASP community. |
| [RESTler](https://github.com/microsoft/rest-api-fuzz-testing) | Stateful API Fuzzer. Resources created by earlier requests are consumed by subsequent requests. Developed by Microsoft. |
| [Dredd](https://github.com/apiaryio/dredd) | Ability to build test similar to unit-test using [hooks](https://dredd.org/en/latest/hooks/index.html#hooks). Hooks allows us to setup context and clean-up after a tests execution  |
| [Schemathesis](https://github.com/schemathesis/schemathesis) | Supports REST and GraphQL |
| [GitLab API Fuzzer](https://docs.gitlab.com/ee/user/application_security/api_fuzzing/) | Build-in API fuzzer. The fuzzer is avaiable in GitLab Ultimate |

## How to use API Fuzzers 
This repository contains 2 examples of API Fuzzer usage:
 - ZAP
 - RESTler
 
I decided to use [spring-petclinic-rest](https://github.com/spring-petclinic/spring-petclinic-rest) for testing. 
[Spring-petclinic-rest](https://github.com/spring-petclinic/spring-petclinic-rest) is a clone of very popular [Spring Pet Clinic Demo Project](https://spring-petclinic.github.io/). 
This app provides REST API and exposes Swagger/OpenAPI specification.  

The app is available on [DockerHub](https://hub.docker.com/r/springcommunity/spring-petclinic-rest). 
Running the app under tests:
```shell script
docker run -p 9966:9966 springcommunity/spring-petclinic-rest
```
Rest API definition: http://localhost:9966/petclinic

API docs: http://localhost:9966/petclinic/v2/api-docs

### OWASP ZAP

ZAP is used for API security testing. Zed Attack Proxy (ZAP) is a free, open-source penetration testing tool. At its core, ZAP is what is known as a “man-in-the-middle proxy.”
For this use case, ZAP is run in headless mode with additional add-ons. During the test, ZAP:
1. Imports the Rest API definition
2. Scans the API
3. Reports issues

This example runs default Scan Policy tuned for APIs:
```shell script
# run the app under tests
docker run --rm -d -p 9966:9966 --name petclinic-rest springcommunity/spring-petclinic-rest

#start fuzzing in debug mode
docker run --rm --link petclinic-rest -v $(pwd):/zap/wrk/:rw -t owasp/zap2docker-weekly zap-api-scan.py -t http://petclinic-rest:9966/petclinic/v2/api-docs -f openapi -r api_zap_report.html -d

#cleanup after fuzzing
docker rm -f petclinic-rest
```
All start-up parameters are well documented on [ZAP webpage](https://www.zaproxy.org/docs/docker/api-scan/).

The test takes a few minutes, depends on your scan policy. 
After successful scan, you can find report `api_zap_report.html` in your working directory.

![ZAP Report](ZAP-report.png?raw=true "ZAP Report")

ZAP Report contains list of issues. For each issue, ZAP provides exact details and a solution.

## Microsoft RESTler

RESTler is a stateful REST API fuzzing tool for automatically testing.
For a given OpenAPI/Swagger specification, RESTler analyzes its entire specification and then generates and executes tests that exercise the service through its REST API.

RESTler provides a docker image but the official docker image is not very convenient for simple usage.  
This example uses [simplified RESTler docker image](https://github.com/wkoszolko/restler-fuzzer-getting-started).

Before you start fuzzing, you have to clone repository [restler-fuzzer-getting-started](https://github.com/wkoszolko/restler-fuzzer-getting-started) 
and build docker image according to [README](https://github.com/wkoszolko/restler-fuzzer-getting-started#readme).

Running Rest API fuzzing:
```shell script
cd ./restler

# run the app under tests
docker run -d --rm -p 9966:9966 --name petclinic-rest springcommunity/spring-petclinic-rest

#start fuzzing 
docker run --rm --link petclinic-rest -v $(pwd):/fuzzer:rw wkoszolko/restler-fuzzer-getting-started fuzz --compilation_config=config.json --settings=engine_settings.json --no_ssl=true

#cleanup after fuzzing
docker rm -f petclinic-rest
```
RESTler has multiple variants of fuzzing:
- smoke tests
- fuzz-lean
- fuzzing

The example runs all tree types of test. After successful scan, you can find tests results and logs in `./restler` directory.
List of aggregated results:
- restler/Test/ResponseBuckets/errorBuckets.json
- restler/Test/ResponseBuckets/runSummary.json
- restler/FuzzLean/ResponseBuckets/errorBuckets.json
- restler/FuzzLean/ResponseBuckets/runSummary.json
- restler/Fuzz/ResponseBuckets/errorBuckets.json
- restler/Fuzz/ResponseBuckets/runSummary.json

## REST Api fuzzing results
ZAP and RESTler fuzzers are able to find multiple API issues. Most of the problems are related to API design and the quality of the implementation.  

Fuzzing [spring-petclinic-rest](https://github.com/spring-petclinic/spring-petclinic-rest) finds the following issues:
- Low quality of API documentation: lack of examples in API docs leads to lower coverage.
- API design issues: using database entities as API input parameters. If the client sets id filed, the API crashes because the id field is set by an identity column.
- Security vulnerabilities e.g improper error handling: error response often provides useful information to an attacker (stack trace, name of libraries, and frameworks).

