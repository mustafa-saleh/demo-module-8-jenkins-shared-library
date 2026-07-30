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
        sh 'docker build -t mustafa199b/demo:jma-2.0 .'
        sh 'echo $PASS | docker login -u $USER --password-stdin'
        sh 'docker push mustafa199b/demo:jma-2.0'
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
        sh 'docker build -t nanatwn/demo-app:jma-2.0 .'
        sh 'echo $PASS | docker login -u $USER --password-stdin'
        sh 'docker push nanatwn/demo-app:jma-2.0'
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


