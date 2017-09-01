# Jenkins Docker slave image

The Dockerfile and Makefile are used to build the image used on Jenkins slaves (a/k/a agents). Uses [jenkinsci/jnlp-slave](https://hub.docker.com/r/jenkinsci/jnlp-slave/).

```sh
# build new image
make build

# push to GCP Container Registry
make push
```

## Why?

It seems that we are having difficulty passing SSH keys to our Jenkins slave during the build of each job.
Our solution is to add an SSH key to our container that is building our images so that they may have access
to our private repositories.

## How to use.

1. Create and upload an [SSH key](https://help.github.com/articles/generating-an-ssh-key/)
2. Name it `github_rsa`
3. Move it to the root of this project
4. Run `make build` to build the image
5. Run `make push` to push the image
