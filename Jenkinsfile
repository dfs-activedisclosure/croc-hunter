#!/usr/bin/groovy

// load pipeline functions
// Requires pipeline-github-lib plugin to load library from github
@Library('github.com/campbelldgunn/jenkins-pipeline')
def pipeline = new org.whiteshieldinc.Pipeline()

podTemplate(label: 'jenkins-pipeline', nodeSelector: 'beta.kubernetes.io/os=linux', containers: [
    containerTemplate(name: 'jnlp', image: 'jenkinsci/jnlp-slave:3.19-1-alpine', args: '${computer.jnlpmac} ${computer.name}', workingDir: '/home/jenkins', resourceRequestCpu: '200m', resourceLimitCpu: '200m', resourceRequestMemory: '256Mi', resourceLimitMemory: '384Mi'),
    containerTemplate(name: 'docker', image: 'docker:18.03', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'golang', image: 'golang:1.8.3', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'helm', image: 'campbelldgunn/k8s-helm:latest', command: 'cat', ttyEnabled: true),
    containerTemplate(name: 'kubectl', image: 'campbelldgunn/k8s-kubectl:latest', command: 'cat', ttyEnabled: true)
],
volumes:[
    hostPathVolume(mountPath: '/var/run/docker.sock', hostPath: '/var/run/docker.sock'),
]){

  node ('jenkins-pipeline') {

    def pwd = pwd()
    def chart_dir = "${pwd}/charts/croc-hunter"

    checkout scm

    // read in required jenkins workflow config values
    def inputFile = readFile('Jenkinsfile.json')
    def config = new groovy.json.JsonSlurperClassic().parseText(inputFile)
    println "pipeline config ==> ${config}"

    // continue only if pipeline enabled
    if (!config.pipeline.enabled) {
        println "pipeline disabled"
        return
    }

    // set additional git envvars for image tagging
    pipeline.gitEnvVars()

    // If pipeline debugging enabled
    if (config.pipeline.debug) {
      println "DEBUG ENABLED"
      sh "env | sort"

      println "Runing kubectl/helm tests"
      container('kubectl') {
        pipeline.kubectlTest()
      }
      container('helm') {
        pipeline.helmConfig()
      }
    }

    def acct = pipeline.getContainerRepoAcct(config)

    // tag image with version, and branch-commit_id
    def tags = pipeline.getContainerTags(config)

    stage ('compile and test') {

      container('golang') {
        sh "go get -u github.com/jstemmer/go-junit-report"
        //sh "go test -v -race ./..."
        sh "go test -v -race ./... | go-junit-report > report.xml"
        sh "make bootstrap build"
      }
    }

    stage ('test deployment') {

      container('helm') {

        // run helm chart linter
        pipeline.helmLint(chart_dir)

        // run dry-run helm chart installation
        pipeline.helmDeploy(
          dry_run       : true,
          name          : config.app.name,
          namespace     : config.app.name,
          version_tag   : tags.get(0),
          chart_dir     : chart_dir,
          replicas      : config.app.replicas,
          cpu           : config.app.cpu,
          memory        : config.app.memory,
          hostname      : config.app.hostname
        )

      }
    }

    stage ('publish container') {

      container('docker') {

        // perform docker login to docker as the docker-pipeline-plugin doesn't work with the next auth json format
        withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: config.container_repo.jenkins_creds_id,
                        usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD']]) {
          sh "docker login -u ${env.USERNAME} -p ${env.PASSWORD} dfsacr.azurecr.io"
        }

        // build and publish container
        pipeline.containerBuildPub(
            dockerfile: config.container_repo.dockerfile,
            host      : config.container_repo.host,
            acct      : acct,
            repo      : config.container_repo.repo,
            tags      : tags,
            auth_id   : config.container_repo.jenkins_creds_id
        )
        // Logging output of the container from the template
        containerLog 'docker'
      }

    }

    if (env.BRANCH_NAME =~ "PR-*" ) {
      stage ('deploy to k8s') {
        container('helm') {
          // Deploy using Helm chart
          pipeline.helmDeploy(
            dry_run       : false,
            name          : env.BRANCH_NAME.toLowerCase(),
            namespace     : env.BRANCH_NAME.toLowerCase(),
            version_tag   : tags.get(0),
            chart_dir     : chart_dir,
            replicas      : config.app.replicas,
            cpu           : config.app.cpu,
            memory        : config.app.memory,
            hostname      : config.app.hostname
          )

          //  Run helm tests
          if (config.app.test) {
            pipeline.helmTest(
              name        : env.BRANCH_NAME.toLowerCase()
            )
          }

          // delete test deployment
          pipeline.helmDelete(
              name       : env.BRANCH_NAME.toLowerCase()
          )
        }
      }
    }

    // deploy only the master branch
    if (env.BRANCH_NAME == 'master') {
      stage ('deploy to k8s') {
        container('helm') {
          // Deploy using Helm chart
          pipeline.helmDeploy(
            dry_run       : false,
            name          : config.app.name,
            namespace     : config.app.name,
            version_tag   : tags.get(0),
            chart_dir     : chart_dir,
            replicas      : config.app.replicas,
            cpu           : config.app.cpu,
            memory        : config.app.memory,
            hostname      : config.app.hostname
          )

          //  Run helm tests
          if (config.app.test) {
            pipeline.helmTest(
              name          : config.app.name
            )
          }
        }
      }
    }
  }
}
