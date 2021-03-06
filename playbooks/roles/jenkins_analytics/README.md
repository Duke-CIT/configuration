# Jenkins Analytics

A role that sets up Jenkins for scheduling analytics tasks. 

This role performs the following steps:

* Installs Jenkins using `jenkins_master`.
* Configures `config.xml` to enable security and use
  Linux Auth Domain.
* Creates Jenkins credentials. 
* Enables the use of Jenkins CLI. 
* Installs a seed job from configured repository, launches it and waits
  for it to finish.

## Configuration

When you are using vagrant you **need** to set `VAGRANT_JENKINS_LOCAL_VARS_FILE`
environment variable. This variable must point to a file containing 
all required variables from this section.

This file needs to contain, at least, the following variables 
(see the next few sections for more information about them): 

* `JENKINS_ANALYTICS_USER_PASSWORD_HASHED` 
* `JENKINS_ANALYTICS_USER_PASSWORD_PLAIN`
* `JENKINS_ANALYTICS_GITHUB_KEY` or `JENKINS_ANALYTICS_CREDENTIALS`

 
### End-user editable configuration 

#### Jenkins user password

You'll need to override default `jenkins` user password, please do that
as this sets up the **shell** password for this user. 

You'll need to set both a plain password and a hashed one.
To obtain a hashed password use the `mkpasswd` command, for example: 
`mkpasswd --method=sha-512`. (Note: a hashed password is required 
to have clean "changed"/"unchanged" notification for this step 
in Ansible.) 

* `JENKINS_ANALYTICS_USER_PASSWORD_HASHED`: hashed password 
* `JENKINS_ANALYTICS_USER_PASSWORD_PLAIN`: plain password

#### Jenkins seed job configuration 

This will be filled as part of PR[#2830](https://github.com/edx/configuration/pull/2830). 
For now go with defaults. 

#### Jenkins credentials

Jenkins contains its own credential store. To fill it with credentials, 
please use the `JENKINS_ANALYTICS_CREDENTIALS` variable. This variable 
is a list of objects, each object representing a single credential.
For now passwords and ssh-keys are supported. 

If you only need credentials to access github repositories
you can override `JENKINS_ANALYTICS_GITHUB_KEY`,
which should contain contents of private key used for 
authentication to checkout github repositories.  

Each credential has a unique ID, which is used to match 
the credential to the task(s) for which it is needed

Examples of credentials variables:
 
    JENKINS_ANALYTICS_GITHUB_KEY: "{{ lookup('file', 'path to keyfile') }}" 
        
    JENKINS_ANALYTICS_CREDENTIALS:
      # id is a scope-unique credential identifier
      - id: test-password
        # Scope must be global. To have other scopes you'll need to modify addCredentials.groovy
        scope: GLOBAL
        # Username associated with this password
        username: jenkins
        type: username-password
        description: Autogenerated by ansible
        password: 'password'
      # id is a scope-unique credential identifier
      - id: github-deploy-key
        scope: GLOBAL
        # Username this ssh-key is attached to
        username: git
        # Type of credential, see other entries for example
        type: ssh-private-key        
        passphrase: 'foobar'
        description: Generated by ansible
        privatekey: |
          -----BEGIN RSA PRIVATE KEY-----
          Proc-Type: 4,ENCRYPTED
          DEK-Info: AES-128-CBC,....

          Key contents
          -----END RSA PRIVATE KEY-----

#### Other useful variables

* `JENKINS_ANALYTICS_CONCURRENT_JOBS_COUNT`: Configures number of 
  executors (or concurrent jobs this Jenkins instance can 
  execute). Defaults to `2`. 

### General configuration 

Following variables are used by this role:

Variables used by command waiting on Jenkins start-up after running
`jenkins_master` role:

    jenkins_connection_retries: 60
    jenkins_connection_delay: 0.5

#### Auth realm

Jenkins auth realm encapsulates user management in Jenkins, that is:

* What users can log in
* What credentials they use to log in

Realm type stored in `jenkins_auth_realm.name` variable.

In future we will try to enable other auth domains, while
preserving the ability to run cli.

##### Unix Realm

For now only `unix` realm supported -- which requires every Jenkins
user to have a shell account on the server.

Unix realm requires the following settings:

* `service`: Jenkins uses PAM configuration for this service. `su` is
a safe choice as it doesn't require a user to have the ability to login
remotely.
* `plain_password`:  plaintext password, **you should change** default values.
* `hashed_password`: hashed password

Example realm configuration:

    jenkins_auth_realm:
      name: unix
      service: su
      plain_password: jenkins
      hashed_password: $6$rAVyI.p2wXVDKk5w$y0G1MQehmHtvaPgdtbrnvAsBqYQ99g939vxrdLXtPQCh/e7GJVwbnqIKZpve8EcMLTtq.7sZwTBYV9Tdjgf1k.


#### Seed job configuration

Seed job is configured in `jenkins_seed_job` variable, which has the following
attributes:

* `name`:  Name of the job in Jenkins.
* `time_trigger`: A Jenkins cron entry defining how often this job should run.
* `removed_job_action`: what to do when a job created by a previous run of seed job
  is missing from current run. This can be either `DELETE` or`IGNORE`.
* `removed_view_action`: what to do when a view created by a previous run of seed job
  is missing from current run. This can be either `DELETE` or`IGNORE`.
* `scm`: Scm object is used to define seed job repository and related settings.
  It has the following properties:
  * `scm.type`: It must have value of `git`.
  * `scm.url`: URL for the repository.
  * `scm.credential_id`: Id of a credential to use when authenticating to the
    repository.     
    This setting is optional. If it is missing or falsy, credentials will be omitted. 
    Please note that when you use ssh repository url, you'll need to set up a key regardless 
    of whether the repository is public or private (to establish an ssh connection
    you need a valid public key). 
  * `scm.target_jobs`: A shell glob expression relative to repo root selecting
    jobs to import.
  * `scm.additional_classpath`: A path relative to repo root, pointing to a
     directory that contains additional groovy scripts used by the seed jobs.

Example scm configuration:

    jenkins_seed_job:
      name: seed
      time_trigger: "H * * * *"
      removed_job_action: "DELETE"
      removed_view_action: "IGNORE"
      scm:
        type: git
        url: "git@github.com:edx-ops/edx-jenkins-job-dsl.git"
        credential_id: "github-deploy-key"
        target_jobs: "jobs/analytics-edx-jenkins.edx.org/*Jobs.groovy"
        additional_classpath: "src/main/groovy"

Known issues
------------

1. Playbook named `execute_ansible_cli.yaml`, should be converted to an
   Ansible module (it is already used in a module-ish way).
2. Anonymous user has discover and get job permission, as without it
   `get-job`, `build <<job>>` commands wouldn't work.
   Giving anonymous these permissions is a workaround for
   transient Jenkins issue (reported [couple][1] [of][2] [times][3]).
3. We force unix authentication method -- that is, every user that can login
   to Jenkins also needs to have a shell account on master.


Dependencies
------------

- `jenkins_master`

[1]: https://issues.jenkins-ci.org/browse/JENKINS-12543
[2]: https://issues.jenkins-ci.org/browse/JENKINS-11024
[3]: https://issues.jenkins-ci.org/browse/JENKINS-22143
