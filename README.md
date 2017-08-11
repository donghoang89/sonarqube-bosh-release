# sonarqube-bosh-release
A BOSHified SonarQube

## Setup
### Prerequsites
You'll need to download the [SonarQube](https://www.sonarqube.org/downloads/) and [Oracle JDK](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html) archives into a src directory such that the structure of this project is:
```
.
├── config
│   └── ...
├── jobs
│   └── ...
├── manifest.yml
├── packages
│   └── ...
└── src
    ├── java
    │   └── jdk1.8.0_144.tar.gz
    └── sonarqube
        └── sonarqube-6.5.zip
```

### manifest.yml
For deployment to [BOSH Lite](https://bosh.io/docs/bosh-lite.html) you'll need to create a manifest.yml that looks something like:
```
---
name: sonarqube-bosh

releases:
- name: sonarqube-bosh
  version: latest

networks:
- name: default
  subnets:
  - range: 10.244.0.0/28
    reserved: [10.244.0.1]
    static: [10.244.0.2]

resource_pools:
- name: default
  stemcell:
    name: bosh-warden-boshlite-ubuntu-trusty-go_agent
    version: latest
  network: default

compilation:
  workers: 2
  network: default

update:
  canaries: 1
  max_in_flight: 2
  canary_watch_time: 1000-30000
  update_watch_time: 1000-30000

jobs:
- name: sonarqube
  templates:
  - name: sonarqube
  instances: 1
  resource_pool: default
  networks:
  - name: default
    static_ips:
    - 10.244.0.2
```

## Release
Now issue the usual BOSH commands to create and upload a release (use --force if there are uncommitted changes):
```
$ bosh create-release [--force]
$ bosh upload-release
$ bosh releases
```

## Deploy
Finally deploy the release (assuming you've uploaded a stemcell):
```
$ bosh -d sonarqube-bosh deploy manifest.yml
```

The deployment will most likely say it's failed:

```
Task xx

10:40:29 | Deprecation: Ignoring cloud config. Manifest contains 'networks' section.
10:40:29 | Preparing deployment: Preparing deployment (00:00:00)
10:40:29 | Preparing package compilation: Finding packages to compile (00:00:00)
10:40:29 | Updating instance sonarqube: sonarqube/<uuid> (0) (canary) (00:01:26)
            L Error: 'sonarqube/0 (<uuid>)' is not running after update. Review logs for failed jobs: sonarqube

10:41:55 | Error: 'sonarqube/0 (<uuid>)' is not running after update. Review logs for failed jobs: sonarqube

Started  Fri Aug 11 10:40:29 UTC 2017
Finished Fri Aug 11 10:41:55 UTC 2017
Duration 00:01:26

Task xx error

Updating deployment:
  Expected task '17' to succeed but state is 'error'

Exit code 1
```

But SonarQube will be running. It's likely the above error is due to monit using the wrond pid (to be fixed).

## Success!
Assuming you've setup a route to the VirtualBox network:
```
$ sudo route add -net 10.244.0.0/16 192.168.50.6 # Mac OS X
$ sudo route add -net 10.244.0.0/16 gw 192.168.50.6 # Linux
```

You should be able to browse to the instance of SonarQube at [http://10.244.0.2:9000](http://10.244.0.2:9000)

