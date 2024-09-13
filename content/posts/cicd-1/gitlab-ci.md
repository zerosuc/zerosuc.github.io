---
title: "gitlab  runner"
date: 2024-08-21T14:51:10-08:00
draft: false
categories: ["gitlab", "runner"]
tags: ["gitlab", "runner"]
---

```
# Download the binary for your system
sudo curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64

# Give it permission to execute
sudo chmod +x /usr/local/bin/gitlab-runner

# Create a GitLab Runner user
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash

# Install and run as a service
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
sudo gitlab-runner start


sudo gitlab-runner register --url http://git.k8s.local/ --registration-token JLKyh3fUix_jzL-GZoRX


```
