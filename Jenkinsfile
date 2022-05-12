pipeline {
  agent any

  options {
    buildDiscarder(logRotator(numToKeepStr: '10', daysToKeepStr: '60'))
    parallelsAlwaysFailFast()
    disableConcurrentBuilds()
  }

  environment {
    REGISTRY = 'git.devmem.ru'
    REGISTRY_URL = "https://${REGISTRY}"
    REGISTRY_CREDS_ID = 'gitea-user'
    IMAGE_OWNER = 'projects'
    IMAGE_BASENAME = 'devmem-ru'
    IMAGE_FULLNAME = "${REGISTRY}/${IMAGE_OWNER}/${IMAGE_BASENAME}"
    DOCKERFILE = '.docker/Dockerfile'
    LABEL_AUTHORS = 'Ilya Pavlov <piv@devmem.ru>'
    LABEL_TITLE = 'My Hugo site'
    LABEL_DESCRIPTION = 'My Hugo site'
    LABEL_URL = 'https://devmem.ru'
    LABEL_CREATED = sh(script: "date '+%Y-%m-%dT%H:%M:%S%:z'", returnStdout: true).toString().trim()
    REVISION = GIT_COMMIT.take(7)

    HUGO_IMAGE = 'klakegg/hugo:ext-alpine-ci'
    ANSIBLE_IMAGE = "${REGISTRY}/projects/ansible:base"
    ANSIBLE_CONFIG = '.ansible/ansible.cfg'
    ANSIBLE_PLAYBOOK = '.ansible/playbook.yml'
    ANSIBLE_INVENTORY = '.ansible/hosts'
    ANSIBLE_CREDS_ID = 'jenkins-ssh-key'
  }

  parameters {
    string(name: 'ANSIBLE_EXTRAS', defaultValue: '', description: 'ansible-playbook extra params')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout([
          $class: 'GitSCM',
          branches: scm.branches,
          extensions: scm.extensions + [
            [$class: 'SubmoduleOption',
              disableSubmodules: false,
              parentCredentials: false,
              recursiveSubmodules: true,
              trackingSubmodules: true,
              reference: ''
            ]
          ],
          userRemoteConfigs: scm.userRemoteConfigs
        ])
      }
    }

    stage('Build assets') {
      steps {
        script {
          image = docker.image("${HUGO_IMAGE}")
          image.pull()
          image.inside {
            sh 'hugo --minify'
            sh """
                apk add --update brotli gzip
                findRegex='.*\\.\\(htm\\|html\\|txt\\|text\\|js\\|css\\|svg\\|xml\\|map\\|json\\)\$'
                archOpts='-f -k --best'
                find public -type f -regex \$findRegex -exec gzip \$archOpts {} \\; -exec brotli \$archOpts {} \\;
            """
          }
        }
      }
      post {
        always {
          sh "docker rmi ${HUGO_IMAGE}"
        }
      }
    }

    stage('Build image') {
      steps {
        script {
          buildDockerImage(
            dockerFile: "${DOCKERFILE}",
            tag: "${REVISION}",
            altTag: 'latest'
          )
        }
      }
    }

    stage('Deploy') {
      agent {
        docker {
          image env.ANSIBLE_IMAGE
          registryUrl env.REGISTRY_URL
          registryCredentialsId env.REGISTRY_CREDS_ID
          alwaysPull true
          reuseNode true
        }
      }
      steps {
        sh 'ansible --version'
        withCredentials([
          usernamePassword(credentialsId: "${REGISTRY_CREDS_ID}", usernameVariable: 'REGISTRY_USER', passwordVariable: 'REGISTRY_PASSWORD')
          ]) {
          ansiblePlaybook(
            colorized: true,
            credentialsId: "${ANSIBLE_CREDS_ID}",
            playbook: "${ANSIBLE_PLAYBOOK}",
            extras: "${params.ANSIBLE_EXTRAS} --syntax-check"
          )
          ansiblePlaybook(
            colorized: true,
            credentialsId: "${ANSIBLE_CREDS_ID}",
            playbook: "${ANSIBLE_PLAYBOOK}",
            extras: "${params.ANSIBLE_EXTRAS}"
          )
        }
      }
    }
  }

  post {
    always {
      emailext(
        to: '$DEFAULT_RECIPIENTS',
        subject: '$DEFAULT_SUBJECT',
        body: '$DEFAULT_CONTENT'
      )
    }
  }
}
