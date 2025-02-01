
# separate the important scripts into shared libraries and move the Helm templates to a shared location, below are the steps:

1. **Create Shared Library Functions:**
    * Move reusable functions from jenkinsfile.build and jenkinsfile.eks to the shared library.
2. **Move Helm Templates to Shared Location:**
    * Move the Helm templates to a shared location in the repository.
3. **Update Jenkinsfiles to Use Shared Library:**
    * Update the jenkinsfile.build and jenkinsfile.eks to use the shared library functions.

Directory Structure:

```
shared-library-repo/
├── vars/
│   ├── deployApp.groovy
│   ├── buildApp.groovy
│   └── seedJob.groovy
├── src/
│   └── org/
│       └── example/
│           └── utils/
│               └── Helper.groovy
├── resources/
│   ├── someTemplate.groovy
│   └── jobTemplate.groovy
├── helm/
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ...
└── config/
    └── services.json
```
Example: buildApp.groovy

```
def call(String service, String version) {
    pipeline {
        agent {
            label 'AGENT-1'
        }
        environment {
            appVersion = version
            region = 'us-east-1'
            account_id = '528757792370'
            project = 'expense'
            component = service
            SONARQUBE_TOKEN = credentials('sonarqube-token') // Assuming you have stored the SonarQube token in Jenkins credentials
        }
        stages {
            stage('Read the Version') {
                steps {
                    script {
                        def packageJson = readJSON file: "${service}/package.json"
                        appVersion = packageJson.version
                        echo "App Version: ${appVersion}"
                    }
                }
            }
            stage('Install Dependencies') {
                steps {
                    dir(service) {
                        sh 'npm install'
                    }
                }
            }
            stage('SonarQube Analysis') {
                environment {
                    SCANNER_HOME = tool 'sonar-6.0' //scanner config
                }
                steps {
                    withSonarQubeEnv('SonarQube') { // 'SonarQube' is the name of the SonarQube server configured in Jenkins
                        dir(service) {
                            sh """
                                $SCANNER_HOME/bin/sonar-scanner \
                                -Dsonar.projectKey=${project}-${environment}-${component} \
                                -Dsonar.sources=. \
                                -Dsonar.host.url=http://your-sonarqube-server:9000 \
                                -Dsonar.login=${SONARQUBE_TOKEN} \
                                -Dsonar.projectVersion=${appVersion} \
                                -Dsonar.language=js \
                                -Dsonar.sourceEncoding=UTF-8
                            """
                        }
                    }
                }
            }
            stage('Build and Push Docker Image') {
                steps {
                    script {
                        dir(service) {
                            sh """
                                aws ecr get-login-password --region $region | docker login --username AWS --password-stdin ${account_id}.dkr.ecr.$region.amazonaws.com
                                docker build -t ${account_id}.dkr.ecr.$region.amazonaws.com/${project}/${environment}/${component}:${appVersion} .
                                docker images
                                docker push ${account_id}.dkr.ecr.$region.amazonaws.com/${project}/${environment}/${component}:${appVersion}
                            """
                        }
                    }
                }
            }
        }
        post {
            always {
                deleteDir()
            }
        }
    }
}
```

Example: deployApp.groovy:

```
def call(String service, String environment, String version) {
    pipeline {
        agent {
            label 'AGENT-1'
        }
        environment {
            appVersion = version
            region = 'us-east-1'
            account_id = '528757792370'
            project = 'expense'
            component = service
        }
        stages {
            stage('Deploy Application') {
                steps {
                    withAWS(region: 'us-east-1', credentials: 'aws-creds-dev') {
                        dir(service) {
                            sh """
                                aws eks update-kubeconfig --region ${region} --name ${project}-${environment}
                                helm upgrade --install ${component} ${libraryResource('helm/templates')} \
                                -f ./${component}-values.yml \
                                --set deployment.image.prefix=${account_id}.dkr.ecr.${region}.amazonaws.com/${project}/${environment}/${component} \
                                --set deployment.image.tag=${appVersion}
                            """
                        }
                    }
                }
            }
        }
        post {
            always {
                deleteDir()
            }
        }
    }
}
```

Example: jenkinsfile.build

```
@Library('shared-library-repo') _

buildApp('backend', '1.0.0')
```

Example: jenkinsfile.eks

```
@Library('shared-library-repo') _

pipeline {
    agent {
        label 'AGENT-1'
    }
    parameters {
        string(name: 'Service', defaultValue: 'backend', description: 'Name of the service to deploy')
        choice(name: 'Environment', choices: ['dev', 'qa', 'prod'], description: 'Deployment environment')
        string(name: 'Version', defaultValue: '1.0.0', description: 'Application image version')
    }
    stages {
        stage('Initialize Environment') {
            steps {
                script {
                    appVersion = params.Version
                    component = params.Service
                    environment = params.Environment
                }
            }
        }
        stage('Deploy Application') {
            steps {
                deployApp(params.Service, params.Environment, params.Version)
            }
        }
    }
    post {
        always {
            deleteDir()
        }
    }
}
```

# split as resuable components

further split buildApp.groovy and deployApp.groovy into smaller, reusable Groovy classes and functions. This approach helps in maintaining a clean and modular codebase, making it easier to manage and update.

Updated Directory Structure:

```
shared-library-repo/
├── vars/
│   ├── buildApp.groovy
│   ├── deployApp.groovy
│   └── seedJob.groovy
├── src/
│   └── org/
│       └── example/
│           ├── build/
│           │   ├── BuildSteps.groovy
│           │   └── SonarQubeSteps.groovy
│           └── deploy/
│               └── DeploySteps.groovy
├── resources/
│   ├── someTemplate.groovy
│   └── jobTemplate.groovy
├── helm/
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ...
└── config/
    └── services.json
```

Example: BuildSteps.groovy

```
package org.example.build

class BuildSteps {
    static void readVersion(script, service) {
        script.dir(service) {
            def packageJson = script.readJSON file: 'package.json'
            script.env.appVersion = packageJson.version
            script.echo "App Version: ${script.env.appVersion}"
        }
    }

    static void installDependencies(script, service) {
        script.dir(service) {
            script.sh 'npm install'
        }
    }

    static void buildAndPushDockerImage(script, service) {
        script.dir(service) {
            script.sh """
                aws ecr get-login-password --region ${script.env.region} | docker login --username AWS --password-stdin ${script.env.account_id}.dkr.ecr.${script.env.region}.amazonaws.com
                docker build -t ${script.env.account_id}.dkr.ecr.${script.env.region}.amazonaws.com/${script.env.project}/${script.env.environment}/${script.env.component}:${script.env.appVersion} .
                docker images
                docker push ${script.env.account_id}.dkr.ecr.${script.env.region}.amazonaws.com/${script.env.project}/${script.env.environment}/${script.env.component}:${script.env.appVersion}
            """
        }
    }
}
```
Example: SonarQubeSteps.groovy

```
package org.example.build

class SonarQubeSteps {
    static void runSonarQubeAnalysis(script, service) {
        script.withSonarQubeEnv('SonarQube') {
            script.dir(service) {
                script.sh """
                    ${script.env.SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=${script.env.project}-${script.env.environment}-${script.env.component} \
                    -Dsonar.sources=. \
                    -Dsonar.host.url=http://your-sonarqube-server:9000 \
                    -Dsonar.login=${script.env.SONARQUBE_TOKEN} \
                    -Dsonar.projectVersion=${script.env.appVersion} \
                    -Dsonar.language=js \
                    -Dsonar.sourceEncoding=UTF-8
                """
            }
        }
    }
}
```

Example: DeploySteps.groovy

```
package org.example.deploy

class DeploySteps {
    static void deployApplication(script, service, environment, version) {
        script.withAWS(region: 'us-east-1', credentials: 'aws-creds-dev') {
            script.dir(service) {
                script.sh """
                    aws eks update-kubeconfig --region ${script.env.region} --name ${script.env.project}-${environment}
                    helm upgrade --install ${service} ${script.libraryResource('helm/templates')} \
                    -f ./${service}-values.yml \
                    --set deployment.image.prefix=${script.env.account_id}.dkr.ecr.${script.env.region}.amazonaws.com/${script.env.project}/${environment}/${service} \
                    --set deployment.image.tag=${version}
                """
            }
        }
    }
}
```
Updated buildApp.groovy

```

import org.example.build.BuildSteps
import org.example.build.SonarQubeSteps

def call(String service, String version) {
    pipeline {
        agent {
            label 'AGENT-1'
        }
        environment {
            appVersion = version
            region = 'us-east-1'
            account_id = '528757792370'
            project = 'expense'
            component = service
            SONARQUBE_TOKEN = credentials('sonarqube-token') // Assuming you have stored the SonarQube token in Jenkins credentials
        }
        stages {
            stage('Read the Version') {
                steps {
                    script {
                        BuildSteps.readVersion(this, service)
                    }
                }
            }
            stage('Install Dependencies') {
                steps {
                    script {
                        BuildSteps.installDependencies(this, service)
                    }
                }
            }
            stage('SonarQube Analysis') {
                environment {
                    SCANNER_HOME = tool 'sonar-6.0' //scanner config
                }
                steps {
                    script {
                        SonarQubeSteps.runSonarQubeAnalysis(this, service)
                    }
                }
            }
            stage('Build and Push Docker Image') {
                steps {
                    script {
                        BuildSteps.buildAndPushDockerImage(this, service)
                    }
                }
            }
        }
        post {
            always {
                deleteDir()
            }
        }
    }
}
```

Updated deployApp.groovy

```
import org.example.deploy.DeploySteps

def call(String service, String environment, String version) {
    pipeline {
        agent {
            label 'AGENT-1'
        }
        environment {
            appVersion = version
            region = 'us-east-1'
            account_id = '528757792370'
            project = 'expense'
            component = service
        }
        stages {
            stage('Deploy Application') {
                steps {
                    script {
                        DeploySteps.deployApplication(this, service, environment, version)
                    }
                }
            }
        }
        post {
            always {
                deleteDir()
            }
        }
    }
}
```

# pipeline logic without declarative pipeline syntax

 we can create a complete Groovy script that defines the pipeline logic without using the declarative pipeline syntax. This approach uses the scripted pipeline syntax, which provides more flexibility and control over the pipeline execution.

```
shared-library-repo/
├── vars/
│   ├── buildApp.groovy
│   ├── deployApp.groovy
│   └── seedJob.groovy
├── src/
│   └── org/
│       └── example/
│           ├── build/
│           │   ├── BuildSteps.groovy
│           │   └── SonarQubeSteps.groovy
│           └── deploy/
│               └── DeploySteps.groovy
├── resources/
│   ├── someTemplate.groovy
│   └── jobTemplate.groovy
├── helm/
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ...
└── config/
    └── services.json
```

Example: buildApp.groovy

```
import org.example.build.BuildSteps
import org.example.build.SonarQubeSteps

def call(String service, String version) {
    node('AGENT-1') {
        env.appVersion = version
        env.region = 'us-east-1'
        env.account_id = '528757792370'
        env.project = 'expense'
        env.component = service
        env.SONARQUBE_TOKEN = credentials('sonarqube-token') // Assuming you have stored the SonarQube token in Jenkins credentials

        try {
            stage('Read the Version') {
                BuildSteps.readVersion(this, service)
            }
            stage('Install Dependencies') {
                BuildSteps.installDependencies(this, service)
            }
            stage('SonarQube Analysis') {
                env.SCANNER_HOME = tool 'sonar-6.0' //scanner config
                SonarQubeSteps.runSonarQubeAnalysis(this, service)
            }
            stage('Build and Push Docker Image') {
                BuildSteps.buildAndPushDockerImage(this, service)
            }
        } finally {
            deleteDir()
        }
    }
}
```

deployApp.groovy

```
import org.example.deploy.DeploySteps

def call(String service, String environment, String version) {
    node('AGENT-1') {
        env.appVersion = version
        env.region = 'us-east-1'
        env.account_id = '528757792370'
        env.project = 'expense'
        env.component = service

        try {
            stage('Deploy Application') {
                DeploySteps.deployApplication(this, service, environment, version)
            }
        } finally {
            deleteDir()
        }
    }
}
```

Example: BuildSteps.groovy

```
package org.example.build

class BuildSteps {
    static void readVersion(script, service) {
        script.dir(service) {
            def packageJson = script.readJSON file: 'package.json'
            script.env.appVersion = packageJson.version
            script.echo "App Version: ${script.env.appVersion}"
        }
    }

    static void installDependencies(script, service) {
        script.dir(service) {
            script.sh 'npm install'
        }
    }

    static void buildAndPushDockerImage(script, service) {
        script.dir(service) {
            script.sh """
                aws ecr get-login-password --region ${script.env.region} | docker login --username AWS --password-stdin ${script.env.account_id}.dkr.ecr.${script.env.region}.amazonaws.com
                docker build -t ${script.env.account_id}.dkr.ecr.${script.env.region}.amazonaws.com/${script.env.project}/${script.env.environment}/${script.env.component}:${script.env.appVersion} .
                docker images
                docker push ${script.env.account_id}.dkr.ecr.${script.env.region}.amazonaws.com/${script.env.project}/${script.env.environment}/${script.env.component}:${script.env.appVersion}
            """
        }
    }
}
```

Example: SonarQubeSteps.groovy

```
package org.example.build

class SonarQubeSteps {
    static void runSonarQubeAnalysis(script, service) {
        script.withSonarQubeEnv('SonarQube') {
            script.dir(service) {
                script.sh """
                    ${script.env.SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=${script.env.project}-${script.env.environment}-${script.env.component} \
                    -Dsonar.sources=. \
                    -Dsonar.host.url=http://your-sonarqube-server:9000 \
                    -Dsonar.login=${script.env.SONARQUBE_TOKEN} \
                    -Dsonar.projectVersion=${script.env.appVersion} \
                    -Dsonar.language=js \
                    -Dsonar.sourceEncoding=UTF-8
                """
            }
        }
    }
}

```

Example: DeploySteps.groovy

```
package org.example.deploy

class DeploySteps {
    static void deployApplication(script, service, environment, version) {
        script.withAWS(region: 'us-east-1', credentials: 'aws-creds-dev') {
            script.dir(service) {
                script.sh """
                    aws eks update-kubeconfig --region ${script.env.region} --name ${script.env.project}-${environment}
                    helm upgrade --install ${service} ${script.libraryResource('helm/templates')} \
                    -f ./${service}-values.yml \
                    --set deployment.image.prefix=${script.env.account_id}.dkr.ecr.${script.env.region}.amazonaws.com/${script.env.project}/${environment}/${service} \
                    --set deployment.image.tag=${version}
                """
            }
        }
    }
}
```

jenkinsfile.build

```
@Library('shared-library-repo') _

buildApp('backend', '1.0.0')
```

jenkinsfile.eks

```
@Library('shared-library-repo') _

deployApp('backend', 'dev', '1.0.0')
```

# complete end-to-end from seed job to build and deployment

```
shared-library-repo/
├── vars/
│   ├── buildApp.groovy
│   ├── deployApp.groovy
│   └── seedJob.groovy
├── src/
│   └── org/
│       └── example/
│           ├── build/
│           │   ├── BuildSteps.groovy
│           │   └── SonarQubeSteps.groovy
│           └── deploy/
│               └── DeploySteps.groovy
├── resources/
│   ├── someTemplate.groovy
│   └── jobTemplate.groovy
├── helm/
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ...
└── config/
    └── services.json

```

Example: services.json

```

{
  "services": [
    {
      "name": "backend",
      "repository": "https://github.com/your-repo/backend.git",
      "branch": "main",
      "buildJenkinsfile": "jenkinsfile.build",
      "deployJenkinsfile": "jenkinsfile.eks",
      "folder": "EXPENSE/DEV"
    },
    {
      "name": "frontend",
      "repository": "https://github.com/your-repo/frontend.git",
      "branch": "main",
      "buildJenkinsfile": "jenkinsfile.build",
      "deployJenkinsfile": "jenkinsfile.eks",
      "folder": "EXPENSE/DEV"
    }
  ]
}

```
Example: jobTemplate.groovy

```
def jobTemplate = { service ->
    // Create the build job
    pipelineJob("${service.folder}/${service.name}-build") {
        definition {
            cpsScm {
                scm {
                    git {
                        remote {
                            url(service.repository)
                            credentials('your-credentials-id')
                        }
                        branches(service.branch)
                    }
                }
                scriptPath(service.buildJenkinsfile)
            }
        }
    }

    // Create the deploy job
    pipelineJob("${service.folder}/${service.name}-deploy") {
        definition {
            cpsScm {
                scm {
                    git {
                        remote {
                            url(service.repository)
                            credentials('your-credentials-id')
                        }
                        branches(service.branch)
                    }
                }
                scriptPath(service.deployJenkinsfile)
            }
        }
    }
}

return jobTemplate

```

Example: seedJob.groovy

```

import groovy.json.JsonSlurper

// Read the JSON configuration file
def jsonFile = new File("${libraryResource('config/services.json')}")
def jsonContent = new JsonSlurper().parseText(jsonFile.text)

// Load the job template
def jobTemplate = evaluate(new File("${libraryResource('resources/jobTemplate.groovy')}"))

// Loop through each service and create the necessary folders and jobs
jsonContent.services.each { service ->
    // Create the folder structure if it doesn't exist
    def folderPath = service.folder.split('/')
    def currentFolder = ''
    folderPath.each { folder ->
        currentFolder = currentFolder ? "${currentFolder}/${folder}" : folder
        folder(currentFolder) {}
    }

    // Create the pipeline jobs using the template
    jobTemplate(service)
}
```

Example: BuildSteps.groovy

```
package org.example.build

class BuildSteps {
    static void readVersion(script, service) {
        script.dir(service) {
            def packageJson = script.readJSON file: 'package.json'
            script.env.appVersion = packageJson.version
            script.echo "App Version: ${script.env.appVersion}"
        }
    }

    static void installDependencies(script, service) {
        script.dir(service) {
            script.sh 'npm install'
        }
    }

    static void buildAndPushDockerImage(script, service) {
        script.dir(service) {
            script.sh """
                aws ecr get-login-password --region ${script.env.region} | docker login --username AWS --password-stdin ${script.env.account_id}.dkr.ecr.${script.env.region}.amazonaws.com
                docker build -t ${script.env.account_id}.dkr.ecr.${script.env.region}.amazonaws.com/${script.env.project}/${script.env.environment}/${script.env.component}:${script.env.appVersion} .
                docker images
                docker push ${script.env.account_id}.dkr.ecr.${script.env.region}.amazonaws.com/${script.env.project}/${script.env.environment}/${script.env.component}:${script.env.appVersion}
            """
        }
    }
}
```
Example: SonarQubeSteps.groovy

```
package org.example.build

class SonarQubeSteps {
    static void runSonarQubeAnalysis(script, service) {
        script.withSonarQubeEnv('SonarQube') {
            script.dir(service) {
                script.sh """
                    ${script.env.SCANNER_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=${script.env.project}-${script.env.environment}-${script.env.component} \
                    -Dsonar.sources=. \
                    -Dsonar.host.url=http://your-sonarqube-server:9000 \
                    -Dsonar.login=${script.env.SONARQUBE_TOKEN} \
                    -Dsonar.projectVersion=${script.env.appVersion} \
                    -Dsonar.language=js \
                    -Dsonar.sourceEncoding=UTF-8
                """
            }
        }
    }
}
```

Example: DeploySteps.groovy

```
package org.example.deploy

class DeploySteps {
    static void deployApplication(script, service, environment, version) {
        script.withAWS(region: 'us-east-1', credentials: 'aws-creds-dev') {
            script.dir(service) {
                script.sh """
                    aws eks update-kubeconfig --region ${script.env.region} --name ${script.env.project}-${environment}
                    helm upgrade --install ${service} ${script.libraryResource('helm/templates')} \
                    -f ./${service}-values.yml \
                    --set deployment.image.prefix=${script.env.account_id}.dkr.ecr.${script.env.region}.amazonaws.com/${script.env.project}/${environment}/${service} \
                    --set deployment.image.tag=${version}
                """
            }
        }
    }
}
```

Example: buildApp.groovy

```
import org.example.build.BuildSteps
import org.example.build.SonarQubeSteps

def call(String service, String version) {
    node('AGENT-1') {
        env.appVersion = version
        env.region = 'us-east-1'
        env.account_id = '528757792370'
        env.project = 'expense'
        env.component = service
        env.SONARQUBE_TOKEN = credentials('sonarqube-token') // Assuming you have stored the SonarQube token in Jenkins credentials

        try {
            stage('Read the Version') {
                BuildSteps.readVersion(this, service)
            }
            stage('Install Dependencies') {
                BuildSteps.installDependencies(this, service)
            }
            stage('SonarQube Analysis') {
                env.SCANNER_HOME = tool 'sonar-6.0' //scanner config
                SonarQubeSteps.runSonarQubeAnalysis(this, service)
            }
            stage('Build and Push Docker Image') {
                BuildSteps.buildAndPushDockerImage(this, service)
            }
        } finally {
            deleteDir()
        }
    }
}
```
Example: deployApp.groovy

```
import org.example.deploy.DeploySteps

def call(String service, String environment, String version) {
    node('AGENT-1') {
        env.appVersion = version
        env.region = 'us-east-1'
        env.account_id = '528757792370'
        env.project = 'expense'
        env.component = service

        try {
            stage('Deploy Application') {
                DeploySteps.deployApplication(this, service, environment, version)
            }
        } finally {
            deleteDir()
        }
    }
}
```
Example: jenkinsfile.build

```
@Library('shared-library-repo') _

buildApp('backend', '1.0.0')
```
Example: jenkinsfile.eks

```
@Library('shared-library-repo') _

deployApp('backend', 'dev', '1.0.0')
```