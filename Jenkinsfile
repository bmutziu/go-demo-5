import java.text.SimpleDateFormat

def props
def label = "jenkins-slave-${UUID.randomUUID().toString()}"
currentBuild.displayName = new SimpleDateFormat("yy.MM.dd").format(new Date()) + "-" + env.BUILD_NUMBER

// namespace: "go-demo-3-build", // Not allowed with declarative
// serviceAccount: "build",

pipeline {
  agent {
    go-demo-3 {
      label "${label}"
      yamlFile "KubernetesPod.yaml"
    }      
  }
  stages {
    stage("test") {
      steps {
        container("helm") {
          sh "Hello!!!"
        }
      }
    }
  }
}



) {
  node(label) {
    stage("build") {
      container("helm") {
        sh "cp /etc/config/build-config.properties ."
        props = readProperties interpolate: true, file: "build-config.properties"
      }
      node("docker") { // Not allowed with declarative
        checkout scm
        k8sBuildImageBeta(props.image)
      }
    }
    stage("func-test") {
      try {
        container("helm") {
          checkout scm
          k8sUpgradeBeta(props.project, props.domain, "--set replicaCount=2 --set dbReplicaCount=1")
        }
        container("kubectl") {
          k8sRolloutBeta(props.project)
        }
        container("golang") {
          k8sFuncTestGolang(props.project, props.domain)
        }
      } catch(e) {
          error "Failed functional tests"
      } finally {
        container("helm") {
          k8sDeleteBeta(props.project)
        }
      }
    }
    if ("${BRANCH_NAME}" == "master") {
      stage("release") {
        node("docker") {
          k8sPushImage(props.image)
        }
        container("helm") {
          k8sPushHelm(props.project, props.chartVer, props.cmAddr)
        }
      }
      stage("deploy") {
        try {
          container("helm") {
            k8sUpgrade(props.project, props.addr)
          }
          container("kubectl") {
            k8sRollout(props.project)
          }
          container("golang") {
            k8sProdTestGolang(props.addr)
          }
        } catch(e) {
          container("helm") {
            k8sRollback(props.project)
          }
        }
      }
    }
  }
}