##Description
>Jenkins script for deployment automation (Continuous Integration and Continuous Deployment). This script has many steps such as pruning docker system, scanning GitLeaks, unit test, SonarQube analysis, build image docker, push to Docker registry, stop & remove docker, running on staging via Kubernetes, and running on production via traditional architecture.
