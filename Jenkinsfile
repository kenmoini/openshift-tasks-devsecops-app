def version, mvnCmd = "mvn -s configuration/cicd-settings-nexus3.xml"
pipeline {
  agent {
    node {
      label 'maven'
    }
  }
  stages {
    stage('Build App') {
      steps {
        git branch: 'eap-7', url: 'http://gogs:3000/gogs/openshift-tasks.git'
        script {
            def pom = readMavenPom file: 'pom.xml'
            version = pom.version
        }
        sh "${mvnCmd} install -DskipTests=true"
      }
    }
    stage('Test') {
      steps {
        sh "${mvnCmd} test"
        step([$class: 'JUnitResultArchiver', testResults: '**/target/surefire-reports/TEST-*.xml'])
      }
    }
    stage('Code Analysis') {
      steps {
        script {
          sh "${mvnCmd} sonar:sonar -Dsonar.host.url=http://sonarqube:9000 -DskipTests=true"
        }
      }
    }
    stage('Archive App') {
      steps {
        sh "${mvnCmd} deploy -DskipTests=true -P nexus3"
      }
    }
    stage('Create Image Builder') {
      when {
        expression {
          openshift.withCluster() {
            openshift.withProject(env.DEV_PROJECT) {
              return !openshift.selector("bc", "tasks").exists();
            }
          }
        }
      }
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.DEV_PROJECT) {
              openshift.newBuild("--name=tasks", "--image-stream=jboss-eap70-openshift:1.5", "--binary=true")
            }
          }
        }
      }
    }
    stage('Build Image') {
      steps {
        sh "rm -rf oc-build && mkdir -p oc-build/deployments"
        sh "cp target/openshift-tasks.war oc-build/deployments/ROOT.war"
        script {
          openshift.withCluster() {
            openshift.withProject(env.DEV_PROJECT) {
              openshift.selector("bc", "tasks").startBuild("--from-dir=oc-build", "--wait=true")
            }
          }
        }
      }
    }
    stage('Create DEV') {
      when {
        expression {
          openshift.withCluster() {
            openshift.withProject(env.DEV_PROJECT) {
              return !openshift.selector('dc', 'tasks').exists()
            }
          }
        }
      }
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.DEV_PROJECT) {
              def app = openshift.newApp("tasks:latest")
              app.narrow("svc").expose();
              def dc = openshift.selector("dc", "tasks")
              while (dc.object().spec.replicas != dc.object().status.availableReplicas) {
                  sleep 10
              }
              openshift.set("triggers", "dc/tasks", "--manual")
            }
          }
        }
      }
    }
    stage('Deploy DEV') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.DEV_PROJECT) {
              openshift.selector("dc", "tasks").rollout().latest();
            }
          }
        }
      }
    }
    stage('Promote to STAGE?') {
      steps {
        timeout(time:15, unit:'MINUTES') {
            input message: "Promote to STAGE?", ok: "Promote"
        }
        script {
          openshift.withCluster() {
            openshift.tag("${env.DEV_PROJECT}/tasks:latest", "${env.STAGE_PROJECT}/tasks:${version}")
          }
        }
      }
    }
    stage('Deploy STAGE') {
      steps {
        script {
          openshift.withCluster() {
            openshift.withProject(env.STAGE_PROJECT) {
              if (openshift.selector('dc', 'tasks').exists()) {
                openshift.selector('dc', 'tasks').delete()
                openshift.selector('svc', 'tasks').delete()
                openshift.selector('route', 'tasks').delete()
              }
              openshift.newApp("tasks:${version}").narrow("svc").expose()
            }
          }
        }
      }
    }
  }
}
