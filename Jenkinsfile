pipeline {
    agent any

    environment {
        // 项目相关变量
        harborRegistry = "harbor.wh02.com"
        harborProject  = "devops"
        imageName      = "spring-boot-helloworld"
        gitRepo        = "https://gitlab.wh02.com/devops/devops-test.git"
        registryCredential = 'harbor-user-credential'
        // K8s 资源变量
        JAVA_OPTS = ""
        CPU_REQUEST = "100m"
        MEMORY_REQUEST = "256Mi"
        CPU_LIMIT = "500m"
        MEMORY_LIMIT = "512Mi"
        // 各环境变量
        DEV_NAMESPACE = "dev"
        DEV_REPLICAS = "1"
        QA_NAMESPACE = "qa"
        QA_REPLICAS = "1"
        PROD_NAMESPACE = "prod"
        PROD_REPLICAS = "2"
    }

    stages {
        stage('拉取代码') {
            steps {
                git url: "${gitRepo}", branch: 'main'
            }
        }
        stage('生成构建标签') {
            steps {
                script {
                    // 获取 git commit 短哈希作为标签
                    build_tag = sh(returnStdout: true, script: 'git rev-parse --short HEAD').trim()
                    fullImageName = "${harborRegistry}/${harborProject}/${imageName}:${build_tag}"
                    echo "镜像完整路径: ${fullImageName}"
                }
            }
        }
        stage('构建') {
            steps {
                sh 'cd app && mvn -B -DskipTests clean package'
            }
        }
        stage('测试') {
            steps {
                sh 'cd app && mvn test'
            }
        }
        // 如有 SonarQube 可解注释
        // stage('SonarQube代码扫描') {
        //     steps {
        //         withSonarQubeEnv('SonarQube-Server') {
        //             sh 'cd app && mvn sonar:sonar'
        //         }
        //     }
        // }
        stage('构建镜像 (containerd)') {
            steps {
                script {
                    sh """
                    cd app
                    nerdctl build -t ${fullImageName} .
                    """
                }
            }
        }
        stage('推送镜像 (containerd)') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${registryCredential}", usernameVariable: 'REG_USER', passwordVariable: 'REG_PASS')]) {
                    script {
                        sh """
                        nerdctl login ${harborRegistry} -u \$REG_USER -p \$REG_PASS
                        nerdctl push ${fullImageName}
                        nerdctl logout ${harborRegistry}
                        """
                    }
                }
            }
        }
        stage('部署到DEV环境') {
            steps {
                input message: "是否部署到DEV环境？", ok: "开始部署"
                script {
                    sh """
                    export APP_NAME=${imageName}
                    export NAMESPACE=${DEV_NAMESPACE}
                    export IMAGE=${fullImageName}
                    export REPLICAS=${DEV_REPLICAS}
                    export JAVA_OPTS="${JAVA_OPTS}"
                    export CPU_REQUEST="${CPU_REQUEST}"
                    export MEMORY_REQUEST="${MEMORY_REQUEST}"
                    export CPU_LIMIT="${CPU_LIMIT}"
                    export MEMORY_LIMIT="${MEMORY_LIMIT}"
                    envsubst < deploy/dev/k8s-deployment-dev.yaml | kubectl apply -f -
                    envsubst < deploy/dev/k8s-service-dev.yaml | kubectl apply -f -
                    """
                }
            }
        }
        stage('部署到QA环境') {
            steps {
                input message: "是否部署到QA环境？", ok: "开始部署"
                script {
                    sh """
                    export APP_NAME=${imageName}
                    export NAMESPACE=${QA_NAMESPACE}
                    export IMAGE=${fullImageName}
                    export REPLICAS=${QA_REPLICAS}
                    export JAVA_OPTS="${JAVA_OPTS}"
                    export CPU_REQUEST="${CPU_REQUEST}"
                    export MEMORY_REQUEST="${MEMORY_REQUEST}"
                    export CPU_LIMIT="${CPU_LIMIT}"
                    export MEMORY_LIMIT="${MEMORY_LIMIT}"
                    envsubst < deploy/qa/k8s-deployment-qa.yaml | kubectl apply -f -
                    envsubst < deploy/qa/k8s-service-qa.yaml | kubectl apply -f -
                    """
                }
            }
        }
        stage('部署到PROD环境') {
            steps {
                input message: "是否部署到PROD环境？", ok: "开始部署"
                script {
                    sh """
                    export APP_NAME=${imageName}
                    export NAMESPACE=${PROD_NAMESPACE}
                    export IMAGE=${fullImageName}
                    export REPLICAS=${PROD_REPLICAS}
                    export JAVA_OPTS="${JAVA_OPTS}"
                    export CPU_REQUEST="${CPU_REQUEST}"
                    export MEMORY_REQUEST="${MEMORY_REQUEST}"
                    export CPU_LIMIT="${CPU_LIMIT}"
                    export MEMORY_LIMIT="${MEMORY_LIMIT}"
                    envsubst < deploy/prod/k8s-deployment-prod.yaml | kubectl apply -f -
                    envsubst < deploy/prod/k8s-service-prod.yaml | kubectl apply -f -
                    """
                }
            }
        }
    }

    post {
        always {
            echo "流水线执行完毕。"
        }
    }
}
