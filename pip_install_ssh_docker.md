# Installing from a private repo using pip, Docker and GitLab CI

You're trying to build a Docker image that uses a Python module from a private repository, e.g. you have something like this in your `requirements.txt`:

```shell
git+ssh://git@private.server.com/repository/my_module@main
```
And in your `Dockerfile` you have the line:

```shell
RUN pip install requirements.txt
```
But when Docker tries to build the image, pip fails with the following error:

```shell
Permission denied, please try again.
git@private.server.com: Permission denied (publickey,password).
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```
The issue is that the Docker image doesn't have access to the SSH private key for the repository, so pip can't download the module.

**Don't try and copy your SSH private key into the Docker image.** This could potentially expose the key to third parties if the image ever gets distributed. Instead, use Docker's SSH pass through functionality to only expose the private key to the build process.

Add the following lines to your `Dockerfile` _before_ the `RUN pip install requirements.txt` line:

```shell
RUN apt update
RUN apt install -y openssh-client
RUN mkdir -p -m 0700 ~/.ssh && ssh-keyscan private.server.com >> ~/.ssh/known_hosts
```
Change `private.server.com` to your server's hostname. These lines will ensure the hostname of the repository is added to the SSH `known_hosts` list inside the image which is required for SSH access.

Now, ensure that pip has access to the SSH key when it runs by modifying the install command and adding a mount type of SSH:

```shell
RUN --mount=type=ssh pip install requirements.txt
```

Finally, you'll need to tell Docker to pass the SSH key through when it builds the image. You can do that with the `--ssh` option. You'll also need to ensure that BuildKit is enabled which can be done by setting an environment variable:

```shell
export DOCKER_BUILDKIT=1
docker build . --ssh default -f docker/Dockerfile
```
The `default` argument just tells Docker to pass your SSH key through using its standard mechanism. If you have issues with that (e.g. it doesn't work in Git Bash on Windows), you can also specify the path to your private SSH key explicitly, for example:

```shell
export DOCKER_BUILDKIT=1
docker build . --ssh default=~/.ssh/id_ed25519 -f docker/Dockerfile
```
Change the path of the key for your machine. 

## Using GitLab CI

You've followed the instructions above and have the Docker image building on your local machine, but how do you get it to work in GitLab CI?

First, navigate to the CI/CD environment variable settings for your project in GitLab and add a new variable.

![GitLab Environment Variable](images/ci_env_var.png)

Call the variable `SSH_DEPLOY_PRIVATE_KEY` and set the value to your SSH private key. Make sure you copy all of the key text, including the `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----`.

![Add SSH Key](images/add_ssh_key.png)

Now, edit your `.gitlab-ci.yml` file in your local project, and add the following section to the stage that builds the Docker image:

```yaml
before_script:
  - 'command -v ssh-agent >/dev/null || ( apt-get update -y && apt-get install openssh-client -y )'
  - eval $(ssh-agent -s)
  - echo "$SSH_DEPLOY_PRIVATE_KEY" | tr -d '\r' | ssh-add -
  - mkdir -p ~/.ssh
  - chmod 700 ~/.ssh
```

That will ensure that the `ssh-agent` is running on the build machine and will install it if not. It will then add the SSH private key to the agent from the environment variable just created.

Lastly, edit the `script` section of the build stage and ensure that the `DOCKER_BUILDKIT=1` environment variable is set and that the `--ssh` option is being passed to the Docker build command:

```yaml
script:
  - export DOCKER_BUILDKIT=1
  - docker build --ssh default . -f docker/Dockerfile
```
That should now enable GitLab CI to successfully build the Docker image.