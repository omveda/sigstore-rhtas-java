

properties([
    parameters([
        string(defaultValue: 'quay.io/ablock/nonroot-jenkins-agent-maven:latest', description: 'Agent Image', name: 'AGENT_IMAGE'),
        string(description: 'apps.cluster-bqcjr.sandbox1219.opentlc.com', name: 'APPS_DOMAIN'),
        string(defaultValue: 'quay.io/vedadashan2/sigstore-rhtas-java', description: 'Image Destination', name: 'IMAGE_DESTINATION'),
        string(defaultValue: 'veda-registry-credentials', description: 'Registry Credentials', name: 'REGISTRY_CREDENTIALS')
    ])
])

podTemplate([
    label: 'non-root-jenkins-agent-maven',
    cloud: 'openshift',
    serviceAccount: 'nonroot-builder',
    containers: [
        containerTemplate(
            name: 'jnlp',
            image: "${params.AGENT_IMAGE}",
            alwaysPullImage: false,
            args: '${computer.jnlpmac} ${computer.name}'
        )
    ],
    volumes: [secretVolume(mountPath: '/var/run/sigstore/cosign',
        secretName: 'oidc-token'
    )]
]) {
    node('non-root-jenkins-agent-maven') {
stage('Setup Environment') {
        script {
            env.COSIGN_FULCIO_URL="https://fulcio-server-trusted-artifact-signer.${params.APPS_DOMAIN}"
            env.COSIGN_REKOR_URL="https://rekor-server-trusted-artifact-signer.${params.APPS_DOMAIN}"
            env.COSIGN_MIRROR="https://tuf-trusted-artifact-signer.${params.APPS_DOMAIN}"
            env.COSIGN_ROOT="https://tuf-trusted-artifact-signer.${params.APPS_DOMAIN}/root.json"
            env.COSIGN_YES="true"
            env.SIGSTORE_FULCIO_URL="https://fulcio-server-trusted-artifact-signer.${params.APPS_DOMAIN}"
            env.SIGSTORE_REKOR_URL="https://rekor-server-trusted-artifact-signer.${params.APPS_DOMAIN}"
            env.REKOR_REKOR_SERVER="https://rekor-server-trusted-artifact-signer.${params.APPS_DOMAIN}"
            env.COSIGN="bin/cosign"
            env.REGISTRY=sh(script: "echo ${params.IMAGE_DESTINATION} | cut -d '/' -f1", returnStdout: true).trim()
            if(params.APPS_DOMAIN == "") {
                currentBuild.result = 'FAILURE'
                error('Parameter APPS_DOMAIN is not provided')
            }

            dir("bin") {
                sh '''
                    #!/bin/bash
                    echo "Downloading cosign"
                    curl -Lks -o cosign.gz https://cli-server-trusted-artifact-signer.$APPS_DOMAIN/clients/linux/cosign-amd64.gz
                    gzip -f -d cosign.gz
                    rm -f cosign.gz
                    chmod +x cosign
                '''
            }

            dir("tuf") {
                deleteDir()
            }

            sh '''
              $COSIGN initialize
            '''
            
            stash name: 'binaries', includes: 'bin/*'
        }
    }

        stage('Checkout') {
            checkout scm
        }

        stage('Build Application') {
            sh '''
              mvn clean package
            '''
        }

        stage('Build and Push Image') {
            withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: params.REGISTRY_CREDENTIALS, usernameVariable: 'REGISTRY_USERNAME', passwordVariable: 'REGISTRY_PASSWORD']]) {
                sh '''
                   podman login -u $REGISTRY_USERNAME -p $REGISTRY_PASSWORD $REGISTRY
                   podman build -t $IMAGE_DESTINATION -f ./src/main/docker/Dockerfile.jvm .
                   podman push --digestfile=target/digest $IMAGE_DESTINATION
                '''
            }
        }

    }
}



