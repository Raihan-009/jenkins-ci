pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
    }

    parameters {
        string(
            name: 'Branch_or_Tag',
            defaultValue: 'main',
            description: 'Git branch or tag to deploy'
        )

        choice(
            name: 'ENVIRONMENT',
            choices: ['qa', 'ppe', 'prod'],
            description: 'Select deployment environment'
        )
    }

    environment {
        REPO_URL    = 'https://github.com/Raihan-009/jenkins-ci.git'
        DEPLOY_ROOT = '/var/jenkins-deploy'
    }

    stages {

        stage('Checkout Application') {
            steps {
                dir('application') {
                    checkout([
                        $class: 'GitSCM',

                        branches: [[
                            name: "${params.Branch_or_Tag}"
                        ]],

                        userRemoteConfigs: [[
                            url: env.REPO_URL
                        ]]
                    ])
                }
            }
        }

        stage('Build') {
            steps {
                sh '''
                    cd application

                    mkdir -p build

                    date -u +'%Y-%m-%dT%H:%M:%SZ' \
                      > build/build-info.txt

                    echo "Built from commit $(git rev-parse --short HEAD)" \
                      >> build/build-info.txt

                    echo "Branch/Tag: ${Branch_or_Tag}" \
                      >> build/build-info.txt

                    echo "Environment: ${ENVIRONMENT}" \
                      >> build/build-info.txt

                    cat build/build-info.txt
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    cd application

                    test -f index.html
                    grep -q "Hello World" index.html

                    echo "Tests passed."
                '''
            }
        }

        stage('Package') {
            steps {
                sh '''
                    tar -czf site.tar.gz \
                      -C application \
                      index.html
                '''

                archiveArtifacts(
                    artifacts: 'site.tar.gz',
                    fingerprint: true
                )
            }
        }

        stage('Deploy') {
            steps {
                script {

                    def safeBranch = params.Branch_or_Tag
                        .replaceAll('refs/heads/', '')
                        .replaceAll('refs/tags/', '')
                        .replaceAll('origin/', '')
                        .replaceAll('/', '-')

                    def envPort = [
                        qa   : '8088',
                        ppe  : '8089',
                        prod : '8090'
                    ]

                    def port = envPort[params.ENVIRONMENT]

                    if (!port) {
                        error(
                            "No port mapping found for environment: " +
                            params.ENVIRONMENT
                        )
                    }

                    def ts = sh(
                        returnStdout: true,
                        script: "date -u +'%Y%m%dT%H%M%SZ'"
                    ).trim()

                    def sha = sh(
                        returnStdout: true,
                        script: '''
                            git -C application \
                              rev-parse --short HEAD
                        '''
                    ).trim()

                    env.SITE_NAME =
                        "${safeBranch}-${params.ENVIRONMENT}-${ts}-${sha}"

                    env.SITE_DIR =
                        "${env.DEPLOY_ROOT}/${env.SITE_NAME}"

                    env.PORT = port

                    echo """
                    Deployment Information
                    ----------------------
                    Branch/Tag : ${params.Branch_or_Tag}
                    Environment: ${params.ENVIRONMENT}
                    Site Name  : ${env.SITE_NAME}
                    Port       : ${env.PORT}
                    """
                }

                sh '''#!/bin/bash
                    set -eux

                    HEX_GW=$(
                        awk 'NR>1 && $2=="00000000" {print $3}' \
                        /proc/net/route \
                        | head -1 \
                        | tr -d ' '
                    )

                    HOST_IP=$(
                        printf '%d.%d.%d.%d\\n' \
                        "0x${HEX_GW:6:2}" \
                        "0x${HEX_GW:4:2}" \
                        "0x${HEX_GW:2:2}" \
                        "0x${HEX_GW:0:2}"
                    )

                    mkdir -p "${SITE_DIR}"

                    cp -f \
                      application/index.html \
                      "${SITE_DIR}/"

                    docker ps -a \
                      --format '{{.Names}}' \
                      | grep "^$(echo ${SITE_NAME} | sed 's/-[0-9].*//')-" \
                      | grep -v "^${SITE_NAME}$" \
                      | xargs -r docker rm -f \
                      || true

                    docker run -d \
                      --name "${SITE_NAME}" \
                      --restart unless-stopped \
                      -p "${PORT}:80" \
                      -v "${SITE_DIR}:/usr/share/nginx/html:ro" \
                      nginx:alpine

                    sleep 3

                    curl -fsS \
                      "http://${HOST_IP}:${PORT}/" \
                      | grep -q "Hello World"

                    echo "Live at http://${HOST_IP}:${PORT}/"
                '''
            }
        }

        stage('Cleanup') {
            steps {
                sh '''
                    SAFE_PREFIX=$(
                        echo "${SITE_NAME}" \
                        | sed 's/-[0-9]\\{8\\}T.*//'
                    )

                    find "${DEPLOY_ROOT}" \
                      -maxdepth 1 \
                      -type d \
                      -name "${SAFE_PREFIX}-*" \
                      ! -name "${SITE_NAME}" \
                      -exec rm -rf {} +
                '''
            }
        }
    }

    post {

        success {
            echo """
            Deployment successful.

            Branch/Tag : ${params.Branch_or_Tag}
            Environment: ${params.ENVIRONMENT}
            Site       : ${env.SITE_NAME}
            Port       : ${env.PORT}
            """
        }

        failure {
            echo 'Failed. Check logs.'
        }

        always {
            cleanWs()
        }
    }
}
