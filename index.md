---
title: "Docker Scout: Remediating CVEs in a Container Image"
description: |
  Learn how to use Docker Scout to analyze and remediate vulnerabilities in a container image.

kind: challenge
playground: docker

createdAt: 2024-05-22
updatedAt: 2024-05-22

difficulty: easy

categories:
  - containers
  - security

tagz:
  - docker-scout
  - docker

tasks:
  install_docker_scout:
    init: true
    run: |
      mkdir -p ~/.docker
      curl -sSfL https://raw.githubusercontent.com/docker/scout-cli/main/install.sh | sh -s --
      git clone https://github.com/docker/scout-demo-service.git --depth 1 /root/scout-demo-service
      cd /root/scout-demo-service
      git checkout 0de180be8a6e8deb3b63cd52b0811b3a23d91c31

  verify_docker_login:
    run: |
      if [ -z $(cat ~/.docker/config.json | jq -r ".auths[].auth") ]; then
        exit 1
      else
        exit 0
      fi

  verify_image_v1:
    needs:
    - verify_docker_login
    hintcheck: |
      if ! docker image inspect scout-demo:v1 > /dev/null 2>&1; then
        echo "To build the image, go to the ~/scout-demo-service directory and run 'docker build -t scout-demo:v1 .'"
      fi
    run: |
      if docker image inspect scout-demo:v1 > /dev/null 2>&1; then
        docker scout cves scout-demo:v1 | grep -oP '\b(\d+) vulnerabilit.+ found' | sed -n 's/\(\d\+\).*/\1/p' | awk '{print $1}' > /tmp/cve_count_v1.txt
        exit 0
      else
        exit 1
      fi

  verify_image_v2:
    needs:
    - verify_image_v1
    hintcheck: |
      if ! docker image inspect scout-demo:v2 > /dev/null 2>&1; then
        echo "To build the image, go to the ~/scout-demo-service directory and run 'docker build -t scout-demo:v2 .'"
      fi
    run: |
      if docker image inspect scout-demo:v2 > /dev/null 2>&1; then
        # echo "Checking if image contains a package.json with id @docker/scout-demo-service"
        # output=$(docker run --rm scout-demo:v2 /bin/sh -c "cat package.json | grep @docker/scout-demo-service | awk -F '\"' '{print \$4}'")
        # if [[ $output != *"No such file or directory"* ]]; then
          echo "v2 image exists, getting CVE count..."
          docker scout cves scout-demo:v2 | grep -oP '\b(\d+) vulnerabilit.+found' | sed -n 's/\(\d\+\).*/\1/p' | awk '{print $1}' > /tmp/cve_count_v2.txt

          CVE_COUNT_V1="$(cat /tmp/cve_count_v1.txt)"
          CVE_COUNT_V2="$(cat /tmp/cve_count_v2.txt)"

          if [ "${CVE_COUNT_V2:-0}" -lt "${CVE_COUNT_V1:-0}" ]; then
            echo "v2 has fewer CVEs than v1"
            exit 0
          else
            echo "v2 has the same or more CVEs than v1"
            exit 1
          fi
        # else
        #   echo "It looks like this image doesn't correspond with the scout-demo app."
        #   exit 1
        # fi
      else
        echo "Image not found"
        exit 1
      fi

---

In this challenge, you will need to scan an image for vulnerabilities using [Docker Scout](https://www.docker.com/products/docker-scout/),
identify the most severe ones, and produce a new version of the image with fewer vulnerabilities in it.

## Prerequisites

Docker Scout expects your Docker installation to be authenticated with Docker Hub.
You can log in using the `docker login` command with a **test Docker Hub account** and a limited-scope access token (`Public Repo Read-only`) that you [can easily generate specifically for this challenge](https://hub.docker.com).
**Usage of personal and especially production Docker Hub accounts is strongly discouraged.**

::remark-box
---
kind: warning
---

When you're done with the challenge, don't forget to log out with `docker logout`.
Or simply terminate the playground VM 😉
::


::simple-task
---
:tasks: tasks
:name: verify_docker_login
---
#active
Waiting for `docker login` to complete...

#completed
Yay! You've logged in 🎉 Now you can use Docker Scout to scan images.
::

## Build the image
The directory `~/scout-demo-service` contains a vulnerable Node.js application that you can use to follow along.
Move into the directory and build the image locally with the tag `scout-demo:v1`.

::simple-task
---
:tasks: tasks
:name: verify_image_v1
---
#active
Waiting for `scout-demo:v1` image to be built...

#completed
Yay! The `scout-demo:v1` container image was built 🎉
::

## Analyze image vulnerabilities

After building the image, use the `docker scout` CLI command to see vulnerabilities detected by Docker Scout.
This challenge uses a vulnerable version of Express and an outdated base image.

::hint-box
---
:summary: Hint 1
---

It's an easy one - `docker scout cves --help` is your friend.
::

::hint-box
---
:summary: Hint 2
---

You can narrow down the results by filtering by package name with `docker scout cves scout-demo:v1 --only-package express`
::


## Fix application vulnerabilities

Can you rebuild the image using a slimmer base image and upgrading your `package.json` dependencies on the way so that the new image has fewer vulnerabilities?

1. Use a slimmer or a most recent version of the current base image that includes fewer vulnerabilities.
2. Update the underlying vulnerable express version to a specific version or later.
3. Build a new image with name `scout-demo:v2`.

::simple-task
---
:tasks: tasks
:name: verify_image_v2
---
#active
Waiting for `scout-demo:v2` image to be built...

#completed
Yay! The `scout-demo:v2` image has fewer vulnerabilities than the original 🎉
::

::hint-box
---
:summary: Hint 3
---

Use a more recent version of the base image, such as:

```
- FROM alpine:3.14
+ FROM alpine:3.19
```
::

::hint-box
---
:summary: Hint 4
---

Update the `package.json` file with the new `express` version.

```
   "dependencies": {
-    "express": "4.17.1"
+    "express": "4.17.3"
   }
```
::

## Analyze the new image

🎉 Congratulations!
If you analyze the new image for CVEs again, you'll see that several vulnerabilities have been fixed.


## What's next?

There's a lot more to discover in Docker Scout,
from third-party integrations, to policy customization,
and runtime environment monitoring in real-time.

Check out the following sections:

- [Image analysis](https://docs.docker.com/scout/image-analysis/)
- [Data sources](https://docs.docker.com/scout/advisory-db-sources/)
- [Docker Scout Dashboard](https://docs.docker.com/scout/dashboard/)
- [Integrations](https://docs.docker.com/scout/integrations/)
- [Policy evaluation](https://docs.docker.com/scout/policy/)
