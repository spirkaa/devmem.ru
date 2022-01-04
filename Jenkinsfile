node {
  stage('Checkout') {
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
  stage('Build') {
    docker.image('klakegg/hugo:ext-alpine-ci').inside {
      sh 'hugo --minify'
    }
  }
}
