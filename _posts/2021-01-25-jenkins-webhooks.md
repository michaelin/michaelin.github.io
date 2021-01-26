---
layout: single
title: Using Jenkins webhooks
tags:
  - Jenkins
  - Getting started
---

It is possible to trigger builds in Jenkins by calling specific endpoints on the Jenkins server. This makes it possible for other servers to trigger builds of specific projects on your server.

## Triggering builds using a URL token

- Configure a unique token for the Jenkins build
  - Configure job.
  - Enable _Trigger builds remotely_.
  - Enter a long, random character sequence in the _Authentication Token_ field.
    - You can use the password generator in your favorite password manager or on [random.org](https://www.random.org/passwords/?num=2&len=16&format=html&rnd=new).
    - The token should be unique for each job.
  - Remember to save.

Trigger the build by calling the following url with curl from a terminal or using your browser:

```text
https://<JENKINS_URL>/job/<JOB_NAME>/build?token=<AUTHENTICATION_TOKEN>
```

A build should now be triggered on the selected job. You can now configure the URL in the service you need to trigger a build from.

The main issue with this option is that it requires `anonymous` users to have full read access to the jobs. This will cause a `HTTP/403` error.

## Using URL tokens without anonymous read access

- Install the [Build Authorization Token Root](https://plugins.jenkins.io/build-token-root) plugin.

Trigger the build on the following url:

```text
https://<JENKINS_URL>/buildByToken/build?job=<JOB_NAME>&token=<AUTHENTICATION_TOKEN>
```

The endpoint will automatically be available to anonymous users, even if anonymous read access is disabled.

## Troubleshooting

- Enable the [Jenkins access log](https://wiki.jenkins.io/display/JENKINS/Access+Logging).
- Check that your firewall allows traffic from the triggering service to your Jenkins server.
