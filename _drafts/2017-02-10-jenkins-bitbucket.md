---
title: Chef Cookbook Pipeline with Jenkins or Gitlab
description: Chef Cookbook Pipeline with Jenkins or Gitlab
header: Chef Cookbook Pipeline with Jenkins or Gitlab
tags: [chef, dev]
---

# Overall Steps
1. Install Jenkins v2.64
2. Go to `Manage Jenkins` and set the following environment variable: `JENKINS_URL = <your jenkins url>`
3. Install [Bitbucket Branch Source Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Bitbucket+Branch+Source+Plugin) v2.1.2
4. Install ChefDK
5. Install [Bitbucket Build Status Notifier Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Bitbucket+Cloud+Build+Status+Notifier+Plugin) v1.3.3
