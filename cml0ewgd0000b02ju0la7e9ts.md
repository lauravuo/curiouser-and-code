---
title: "Hacking the Orb"
datePublished: Sat Nov 14 2020 22:00:00 GMT+0000 (Coordinated Universal Time)
cuid: cml0ewgd0000b02ju0la7e9ts
slug: hacking-the-orb
canonical: https://dev.to/lauravuo/hacking-the-orb-54o1
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1769748782811/36ce9f54-0d34-4791-8f5d-eb020e5c3041.webp
tags: circleci, ci-cd

---

I remember times when setting up CI meant long hours of server setup, Jenkins studies, XML configuration pain, and Java memory issues. No wonder continuous testing and integration was not the first thing that came to mind when starting a new project.

Luckily, nowadays things are different. Services like [GitHub Actions](https://github.com/features/actions), [CircleCI](https://circleci.com/) or [Travis](https://travis-ci.org/) (just to name a few) are cloud services that handle the CI flows swiftly for you. Pipeline configurations can be stored neatly next to your code in the version control repository and the changes to them are picked up automatically whenever new commits are introduced. Furthermore, containerization technologies have simplified setting up external dependencies, so even complex end-2-end test environments can be configured only with a couple of lines of YAML.

**And the best part is that these services are usually free for open source projects.**

However, one thing that has puzzled me over the years when using the aforementioned services is how to avoid [DRY a.k.a. Don't Repeat Yourself](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) principle. Many projects have similar CI configurations especially if they are related or implemented using the same technology. In the past, I have found myself copy-pasting configurations between projects or repeating similar steps over and over again and this is something I would rather always avoid.

Let's take an example from the world of Node.js. Unit testing CI process for a node project consists typically of the following steps:

1. Select the base OS and OS version for the job
1. Select (and install) Node.js version
1. Clone the project sources
1. (If a cache is found) load cached dependencies 
1. Install dependencies
1. Run tests
1. Save dependencies to cache

Steps 1-2 may be combined if you use a system that allows the jobs to be run in a container: the base container can have all the global bells and whistles preinstalled that your project needs. But the rest of the steps you usually cannot avoid defining and actually only one, step number 6, interests me as the application or service developer. After figuring out and defining this process for the first time, I would like to be able to reuse the configuration in multiple projects.

**Fortunately, the CI services are constantly introducing new features and most of them have enabled the sharing of common modules in a way or another.**

Especially before GitHub Actions was made available for all GitHub users, my standard choice of CI was CircleCI. During last [Hacktoberfest](https://hacktoberfest.digitalocean.com/) I happened to notice that CircleCI had an extra challenge for crafting [orbs](https://circleci.com/orbs/). I got intrigued and it turned out that the orbs were just the thing I had been missing a few years back: *"A reusable package of YAML configuration that condenses repeated pieces of config into a single line of code."*

So I decided to give the orbs a go and replaced the unit test job in one of my personal Node.js projects with a job from an orb. The CI configuration was trimmed exactly as I wanted: I got rid of the lines related to cloning, dependencies installation, and cache handling, and all that was left was the test command. The CI configuration was trimmed by 15 lines and everything worked as before the change.

![Change commit](https://cdn.hashnode.com/res/hashnode/image/upload/v1769748779952/3ccb83c9-5ffb-4203-9e6c-69780068c246.png)

*[Example of orb usage](https://github.com/lauravuo/react-project-template/commit/d58224afe964fd72eea4bab0505cc8280cc0335b?branch=d58224afe964fd72eea4bab0505cc8280cc0335b&diff=split)*

In this case, the common functionality was authored by CircleCI. [Node orb](https://circleci.com/developer/orbs/orb/circleci/node) that I utilized in my personal project above provides Node.js related basic workflows related to dependency management and testing. But how about creating an own orb? The Hacktoberfest challenge was just about that, i.e. providing some meaningful CI functionality as an orb module that could be used by multiple repositories.

**As it happened, I had just come across an interesting documentation tool, [alexjs](https://alexjs.com/), that lints documentation files for insensitive and inconsiderate writing.**

Alex helps to find *"gender favouring, polarising, race related, religion inconsiderate, or other unequal phrasing"*. I thought that Alex would be a good addition to many of my projects, especially as I am a non-native English speaker and take gladly all the help I can get when authoring the project documentation. Also, the use of alex requires the npm toolchain as it is implemented with Node.js. So the additional motivation to implement an orb for alex was to use it in projects based on other technologies than Node.js.

CircleCI hosts [a registry](https://circleci.com/developer/orbs) for all available orbs so before you can use your self-authored orb in a CI workflow, you need to publish it to the registry. The easiest way to launch your orb development is to install [circleci cli](https://circleci.com/docs/2.0/local-cli/#installation) and use [an orb development kit](https://circleci.com/docs/2.0/orb-author/#orb-development-kit) that sets up [a project template](https://github.com/CircleCI-Public/Orb-Project-Template) for you to get started with the development. Make sure to [configure circleci cli](https://circleci.com/docs/2.0/local-cli/#configuring-the-cli) first if you try to use the development kit at home.

**The orb project template may seem overwhelming at first.**

In the source folder there are placeholders for the following sections:

| concept   | description                                                        |
|-----------|--------------------------------------------------------------------|
| commands  | steps and their parameters and mapping of those to shell scripts |
| examples  | use-case examples, displayed in registry documentation              |
| executors | environment in which the steps of a job will be run    |
| jobs      | full jobs definitions that may combine steps even from other orbs  |
| scripts   | shell scripts that implement the actual command functionality      |
| tests   | test scripts for the shell scripts      |

For me it took a while to get my head around these concepts in order to achieve the end result I wanted: provide a preconfigured job that would do the code checkout and linting with alex. The target was that the orb user could just import the orb and call the job in the workflow just as with Node.js testing example before.

The other thing that was a bit challenging to get working at first was the orb CI pipeline. The template project is preconfigured to use CircleCI for orb linting and testing. It even packs and publishes the development version of the orb to the registry and executes the integration tests with an actually deployed version of the orb before publishing the final production version. (I'd say nicely designed flow :blush:)

![Testing steps in CI](https://cdn.hashnode.com/res/hashnode/image/upload/v1769748781132/d47da667-b527-40e0-8f03-319f43e5ba85.png)

*CI execution log after successful PR merge and deployment*

However, with some documentation browsing, detective work and just trying out things, I finally got the things working as I had planned and as a result I was able to see [my orb in the registry](https://circleci.com/developer/orbs/orb/lauravuo/alexjs-orb). One key thing was to learn how to setup the CircleCI credentials to [the organization context](https://circleci.com/docs/2.0/contexts/#creating-and-using-a-context) so that CI was able to publish the orb on my behalf successfully.

Using the orb is now straightforward:

```yml
version: 2.1

orbs:
  alexjs-orb: lauravuo/alexjs-orb@0.0.1

workflows:
  lint:
    jobs:
      alexjs-orb/lint
```

And if all goes well, the job succeeds and logs can be observed from the CI output:

```bash
...
CHANGELOG.md: no issues found
README.md: no issues found
src/README.md: no issues found
src/commands/README.md: no issues found
src/examples/README.md: no issues found
src/executors/README.md: no issues found
src/jobs/README.md: no issues found
src/scripts/README.md: no issues found
src/tests/README.md: no issues found
src/tests/test.md: no issues found
CircleCI received exit code 0
```

The sources for my orb can be found in [GitHub](https://github.com/lauravuo/alexjs-orb) in case you want to take a closer look. In fact, currently all orbs published to the orb registry are open source. There is a definite need for private orbs as well, so it is interesting to see what happens in this front in the future.

What about you? Which CI system do you use and does it have similar support for shared functionality?

<span>Cover image by <a href="https://unsplash.com/@patmcmanaman?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Patrick McManaman</a> on <a href="https://unsplash.com/s/photos/circle?utm_source=unsplash&amp;utm_medium=referral&amp;utm_content=creditCopyText">Unsplash</a></span>