## Add Docker Engine as gitlab-runner

### Prepare node 
- First of all, bring up a new instance where gitlab-runner will be running
- Install gitlab-runner binary on the system
```
# curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
# apt-get install gitlab-runner
```

- Install Docker on ubuntu system
```
# apt -y install apt-transport-https ca-certificates curl gnupg-agent software-properties-common
# add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"apt update
# apt install docker-ce docker-ce-cli containerd.io
# systemctl docker status
```
- Run the below command to register
-
`--registration-token` can be anything. Need to copy it from `gitlab.example.com->project->setting->CI/CD pipeline->Runner->Dedicated Runner` 
```
# gitlab-runner register --non-interactive --url "http://gitlab.example.com/" --registration-token "dPM56-zd8QRHyA6zvD7a" --executor "docker" --docker-image alpine:latest --description "docker-runner" --tag-list "docker,aws" --run-untagged="true" --locked="false" –access-level="not_protected"
```

- Update the `config.toml` file to udate the host file with `gitlab.example.com` to communicate
```
# vim /etc/gitlab-runner/config.toml

[[runners]]
  name = "docker-runner1"
  url = "http://gitlab.example.com/"
  token = "oHHWdyBaduHZ6Piv2ck4"
  executor = "docker"
  [runners.custom_build_dir]
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
    [runners.cache.azure]
  [runners.docker]
    tls_verify = false
    image = "omvedi25/alpine:v1"
    extra_hosts = ["gitlab.example.com:192.168.1.113"]  <---  Add the gitlab ip and hostname
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/cache"]
    shm_size = 0
```

- Restart the gitlab-runner
```
# systemctl restart gitlab-runner
# systemctl status gitlab-runner
```

