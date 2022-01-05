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
      def appImage = docker.build("devmem-ru:${env.GIT_COMMIT}", "-f ./compose/Dockerfile ./public")
      appImage.push()
      appImage.push('latest')
    }
  }
}
