pipeline {
    agent any

    tools {
        maven 'Maven'
        jdk 'JDK8'
    }

    environment {
        // 构建信息
        BUILD_VERSION = "${BUILD_ID}-${env.BUILD_TIMESTAMP}"
        REPO_URL = 'git@github.com:backend-ex/spot_exchange_web.git'

        // 根据不同环境设置变量
        // 不再将 JSON 对象存储在环境变量中
        // DEPLOY_CONFIG = readJSON file: "jenkins/configs/test-config.json"
    }

    stages {
        stage('初始化') {
            steps {
                script {
                    echo "开始部署 ${SERVICE_NAME} 到 test 环境"
                    echo "构建版本: ${BUILD_VERSION}"
                    echo "Git分支: ${BRANCH}"

                    // 备份配置文件到临时位置
                    sh '''
                        mkdir -p /tmp/jenkins-configs
                        cp -r jenkins/configs/* /tmp/jenkins-configs/
                    '''

                    // 读取服务配置
                    def servicesConfig = readJSON file: 'jenkins/configs/services.json'
                    def allServices = servicesConfig.collect { it.toString() }
                    
                    // 正确处理 ALL 参数或者单个服务参数，避免使用被拒绝的toJson方法
                    if (params.SERVICE_NAME == 'ALL') {
                        // 直接在环境中存储所有服务的字符串表示
                        env.ALL_SERVICES = "true"
                        env.SINGLE_SERVICE = ""
                    } else {
                        // 存储单个服务
                        env.ALL_SERVICES = "false"
                        env.SINGLE_SERVICE = params.SERVICE_NAME
                    }
                    
                    // 调试信息
                    echo "所有服务: ${allServices}"
                    echo "部署全部服务: ${env.ALL_SERVICES}"
                    echo "单个服务: ${env.SINGLE_SERVICE}"
                }
            }
        }

        stage('代码检出') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: "${BRANCH}"]],
                    extensions: [],
                    userRemoteConfigs: [[
                        url: REPO_URL,
                        credentialsId: 'github-jenkins-key'
                    ]]
                ])

                // 恢复配置文件
                sh '''
                    mkdir -p jenkins/configs
                    cp -r /tmp/jenkins-configs/* jenkins/configs/
                '''

                // 保存提交信息
                sh '''
                    git log -1 --pretty=format:"%H | %an | %ad | %s" > commit_info.txt
                '''
            }
        }

        stage('依赖构建') {
            steps {
                dir('.') {
                    sh 'mvn clean install -DskipTests=true -Dmaven.test.skip=true'
                }
            }
        }

        stage('服务构建') {
            steps {
                script {
                    // 根据环境变量决定要部署的服务
                    def servicesToDeploy
                    if (env.ALL_SERVICES == "true") {
                        // 读取所有服务
                        def servicesConfig = readJSON file: 'jenkins/configs/services.json'
                        servicesToDeploy = servicesConfig.collect { it.toString() }
                    } else {
                        // 使用单个服务
                        servicesToDeploy = [env.SINGLE_SERVICE]
                    }

                    def serviceDirectoryMapping = readJSON file: 'jenkins/configs/service-directory-mapping.json'

                    servicesToDeploy.each { serviceName ->
                        stage("构建 ${serviceName}") {
                            // 根据服务名获取对应目录
                            def serviceDir = serviceDirectoryMapping[serviceName] ?: serviceName

                            echo "开始构建服务: ${serviceName}，对应目录: ${serviceDir}"
                            dir("${serviceDir}/${serviceName}") {
                                if (params.SKIP_TESTS) {
                                    sh "mvn clean package -DskipTests"
                                } else {
                                    sh "mvn clean package"
                                }

                                // 归档制品
                                archiveArtifacts artifacts: "target/*.jar", fingerprint: true

                                // 生成部署包
                                sh """
                                    mkdir -p ${WORKSPACE}/deploy-packages/${serviceName}
                                    cp ${WORKSPACE}/../deploy/${serviceName}.jar ${WORKSPACE}/deploy-packages/${serviceName}/
                                    cp src/main/resources/application-prod.properties ${WORKSPACE}/deploy-packages/${serviceName}/ 2>/dev/null || true
                                """
                            }
                        }
                    }
                }
            }
        }

        stage('分发部署') {
            steps {
                script {
                    // 根据环境变量决定要部署的服务
                    def servicesToDeploy
                    if (env.ALL_SERVICES == "true") {
                        // 读取所有服务
                        def servicesConfig = readJSON file: 'jenkins/configs/services.json'
                        servicesToDeploy = servicesConfig.collect { it.toString() }
                    } else {
                        // 使用单个服务
                        servicesToDeploy = [env.SINGLE_SERVICE]
                    }
                    
                    servicesToDeploy.each { serviceName ->
                        stage("部署 ${serviceName}") {
                            def deployConfig = readJSON file: "jenkins/configs/test-config.json"
                            // 获取该服务的服务器列表
                            def servers = deployConfig.services[serviceName]?.servers ?: []
                            if (servers.isEmpty()) {
                                echo "警告: ${serviceName} 在 test 环境中没有配置服务器"
                                return
                            }

                            servers.each { server ->
                                stage("部署到 ${server.host}:${server.port}") {
                                    deployToServer(serviceName, server)
                                }
                            }
                        }
                    }
                }
            }
        }
    }
}

// 部署到单台服务器的方法
def deployToServer(serviceName, serverConfig) {
    def host = serverConfig.host
    def port = serverConfig.port ?: 22
    def deployPath = serverConfig.deployPath ?: "/opt/apps/${serviceName}"
    def rs = serverConfig.restartScript

    echo "部署 ${serviceName} 到服务器 ${host}"

    // 使用SSH连接服务器执行部署
    sshagent(['deploy-server-key']) {
        sh """
            # 创建目标目录（如果不存在）
            ssh -p 22 -o StrictHostKeyChecking=no ec2-user@${host} '
                sudo mkdir -p ${deployPath}/tmp
                sudo chown -R ec2-user:ec2-user ${deployPath}
                sudo chmod 755 ${deployPath}
            '
                
            # 上传部署包
            scp -P ${port} -o StrictHostKeyChecking=no \\
                ${WORKSPACE}/deploy-packages/${serviceName}/* \\
                ec2-user@${host}:${deployPath}/tmp/

            # 执行远程部署脚本
            ssh -p ${port} -o StrictHostKeyChecking=no ec2-user@${host} << 'ENDSSH'
                cd /home/ec2-user/nokex-app
                # 执行部署脚本并记录PID
                sudo ./"${rs}".sh tmp > "/tmp/deploy_${serviceName}.log" 2>&1 &
                DEPLOY_PID=\$!

                # 等待30秒让Java应用启动
                sleep 30

                # 查找并终止tail -f进程
                TAIL_PID=\$(ps -ef | grep "tail -f" | grep -v grep | awk '{print \$2}')
                if [ -n "\$TAIL_PID" ]; then
                    # 检查该进程是否是我们的部署脚本的子进程
                    PARENT_PID=\$(ps -o ppid= -p "\$TAIL_PID" | tr -d ' ')
                    if [ "\$PARENT_PID" = "\$DEPLOY_PID" ]; then
                        kill "\$TAIL_PID" 2>/dev/null && echo "已终止tail -f进程: \$TAIL_PID"
                    fi
                fi

                echo "部署完成，应用已启动。"

ENDSSH
        """
    }
}