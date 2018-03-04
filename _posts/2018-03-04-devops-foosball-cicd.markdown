---
layout: post
title:  "Devops for foosball: CI and CD"
date:   2018-03-4 11:45:22 -0400
categories: devops
---

There are tons of devops tools at our disposal, including tools for automating server configuration, testing software builds, and deploying whole applications and systems. But, do you know when to use what tool, and what problems each devops tool tries to solve?

In this series of posts, I will touch on the issues we face when developing software in an Agile environment, and I identify a devops solution to address that issue. Let's pretend that we're building a [foosball scoreboard system](https://github.com/IQ-Inc/foosball-scoreboard-demo), or something to keep track of foosball game scores. Yes, that's a real foosball scoreboad, written in Clojure, that talks with an Arduino hardware platform. But in our hypothetical, we're a team who builds foosball scoreboards for amateur and professional users, and we are interested in devops tools to improve our development process. We'll first focus on continuous integration and delivery services, one of the most relied-upon devops tools for Agile development teams.

## Continuous integration and delivery

Our team frequently adds new scoreboard features for our end-users. To help us implement new features, we introduce in-house and third-party libraries into the software build process. Of course, like any good software team, we also write automated tests for our foosball system. The tests show us that a feature works, and they serve as living documentation for how software modules perform. As we work on our scoreboard feature, we want to integrate our work with our co-workers' features. Not only does the integration process ensure that eveything jives, but it also brings in the dependencies and tests that our teammates can use.

Before we push our changes to the mainline, we want to build the repository and run the automated test suite. Running the tests on our own machines can be tedious, particularly if builds and integration tests take a long time to complete. But, how can we be sure we didn't break anything if we don't build the code and run the tests? No one wants to introduce bugs, so how can we quickly integrate features and ensure that we didn't break the build? Then, how do we prepare a release for our scoreboard users?

We need a tool for continuous integration (CI) and continuous delivery (CD). When we push a feature to the repository, A CI tool builds the code and runs all of our automated tests. We can configure the CI process to build and test every pull request, or if we want better insight, every batch of commits. If the build or tests fail, we get a Slack or email notification. A CI tool gives us insight into the health of our build, and it immediately alerts the team if there are issues.

We also need a tool for continuous delivery. After building and testing our software, our CD process deploys the application for our end-users. Since our foosball scoreboard runs on an end-user's local machine, we simply [make the binary available for download](https://github.com/IQ-Inc/foosball-scoreboard-demo/releases). However, we could imagine a future foosball service that runs in the cloud. A CD process could deploy that foosball service directly to a cloud platform, making it immediately accessible in a production environment. Most importantly, a CD process removes the tedium of preparing a release build for our users. The automatic process is free from human error once it's established.

Our foosball team uses [Travis](https://travis-ci.org/IQ-Inc/foosball-scoreboard-demo), a CI / CD service that can build, test, and deploy our application. Travis is one of a few services that treats the CI / CD definition as a software artifact. [Our configuration](https://github.com/mciantyre/foosball-scoreboard-demo/blob/master/.travis.yml) is checked directly into the repository. As of this writing, it looks something like this:

```yaml
language: clojure
script: lein do uberjar, test :all
deploy:
  provider: releases
  api_key:
    secure: bENcn3kvRKSlhXMzXf/efKbhONSy7JnhxEYW63/SZrjM45OFJ+yWSqC4dLhUqLacjKknGyatnp7Pkf6dwK88TzIxCpeFPk4d7OxYio2L52F5pm0yIR5Dz2ac/nm9SENRkX3uqkRol9iDfSJJ+8+IpaMygMm2NO9t9JHa1AppGcl+yQaCKs8sW9DQs/hVfXOGnsnoq6dQ+6GV0wLnftYTsonH/G92iq6W8fE71JWZr3oduo4tcbFDLBHRVPpO1cXPlo2FOJKuSsmgBJZy4HsVyR7ak1O28JRtfAhFD8bow4ucm+4eYuEYWhc3QlrhcWj0XbDkyl79m9ALhYSzauDfyUVkxHweKtImxeuGSzRRWxoZEvzHN97usXWF4Yt30Y6xPt6U3D/E8SKL+dOQlRSq3CD3mEEHddm8ZAVYO40YkSYOxoSWTzjUgMobUMw9hUScl+srF7keiN9YjNx5KjTNrh7MU/PlFH9ab2zcuH6lNWHAuB54yUnCReNA3YNRkair0DP7dMZxh6RBtwZMs4+TnHL9FY2v1Gw84XpOJka4jbOz6J1pZ4myDSyssPbNue7XMAYYDllPDUTAFsJ6hy8aVOeCQd+mvastznHOxaJgLeg1oUzLAdyAflQSAi1gY/wlyrwoEycXxBZqCszN4Bqrej9QXjhFrcsZJq8olnO9YhI=
  file: target/foosball-score.jar
  skip_cleanup: true
  on:
    repo: mciantyre/foosball-scoreboard-demo
    tags: true
```

We define the build as a Clojure project with the `language` key. For each batch of commits pushed to remote branches, run the scrip `lein do uberjar, test :all` to download dependencies, build the application, and run all the tests. We also specify a deploy to our GitHub repository's releases page. Whenever we tag a commit, upload the JAR file at `target/foosball-score.jar` so that our end-users can download the new release from GitHub. Travis uses the secure API key to interact with our repository.

Rather than maintaining a document that describes how to configure the CI / CD process, we can refer directly to the configuration file. Since it's checked into git, it follows the development of the software, and it has the same traceability of the source code.

There are many CI / CD services that are available for free and for sale. [Jenkins](https://jenkins.io) is another popular, well-supported, and free option for build, test, and deployment automation. The Jenkins ecosystem has plugins for almost every job you want to achieve. Unlike Travis, Jenkins jobs are normally defined in a graphical interface. Jenkins is also designed for running on your own infrastructure, whereas Travis is normally utilized as a hosted CI / CD solution.

For our simple foosball scoreboard application, we picked the easiest CI / CD process available. However, it is important to evaluate CI / CD solutions for your needs. We can compare Travis, Jenkins, and other CI / CD services across multiple dimensions, such as

- configuration management; is it a graphical configuration, or is it checked into the source code?
- hosting options; is it hosted on our infrastructure, or is it hosted off-site?
- what kinds of version control systems and services does it support?
- can it support our software language and build system?
- is it free and open source, or is it closed-source and licensed?
