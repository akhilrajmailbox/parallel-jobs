




### Jenkins Installation (master or slave need to configure accordingly)

[ref](https://www.jenkins.io/doc/book/installing/linux/#debianubuntu)

[jenkins slave on gcp](https://cloud.google.com/solutions/using-jenkins-for-distributed-builds-on-compute-engine)

```shell
# dependencies for jenkisnfile stages
apt-get update
apt-get install wget gettext jq apt-transport-https ca-certificates curl gnupg-agent software-properties-common -y
# java installation
apt-get install openjdk-11-jdk -y
java -version
# docker installation
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
add-apt-repository    "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io -y
docker version
# kubectl install (1.16 version here)
curl -LO https://dl.k8s.io/release/v1.16.0/bin/linux/amd64/kubectl
install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
kubectl version --client
# jenkins installation
wget -q -O - https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo apt-key add -
sh -c 'echo deb https://pkg.jenkins.io/debian-stable binary/ > \
    /etc/apt/sources.list.d/jenkins.list'
apt-get update
apt-get install jenkins -y
systemctl start jenkins
systemctl status jenkins
# grant permission for jenkins group
usermod -aG docker jenkins
```

### ssl termination

```shell
# Self-Signed SSL Certificate
openssl req -newkey rsa:4096 \
            -x509 \
            -sha256 \
            -days 365 \
            -nodes \
            -out jenkins.crt \
            -keyout jenkins.key \
            -subj "/C=IN/ST=Karnataka/L=Bangalore/O=Security/OU=IT Department/CN=www.jenkins.local"

# Convert SSL keys to PKCS12 format
openssl pkcs12 -export -out jenkins.p12 \
    -passout 'pass:your-secret-password' \
    -inkey jenkins.key \
    -in jenkins.crt \
    -name www.jenkins.local

# Convert PKCS12 to JKS format
keytool -importkeystore -srckeystore jenkins.p12 \
    -srcstorepass 'your-secret-password' \
    -srcstoretype PKCS12 \
    -srcalias www.jenkins.local \
    -deststoretype JKS \
    -destkeystore jenkins.jks \
    -deststorepass 'your-secret-password' \
    -destalias www.jenkins.local

# create a keystore and save the JKS file
mkdir /var/lib/jenkins/keystore
cp jenkins.jks /var/lib/jenkins/keystore/
chown -R jenkins: /var/lib/jenkins/keystore
chmod 700 /var/lib/jenkins/keystore
chmod 600 /var/lib/jenkins/keystore/jenkins.jks
```

# Modify Jenkins Configuration for SSL

```shell
vim /etc/default/jenkins
```

```code
HTTP_PORT=-1
JENKINS_ARGS="--webroot=/var/cache/$NAME/war --httpsPort=8443 --httpsKeyStore=/var/lib/jenkins/keystore/jenkins.jks --httpsKeyStorePassword=your-secret-password --httpPort=$HTTP_PORT"
```

[ssl ref](https://wiki.wocommunity.org/display/documentation/Installing+and+Configuring+Jenkins)
[ssl ref 1](https://devopscube.com/configure-ssl-jenkins/)



### Automatically Generating Jenkins Jobs (Jenkins Job Builder)

#### Install and Configure

```shell
# install
apt-get install python3-pip --fix-missing
pip3 install --user jenkins-job-builder
```

```shell
# configure
vim jenkins_jobs.ini
```

```code
[job_builder]
ignore_cache=True
keep_descriptions=False
recursive=False
allow_duplicates=False
update=all

[jenkins]
user=custom_user
password=token
url=https://localhost:8443
query_plugins_info=False
```


#### example

```code
- project:
    name: my_project
    jobs:
        - 'multibranch':
            customer: customer_1
            disable_job: false
            disable_master: true
            description: 'pipeline cicd job for {customer}'

- job-template:
    # default variable
    disable_job:
    description:
    disable_master:
    git_credentials_id: 'mytestid'
    job_folder: 'mygroup'
    # job configuration
    name: '{job_folder}/{customer}'
    id: 'multibranch'
    description: '{description}'
    display-name: '{customer}'
    periodic-folder-trigger: '1m'
    project-type: multibranch
    number-to-keep: '10'
    script-path: Jenkinsfile
    disabled: '{disable_job}'
    scm:
      - git:
          url: 'https://example.com/sample.git'
          credentials-id: '{git_credentials_id}'
          discover-tags: true
          property-strategies:
            named-branches:
              exceptions:
                - exception:
                    branch-name: 'master,main'
                    properties:
                      - suppress-scm-triggering: '{disable_master}'

- job:
    name: mygroup
    project-type: folder
```

```shell
# create jenkins job
jenkins-jobs --conf keystore/jenkins_jobs.ini update jenkins-jobs/Jenkinsjob

# update / create new jobs and delete old jobs
jenkins-jobs --conf keystore/jenkins_jobs.ini update jenkins-jobs/Jenkinsjob --delete-old
```



[ref 1](https://medium.com/slalom-build/automatically-generating-jenkins-jobs-d30d4b0a2b49)
[Jenkins Job Builder install & config official](https://jenkins-job-builder.readthedocs.io/en/latest/installation.html)
[Jenkins Job Builder definition official](https://docs.openstack.org/infra/jenkins-job-builder/definition.html)
[Job DSL Plugin official](https://jenkinsci.github.io/job-dsl-plugin/#path/folder)
[Job DSL Plugin ref 1](https://www.digitalocean.com/community/tutorials/how-to-automate-jenkins-job-configuration-using-job-dsl)
[Job DSL Plugin ref 2](https://metadrop.net/en/articles/jenkins-and-job-dsl-plugin)