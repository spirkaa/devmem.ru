node {
  env.REGISTRY = 'git.devmem.ru'
  env.REGISTRY_CREDS_ID = 'gitea-user'
  env.IMAGE_OWNER = 'cr'
  env.IMAGE_NAME = 'devmem-ru'
  env.DOCKERFILE = './.docker/Dockerfile'
  env.LABEL_AUTHORS = 'Ilya Pavlov <piv@devmem.ru>'
  env.LABEL_TITLE = 'devmem-ru'
  env.LABEL_DESCRIPTION = 'My hugo site'
  env.LABEL_URL = 'https://devmem.ru'
  env.LABEL_CREATED = sh(script: "date '+%Y-%m-%dT%H:%M:%S%:z'", returnStdout: true).toString().trim()

  env.HUGO_IMAGE = 'klakegg/hugo:ext-alpine-ci'
  env.ANSIBLE_IMAGE = 'cytopia/ansible:latest-infra'
  env.ANSIBLE_CONFIG = '.ansible/ansible.cfg'
  env.ANSIBLE_PLAYBOOK = '.ansible/playbook.yml'
  env.ANSIBLE_INVENTORY = '.ansible/hosts'
  env.ANSIBLE_CREDS_ID = 'jenkins-ssh-key'

  properties([
    buildDiscarder(
      logRotator(
        daysToKeepStr: '60',
        numToKeepStr: '10'
      )
    )
  ])

  stage('Checkout') {
    def scmVars = checkout([
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

    env.GIT_COMMIT = scmVars.GIT_COMMIT.take(7)
    env.GIT_URL = scmVars.GIT_URL.toString().trim()
  }

  stage('Build') {
    // Build Hugo site
    image = docker.image("${env.HUGO_IMAGE}")
    image.pull()
    image.inside {
      sh 'hugo --minify'
      // Compress assets
      sh """
          apk add --update brotli gzip
          findRegex='.*\\.\\(htm\\|html\\|txt\\|text\\|js\\|css\\|svg\\|xml\\|map\\|json\\)\$'
          archOpts='-f -k --best'
          find public -type f -regex \$findRegex -exec gzip \$archOpts {} \\; -exec brotli \$archOpts {} \\;
      """
    }
    // Build image
    docker.withRegistry("https://${env.REGISTRY}", "${env.REGISTRY_CREDS_ID}") {
      env.DOCKER_BUILDKIT = 1
      env.CACHE_FROM = "${env.REGISTRY}/${env.IMAGE_OWNER}/${env.IMAGE_NAME}:latest"

      def myImage = docker.build(
        "${env.IMAGE_OWNER}/${env.IMAGE_NAME}:${env.GIT_COMMIT}",
        "--label \"org.opencontainers.image.created=${env.LABEL_CREATED}\" \
        --label \"org.opencontainers.image.authors=${env.LABEL_AUTHORS}\" \
        --label \"org.opencontainers.image.url=${env.LABEL_URL}\" \
        --label \"org.opencontainers.image.source=${env.GIT_URL}\" \
        --label \"org.opencontainers.image.version=${env.GIT_COMMIT}\" \
        --label \"org.opencontainers.image.revision=${env.GIT_COMMIT}\" \
        --label \"org.opencontainers.image.title=${env.LABEL_TITLE}\" \
        --label \"org.opencontainers.image.description=${env.LABEL_DESCRIPTION}\" \
        --progress=plain \
        --cache-from ${env.CACHE_FROM} \
        -f ${env.DOCKERFILE} ."
      )
      myImage.push()
      myImage.push('latest')
      // Untag and remove image by sha256 id
      sh "docker rmi -f \$(docker inspect -f '{{ .Id }}' ${myImage.id})"
    }
  }

  stage('Deploy') {
    withCredentials([usernamePassword(credentialsId: "${env.REGISTRY_CREDS_ID}", usernameVariable: 'REGISTRY_USER', passwordVariable: 'REGISTRY_PASSWORD')]) {
      image = docker.image("${env.ANSIBLE_IMAGE}")
      image.pull()
      image.inside {
        sh 'ansible --version'
        ansiblePlaybook(
          colorized: true,
          playbook: "${env.ANSIBLE_PLAYBOOK}",
          inventory: "${env.ANSIBLE_INVENTORY}",
          credentialsId: "${env.ANSIBLE_CREDS_ID}",
        )
      }
    }
  }
}
