# Prow deployment role

This role deploys prow in a kubernetes cluster created using kubevirt-ci
It contains variables that define a set of manifests to upload
to deploy prow, then uses kubernetes_crud role as primary interface with
the staging cluster to upload the manifests.

## Role structure

From the role root directory

    defaults/main.yaml

Contains a few variable useful for the deployment, but most importantly the
list of manifest to upload for the prow deployment. The list references
manifest location in other parts of the role tree and is directly passed
as input to kubernetes_crud role

    files/
    
Is the directory containing static configuration files 

    files/manifests
    
Contain static manifests.

    files/jobs
    
Is a symbolic link to the job configuration sub-tree already existing under
the "prow" role

    templates/manifests
    
Contains parametrized manifests that will be rendered by kubernetes_crud
role using variables passed to the role.
    
    molecule/default
    
Contains the main scenario for local or automated testing.

Beside passing manifests as steps, the role performs small setup/cleanup
tasks, primarily passing variables around.

## How to launch a deployment

First you need to create a deployment configuration file.
The role will look by default in

    /etc/prow-deploy/deploy_config.yaml
    
An example can be found in directory

    molecule/default/vars/deploy_config.yaml
    
be sure to specify a githubToken.
    
When the configuration is in place launch

    cd /path/to/project-infra/github/ci/
    ansible-playbook -v prow-deploy/prow-deploy.yaml
    
to use the default config location, or:
    

    ansible-playbook -v prow-deploy/prow-deploy.yaml -e deploy_config_path=/path/to/deploy_config.yaml
    -- OR --
    export DEPLOY_CONFIG_PATH=/path/to/deploy.yaml
    ansible-playbook -v prow-deploy/prow-deploy.yaml 

To specify a different location

## How to test the role

The role is tested using molecule.
Molecule will take care of all the test task. The kubevirtci cluster will require at 
least 16G of memory to run properly.
The user needs to be able to sudo to root without password.
From the role root directory launch

    molecule prepare
   
To launch the kubevirtci cluster and prepare the nodes
properly. This is a protected action, it cannot be done twice.
The natural flow is that you can prepare again an instance only
after you destroy it, so if you need to prepare again but no destroy
has been issued, you need to call

    molecule reset
    
To tell molecule to start from scratch.


For prow deployment, a real github token is a major requirement. Many
service will try to access github using the token before starting.
Create a github token for your account, with at most read:user scope. Any
additional permission will make the test environment interfere with the production
repositories, like adding automatic comments.
Put you token into a yaml file e.g.

    cat > /tmp/token_file.yaml << EOF
    githubToken: 2391rj0lvksldkfj0-w9ejgpkljvblskgj
    EOF
   
Then start deployment by passing the file as extra args file

    molecule converge -- -e @/tmp/token_file.yaml
    
This will launch the prow deployment itself, will wait for the deployment
to settle and then will collect some information in the
artifacts dir.

    molecule verify
    
Will launch a set of tests to verify that the deployment
works correctly. At the moment only smoke tests are available
Some tests are using test-infra commands, which
by default is located in

    /workspace/test-infra

    molecule verify -- -e testinfra_dir=/path/to/test-infra

Will let you specify a different directory

    molecule cleanup 
    
Will remove prow-namespace, so that prow can be eventually
deployed again in the same cluster

    molecule destroy
    
Will tear down the kubevirt ci cluster completely

    molecule test
    
Will launch all the above step automatically in sequence.
If you need to launch test with a different deploy config, you need to
export an environment variable instead

    export DEPLOY_CONFIG_PATH=/path/to/config
    molecule test


## How to debug the services in live cluster

Behaviour of prow services in pods proved to be less than reliable, with
services unable to start properly, but not reporting any errors in the logs
and marking the pod as "Running"

In those cases, the best way to debug a service is to jump directly on the pod
executing a shell and running the service manually.

### Process overview

We need to enter the pod, download the service code, and run it manually with "go run".
The service will log to stdout and any error will block the execution, giving reason
on its behaviour
Unfortunately, there is no way to stop the pod entrypoint, as any attempt to interfere with
process 1 (the service) will cause the pod termination, but it's possible to start
another process with the same service in parallel.

### Container base image caveat

The container base image of a service is a quite obsolete version of alpine (3.6).
The base image uses musl libc instead of glibc and has very old version of basic tools.
The pod entrypoint is normally a statically compiled binary that requires limited memory and doesn't
rely on the (very old) dynamic libraries offered by the base images when launched.
We need to launch the service using "go run" instead of bazel, because bazel requires glibc.

### Increase pod memory

Launching service with "go run" will require more memory that the default 512Mb
So we need to change the request and limit on the deployment configuration of the pod
you want to debug to at least 1G. If a "go run" attempt is killed by an unexplained signal, more
memory will be needed.
This can be done on the manifests, or directly editing the deployment.
If the manifest is changed the deployment will need to be cleaned up and restart.
If it is done directly, kubernetes will redeploy the pods automatically, but a cleanup will
wipe the configuration.

### Update base image

Once memory requirements are satisfied, the next step is to git clone the service code and install
go runtime.
The go runtime provided with the base alpine image is very old (pre 1.11) and will not be able to run
the code correctly, so the release must be updated.
A script at

    files/debug_prepare.sh
    
will do this automatically
it is enough to launch it inside the pod, as in the following example

    cd $KUBEVIRTCI_DIR
    POD=deck-74fd5d678d-g8dz6
    ./cluster-up/kubectl.sh -n kubevirt-prow cp debug_prepare.sh $POD:/
    ./cluster-up/kubectl.sh -n kubevirt-prow exec $POD -- chmod +x /debug_prepare.sh
    ./cluster-up/kubectl.sh -n kubevirt-prow exec $POD -- sh -c /debug_prepare.sh

The script will replace repository to a viable version, update the packaging tool,
install basic tools, and clone the test-infra code inside the container.
Once the script has finished, we can run a interactive shell, change to the parent directory of
the service, then launch the service
 
    ./cluster-up/kubectl.sh -n kubevirt-prow exec -it $POD -- sh
    # cd /srv/test-infra/prow/cmd
    # go run ./$COMMAND

Logs will show on stdout.