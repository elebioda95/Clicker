#!/usr/bin/env groovy


def label="clicker-${UUID.randomUUID().toString()}"
def gitCommit 
podTemplate(yaml: '''
    apiVersion: v1
    kind: Pod
    spec:
      containers:
      - name: node
        image: node:10-alpine
        ttyEnabled: true
        command:
        - /bin/sh
        args:
        - -c cat
      - name: dind
        image: docker:20-dind
        privileged: true
        env:
        - name: DOCKER_TLS_CERTDIR
          value: ''
      nodeSelector:
        kubernetes.io/hostname: ip-10-0-2-202.ec2.internal      
''')
{
  timeout(time: 4, unit: 'HOURS') {
    node(label) {
      stage ("Checkout SCM") {
        def scmVars = checkout scm
        def lastCommit = sh script: 'git log -1 --pretty=%B', returnStdout: true
        echo ("last commit: ${lastCommit}")
        echo ("commit HASH: ${scmVars.GIT_COMMIT}")
        gitCommit = scmVars.GIT_COMMIT
      }

      stage('Test project') {
        container('node') {
            sh 'npm ci --no-optional'
            def passed = sh script: 'npm test a -- --coverage --coverageReporters="json-summary"', returnStatus: true
            if (passed != 0) {
                  currentBuild.result = 'ABORTED'
                  error('Failed unit tests!')
            }
          }
        }

      stage('Coverage above 70%') {
        container('node') {
          sh 'apk add jq'
          sh 'ls -la'
          def coverage = sh script: 'cat coverage/coverage-summary.json | jq \'.total.lines.pct\'', returnStdout: true
          echo ("coverage: ${coverage}")
        }
      }


      stage('Build Project') {
        container('dind') {
          sh 'docker build .'
        }
      }

    }
  }
}