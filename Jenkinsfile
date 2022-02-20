pipeline {
  agent {
    kubernetes {
      yamlFile 'build-agent.yaml'
      defaultContainer 'maven'
      idleMinutes 1
    }
  }
  stages {
    stage('Build') {
      parallel {
        stage('Compile') {
          steps {
            container('maven') {
              sh 'mvn compile'
            }
          }
        }
      }
    }
    stage('Static analysis') {
      parallel {
        stage('Unit Tests') {
          steps {
            container('maven') {
              sh 'mvn test'
            }
          }
        }
        stage('SCA Software Composition Analysis') {
          steps {
            container('maven') {
              catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
                sh('mvn org.owasp:dependency-check-maven:check')
              }
            }
          }
          post {
            always {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/dependency-check-report.html', fingerprint: true, onlyIfSuccessful: true
            }
          }
        }
        stage('OSS License Checker') {
          steps {
            container('license-finder') {
              sh('ls -la')
              sh('''#!/bin/bash --login
                      /bin/bash --login
                      rvm use default
                      gem install license_finder
                      license_finder
              ''')
            }
          }
        }
        stage('SBOM generation and publish to Dependency Tracker') {
          steps {
            container('maven') {
              sh('mvn org.cyclonedx:cyclonedx-maven-plugin:makeAggregateBom')
            }
          }
          post {
            success {
              dependencyTrackPublisher projectName: 'sample-spring-app', projectVersion: '1.0.0', artifact: 'target/bom.xml', autoCreateProjects: true, synchronous: true
              archiveArtifacts allowEmptyArchive: true, artifacts: 'target/bom.xml', fingerprint: true, onlyIfSuccessful: true
            }
          }
        }
        stage('Project SAST') {
          steps {
            container('sast-scan') {
              sh('scan --type java,deepscan --build')
            }
          }
          post {
            success {
              archiveArtifacts allowEmptyArchive: true, artifacts: 'reports/*', fingerprint: true, onlyIfSuccessful: true
            }
          }
        }
      }
    }
    stage('Package') {
      parallel {
        stage('Create Jarfile') {
          steps {
            container('maven') {
              sh 'mvn package -DskipTests'
            }
          }
        }
        stage('Build container image') {
            steps {
                container('kaniko') {
                    sh('/kaniko/executor -f ./Dockerfile -c . --insecure --skip-tls-verify --force --cache=true --destination=docker.io/oscmedgon/dso-demo')
                }
            }
        }
      }
    }

    stage('Deploy to Dev') {
      steps {
        // TODO
        sh "echo done again"
      }
    }
  }
}
