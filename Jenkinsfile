node {
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
    docker.image('klakegg/hugo:ext-alpine-ci').inside {
      sh 'hugo --minify'
      // Compress assets
      sh """
          apk add --update brotli gzip
          findRegex='.*\\.\\(htm\\|html\\|txt\\|text\\|js\\|css\\|svg\\|xml\\|map\\|json\\)\$'
          archOpts='-f -k --best'
          find public -type f -regex \$findRegex -exec gzip \$archOpts {} \\; -exec brotli \$archOpts {} \\;
      """

    }
    docker.withRegistry('https://registry.home.devmem.ru') {
      env.DOCKER_BUILDKIT = 1
      def myImage = docker.build("devmem-ru:${env.GIT_COMMIT}", "--progress=plain --cache-from registry.home.devmem.ru/devmem-ru:latest -f ./.docker/Dockerfile .")
      myImage.push()
      myImage.push('latest')
      // Untag and remove image by sha256 id
      sh "docker rmi -f \$(docker inspect -f '{{ .Id }}' ${myImage.id})"
    }
  }

  stage('Deploy') {
    env.ANSIBLE_CONFIG = '.ansible/ansible.cfg'
    docker.image('cytopia/ansible:latest-infra').inside {
      sh 'ansible --version'
      ansiblePlaybook(
        playbook: '.ansible/playbook.yml',
        inventory: '.ansible/hosts',
        credentialsId: 'jenkins-ssh-key',
        colorized: true)
    }
  }
}
