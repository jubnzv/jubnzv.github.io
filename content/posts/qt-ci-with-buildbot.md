---
title: "Creating CI pipelines for C++/Qt applications with Buildbot"
date: 2021-01-18T09:23:31+03:00
toc: true
tags: [ci, buildbot]
---

## Introduction

I recently had to set up a Continuous Integration (CI) system to automate the build and testing of several proprietary C++ project based on Qt. I needed a self-hosted solution that works on a local network, and is as customizable and flexible as possible. Among other alternatives, I chose [Buildbot](https://buildbot.net/).

Buildbot is a Python framework for creating CI.
The task was not so simple, as there are very few tutorials on installing and configuring Buildbot. So I wrote this small article. It gives a very cursory overview of the framework and provides practical step-by-step guide to create your own configuration for Qt project. I assume that you are already familiar with Buildbot, otherwise you'll need to read [the official documentation](https://docs.buildbot.net/) while reading this article.

There won't be a full configuration and demo project here, because it depends on your project and your local area network infrastructure.
However, using this guide, you can set up your own CI pipeline without reading all the documentation.

In the first part we'll configure CI server and install the necessary components.
Next we'll take a closer look at the pipeline configuration process and create a very simple config.

As a result, we will get a CI that provides:
- build of the application with dependent components using different versions of Qt and toolchains
- extensibility and flexible configuration: the resulting system can be easily extended by running static analyzers, launching unit tests, etc.
- functioning withing a local network without external dependencies

To install, we'll need a Linux server connected to Internet. I'll use Debian 10 in this article.
I also assume that you are using git as version control system, however, the configuration for other VCS will not be much different.

## Preparing the server

We will install the Buildbot on the latest Debian Buster 10. For other distributions, the differences will only be the commands associated with using the package manager.

At the time of writing the actual version of Buildbot is 2.10.

For the demonstration we will deploy both Buildbot master and Buildbot worker on the same machine.
For a real system you'll probably want to use multiple workers on separate servers. This allows you to speed up the CI process and use different operating systems or environment settings for workers. This can be easily achieved because the configuration of the workers will be actually the same.

### Installing the Master

Let's start by installing Buildbot master. Buildbot master is a daemon that runs a web server that allows the end user to start new builds and to control the behavior of the Buildbot instance. The master also distributes builds to the workers.

First we need to install the required system packages:

```bash
apt-get update && apt-get install -y python3 python3-pip
```

Next, we will install the Buildbot components. We will use the `pip` package manager to get a more recent version:

```bash
pip3 install Buildbot[all] Buildbot-waterfall-view Buildbot-www
```

We need to create a user from which the build process will be performed. We also want to add him in the required groups:

```bash
useradd buildbot
sudo usermod -aG docker buildbot
sudo su buildbot
```

Now create a master:

```bash
cd /home/buildbot
buildbot create-master master
```

At this point, we will stop and configure worker on the same server. We will return to the master in a separate section.

### Installing a Worker

A worker is a daemon that connects to the master daemon and performs builds whenever master tells him to do so.

First install the system packages:

```bash
apt-get update && apt-get install -y ssh git python3 python3-pip
```

Then install the Buildbot components:

```bash
pip3 install buildbot-worker
```

We will also need to create a [secrets](https://docs.Buildbot.net/latest/manual/secretsmanagement.html) file to keep the new worker's password:

```bash
mkdir -p /home/buildbot/secrets/
echo 'password-1234' > /home/buildbot/secrets/local-worker-password
chmod 700 /srv/buildbot/secrets/local-worker-password
```

Finally, we can create a worker:

```bash
cd /home/buildbot/
Buildbot-worker create-worker worker localhost local-worker 'password-1234'
```

Write the IP address and port of the Buildbot master in `worker/buildbot.tac`:

```
buildmaster_host = 'localhost'
port = 9995
```

Now let's prepare the docker images needed in our CI pipeline.

#### Preparing docker images for Qt builds

Most code bases eventually migrate to new versions of the frameworks, being compiled by later versions of toolchains. You will probably also have to support building your application for multiple platforms, which is typical for embedded systems. This complicates our task a bit. Now we also want to build our project in multiple build environments to make sure that the new changes don't break backward compatibility on some platforms. To achieve this we will use docker.

Fortunately, there is a set of Dockerfile's [rabits/dockerfiles](https://github.com/rabits/dockerfiles) that contain a development environment with different versions of Qt.
This will make our task easier.
We will only need to download docker images with the required versions of Qt and add toolchain and additional dependencies needed to build our project.

Let's look at an example. We will create a sample Dockerfile with the `clang++` compiler from the Ubuntu package repository:

```dockerfile
FROM rabits/qt:5.13-desktop

# Install the required dependencies. Replace with your own components.
RUN apt-get update &&           \
    apt-get install -y clang++
```

When you're done writing your Dockerfile, start building a new image and tag it with a unique name:

```bash
docker build -t 'demo-project:qt-5.13-clang' .
```

Create docker images for all the required build environments and proceed to the next step.

### Configuring systemd

We also want to start a master and workers on a boot time. For this, we'll create systemd unit files so that the server's init system can manage the Buildbot processes.

First, we'll create and open a file named `/etc/systemd/system/buildbot-master.service`:

```systemd
[Unit]
Description=Buildbot Master
Wants=network.target
After=network.target

[Service]
User=Buildbot
Group=Buildbot
WorkingDirectory=/home/buildbot/master/
ExecStart=/usr/local/bin/buildbot start --nodaemon
ExecReload=/bin/kill -HUP $MAINPID
Restart=always
User=Buildbot

[Install]
WantedBy=multi-user.target
```

Next, we'll create `/etc/systemd/system/buildbot-worker.service` for the worker:

```systemd
[Unit]
Description=Buildbot Worker
Wants=network.target
After=network.target

[Service]
User=Buildbot
Group=Buildbot
WorkingDirectory=/home/buildbot/worker
ExecStart=/usr/local/bin/buildbot-worker start --nodaemon
Restart=always
User=Buildbot

[Install]
WantedBy=multi-user.target
```

Then reload systemd and add our services to autostart:

```bash
systemctl daemon-reload
systemctl enable --now Buildbot-worker
systemctl enable --now Buildbot-master
```

## Setting up a CI pipeline

Now let's move on to configuring the CI pipeline on the Buildbot master side.
First we'll copy sample `master.cfg.sample` config to `master.cfg`:

```bash
cp -v /home/buildot/master/master.cfg{.sample,}
```

Then, we'll work with `/home/buildot/master/master.cfg` that is a long Python script.
Open it with [your favorite text editor](www.vim.org) and proceed to the next section.

### General settings

First, we'll need to set up general settings with information about our environment. Add the appropriate URLs to your CI server and VCS web interface:

```python
c = BuildmasterConfig = {}
c["title"] = "Demo Project"
c["titleURL"] = "http://git-server/demo-user/demo-project/"
c["BuildbotURL"] = "http://buildbot-server:8010/"
c["www"] = dict(port=8010, plugins=dict(
    waterfall_view={},
    console_view={},
    grid_view={}))
c["db"] = {"db_url": "sqlite:///state.sqlite"}
c["secretsProviders"] = [secrets.SecretInAFile(dirname="/home/buildbot/secrets")]
c["workers"] = [worker.Worker("local-worker", util.Secret("local-worker-password"))]
c["protocols"] = {"pb": {"port": 9991}}
```

### Code bases

A [codebase](https://docs.buildbot.net/latest/manual/concepts.html) is a collection of related files and their history tracked as a unit by version control systems. The files and their history are stored in one or more repositories.

Add the necessary codebases for your projects:

```python
all_repositories = { 'ssh://git-server/demo-user/demo-project': 'demo-project' }
def codebaseGenerator(chdict):
    return all_repositories[chdict['repository']]
c['codebaseGenerator'] = codebaseGenerator
```

You will also need to create [change source](https://docs.buildbot.net/latest/manual/configuration/changesources.html) instances to make Buildbot know about our code bases.
Add this line in the config and proceed to the next section:

```python
c['change_source'] = [changes.GitPoller(str(url), pollinterval=60) for url in all_repositories.keys()]
```

### Schedulers

[Schedulers](https://docs.buildbot.net/latest/manual/configuration/schedulers.html) are responsible for initiating builds on builders. They decide how to react to incoming changes in your code bases and how to start workers.

In this section we will create two schedulers: "manual" [ForceScheduler](https://docs.buildbot.net/latest/manual/configuration/schedulers.html#forcescheduler-scheduler) and [SingleBranchScheduler](https://docs.buildbot.net/latest/manual/configuration/schedulers.html#singlebranchscheduler) that reacts on new commits in the `master`.

First, let's create `ForceScheduler` that will start build process after manual build request in the web interface:

```python
c["schedulers"] = []
c["schedulers"].append(schedulers.ForceScheduler(
    name="forced-scheduler",
    properties=[
            util.StringParameter(
                name='buildname',
                label='Build Name:',
                required=False,
            ),
    ],
    codebases=[
        forcesched.CodebaseParameter(
            codebase=name,
            branch="master",
            repository=forcesched.FixedParameter(name="repository", default="", hide=True),
            project=forcesched.FixedParameter(name="project", default="", hide=True),
            revision=forcesched.FixedParameter(name="revision", default="", hide=True),
        ) for name in ["demo-project"]
    ],
    builderNames=["local-builder"]))
```

Next we'll add a `SingleBranchScheduler` for the `master` branch:

```python
# Branch schedulers that watch over changes in projects.
c["schedulers"].append(schedulers.SingleBranchScheduler(
    name="changes-scheduler",
    # The scheduler will wait for this many seconds before starting the build.
    # If new changes are made during this interval, the timer will be restarted.
    treeStableTimer=60,
    change_filter=util.ChangeFilter(branch="master"),
    codebases=["demo-project"],
    builderNames=["local-builder"]))
```

After that, we can describe the commands included in our pipeline.

### Build steps

Buildbot performs commands execution in the pipeline via [Builders](https://docs.buildbot.net/latest/manual/concepts.html#concepts-builder).

We'll create a builder for our worker:

```python
c["builders"] = []
test_factory = util.BuildFactory()
c["builders"].append(util.BuilderConfig(
    name="local-builder",
    workernames=["local-worker"],
    factory=test_factory))
```

Next let's add some pipeline commands to the created builder.

First, we'll need to clone Git repository with our codebase. Add the following configuration:

```python
test_factory.addStep(steps.Git(
    name="git clone",
    repourl="ssh://local-git/demo-user/demo-project",
    alwaysUseLatest=True,
    method="copy",
    mode="full",
    codebase="demo-project",
    branch="master"))
```

By default, Buildbot worker will create `/home/buildbot/worker/local-builder/build/` directory and clone the repository there.

Next, let's add a step to build our application in the docker container created earlier:

```python
docker_command = """
docker run --privileged --rm -i                    \
--mount "type=bind,src=%(src_workdir)s,dst=/build" \
--workdir=/build/build                             \
--user "$(id -u):$(id -g)"                         \
%(container_name)s                                 \
/bin/bash -c "qmake && make -j`nproc`"
"""

test_factory.addStep(steps.ShellCommand(
    name="Qt 5.13 build",
    command=docker_command % {"src_workdir": os.path.join("/home/buildbot/worker/local-builder/"),
                              "container_name": "demo-project:qt-5.13-clang"}))
# ... add the same steps for other containers
```

The `docker_command` present here is a bit tricky. `–user "$(id -u):$(id -g)"` tells the container to run with the current user id and group id which are obtained dynamically through bash command substitution by running the `id -u` and `id -g` and passing on their values. We need this to avoid problems with permissions, because we want to compile source to object files and save them mounted volume.

## Testing the system

In the end, we got a minimal CI pipeline that should work right now.

Restart master instance:

```bash
systemctl restart buildbot-master
```

And open web interface, specified in the configuration. If all work correctly, you should see something like this:

![](/posts/qt-ci-with-buildbot/welcome-screen.png)

If the interface is not displayed, I recommend looking at the logs by this way:

```bash
tail -f /home/buildbot/master/twistd.log
```

Most often, errors are well described in the logs, and you can easily find problems in your config.
