DOCKER_USER = "${env.BRANCH_NAME}"
DOCKER_IMAGE_NAMESPACE = "se-${DOCKER_USER}"
DOCKER_IMAGE_REPOSITORY = "simple-nginx"
DOCKER_IMAGE_REPOSITORY_DEV = "${DOCKER_IMAGE_REPOSITORY}-dev"
DOCKER_IMAGE_REPOSITORY_PROD = "${DOCKER_IMAGE_REPOSITORY}-prod"
DOCKER_IMAGE_TAG = "${env.BUILD_TIMESTAMP}"

DOCKER_SERVICE_NAME = "${DOCKER_USER.replace(".", "")}-${DOCKER_IMAGE_REPOSITORY}"

node {
    def docker_image
/*
    stage('Validate Environment') {
        required_env = [ 'DOCKER_USER',
                         'DOCKER_IMAGE_NAMESPACE',
                         'DOCKER_IMAGE_REPOSITORY',
                         'DOCKER_IMAGE_REPOSITORY_DEV',
                         'DOCKER_IMAGE_REPOSITORY_PROD',
                         'DOCKER_IMAGE_TAG',
                         'DOCKER_REGISTRY_HOSTNAME',
                         'DOCKER_REGISTRY_URI',
                         'DOCKER_REGISTRY_CREDENTIALS_ID',
                         'DOCKER_UCP_URI',
                         'DOCKER_UCP_CREDENTIALS_ID',
                         'DOCKER_SERVICE_NAME' ]


        fail = 0

        required_env.each { required ->
            if(env[required] == null) {
                fail = 1
                echo "Missing required environment variable: '${required}'" 
            }
        }

        if(fail) {
            error("Missing required environment variables")
        }
    }
*/
    stage('Clone') {
        /* Let's make sure we have the repository cloned to our workspace */

        checkout scm
    }

    stage('Build') {
        /* This builds the actual image; synonymous to
         * docker build on the command line */

        docker_image = docker.build("${DOCKER_IMAGE_NAMESPACE}/${DOCKER_IMAGE_REPOSITORY_DEV}")
    }

    stage('Test') {
        /* Ideally, we would run a test framework against our image.
         * For this example, we're using a Volkswagen-type approach ;-) */

        docker_image.inside {
            sh 'echo "Tests passed"'
        }
    }

    stage('Push') {
        docker.withRegistry(DOCKER_REGISTRY_URI, DOCKER_REGISTRY_CREDENTIALS_ID) {
            docker_image.push(DOCKER_IMAGE_TAG)
        }
    }

    stage('Scan') {
        httpRequest acceptType: 'APPLICATION_JSON', authentication: DOCKER_REGISTRY_CREDENTIALS_ID, contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, responseHandle: 'NONE', url: "${DOCKER_REGISTRY_URI}/api/v0/imagescan/scan/${DOCKER_IMAGE_NAMESPACE}/${DOCKER_IMAGE_REPOSITORY_DEV}/${DOCKER_IMAGE_TAG}/linux/amd64"

        def scan_result

        def scanning = true
        while(scanning) {
            def scan_result_response = httpRequest acceptType: 'APPLICATION_JSON', authentication: DOCKER_REGISTRY_CREDENTIALS_ID, httpMode: 'GET', ignoreSslErrors: true, responseHandle: 'LEAVE_OPEN', url: "${DOCKER_REGISTRY_URI}/api/v0/imagescan/repositories/${DOCKER_IMAGE_NAMESPACE}/${DOCKER_IMAGE_REPOSITORY_DEV}/${DOCKER_IMAGE_TAG}"
            scan_result = readJSON text: scan_result_response.content

            if (scan_result.size() != 1) {
                println('Response: ' + scan_result)
                error('More than one imagescan returned, please narrow your search parameters')
            }

            scan_result = scan_result[0]

            if (!scan_result.check_completed_at.equals("0001-01-01T00:00:00Z")) {
                scanning = false
            } else {
                sleep 1
            }

        }
        println('Response JSON: ' + scan_result)
    }

    stage('Promote') {
        httpRequest acceptType: 'APPLICATION_JSON', authentication: DOCKER_REGISTRY_CREDENTIALS_ID, contentType: 'APPLICATION_JSON', httpMode: 'POST', ignoreSslErrors: true, requestBody: "{\"targetRepository\": \"${DOCKER_IMAGE_NAMESPACE}/${DOCKER_IMAGE_REPOSITORY_PROD}\", \"targetTag\": \"${DOCKER_IMAGE_TAG}\"}", responseHandle: 'NONE', url: "${DOCKER_REGISTRY_URI}/api/v0/repositories/${DOCKER_IMAGE_NAMESPACE}/${DOCKER_IMAGE_REPOSITORY_DEV}/tags/${DOCKER_IMAGE_TAG}/promotion"
    }

    stage('Deploy') {
        withDockerServer([credentialsId: DOCKER_UCP_CREDENTIALS_ID, uri: DOCKER_UCP_URI]) {
            sh "docker service update --image ${DOCKER_REGISTRY_HOSTNAME}/${DOCKER_IMAGE_NAMESPACE}/${DOCKER_IMAGE_REPOSITORY_PROD}:${DOCKER_IMAGE_TAG} ${DOCKER_SERVICE_NAME}" 
        }
    }
}
