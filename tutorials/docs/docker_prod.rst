.. _docker_development:

Docker for Production CI: Zero to Hero
***************************************************

- `Overview`_
- `Preliminaries`_
- `Important Docker Commands`_
- `Exploring Your First Container`_
- `Understanding ROS Docker Images`_
- `For Docker-Based Development`_
- `For Docker-Based Deployment`_
- `Conclusion`_
- `Appendix`_

Overview
========

This tutorial is a hands on walkthrough of many of the optimizations that typical ROS teams use when using Docker as part of CI infrastructure.
This is not a comprehensive guide because some information is specific to the Github, Gitlab, or CircleCI.

Some other useful resources:

- `Best Practices for Dockerfile instructions <https://docs.docker.com/develop/develop-images/instructions/>`_ A brief set of steps to follow when writing Dockerfiles
- `Optimizing builds with cache management <hhttps://docs.docker.com/build/cache//>`_ for handling docker cache.
- `Dockerfile best practices <https://sysdig.com/blog/dockerfile-best-practices/>`_ for improving docker security

Preliminaries
=============

If you aren't familiar with ``docker_dev`` [TODO LINK], study up!

What does using docker in "production" mean? 

- You use docker as part of a continuous integration pipeline that builds and test ROS code every time you commit
- You use docker to deploy ROS code to an embedded robot that has limited network connectivity, so image size is important
- You have closed source code and want to ensure your docker images expose as little intellectual property in case they are compromised
- You supply docker images to another internal team regularly

This tutorial will build on the previous dockerfile, taking it from easy and workable to blazing fast, but complicated.
If you don't need the speed optimizations, don't spend the time adding these features as they take time to maintain.


Picking the starting image
==========================

When picking which image to start from, consider what that layer will be used for:
1) If it's used by developer, using the full desktop image provides all the build in tools
2) If it's used for deployment, and image size in a concern, start with ros:<ros-distro>-core

Taking a core image and installing a ton of dependencies every CI run takes a lot of time.
It is worth trying both options in CI and testing which is faster before making a decision.


Using rosdep to only install the minimum runtime
================================================

When deploying images, only the runtime dependencies need to be installed. Installing build and test dependencies is a waste of space.
Typical incantations of rosdep look like the following:

.. code-block:: bash

  rosdep install --from-paths src --ignore-src -y

In order to install only runtime dependencies, consider using `--dependency-types=exec`.

.. code-block:: bash

  rosdep install --from-paths src --ignore-src -y --dependency-types=exec
    

When using this flag, it is only worthwhile if packages separate out their build and runtime dependencies.
These dependency types are shown in REP-147: https://ros.org/reps/rep-0149.html

As an example, a package that uses boost program options should specify the following if boost-program-options show up in a library in the `PUBLIC` header:

.. code-block:: xml

  <exec_depend>libboost-program-options</exec_depend>
  <buildtool_depend>libboost-program-options-dev</buildtool_depend>
  <buildtool_export_depend>libboost-program-options-dev</buildtool_export_depend>

Many packages do not do this, and instead use the generic `<depend>` for all all the `dev` level dependencies:

.. code-block:: xml

  <depend>libboost-program-options-dev</depend>

If program options is only used in a Node (executable), this does not need to given as a `buildtool_export_depend`:

.. code-block:: xml

  <exec_depend>libboost-program-options</exec_depend>
  <buildtool_export_depend>libboost-program-options-dev</buildtool_export_depend>

If you notice any open source packages unnecessarily marking development dependencies with a generic `<depend>` attribute, please consider contributing fixes.
Sometimes, dev packages are all that is available in the rosdep sources: https://github.com/ros/rosdistro/blob/master/rosdep/base.yaml
Consider contributing new rosdep rules for runtime dependencies if they are missing for dev packages.



Speeding up rosdep installs with tuned COPY or mounts for cache preservation for CI
===================================================================================

In this section, we will take the previous dockerfile that cloned NAV2, and instead modify it preserve cache better.

First off, because most CI jobs clone the repository before starting the job, a common pattern is to use `COPY` from the local directory instead of `git clone`.
The COPY command will COPY all of NAV2 into the image build environment.

.. code-block:: bash

  WORKDIR /root/nav2_ws 
  COPY . ./src/navigation2
  RUN rosdep init
  RUN apt update && apt upgrade -y \
      && rosdep update \
      && rosdep install -y --ignore-src --from-paths src -r

Go ahead and rebuild the dockerfile.

Now, change a single line in the NAV2 README and rebuild.


  => [internal] load build definition from Dockerfile                                                                                                                                                                                                                                 0.0s
  => => transferring dockerfile: 480B                                                                                                                                                                                                                                                 0.0s
  => [internal] load metadata for docker.io/library/ros:rolling-ros-core                                                                                                                                                                                                              0.0s
  => [internal] load .dockerignore                                                                                                                                                                                                                                                    0.0s
  => => transferring context: 229B                                                                                                                                                                                                                                                    0.0s
  => [1/7] FROM docker.io/library/ros:rolling-ros-core                                                                                                                                                                                                                                0.0s
  => [internal] load build context                                                                                                                                                                                                                                                    0.4s
  => => transferring context: 1.79MB                                                                                                                                                                                                                                                  0.3s
  => CACHED [2/7] RUN apt update       && DEBIAN_FRONTEND=noninteractive apt install -y --no-install-recommends --no-install-suggests     ros-dev-tools     wget                                                                                                                      0.0s
  => CACHED [3/7] WORKDIR /root/nav2_ws                                                                                                                                                                                                                                               0.0s
  => CACHED [4/7] RUN mkdir -p ~/nav2_ws/src                                                                                                                                                                                                                                          0.0s
  => [5/7] COPY . ./src/navigation2 
  => [6/7] RUN rosdep init   


You should observe some steps are cached, but it calls the RUN command with `apt update` and `rosdep install` again!
But, we didn't change anything that affects what packages need to be installed, so why is it doing that?

This is called breaking the docker build cache, and is a cause of increase in CI execution times for many docker-based build systems.
On my computer, this stage took 348 seconds, which is signficant.
So, how do we fix it?

Well, the way `rosdep` install works is by examining the dependencies in the `package.xml`.
Thus, only the `package.xml` files should be copied in.

What if we try this:

.. code-block:: bash

  COPY **/package.xml ./src/navigation2/

Although `COPY` supports `glob` operations, it won't preserve directory structure, so that doesn't work.
https://github.com/moby/moby/issues/29211

Instead, we need all the `package.xml` files. With a big package such as NAV2, this is super tedious, so instead, script it!

.. code-block:: bash

  find . -type f -name package.xml | sort | xargs -I {} echo COPY {} {}

.. code-block:: bash

  COPY ./nav2_amcl/package.xml ./nav2_amcl/package.xml
  COPY ./nav2_behaviors/package.xml ./nav2_behaviors/package.xml
  COPY ./nav2_behavior_tree/package.xml ./nav2_behavior_tree/package.xml
  ...

Valid approach:

  WORKDIR /root/nav2_ws

  COPY ./nav2_amcl/package.xml ./nav2_amcl/package.xml
  COPY ./nav2_behaviors/package.xml ./nav2_behaviors/package.xml
  # Add the rest of the COPY commands here...

  RUN rosdep init
  RUN apt update && apt upgrade -y \
      && rosdep update \
      && rosdep install -y --ignore-src --from-paths src -r
  WORKDIR /root/nav2_ws 
  COPY **/*.package.xml ./src/navigation2
  RUN rosdep init
  RUN apt update && apt upgrade -y \
      && rosdep update \
      && rosdep install -y --ignore-src --from-paths src -r

Now, re-run the build, change a README, and observe the cache is preserved.

An improved alternative to COPY is bind mounts:
https://docs.docker.com/build/guide/mounts/

With slightly altered syntax using bind-mounts

.. code-block:: bash

  find . -type f -name package.xml | sort | xargs -I {} echo --mount=type=bind,source={},target={} \\

.. code-block:: bash

RUN --mount=type=bind,source=./nav2_amcl/package.xml,target=./nav2_amcl/package.xml \
  --mount=type=bind,source=./nav2_behaviors/package.xml,target=./nav2_behaviors/package.xml \
  # Add the rest of the mounts here...
  --mount=type=bind,source=./navigation2/package.xml,target=./navigation2/package.xml \
  apt update && apt upgrade -y \
  && rosdep update \
  && rosdep install -y --ignore-src --from-paths src -r


Now, when you change any code outside of a package.xml, this time-expensive layer is preserved. Woohoo!

Improving apt install speed by caching packages
===============================================

Each time the `Dockerfile` calls `apt-get update`, it must download packages.
Because internet bandwidth is finite, this can also slow down CI. 

Instead, add an apt cache mount as recommneded by builtkit: https://github.com/moby/buildkit/blob/master/frontend/dockerfile/docs/reference.md#example-cache-apt-packages

.. code-block:: bash

  RUN rm -f /etc/apt/apt.conf.d/docker-clean; echo 'Binary::apt::APT::Keep-Downloaded-Packages "true";' > /etc/apt/apt.conf.d/keep-cache

  RUN -mount=type=cache,target=/var/cache/apt,sharing=locked \
  --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt update && apt upgrade -y \
    && rosdep update \
    && rosdep install -y --ignore-src --from-paths src -r




Making use of multistage builds to separate the development build and colcon build
====================================================================================

For development and CI testing, where NAV2 is compiled, both jobs require the same `apt` packages. But, only the CI needs to build.
Instead of maintaining two `Dockerfiles`, you can make use of multi-stage builds to remove code duplication.
https://docs.docker.com/build/guide/multi-stage/

To start, lets look as some docker pseudocode of the what we will do.

.. code-block:: bash

  FROM ros:rolling as base

  RUN apt update && rosdep install

  FROM base as build

  RUN colcon build
  RUN colcon test


The `FROM ros:rolling as base` creates a base stage that can be built by itself and used by developers.
Then, CI can run the `build` stage with does the build and test. Because `build` starts from `base`,
docker will automatically build `base` first before `build`.

.. code-block:: bash

  # Developers use this command
  docker build . --target base

  # CI uses this command
  docker build . --target build

Using multistage builds for a minimal runtime
=============================================

How do you combine multi-stage builds with using rosdep to only install the minimal runtime?

Here is one way, with pseudocode:


.. code-block:: bash

  FROM ros:rolling-core as deploy-base

  # Install only runtime dependencies in this layer
  RUN apt update && rosdep install --dependency-types=exec

  FROM deploy-base as dev-base

  # Install the remaining build and test dependencies in this layer
  RUN apt update && rosdep install

  FROM dev-base as build

  RUN colcon build
  RUN colcon test

  FROM deploy-base as deploy

  # Copy the build artifacts from the build into a deploy layer
  COPY --from build install /opt/navigation2

By separating out the installation of runtime and build/test requirements into two layers, those dependencies can be used as needed.

Remove unnecessary files from deployment
==================================

By default, colcon build will generate all artifacts in to the `install` directory. 
If you follow the ROS tutorials, C++ header files will be part of the default `all` CMake component and installed in the image.

What is the actual different between `libboost-program-options-dev` and `libboost-program-options-dev`?
Using the following command, you can see it comes with manpages, headers, cmake files, 
dpkg -L libboost1.74-dev

For deployment, this is quite unnecessary. There are two options:
1) Modify the CMakeLists of all the packages to split them into the runtime and development libraries.
2) Remove the extra files in Docker

Option 1 is the "standard" way that package maintainers do this, but multi-target libraries are not common in ROS.
Option 2 is much less code to maintain, and perfectly fine for internal use. 

.. code-block:: bash

  # C++
  RUN find install -type f -name *.hpp | xargs rm
  RUN find install -type f -name *.h | xargs rm

  # CMake
  RUN find install -type f -name *.cmake | xargs rm

  # If you pre-compile python code
  RUN find install -type f -name *.py | xargs rm

When using this technique, make sure to use a multi-stage build otherwise the previous layers can just be extracted and your implementation details exposed!

Exposing specific ports instead of net=host
===========================================

While `--net=host` is a great option to provide network access during development, it's more secure to only expose the specific ports.

When using DDS, the ports are not static, but there is a calculator to figure out which ports to expose:

https://docs.ros.org/en/rolling/Concepts/Intermediate/About-Domain-ID.html#domain-id-to-udp-port-calculator

If you have one ROS 2 process, it uses on participant.

For example, when you run `ros2 run demo_nodes_cpp talker`, the following ports are used:


Looking at the process and network ports exposed:
  $ sudo lsof -n -i | grep talker
  talker    110592            ryan   12u  IPv4 392495      0t0  UDP *:7400 
  talker    110592            ryan   13u  IPv4 392499      0t0  UDP *:7412 
  talker    110592            ryan   15u  IPv4 392500      0t0  UDP *:7413 
  talker    110592            ryan   16u  IPv4 392504      0t0  UDP *:59050 
  talker    110592            ryan   17u  IPv4 392505      0t0  UDP 192.168.1.5:50658 
  talker    110592            ryan   18u  IPv4 392506      0t0  UDP 192.168.1.111:60996 

To start simple, with a hard coded `cmd` to launch three process all on `DOMAIN_ID` 0, it would use the following ports:

# Multicast
- 7400
- 7401
# Participant 0 unicast
- 7410
- 7411
# Participant 1 unicast
- 7412
- 7413
# Participant 2 unicast
- 7414
- 7413



Thus, using that in a docker run command, instead of `net=host`.

  $ docker run -p 7400:7401/udp -p 7410:7414/udp

Use apt-get and apt-install in CI
=================================

The `apt update` and `apt install` commands are not intended to be used CI. Replace these with `apt-get update` and `apt-get install`

Prevent duplication by re-using local scripts
=============================================

So far, we recommended splitting up runtime and development install into two different layers.
This causes code duplication because much of the commands are the same.

Making use of a container registry for pulling pre-built images
==============================================================

Using a dockerignore to ignore files unnecessary for build
==========================================================

Deploy in Release mode
======================

Make sure to build in release mode for deployment. It's handy to use `colcon-mixin`

Use Ninja (and gold?) for speed
======================

Export test results in CI JUnit format
======================


Maintainability anti-patterns for Production Packages
=====================================================

1. Do not use `rosdep install -r` in CI. This allows contributions to your `package.xml` to contain invalid rosdep keys. Instead, use `--skip-keys` for any unknown keys, and enforce the rest can resolve.
1. Do not use a docker `RUN` command to install package dependencies. This duplicates what is supposed to be done in rosdep, but is not re-usable to any other tools except that single Dockerfile.


Linting your Dockerfile
=======================

Wow, there was a ton of stuff in this dockerfile. Are there any tools to automate some of these best practices?

Sure! Use hadolint: https://github.com/hadolint/hadolint

You can integrate it with your CI or developerment workflow: https://github.com/hadolint/hadolint?tab=readme-ov-file#integrations



Conclusion
==========

At the end of this, you should be able to now:

- Pick a good starting image for the right use case
- Use rosdep to only install the minimum runtime packages
- Prevent cache-busting when using rosdep
- Use an apt cache for to skip downloading apt packages
- Use multistage builds for development,  CI, and minimal runtime
- Remove unnecessary files for deployment
- Expose only certain network ports for DDS
- Be aware of common anti-patterns

Its useful to note at this point that the ``--privileged`` flag is a real hammer. If you want to avoid running this, you can find all the individual areas you need to enable for visualization to work.
Also note that ``--privileged`` also makes it easier to run hardware interfaces like joysticks and sensors by enabling inputs from the host operating system that are processing those inputs.
If in production, you cannot use a hammer, you may need to dig into your system a bit to allow through only the interfaces required for your hardware.

As for potential steps forward: 

- Try adding some production optimizations to speed up your docker workflow
- Try reducing the image size of your repositories
- Hunt through your dependency chain to clean up runtime dependencies to only what is necessary

Each development, CI and production environment is unique.
This guide is a good starting point, however certain applications will require either more minimal image sizes, faster CI, higher security, or other concerns.

We hope this helps you speed up your development and increase safety in production!

-- Your Friendly Neighborhood Navigators

Appendix
========

Nav2 Production Development, CI, and Deployment Image
----------------------

This puts together all of the techniques used above into a single container.
Make sure to only use the techniques that benefit your use case because some have a maintenance burden!

.. code-block:: bash

  ARG ROS_DISTRO=rolling
  FROM ros:${ROS_DISTRO}-ros-core as deploy-base

  RUN apt update \
    && DEBIAN_FRONTEND=noninteractive apt install -y --no-install-recommends --no-install-suggests \
    ros-dev-tools \
    wget

  RUN rosdep init

  WORKDIR /root/nav2_ws 
  RUN mkdir -p ~/nav2_ws/src

  # Install only runtime dependencies in this layer
  RUN --mount=type=cache,target=/var/cache/apt,sharing=locked \
    --mount=type=cache,target=/var/lib/apt,sharing=locked \
    apt-get update && apt-get upgrade -y \
    && rosdep update \
    && rosdep 

  RUN apt update && rosdep install --dependency-types=exec

  FROM deploy-base as dev-base

  # Install the remaining build and test dependencies in this layer
  RUN apt update && rosdep install

  FROM dev-base as build

  RUN colcon build
  RUN colcon test

  FROM deploy-base as deploy

  # Copy the build artifacts from the build into a deploy layer
  COPY --from build install /opt/navigation2



  RUN apt update \
      && DEBIAN_FRONTEND=noninteractive apt install -y --no-install-recommends --no-install-suggests \
    ros-dev-tools \
    wget
    
  WORKDIR /root/nav2_ws 
  RUN mkdir -p ~/nav2_ws/src
  RUN git clone https://github.com/ros-planning/navigation2.git --branch main ./src/navigation2
  RUN rosdep init
  RUN apt update && apt upgrade -y \
      && rosdep update \
      && rosdep install --from-paths src --ignore-src -y --dependency-types=exec

Nav2 Deployment Image
---------------------

This image either downloads and installs Nav2 (Rolling; from source) or installs it (from binaries) to have a self contained image of everything you need to run Nav2.
From here, you can go to the :ref:`getting_started` to test it out! 

.. code-block:: bash

  ARG ROS_DISTRO=rolling
  FROM ros:${ROS_DISTRO}-ros-core

  RUN apt update \
      && DEBIAN_FRONTEND=noninteractive apt install -y --no-install-recommends --no-install-suggests \
    ros-dev-tools \
    wget

  # For Rolling or want to build from source a particular branch / fork
  WORKDIR /root/nav2_ws 
  RUN mkdir -p ~/nav2_ws/src
  RUN git clone https://github.com/ros-planning/navigation2.git --branch main ./src/navigation2
  RUN rosdep init
  RUN apt update && apt upgrade -y \
      && rosdep update \
      && rosdep install -y --ignore-src --from-paths src -r
  RUN . /opt/ros/${ROS_DISTRO}/setup.sh \
      && colcon build --symlink-install

  # For all else, uncomment the above Rolling lines and replace with below
  # RUN rosdep init
  # RUN apt update && apt upgrade -y \
  #     && rosdep update \
  #     && apt install \
  #         ros-${NAV2_BRANCH}-nav2-bringup \
  #         ros-${NAV2_BRANCH}-navigation2 \
  #         ros-${NAV2_BRANCH}-turtlebot3-gazebo


