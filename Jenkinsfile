

stage('Configure') {
    abort = false
    inputConfig = input id: 'InputConfig', message: 'Docker registry and Anchore Engine configuration', parameters: [string(defaultValue: 'https://index.docker.io/v1/', description: 'URL of the docker registry for staging images before analysis', name: 'dockerRegistryUrl', trim: true), string(defaultValue: 'docker.io', description: 'Hostname of the docker registry', name: 'dockerRegistryHostname', trim: true), string(defaultValue: 'hcheungl3harris/testdummy_anchore', description: 'Name of the docker repository', name: 'dockerRepository', trim: true), credentials(credentialType: 'com.cloudbees.plugins.credentials.impl.UsernamePasswordCredentialsImpl', defaultValue: '', description: 'Credentials for connecting to the docker registry', name: 'dockerCredentials', required: true)]

    for (config in inputConfig) {
        if (null == config.value || config.value.length() <= 0) {
          echo "${config.key} cannot be left blank"
          abort = true
        }
    }

    if (abort) {
        currentBuild.result = 'ABORTED'
        error('Aborting build due to invalid input')
    }
}

node {
  def app
  def dockerfile
  def anchorefile
  def repotag
  def dockerRegistryUrl = "https://index.docker.io/v1/"
  def dockerRegistryHostname = "docker.io"
  def dockerRepository = "hcheungl3harris/testdummy_anchore"
  def anchoreEngineUrl = "http://henry.anchore-testing123.net:8228/v1"

  try {
    stage('Checkout') {
      // Clone the git repository
      checkout scm
      def path = sh returnStdout: true, script: "pwd"
      path = path.trim()
      dockerfile = path + "/Dockerfile"
      anchorefile = path + "/anchore_images"
      echo "this is the repotag ${path}"
    }

    stage('Build') {
      // Build the image and push it to a staging repository
      repotag = inputConfig['dockerRepository'] + ":${BUILD_NUMBER}"
      echo "this is the repotag ${repotag}"
      docker.withRegistry(inputConfig['dockerRegistryUrl'], inputConfig['dockerCredentials']) {
        app = docker.build(repotag)
        app.push()
      }
    }

    stage('Parallel') {
      parallel Test: {
        app.inside {
            sh 'echo "Dummy - tests passed"'
        }
      },
      Analyze: {
        writeFile file: anchorefile, text: dockerRegistryHostname + "/" + repotag + " " + dockerfile
        anchore name: anchorefile, engineurl: anchoreEngineUrl, engineCredentialsId: 'anchore', annotations: [[key: 'added-by', value: 'jenkins']]
      }
    }
  } finally {
    stage('Cleanup') {
      // Delete the docker image and clean up any allotted resources
      sh script: "docker rmi " + repotag
    }
  }
}
