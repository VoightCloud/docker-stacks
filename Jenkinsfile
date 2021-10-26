String imageVersion = "1.1"
String imageName = "base-notebook"
String imageRepo = "voight"
String nexusServer = "nexus.voight.org:9042"

stage('Build') {
    stage('Git Checkout') {
        def scmVars = checkout([
                $class           : 'GitSCM',
                userRemoteConfigs: scm.userRemoteConfigs,
                branches         : scm.branches,
                extensions       : scm.extensions
        ])

        // used to create the Docker image
        env.GIT_BRANCH = scmVars.GIT_BRANCH
        env.GIT_COMMIT = scmVars.GIT_COMMIT
    }
    stash name: 'scm', includes:'*'

    buildArm(imageName, imageVersion, imageRepo, nexusServer, "docker-build-arm${UUID.randomUUID().toString()}")
    buildAMD(imageName, imageVersion, imageRepo, nexusServer, "docker-build-x86_64${UUID.randomUUID().toString()}")
    createManifest(imageName, imageVersion, imageRepo, nexusServer)
}

def buildArm(imageName, imageVersion, imageRepo, nexusServer, dockerLabel) {
    podTemplate(
            label: dockerLabel,
            containers: [
                    containerTemplate(name: 'docker',
                            image: 'docker:20.10.9',
                            alwaysPullImage: false,
                            ttyEnabled: true,
                            command: 'cat',
                            envVars: [containerEnvVar(key: 'DOCKER_HOST', value: "unix:///var/run/docker.sock")],
                            privileged: true),
                    containerTemplate(name: 'jnlp', image: 'jenkins/inbound-agent:latest-jdk11', args: '${computer.jnlpmac} ${computer.name}')
            ],
            volumes: [
                    hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')
            ],
            nodeSelector: 'kubernetes.io/arch=arm64'
    ) {
        node(dockerLabel) {
            stage('Push') {
                container('docker') {
                    docker.withRegistry("https://${nexusServer}", 'NexusDockerLogin') {
                        unstash 'scm'
                        dir(imageName) {
                            image = docker.build("${imageRepo}/${imageName}:${imageVersion}-arm64")
                            image.push("${imageVersion}-arm64")
                            image.push("arm64-latest")
                        }
                    }
                }
            }
        }
    }
}

def buildAMD(imageName, imageVersion, imageRepo, nexusServer, dockerLabel) {
    podTemplate(
            label: dockerLabel,
            containers: [
                    containerTemplate(name: 'docker',
                            image: 'docker:20.10.9',
                            alwaysPullImage: false,
                            ttyEnabled: true,
                            command: 'cat',
                            envVars: [containerEnvVar(key: 'DOCKER_HOST', value: "unix:///var/run/docker.sock")],
                            privileged: true),
                    containerTemplate(name: 'jnlp', image: 'jenkins/inbound-agent:latest-jdk11', args: '${computer.jnlpmac} ${computer.name}')
            ],
            volumes: [
                    hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')
            ],
            nodeSelector: 'kubernetes.io/arch=amd64'
    ) {
        node(dockerLabel) {

            stage('Push') {
                container('docker') {
                    docker.withRegistry("https://${nexusServer}", 'NexusDockerLogin') {
                        unstash 'scm'
                        dir(imageName) {
                            image = docker.build("${imageRepo}/${imageName}:${imageVersion}-amd64")
                            image.push("${imageVersion}-amd64")
                            image.push("amd64-latest")
                        }
                    }
                }
            }
        }
    }
}

def createManifest(imageName, imageVersion, imageRepo, nexusServer, dockerLabel) {
    podTemplate(
            label: dockerLabel,
            containers: [
                    containerTemplate(name: 'docker',
                            image: 'docker:20.10.9',
                            alwaysPullImage: false,
                            ttyEnabled: true,
                            command: 'cat',
                            envVars: [containerEnvVar(key: 'DOCKER_HOST', value: "unix:///var/run/docker.sock")],
                            privileged: true),
                    containerTemplate(name: 'jnlp', image: 'jenkins/inbound-agent:latest-jdk11', args: '${computer.jnlpmac} ${computer.name}')
            ],
            volumes: [
                    hostPathVolume(hostPath: '/var/run/docker.sock', mountPath: '/var/run/docker.sock')
            ],
            nodeSelector: 'kubernetes.io/arch=amd64'
    ) {
        node(dockerLabel) {
            stage('Manifest') {
                container('docker') {
                    docker.withRegistry("https://${nexusServer}", 'NexusDockerLogin') {
                        sh "docker pull ${imageRepo}/${imageName}:${imageVersion}-arm64"
                        sh "docker pull ${imageRepo}/${imageName}:${imageVersion}-amd64"

                        sh "docker manifest create --insecure ${nexusServer}/${imageRepo}/${imageName}:latest -a ${nexusServer}/${imageRepo}/${imageName}:amd64-latest -a ${nexusServer}/${imageRepo}/${imageName}:arm64-latest"
                        sh "docker manifest push --insecure ${nexusServer}/${imageRepo}/${imageName}:latest"

                        sh "docker manifest create --insecure ${nexusServer}/${imageRepo}/${imageName}:${imageVersion} -a ${nexusServer}/${imageRepo}/${imageName}:${imageVersion}-amd64 -a ${nexusServer}/${imageRepo}/${imageName}:${imageVersion}-arm64"
                        sh "docker manifest push --insecure ${nexusServer}/${imageRepo}/${imageName}:${imageVersion}"
                    }
                }
            }
        }
    }
}


