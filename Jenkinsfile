pipeline {
  agent {
    kubernetes {
      label 'regression-gitbase'
      inheritFrom 'performance'
      defaultContainer 'regression-gitbase'
      containerTemplate {
        name 'regression-gitbase'
        image 'srcd/regression-gitbase:v0.1.0-beta.2'
        ttyEnabled true
        command 'cat'
      }
    }
  }
  environment {
    GOPATH = "/go"
    GO_IMPORT_PATH = "github.com/src-d/regression-gibase"
    GO_IMPORT_FULL_PATH = "${env.GOPATH}/src/${env.GO_IMPORT_PATH}"
  }
  stages {
    stage('Run') {
      steps {
        sh '/bin/regression --complexity=2 --csv local:HEAD'
      }
    }
    stage('Plot') {
      steps {
        script {
          plotFiles = findFiles(glob: "plot_*.csv")
          plotFiles.each {
            echo "plot ${it.getName()}"
            sh "cat ${it.getName()}"
            plot(
              group: 'performance',
              csvFileName: it.getName(),
              title: it.getName(),
              numBuilds: '100',
              style: 'line',
              csvSeries: [[
                displayTableFlag: false,
                exclusionValues: '',
                file: it.getName(),
                inclusionFlag: 'OFF',
              ]]
            )
          }
        }
      }
    }
  }
}
