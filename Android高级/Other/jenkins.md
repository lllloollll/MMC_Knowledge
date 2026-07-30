```bash
pipeline {
    agent { label 'mobile_app_build' }

    // tools {
        // gradle 'Gradle8.7_mac'
        // jdk 'jdk21' // 根据项目需要调整 JDK 版本
    // }

    environment {
        // Git 仓库配置
        GIT_REPO_URL = 'http://gitlab.casdon.com.cn/frontend/application/android/casdon-application.git'
        GIT_REPO_CREDENTIALS_ID = '41c34b18-1bd1-4bf0-b880-56f446c55bc6'

        // Android SDK 路径（根据 Jenkins 节点配置调整）
        JAVA_HOME= '/Applications/Android Studio.app/Contents/jbr/Contents/Home'
        GRADLE_HOME = '/Users/android_dev/.gradle/gradle-8.7'
        ANDROID_HOME = '/Users/android_dev/android-sdk'
        PATH = "${ANDROID_HOME}/cmdline-tools/latest/bin:${ANDROID_HOME}/platform-tools:$GRADLE_HOME/bin:$JAVA_HOME:$PATH"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '20'))
        timestamps()
        disableConcurrentBuilds()
    }

    stages {
        stage('Debug Healthy Node') {
            steps {
                sh '''
                echo "===== OS & ARCH ====="
                uname -a

                echo "===== JAVA_HOME ====="
                echo $JAVA_HOME

                echo "===== ANDROID_HOME ====="
                echo $ANDROID_HOME
                ls -l $ANDROID_HOME

                echo "===== SDKMANAGER ====="
                which sdkmanager || find / -name sdkmanager 2>/dev/null

                echo "===== GRADLE ====="
                gradle -v || ./gradlew -v
                '''
            }
        }
        // 初始化阶段：解析分支名称并设置构建配置
        stage('Init') {
            steps {
                script {
                    // ✅ 检查是否是删除操作（after 为全 0 表示删除）
                    // 检测是否为删除标签/分支操作
                    // 删除操作时，GitLab Webhook 会发送 after=0000000000000000000000000000000000000000
                    def afterCommit = env.git_after ?: env.GIT_AFTER ?: ''

                    // 或者检测 ref 变量是否包含 deleted
                    def ref = env.ref ?: ''

                    if (afterCommit == '0000000000000000000000000000000000000000' || 
                    ref.contains('deleted') || 
                    ref.contains('/tags/') && afterCommit.isEmpty()) {
                        echo "🗑️ 检测到删除操作（标签或分支），跳过构建"
                        currentBuild.result = 'ABORTED'
                        error("删除操作，不触发构建")
                    }

                    // 判断是标签还是分支
                    def isTag = env.ref?.startsWith('refs/tags/')

                    if (isTag) {
                        // 目前标签只打正式包
                        env.BRANCH_NAME = 'prod'
                        // 标签触发：提取标签名
                        def tagName = env.ref.replace('refs/tags/', '')
                        echo "🏷️ 检测到标签: ${tagName}，构建 Release 包"
                    } else{
                        // 设置分支和构建配置（从环境变量获取）
                        env.BRANCH_NAME = env.ref ? env.ref.split('/')[-1] : (env.GIT_BRANCH ? env.GIT_BRANCH.split('/')[-1] : 'dev')
                    } 
                    if (env.BRANCH_NAME == 'prod') {
                        env.BUILD_ENV = 'prod'
                        env.BUILD_TYPE = 'Release'
                        env.FLAVOR = 'Prod'
                        echo "检测到 prod 分支，构建 prodRelease"
                    } else {
                        // 非 prod 分支（包括 dev、main、其他分支）都按 dev 环境 Debug 包处理
                        env.BUILD_ENV = 'dev'
                        env.BUILD_TYPE = 'Debug'
                        env.FLAVOR = 'Dev'
                        if (env.BRANCH_NAME == 'dev') {
                            echo "检测到 dev 分支，构建 devDebug"
                        } else {
                            echo "非 prod/dev 分支 (${env.BRANCH_NAME})，默认构建 devDebug"
                        }
                    }


                    echo "分支: ${env.BRANCH_NAME}"
                    echo "构建环境: ${env.BUILD_ENV}"
                    echo "构建类型: ${env.BUILD_TYPE}"
                }
            }
        }

        // 代码拉取阶段
        stage('Code Pull') {
            steps {
                script {
                    checkout([
                        $class: 'GitSCM',
                        branches: [[name: "*/${env.BRANCH_NAME}"]],
                        doGenerateSubmoduleConfigurations: false,
                        extensions: [],
                        submoduleCfg: [],
                        userRemoteConfigs: [
                            [
                                credentialsId: "${env.GIT_REPO_CREDENTIALS_ID}",
                                url: "${env.GIT_REPO_URL}"
                            ]
                        ]
                    ])
                }
            }
        }

        // 设置版本信息（必须在代码拉取之后）
        stage('Set Version') {
            steps {
                script {
                    // 设置版本信息（此时已有代码）
                    env.VERSION_NAME = sh(script: 'git describe --tags --abbrev=0 2>/dev/null || echo "1.0.0"', returnStdout: true).trim()
                    env.VERSION_CODE = sh(script: 'git tag | wc -l | tr -d " "', returnStdout: true).trim().toInteger() + 69
                    env.GIT_COMMIT_SHORT = sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                    env.BUILD_TIME = sh(script: 'date +"%Y%m%d_%H%M%S"', returnStdout: true).trim()

                    echo "版本号: ${env.VERSION_NAME} (${env.VERSION_CODE})"
                    echo "Git Commit: ${env.GIT_COMMIT_SHORT}"
                    echo "构建时间: ${env.BUILD_TIME}"
                }
            }
        }

        stage('Accept Android Licenses') {
            steps {
                sh '''
                yes | $ANDROID_HOME/cmdline-tools/latest/bin/sdkmanager --licenses
                '''
            }
        }

        stage('Environment Check') {
            steps {
                echo "检查环境..."
                sh '''
                    echo "Java 版本:"
                    java -version
                    echo "Gradle 版本:"
                    gradle --version
                    echo "Android SDK 路径: ${ANDROID_HOME}"
                    echo "JAVA_HOME=${JAVA_HOME}"
                    gradle -v
                '''
            }
        }

        stage('Setup Signing') {
            steps {
                script {
                    withCredentials([
                        file(credentialsId: 'SIGN_KEYSTORE', variable: 'KEYSTORE_FILE'),
                        string(credentialsId: 'SIGN_STORE_PASSWORD', variable: 'STORE_PWD'),
                        string(credentialsId: 'SIGN_KEY_ALIAS', variable: 'KEY_ALIAS'),
                        string(credentialsId: 'SIGN_KEY_PASSWORD', variable: 'KEY_PWD'),
              string(credentialsId: 'THING_SMART_APPKEY', variable: 'SMART_APPKEY'),
                        string(credentialsId: 'THING_SMART_SECRET', variable: 'SMART_SECRET')
                    ]) {
                        // 将签名文件从凭据复制到工作区
                        sh '''
                            cp ${KEYSTORE_FILE} ${WORKSPACE}/signing.keystore
                            chmod 600 ${WORKSPACE}/signing.keystore
                        '''

                        // 创建 local.properties 用于签名配置
                        writeFile file: "local.properties", text: """
                            STORE_FILE=${WORKSPACE}/signing.keystore
                            STORE_PASSWORD=${STORE_PWD}
                            KEY_ALIAS=${KEY_ALIAS}
                            KEY_PASSWORD=${KEY_PWD}
                  THING_SMART_APPKEY=${SMART_APPKEY}
                            THING_SMART_SECRET=${SMART_SECRET}
                        """
                    }
                }
            }
        }

        stage('Clean') {
            steps {
                echo "清理构建..."
                sh """
                    gradle clean --no-daemon
                """
            }
        }

        stage('Build APK') {
            steps {
                script {
                    def taskName = "assemble${env.BUILD_ENV}${env.BUILD_TYPE}"
                    echo "开始构建: ${taskName}"
                    withCredentials([
                        string(credentialsId: 'SIGN_STORE_PASSWORD', variable: 'STORE_PWD'),
                        string(credentialsId: 'SIGN_KEY_ALIAS', variable: 'KEY_ALIAS'),
                        string(credentialsId: 'SIGN_KEY_PASSWORD', variable: 'KEY_PWD')
                    ]) {
                        sh """
                            gradle ${taskName} \
                                -PstoreFile=${WORKSPACE}/signing.keystore \
                                -PstorePassword=${STORE_PWD} \
                                -PkeyAlias=${KEY_ALIAS} \
                                -PkeyPassword=${KEY_PWD} \
                                --stacktrace --no-daemon
                        """
                    }
                }
            }
        }

        stage('Collect Artifacts') {
            steps {
                script {
                    def _env = env.BUILD_ENV.toLowerCase()
                    def _buildType = env.BUILD_TYPE.toLowerCase()
                    def _apkName = "casdon_${_buildType}_arm64-v8a_*.apk"
                    env.APK_PARENT_PATH = "app/build/outputs/apk/${_env}/${_buildType}"
                    def apkPattern = "${env.APK_PARENT_PATH}/${_apkName}"

                    env.JOB_NAME = _apkName

                    archiveArtifacts artifacts: apkPattern, fingerprint: true

                    // 获取 APK 文件信息用于上传
                    // ✅ 直接构造 APK_PATH（不使用 findFiles）
                    def apkFiles = sh(
                        script: "ls ${apkPattern}",
                        returnStdout: true
                    ).trim().split('\n')

                    if (apkFiles.size() == 0 || apkFiles[0].isEmpty()) {
                        error "❌ 未找到 arm64-v8a APK"
                    }
                    echo "归档 APK 产物：${apkFiles}"
                    env.APK_PATH = apkFiles[0]
                    env.APK_NAME = env.APK_PATH.split('/').last()
                    echo "✅ 找到 APK: ${env.APK_PATH} name:${env.APK_NAME}"
                    def name = env.APK_NAME.replace(".apk",'')
                    def parts = name.split('_')
                    env.PARSED_VERSION_NAME = parts[-2]
                    env.PARSED_VERSION_CODE = parts[-1]
                    echo "解析版本名称：${env.PARSED_VERSION_NAME}，版本号为：${env.PARSED_VERSION_CODE}"
                }
            }
        }

        stage('Upload to Pgyer') {
            when {
                expression { env.APK_PATH != null && env.APK_PATH != '' }
            }
            steps {
                script {
                    echo "上传到蒲公英..."

                    // 蒲公英 API 上传 (https://www.pgyer.com/doc/view/api#uploadApp)
                    withCredentials([
                        string(credentialsId: 'PGYER_API_KEY', variable: 'PGYER_API_KEY')
                    ]) {
                        def uploadResponse = sh(
                                                script: """
                                                    set -e
                                                    curl -s -S -X POST \
                                                        https://www.pgyer.com/apiv2/app/upload \
                                                        -F "_api_key=${PGYER_API_KEY}" \
                                                        -F "file=@${APK_PATH}" \
                                                        -F "buildUpdateDescription=Branch: ${BRANCH_NAME}, Commit: ${GIT_COMMIT_SHORT}, Build: ${BUILD_TYPE},VersionName:${PARSED_VERSION_NAME},VersionCode:${PARSED_VERSION_CODE}\n${VERSION_NOTES}"
                                                """,
                                                returnStdout: true,
                                                returnStatus: false
                                            ).trim()

                                            echo "蒲公英上传响应: ${uploadResponse}"

                                            // 解析响应
                                            def json
                                            try{
                                                json = new groovy.json.JsonSlurper().parseText(uploadResponse)
                                            }catch(Exception e){
                                                echo "⚠️蒲公英上传响应解析失败: ${e.message}"
                                            }
                                            if (json.code == 0) {
                                                echo "上传成功!"
                                                def url = "https://www.pgyer.com/${json.data.buildShortcutUrl}"
                                                env.JOB_URL = url
                                                echo "应用短链接: ${url}"
                                                env.PGYER_UPLOAD_STATUS = "✅ 成功 [点击下载](${url})"
                                            } else {
                                                error "蒲公英上传失败: ${json.message}"
                                            }

                                }
                }
            }
        }
    }

    post {
        always {
            echo '构建完成'
            // 清理签名文件
            sh 'rm -f ${WORKSPACE}/signing.keystore || true'
            // 复制 APK 到 apk 目录（清理前执行）
        script {
            // 创建 apk 目录并复制文件
            env.SAVE_PATH = "/Users/jenkins/jenkins_agent/workspace/apk_backup/${env.BUILD_TYPE.toLowerCase()}/v${env.PARSED_VERSION_NAME}_${env.PARSED_VERSION_CODE}"
            env.APK_FULL_PATH = "${WORKSPACE}/${env.APK_PARENT_PATH}"

            echo "APK_PARENT_PATH: ${env.APK_PARENT_PATH}"
            echo "APK_FULL_PATH: ${env.APK_FULL_PATH}"
            echo "SAVE_PATH: ${env.SAVE_PATH}"

            sh '''
                mkdir -p ${SAVE_PATH}
                cp -rf ${APK_FULL_PATH}/* ${SAVE_PATH}/ 2>/dev/null || true
                echo "已复制 APK 到 ${SAVE_PATH} 目录"
                ls -la ${SAVE_PATH}/
            '''
        }
            // 清理工作空间
            cleanWs()
        }
        success {
            script {
                echo '构建成功!'

                // 获取构建产物信息
                echo "apk包:{env.APK_PATH}"
                echo "分支: ${env.BRANCH_NAME}"
                echo "环境: ${env.BUILD_ENV}"
                echo "类型: ${env.BUILD_TYPE}"
            }
        }
        failure {
            echo '构建失败!'
        }
        unstable {
            echo '构建不稳定'
        }
    }
}
```

添加版本信息：

```bash

```
