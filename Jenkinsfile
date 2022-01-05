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
    }
    docker.withRegistry('https://registry.home.devmem.ru') {
      def myImage = docker.build("devmem-ru:${env.GIT_COMMIT}", "-f ./compose/Dockerfile ./public")
      myImage.push()
      myImage.push('latest')
      // Untag and remove image by sha256 id
      sh "docker rmi -f \$(docker inspect -f '{{ .Id }}' ${myImage.id}) nginx:alpine"
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
