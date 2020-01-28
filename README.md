# mtail-boshrelease

PoC of mtail: https://github.com/google/mtail/


# Creating and using a release:

In order to test and create a "non final" (dev) release, run:

```
# Update or sync blobs
./update-blobs.sh
# Create a dev release
bosh  create-release --force --tarball=/tmp/release.tgz
# Upload release to bosh director
bosh -e <bosh-env> upload-release /tmp/release.tgz
```

You can deploy an empty vm with this example manifest to provide
a metric per log file with the number of lines (graphite is also
enabled here!):

```
name: mtail

releases:
- name: mtail
  version: latest

instance_groups:
- name: mtail
  instances: 1
  vm_type: small
  stemcell: default
  vm_extensions: []
  azs:
  - z1
  - z2
  - z3
  networks:
  - name: default
  jobs:
  - name: mtail
    release: mtail
  properties:
    mtail:
      port: 3903
      graphite: "localhost:5555"
      progs:
      - name: mtaillogs
        log: "/var/vcap/sys/log/mtail/*.log"
        config: |
          counter filename_lines by filename
          /$/ {
            filename_lines[getfilename()]++
          }

stemcells:
- alias: default
  name: bosh-google-kvm-ubuntu-xenial-go_agent
  version: latest

update:
  canaries: 1
  max_in_flight: 1
  serial: false
  canary_watch_time: 1000-60000
  update_watch_time: 1000-60000
```

A more realistic purpose is to use this release to provide metrics from the gorouter
access logs by using a runtime-config:

```
releases:
- name: mtail
  version: "0+dev.4"

addons:
- name: mtail
  jobs:
  - name: mtail
    release: mtail
    properties:
      mtail:
        progs:
        - name: gorouter
          log: "/var/vcap/sys/log/gorouter/access.log"
          config: |
            counter gorouter_http_requests_total
            counter gorouter_http_requests by route, request_method, status_code
            counter gorouter_http_requests_app by route
            /^/ +
            /(?P<route>[0-9A-Za-z\.:-]+) / +
            /- / +
            /\[(\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}.\d{3}(\+|-)\d{4})\] / +
            /"(?P<request_method>[A-Z]+) (?P<request_uri>\S+) (?P<http_version>HTTP\/[0-9\.]+)" / +
            /(?P<status_code>\d{3}) / +
            /.*$/ {
                gorouter_http_requests_total++
                gorouter_http_requests[$route][$request_method][$status_code]++
                gorouter_http_requests_app[$route]++
            }

  include:
    deployments:
    - cf
    jobs:
    - name: gorouter
      release: cf
```

# Author

(c) Jose Riguera Lopez jose.riguera@springernature.com

Springernature Engineering Enablement

MIT License