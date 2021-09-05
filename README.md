## Ansible K6

Simple Ansible role to install, configure, and start [k6](https://k6.io/).

There are two ways of provisioning and automating a K6 deployment in this role: systemd and cron.

* Using systemd: Install your own script.js and configure this role via `k6_systemd_script_file` variable. See [defaults/main.yml](./defaults/main.yml) for example.
* Using cron: Configured via `k6_cron_scripts` variable. See [defaults/main.yml](./defaults/main.yml) for example.

### Configure K6

#### Cron

We can set multiple k6 instances within a single host via `k6_cron_scripts`.

```yaml
---
- name: Example K6 Playbook
  vars:

    # Ref: https://k6.io/docs/using-k6/options/
    k6_cron_scripts_global_env:
      K6_OUT: statsd
      K6_STATSD_ADDR: localhost:8130
      K6_STATSD_ENABLE_TAGS: "true"
      K6_STATSD_TAG_BLOCKLIST: "vu,iter"

    k6_cron_scripts:
      - name: http-call
        env:
          K6_DISCARD_RESPONSE_BODIES: "true"
          K6_NO_CONNECTION_REUSE: "true"
          K6_DNS: "ttl=2s,select=random,policy=preferIPv4"
          K6_RPS: 10
          K6_DURATION: 1h
          K6_VUS: 2
        schedule:
          user: root
          minute: 0 # Hourly, with K6_DURATION set to 1h, this loadtest will run forever
          hour: "*"
          day: "*"
          month: "*"
          weekday: "*"
        script: |
          import http from 'k6/http';
          import { check } from 'k6';
          import { group } from 'k6';

          http.setResponseCallback(http.expectedStatuses(200));

          export default function () {
            let params = {
              headers: {
                'X-Request-Xyz': '123',
              },
            };

            group('http-from-gcp-to-gcp', function () {
              let res = http.get('https://example.com/loadtest', params);
              check(res, {
                'http_success': (res) => res.status == 200,
              });
            });
          }

  roles:
    - k6
```

#### Systemd

We can configure an instance of k6 systemd service unit.

This approach expects a k6 loadtest script location specified in `k6_systemd_script_file`.

### Custom xk6 Build

We can build custom `k6` binary using [xk6](https://k6.io/blog/extending-k6-with-xk6/) to add customization via extensions.

Typically, we'll want to:
1. Build
2. Generate sha256sum
3. Upload both the build and sha256sum file
4. Configure `k6_package_url` and/or `k6_version`
5. Voila~

#### Example Build (--with yjuba/xk6-dns)

```sh
go install go.k6.io/xk6/cmd/xk6@latest

xk6 build v0.33.0 --with github.com/yjuba/xk6-dns
```

In this example, we store build artifacts in `s3://${S3_BUCKET_NAME}/pkg/k6/k6-${BUILD_VERSION}-linux-amd64.tar.gz`

```sh
export S3_BUCKET_NAME=my-custom-bucket # To store the custom xk6 k6 binaries
export BASE_VERSION=v0.33.0 # Original base for xk6 to build from
export BUILD_VERSION=${BASE_VERSION}-my-custom-build # Our custom version

# Prepare build workspace
mkdir k6-${BUILD_VERSION}-linux-amd64
cd k6-${BUILD_VERSION}-linux-amd64

# Build the custom binary
xk6 build $BASE_VERSION --with github.com/yjuba/xk6-dns
cd ..
tar cvzf k6-${BUILD_VERSION}-linux-amd64.tar.gz k6-${BUILD_VERSION}-linux-amd64
sha256sum k6-${BUILD_VERSION}-linux-amd64.tar.gz >k6-${BUILD_VERSION}-linux-amd64.tar.gz.sha256

# Deploy
aws s3 cp k6-${BUILD_VERSION}-linux-amd64.tar.gz s3://${S3_BUCKET_NAME}/pkg/k6/
aws s3 cp k6-${BUILD_VERSION}-linux-amd64.tar.gz.sha256 s3://${S3_BUCKET_NAME}/pkg/k6/
```

Proceed to update `k6_version` variable in [defaults/main.yml](./defaults/main.yml)

### License

```
Copyright (c) 2021 William Chanrico.

This file is part of ansible-k6.
See https://github.com/williamchanrico/LICENSE for further info.
```
