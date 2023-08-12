# Dynamic Choice Parameter in Jenkins Pipeline

## Introduction

Choice parameters for running Jenkins Pipelines that changes based on external factors are handy in cases where the parameters relies on third parties.

In the recent versions of Jenkins security has become more enforced and for good reasons so. This solution works within the default restrictions with the caveat of a required job to be scheduled on the OS. Also, this solution does not rely on the Pipeline to be pre-run, a Seed Job or such, for the parameters to update.

## Strategy

1. Setup a Jenkins Instance with Docker
2. Create necessary directories and files in the container
3. Schedule a job on the OS to maintain updates
4. Showcase with an example Jenkins Pipeline

## Step 1 - Setup a Jenkins Instance with Docker

Start a docker container and extract the admin password that is printed on the console

```bash
docker run -p 8080:8080 -p 50000:50000 --restart=on-failure jenkins/jenkins
```

Goto `http://localhost:8080` and enter the admin password.

Pick “Recommended Plugins”.

Set admin credentials (these are not important for this example).

Go to http://localhost:8080/manage/pluginManager/available and install plugin “Active Choices”. 

## Step 2 - Create Necessary Directories and Files in the Container

Exec into the container

```bash
[ofd@PF2DNJJ3 ~]$ docker ps
CONTAINER ID   IMAGE             COMMAND                  CREATED         STATUS         PORTS                                              NAMES
2123a5ff36fa   jenkins/jenkins   "/usr/bin/tini -- /u…"   4 minutes ago   Up 4 minutes   0.0.0.0:8080->8080/tcp, 0.0.0.0:50000->50000/tcp   optimistic_ramanujan

[ofd@PF2DNJJ3 ~]$ docker exec -it 2123a5ff36fa /bin/bash
```

Run the following commands in the container which will create necessary directories and files.

```bash
mkdir -p /var/jenkins_home/deployment/packages/
cd /var/jenkins_home/deployment/packages/
touch deployment1.war deployment2.war deployment3.war
touch /var/jenkins_home/deployment/options.conf
exit
```

## Step 3 - Schedule a Job on the OS to Maintain Updates

The following command will install Cron and Nano in the container

```bash
docker exec -u 0 f1066de41a60 bash -c 'apt update \
&& apt install cron nano -y \
&& service cron start'
```

Now, access the container as Jenkins `docker exec -it f1066de41a60 /bin/bash`, run `crontab -e` and paste the following line

```bash
* * * * * ls /var/jenkins_home/deployment/packages/ >> /var/jenkins_home/deployment/options.conf
```

Press `CTRL + X` followed by `y` to save changes. Finish off by exiting the container by entering the command `exit` .

## Step 4 - Showcase with an Example Jenkins Pipeline

Browse to the location [http://localhost:8080](http://localhost:8080/manage/pluginManager/available) and hit *+ New Item* in the left corner.

Enter “dynamic-dropdown” as the name of the Pipeline and select “Pipeline” followed by OK.

Next, scroll down to the bottom of the configuration page and insert the following script and click *Save .*

```bash
properties([
    parameters([
        [$class: 'ChoiceParameter', 
         choiceType: 'PT_SINGLE_SELECT', 
         description: 'Select a file from the directory', 
         filterable: false, 
         name: 'SELECTED_FILE', 
         randomName: 'choice-parameter-351463267', 
         script: [
             $class: 'GroovyScript', 
             fallbackScript: [
                 classpath: [], 
                 script: 'return ["ERROR"]'
             ], 
        script: [
            classpath: [], 
            script: '''
return new File("/var/jenkins_home/deployment/options.conf").readLines()
            '''
        ]
         ]
        ]
    ])
])

pipeline {
    agent any

    stages {
        stage('Display Selected File') {
            steps {
                echo "You selected: ${params.SELECTED_FILE}"
            }
        }
    }
}
```

Lastly, we need to Approve the script to be run. Do this by executing the Pipeline by going to http://localhost:8080/job/dynamic-dropdown/build click Build .

Expectedly, the Build should fail. Go to http://localhost:8080/manage/scriptApproval/ and approve the script.

Rerun the Pipeline and the dropdown menu should now be functioning as intended.

Run the Pipeline selecting deployment2.war from the dropdown and the console output should print the selected parameter *You selected: deployment2.war*

### Verification

The parameter should update once every minute based on the contents of the *packages* folder. To verify this rename one of the example files in the container

```bash
docker exec f1066de41a60 bash -c 'mv \
 /var/jenkins_home/deployment/packages/deployment1.war \
 /var/jenkins_home/deployment/packages/deployment1-renamed.war'
```

After waiting for one minute goto http://localhost:8080/job/dynamic-dropdown/build and verify the change has taken effect in the dropdown menu. The file *deployment1.war* should be gone and instead there should be a file named *deployment1-renamed.war*.
