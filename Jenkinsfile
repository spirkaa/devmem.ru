node {
  env.REGISTRY = 'registry.home.devmem.ru'
  env.IMAGE_NAME = 'devmem-ru'
  env.HUGO_IMAGE = 'klakegg/hugo:ext-alpine-ci'
  env.DOCKERFILE = './.docker/Dockerfile'
  env.ANSIBLE_IMAGE = 'cytopia/ansible:latest-infra'
  env.ANSIBLE_CONFIG = '.ansible/ansible.cfg'
  env.ANSIBLE_PLAYBOOK = '.ansible/playbook.yml'
  env.ANSIBLE_INVENTORY = '.ansible/hosts'
  env.ANSIBLE_CREDENTIALS_ID = 'jenkins-ssh-key'

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
  }

  stage('Build') {
    // Build Hugo site
    docker.image("${env.HUGO_IMAGE}").inside {
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
    docker.withRegistry("https://${env.REGISTRY}") {
      env.DOCKER_BUILDKIT = 1
      env.CACHE_FROM = "${env.REGISTRY}/${env.IMAGE_NAME}:latest"

      def myImage = docker.build("${env.IMAGE_NAME}:${env.GIT_COMMIT}", "--progress=plain --cache-from ${env.CACHE_FROM} -f ${env.DOCKERFILE} .")
      myImage.push()
      myImage.push('latest')
      // Untag and remove image by sha256 id
      sh "docker rmi -f \$(docker inspect -f '{{ .Id }}' ${myImage.id})"
    }
  }

  stage('Deploy') {
    docker.image("${env.ANSIBLE_IMAGE}").inside {
      sh 'ansible --version'
      ansiblePlaybook(
        playbook: "${env.ANSIBLE_PLAYBOOK}",
        inventory: "${env.ANSIBLE_INVENTORY}",
        credentialsId: "${env.ANSIBLE_CREDENTIALS_ID}",
        colorized: true)
    }
  }
}
