

properties([
    parameters([
        string(defaultValue: 'quay.io/ablock/nonroot-jenkins-agent-maven:latest', description: 'Agent Image', name: 'AGENT_IMAGE'),
        string(description: 'Cluster Apps Domain', name: 'APPS_DOMAIN'),
        string(defaultValue: '', description: 'Image Destination', name: 'IMAGE_DESTINATION'),
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
            env.COSIGN_REKOR_URL="https://rekor-server-trusted-artifact-signer.${params.APPS_DOMAIN}"
            env.COSIGN_MIRROR="https://tuf-trusted-artifact-signer.${params.APPS_DOMAIN}"
            env.COSIGN_ROOT="https://tuf-trusted-artifact-signer.${params.APPS_DOMAIN}/root.json"
            env.COSIGN_YES="true"
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
                    echo "hello world" > key
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

        stage('Sign Artifacts') {
            unstash 'binaries'

            // Sign Jar
            sh '''
            echo "111"
            cat bin/key
            # $COSIGN sign-blob $(find target -maxdepth 1  -type f -name '*.jar') --identity-token=/var/run/sigstore/cosign/id-token
            foojar=$(find target -maxdepth 1  -type f -name '*.jar')
            echo $foojar
            '''

            // Sign Container Image and Attest SBOM
            withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: params.REGISTRY_CREDENTIALS, usernameVariable: 'REGISTRY_USERNAME', passwordVariable: 'REGISTRY_PASSWORD']]) {
                sh '''
                   set +x

                   DIGEST_DESTINATION="$(echo $IMAGE_DESTINATION | cut -d \":\" -f1)@$(cat target/digest)"
                   $COSIGN login -u $REGISTRY_USERNAME -p $REGISTRY_PASSWORD $REGISTRY

                   # $COSIGN sign --identity-token=/var/run/sigstore/cosign/id-token $DIGEST_DESTINATION

                   # $COSIGN attest --identity-token=/var/run/sigstore/cosign/id-token --predicate=./target/classes/META-INF/maven/com.redhat/sigstore-rhtas-java/license.spdx.json -y --type=spdxjson $DIGEST_DESTINATION
                '''
            }
        }

        stage('Verify Signature') {
            sh '''
            echo "222"
            cat bin/key
            echo $IMAGE_DESTINATION
            # $COSIGN verify  --certificate-identity=ci-builder@redhat.com $IMAGE_DESTINATION
            '''
        }
    }
}



