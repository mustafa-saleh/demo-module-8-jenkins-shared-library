# Jenkins Shared Library

This project is for the DevOps Bootcamp demo for:

Build Automation & CI/CD with Jenkins - [DevOps Bootcamp](https://techworld-with-nana.teachable.com/p/devops-bootcamp)

A Jenkins Shared Library is a collection of reusable Groovy scripts that you can import into your Jenkins Pipelines. It helps eliminate code duplication, enforce organizational standards, and keep your Jenkins files clean and maintainable.

## Standard Repository Structure

For Jenkins to recognize your library, your Git repository must follow this specific folder structure:

- `vars/`: Contains global variables and custom steps (scripts named yourStepName.groovy). This is the most common place to put reusable functions.
- `src/`: Contains standard Java/Groovy object-oriented classes (e.g., com/company/utils/Helper.groovy) for complex logic.
- `resources/`: Contains non-Groovy files (JSON, XML, Shell scripts) that your pipeline needs to load dynamically.

## Step-by-Step Example

### Create the Custom Step

Let's extract the logic from the below Jenkins file to a shared library that can be referenced in pipelines.

Jenkinsfile:

```groovy
def gv

pipeline {   
    agent any

    tools {
        maven 'maven-3.9.16'
    }

    stages {
        stage("init") {
            steps {
                script {
                    gv = load "script.groovy"
                }
            }
        }

        stage("build jar") {
            steps {
                script {
                    gv.buildJar()
                }
            }
        }

        stage("build image") {
            when {
                expression { 
                    BRANCH_NAME == 'main'
                }
            }

            steps {
                script {
                    gv.buildImage()
                }
            }
        }       
    }
} 
```

script.groovy:

```groovy
def buildJar() {
    echo 'building the application...'
    sh 'mvn package'
}

def buildImage() {
    echo "building the docker image..."
    withCredentials([usernamePassword(credentialsId: 'docker-hub', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
        sh 'docker build -t mustafa199b/demo:jma-3.0 .'
        sh 'echo $PASS | docker login -u $USER --password-stdin'
        sh 'docker push mustafa199b/demo:jma-3.0'
    }
}

return this
```

In Jenkins shared library, create a new file "vars/buildJar.groovy" with the following

```groovy
#!/user/bin/env groovy

def call() {
    echo 'building the application...'
    sh 'mvn package'
}
```

create a new file "vars/buildImage.groovy" with the following

```groovy
#!/user/bin/env groovy

def call() {
    echo "building the docker image..."
    withCredentials([usernamePassword(credentialsId: 'docker-hub-repo', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
        sh 'docker build -t mustafa199b/demo:jma-3.0 .'
        sh 'echo $PASS | docker login -u $USER --password-stdin'
        sh 'docker push mustafa199b/demo:jma-3.0'
    }
}
```

note: the file name will later be used to invoke the function in the pipeline.

### Configure Jenkins

To make the shared library available in Jenkins, we've to configure it with the following

1. Go to Manage Jenkins ➔ System (or Configure System).
2. Scroll down to Global Pipeline Libraries.
3. Click Add.
4. Name your library (e.g., my-shared-library).
5. Specify the default version (e.g., main or a specific tag).
6. Select Modern SCM and provide your Git repository URL.

### Use it in a Jenkinsfile

Import the library at the very top of your pipeline using the "@Library" annotation:

```groovy
@Library('my-shared-library') _     // _ is required since there're no statements between the "@Library" & "pipeline" declarations

pipeline {   
    agent any

    tools {
        maven 'maven-3.9.16'
    }

    stages {

        stage("build jar") {
            steps {
                script {
                    buildJar()
                }
            }
        }

        stage("build image") {
            when {
                expression { 
                    BRANCH_NAME == 'main'
                }
            }

            steps {
                script {
                    buildImage()
                }
            }
        }       
    }
} 
```

Note that the "script.groovy" have been removed & now the functions are referenced from the shared library.

We can further breakdown the logic to a smaller steps to prevent duplications and allow the code to be reused. The "buildImage" function takes care of creating the docker image, signin to the repository & pushing the created image. We can split this logic to 3 functions "buildDockerImage", "dockerLogin", "dockerPush". 

Create a new package in src folder "com.example" that contains the "Docker.groovy" class below:

```groovy
#!/user/bin/env groovy
package com.example

class Docker implements Serializable {

    def script

    Docker(script) {
        // pass execution context as a script from the caller to allow access to jenkins modules
        this.script = script
    }

    def buildDockerImage(String imageName) {
        script.echo "building the docker image..."
        script.sh "docker build -t $imageName ."
        }

    def dockerLogin() {
        script.withCredentials([script.usernamePassword(credentialsId: 'docker-hub', passwordVariable: 'PASS', usernameVariable: 'USER')]) {
            script.sh "echo '${script.PASS}' | docker login -u '${script.USER}' --password-stdin"
        }
    }

    def dockerPush(String imageName) {
        script.sh "docker push $imageName"
    }
}
```

Now let's update the "buildImage" function with the following:

```groovy
#!/user/bin/env groovy

import com.example.Docker

def call(String imageName) {
    return new Docker(this).buildDockerImage(imageName)
}
```

Create new file "dockerLogin.groovy"

```groovy
#!/user/bin/env groovy

import com.example.Docker

def call() {
    return new Docker(this).dockerLogin()
}
```

Create new file "dockerPush.groovy"

```groovy
#!/user/bin/env groovy

import com.example.Docker

def call(String imageName) {
    return new Docker(this).dockerPush(imageName)
}
```

Update the Jenkins file to call the new functions and pass the "imageName" as a parameter

```groovy
@Library('my-shared-library') _     // _ is required since there're no statements between the "@Library" & "pipeline" declarations

pipeline {   
    agent any

    tools {
        maven 'maven-3.9.16'
    }

    stages {

        stage("build jar") {
            steps {
                script {
                    buildJar()
                }
            }
        }

        stage("build & push image") {
            when {
                expression { 
                    BRANCH_NAME == 'main'
                }
            }

            steps {
                script {
                    buildImage 'mustafa199b/demo:jma-3.0'
                    dockerLogin()
                    dockerPush 'mustafa199b/demo:jma-3.0'
                }
            }
        }       
    }
} 
```

You can reference a shared library directly inside your Jenkinsfile without configuring it globally in the Jenkins UI. This is called a Dynamic Retrieval or Inline Library Definition.This method is highly useful for testing library changes on a feature branch or keeping your pipeline definitions completely standalone.

Remove the Global library definition "my-shared-library" from Jenkins UI and modify the Jenkinsfile as below

```groovy
library identifier: 'my-shared-library@main', retriever: modernSCM([
    $class: 'GitSCMSource',
    remote: 'https://github.com/mustafa-saleh/demo-module-8-jenkins-shared-library.git',
    credentialsId: 'github-repo'
])

def gv

pipeline {   
    agent any

    tools {
        maven 'maven-3.9.16'
    }

    stages {

        stage("build jar") {
            steps {
                script {
                    buildJar()
                }
            }
        }

        stage("build & push image") {
            when {
                expression { 
                    BRANCH_NAME == 'main'
                }
            }

            steps {
                script {
                    buildImage 'mustafa199b/demo:jma-3.0'
                    dockerLogin()
                    dockerPush 'mustafa199b/demo:jma-3.0'
                }
            }
        }       
    }
} 
```
