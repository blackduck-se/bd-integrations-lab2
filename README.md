# Black Duck Software Integration Lab 2

The goal of this lab is to provide hands on experience integrating a **Polaris** scan into a **GitLab** pipeline using the [Black Duck Security Scan Template](https://gitlab.com/blackduck-inc/black-duck-security-scan) and demonstrating its post scan capabilities. As part of this lab, we will:
1. setup a GitLab pipeline and execute a full scan
2. break the build based on a [policy](https://polaris.blackduck.com/developer/default/polaris-documentation/t_post_scan_policies) defined in Polaris
3. review the findings in Polaris
4. review the exported SARIF report
5. introduce a vulnerable code change, triggering a PR scan which adds a comment to the Pull Request

This repository contains everything you need to complete the lab except for the prerequisites listed below.

# Prerequisites

1. [signup](https://gitlab.com/users/sign_up) for a free GitLab Account
2. [create](https://polaris.blackduck.com/developer/default/polaris-documentation/t_make-token) a Polaris Access Token

# Clone repository

1. Clone this repository into your GitLab account via _GitLab → Projects → New Project → Import Project → Repository by URL_
   - repostiory url: https://github.com/blackduck-se/bd-integrations-lab2.git
   - username & password not needed
   - optionally change project slug (repository name)
   - set visbility to _public_ (for debugging purposes)

# Setup pipeline

1. Create a project access token with Developer role and API scope via _GitLab → Project → Settings → Access Token → Add New Token_
2. Add the following variables via _GitLab → Project → Settings → CI/CD → Variables_. Be sure to select **masked** for the tokens and unset **protect** for all three.
   - POLARIS_SERVERURL
   - POLARIS_ACCESSTOKEN
   - GITLAB_USER_TOKEN
3. Add a coverity.yaml to the project repository via _GitLab → Project → New file (plus icon top middle)_

```
capture:
  build:
    clean-command: mvn -B clean
    build-command: mvn -B -DskipTests package
analyze:
  checkers:
    webapp-security:
      enabled: true
```

4. Create a new pipeline via _GitLab → Project → Build → Pipeline Editor → Configure Pipeline_

```
include:
- project: blackduck-inc/black-duck-security-scan
ref: v2.0.0
file: templates/security_scan.yml

stages:
- build
- test
- security
- deploy

variables:
SCAN_BRANCHES: "/^(main|master|develop|stage|release)$/"

cache:
paths:
- .m2/repository/
- target/

image: maven:3-eclipse-temurin-21

build:
stage: build
script: mvn -B compile

test:
stage: test
script: mvn -B test

deploy:
stage: deploy
only:
variables:
- $CI_COMMIT_REF_NAME =~ $SCAN_BRANCHES
script: mvn -B install

polaris:
stage: security
rules:
- if: (($CI_COMMIT_BRANCH =~ $SCAN_BRANCHES && $CI_PIPELINE_SOURCE != 'merge_request_event') ||
($CI_MERGE_REQUEST_TARGET_BRANCH_NAME =~ $SCAN_BRANCHES && $CI_PIPELINE_SOURCE == 'merge_request_event'))
variables:
BRIDGE_POLARIS_SERVERURL: $POLARIS_SERVERURL
BRIDGE_POLARIS_ACCESSTOKEN: $POLARIS_ACCESSTOKEN
BRIDGE_POLARIS_ASSESSMENT_TYPES: 'SAST,SCA'
BRIDGE_POLARIS_APPLICATION_NAME: $CI_PROJECT_NAMESPACE-$CI_PROJECT_NAME
BRIDGE_POLARIS_PRCOMMENT_ENABLED: 'true'
BRIDGE_POLARIS_REPORTS_SARIF_CREATE: 'true'
BRIDGE_GITLAB_USER_TOKEN: $GITLAB_USER_TOKEN
# INCLUDE_DIAGNOSTICS: 'true'
before_script:
- apt-get -qq update && apt-get install -y curl file unzip
extends: .run-black-duck-tools
#artifacts:
#  name: "bridge-logs"
#  when: always
#  paths:
#    - .bridge/
#  expire_in: 30 days
```

# Full Scan

1. Monitor your pipeline run and wait for the scan to complete via _GitLab → Project → Build → Pipelines_ **Milestone 1** :heavy_check_mark:
   - Note that the scan completes and the pipeline passes. This is because the default policy is "notify on critical & high issues".
2. Log into Polaris and [create a policy](https://polaris.blackduck.com/developer/default/polaris-documentation/t_post_scan_policies) that breaks the build and assign it to your project.
3. Run the pipeline again. Once it completes, select the failed polaris step to see policy enforcement and a failed pipeline. **Milestone 2** :heavy_check_mark:
4. Click on the link in the console output to view the findings in Polaris. **Milestone 3** :heavy_check_mark:
5. Download the SARIF Report via _GitLab → Project → Build → Download Artifacts (download icon right side)_ **Milestone 4** :heavy_check_mark:

# PR scan

1. Edit pom.xml via _GitLab → Project → Code → Repository → pom.xml → Edit button upper right_
   - change log4j version from 2.14.1 to 2.15.0
2. Change target branch to main-patch1, leave _start merge request_ checked, then click on _Commit Changes_
3. Review changes and click on _Create Merge Request_
4. Monitor pipeline run via _GitLab → Project → Build → Pipelines_
5. Once pipeline completes, navigate back to MR and see the MR comment via _GitLab → Project → Merge requests_ **Milestone 5** :heavy_check_mark:

# Congratulations

You have now configured a Polaris scan in a GitLab pipeline and demonstrated all the functionality of the [Black Duck Security Scan Template](https://gitlab.com/blackduck-inc/black-duck-security-scan). :clap: :trophy:
