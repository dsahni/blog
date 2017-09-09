---
title: Chef Cookbook Pipeline with Jenkins or Gitlab
description: Chef Cookbook Pipeline with Jenkins or Gitlab
header: Chef Cookbook Pipeline with Jenkins or Gitlab
tags: [chef, dev]
---

## Pre-reqs
1. Provision and setup Jenkins
2. Provision and setup Gitlab
3. Have a chef-repo ready
4. Have a cookbook ready

## Setup Jenkins

### Plugins
The following plugin)s will be needed (along with versions that were tested at the time of this writing):
1. Git (2.4.0)
2. Gitlab Plugin (1.1.28)
3. ANSI Color

### Tools
SSH into the Jenkins server and install the following:
1. ChefDK (0.10.0)
2. Docker
3. kitchen-docker
4. jsonlint

```
# Fetch and install the Chef Development Kit
wget <chefdk url> | sudo yum install <chefdk>

# Install Docker from the CentOS 7 yum repo
sudo yum install <docker>

# Add the `jenkins` user to group `docker`


# Install `kitchen-docker` plugin for Test Kitchen
chef gem install kitchen-docker

# Install the `jsonlint` gem for validating JSON syntax
chef gem install jsonlint
```
Don't forget to add the Gitlab server to `known_hosts`!

### Chef Server credentials and settings
Create a directory on the Jenkins server to store our Chef Server credentials. This will be used by Berkshelf to upload our tested and approved cookbooks.

```
su - jenkins
mkdir -p ~/chef-configurations/.chef
mkdir -p ~/chef-configurations/knife
mkdir -p ~/chef-configurations/berks
mkdir -p ~/chef-configurations/scripts
```
We'll need the following in our `chef-configurations` dir:
* berks-east-nonprod.json
* berks-west-nonprod.json
* adminuser.pem
* knife-east-nonprod.rb
* knife-west-nonprod.rb

Example berks-east-nonprod.json:
```
{
	"chef": {
		"chef_server_url": "https://api.chef.io/organizations/dsahni-east-nonprod",
		"node_name": "adminuser",
		"client_key": "~/chef-configurations/adminuser.pem"
	},
	"ssl": {
		"verify": false
	}
}
```

Example knife-east-nonprod.rb:
```
log_level                :info
log_location             STDOUT
node_name                "adminuser"
client_key               "~/chef-configurations/adminuser.pem"
chef_server_url          "https://api.chef.io/organizations/dsahni-east-nonprod"
```

## Setup Gitlab
We'll need to create a separate user that Jenkins will be able to authenticate with. This 'service account' user will be posting comments on our projects, detailing if our builds were successful or not.

1. Create a new user in Gitlab ('Admin Area' > 'New User') called `jenkins`
2. Grab this user's Private Token, which is needed for the Jenkins -> Gitlab API interaction
  * Assuming you are an admin user, impersonate the newly created user byt clicking on `jenkins` user and `impersonate`. Then grab the user's Private Token from 'Account' on the left
3. Setup the Gitlab plugin in Jenkins by navigating to: 'Manage Jenkins' > 'Configure System' > 'Gitlab'
  * Gitlab Host URL
  * API Token will be the `jenkins` user's Private Token from Gitlab
  * Leave 'Ignore SSL Errors' checked
4. Add the SSH key of whatever user the Jenkins server runs as. In most standard installation, this user is also `jenkins`. It could also be `tomcat` if using a Bitnami image.
  * Paste the `id_rsa.pub` key from  the Jenkins server into Gitlab

```
ssh <$JENKINS_SERVER>
su - jenkins

# If this user does not have a public SSH key, generate one: `ssh-keygen -t rsa`
cat ~/.ssh/id_rsa.pub
```


## Setting up a Cookbook pipeline

Setup a new git repo, commit your cookbook into the new repo.

### Creating the Verify job
Name it "$COOKBOOK_NAME-verify".

1. Choose Git as SCM, add the Repository URL (ends in .git)
TODO: Need to secure this by authing with jenkins user on Gitlab

2. Add one more Respository URL: `${gitlabSourceRepoURL}`
This will be used to run on forked merge requests

3. Branches to build. Branch Specifier: `${gitlabSourceRepoName}/${gitlabSourceBranch}`

4. Check 'Build when a change is pushed to Gitlab', and check the following:
  * Build on Merge Request Events
  * Rebuild open Merge Events: `On push to source branch`
  * Enable [ci-skip]
  * Set build description to build cause (eg. Merge request or Git Push )
  * Add note with build status on merge requests
  * Vote added to note with build status on merge requests


5. Add `Execute Shell` as a build step, and have the following, either embedded or in a shell script for easier management:

```
#export PATH=/opt/chefdk/embedded/bin:$PATH

# Syntax
chef exec ruby -c ./*

#Lint
chef exec foodcritic .
chef exec rubocop

#Unit
chef exec rspec --format documentation --color
```

### Creating the Deliver job
Name it "$COOKBOOK_NAME-deliver".

1. Check 'This Build is Parametrized'. This will enable us to run this job ad-hoc, instead of just relying on a trigger. Useful if it fails for an unintended reason and needs to be re-run!

  Setup the following Text Parameter and Default Values (replacing URLs and Project names of your own, as necessary):
  * gitlabSourceRepoName
    * cookbooks/`$COOKBOOKNAME`
  * gitlabBranch
    * master
  * gitlabSourceRepoURL
    * git@`$GITLAB_URL`:cookbooks/`$COOKBOOKNAME`.git
  * gitlabSourceBranch
    * master
  * gitlabTargetBranch
    * master
  * gitlabActionType
    * Push

2. Add and check blah blah

3. Add `Execute Shell` as a build step, and have the following, either embedded or in a shell script for easier management:

```
#export PATH=/opt/chefdk/embedded/bin:$PATH

# Syntax
chef exec ruby -c ./*

#Lint
chef exec foodcritic .
chef exec rubocop

#Unit
chef exec rspec --format documentation --color

# Integration - only if kitchen-docker driver is specified
if grep --quiet docker .kitchen.yml; then
chef exec kitchen test
fi

# Berkshelf Upload
if [ -f Berksfile.lock ];
  then
    chef exec berks update
  else
    chef exec berks install
fi

chef exec berks upload -c ~/chef-configurations/berks-east-nonprod.json
chef exec berks upload -c ~/chef-configurations/berks-west-nonprod.json
```

### Setup Project Webhooks in Gitlab
We'll now need a way for Gitlab to trigger our Jenkins jobs based on activity that happens on our Projects. We want the Verify job to trigger only on the merge request's master branch, and the Deliver job to trigger on the project's master branch only when a human has approved and merged the request.

**Verify Webhook**
Under 'Settings > Web Hooks' for the project, add a webhook to the Jenkins project URL:
* URL: `http://$JENKINSURL/jenkins/project/$COOKBOOKNAME-verify`
* Check `Push Events` and `Merge Request Events`
* Uncheck `Enable SSL Verification`

**Delivery Webhook**
Add another webhook similar to above.
* URL: `http://$JENKINSURL/jenkins/project/$COOKBOOKNAME-deliver`
* Check `Push Events`
* Uncheck `Enable SSL Verification`

## Further Mods

### Filter Verify/Deliver jobs

### Chef Sync Job
This job can be scheduled, or run ad-hoc. Its purpose is to make sure all Chef servers have the same cookbooks and versions distributed amongst them.

### Promote Job


### Chef-Repo pipeline

Create a new Jenkins project called "chef-repo", copying from the existing "$COOKBOOKNAME-deliver" project so we have a base to start from.

Edit the following sections:
1. Build Parameters
  * gitlabSourceRepoName
  * gitlabSourceRepoURL
2. Source Code management
   * Repository URL

The shell commands in "Execute Shell" will look like the following:

```
# Check syntax for data bags, roles and environments
for d in data_bags/*/; do
chef exec jsonlint "$d"/*
done

chef exec jsonlint ./roles/*.json
chef exec jsonlint ./environments/*.json

# Upload data bags, roles and environments
chef exec knife upload -c ~/chef-configurations/knife/knife-ftw-poc.rb data_bags environments roles
```

Next, add the webhook in Gitlab for the repo, similar to what we did before.

* `http://54.165.127.80/jenkins/project/chef-repo-awsloft`
* "Push events"
