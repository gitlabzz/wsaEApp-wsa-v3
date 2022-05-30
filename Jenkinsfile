node('APIM-Python-Docker') {

    def mvnHome
    String branchName
    String targetEnvironment = 'SIT'
    def pullRequest
    def apimHost
    def gitRepoProtocol = "http"
    def gitRepo = "gitlab.asim.com/root/wsaEApp-wsa-v3.git"

    def nonProdSITBranches = ['DEV', 'SIT']
    def nonProdUATBranches = ['UAT']
    def prodBranches = ['PROD', 'MAIN']

    def backends = [
            SIT : [BACKEND_PROTOCOL: "http", BACKEND_HOST: "172.16.63.166", BACKEND_PORT: "8080"],
            UAT : [BACKEND_PROTOCOL: "http", BACKEND_HOST: "172.16.63.166", BACKEND_PORT: "8080"],
            PROD: [BACKEND_PROTOCOL: "http", BACKEND_HOST: "172.16.63.166", BACKEND_PORT: "8080"]
    ]

    def healthCheckEndpoints = [
            SIT : "http://sit.apim.demo.bct:8080/healthcheck",
            UAT : "http://uat.apim.demo.bct:8080/healthcheck",
            PROD: "http://prod.apim.demo.bct:8080/healthcheck",
    ]

    stage('Initialize') {
        branchName = BRANCH_NAME
        echo "checking if it's a pull request branch!"
        if (branchName.toUpperCase().startsWith("PR-")) {
            echo "found pull request '${branchName}', so targetting it to the 'SIT' environment!!!"
            pullRequest = branchName.substring(branchName.lastIndexOf("-") + 1)
            echo "Pull request is   ========================================>  ${pullRequest}."
        } else {
            echo "${branchName} is not a pull request."
        }
    }

    stage("Checkout Code (${pullRequest ? 'PR-' + pullRequest : branchName})") {
        if (pullRequest) {
            echo "Checking out pull request '${branchName}'"
            try {
                git branch: '${BRANCH_NAME}', credentialsId: 'gitlab.vv0053.userid.password', url: "${gitRepoProtocol}://${gitRepo}"
            } catch (exception) {
                sh '''
                    git fetch origin +refs/pull/''' + pullRequest + '''/merge
                    git checkout FETCH_HEAD
                '''
                branchName = "sit"
                echo "targeting build for pull request ${pullRequest} to '${branchName}' environment"
            }
            echo "Check out for pull request '${BRANCH_NAME}' is successfully completed!"

        } else {
            echo "Checking out branch '${BRANCH_NAME}'"
            git branch: '${BRANCH_NAME}', credentialsId: 'gitlab.vv0053.userid.password', url: "${gitRepoProtocol}://${gitRepo}"
            echo "Check out for '${BRANCH_NAME}' is successfully completed!"
        }
    }

    stage("Prepare Environment (${branchName})") {
        echo "Preparing build environment for branch ---------------------------------> '${branchName}'"

        if (nonProdSITBranches.contains(branchName.toUpperCase())) {
            targetEnvironment = 'SIT'
        } else if (nonProdUATBranches.contains(branchName.toUpperCase())) {
            targetEnvironment = 'UAT'
        } else if (prodBranches.contains(branchName.toUpperCase())) {
            targetEnvironment = 'PROD'
        } else {
            echo "couldn't find mapping for branch '${branchName}', so mapping to SIT"
            targetEnvironment = 'SIT'
        }

        echo "Setting target environment for '${branchName}' as ---------------------------------> '${targetEnvironment}'"

        mvnHome = tool 'M3'

        def apimHostEnvVar = "APIM_${targetEnvironment}_HOST"
        echo "Using Global Environment Variable: '${apimHostEnvVar}'"
        apimHost = env.getProperty(apimHostEnvVar)
        echo "The Environment Variable: '${apimHostEnvVar}' is set to '${apimHost}'"

        withCredentials([usernamePassword(credentialsId: "APIM_ADMIN_USERNAME_PASSWORD_${targetEnvironment}_ENV", passwordVariable: 'password', usernameVariable: 'username')]) {
            // Referred in env.properties file, used by APIM CLI to login
            env.APIM_HOST = apimHost

            // Referred in env.properties file, used by APIM CLI to login
            env.APIM_ADMIN_USER = username

            // Referred in env.properties file, used by APIM CLI to login
            env.APIM_ADMIN_PASSWORD = password

            // Required by APIM CLI Utility to load config for given environment
            env.AXWAY_APIM_CLI_HOME = "src/main/environments/${targetEnvironment.toLowerCase()}"

            // To load environment specific WSDL, referred in api-config.json for apiSpecification.resource: src/main/resources/${ENV}.wsdl.url
            env.ENV = targetEnvironment.toLowerCase()

            echo "Setting 'AXWAY_APIM_CLI_HOME' to --------------------------------------> ${AXWAY_APIM_CLI_HOME}"
            echo "Using 'conf/env.properties' file from ---------------------------------> '${AXWAY_APIM_CLI_HOME}'"
            echo "Setting APIM_HOST to --------------------------------------------------> ${apimHost}"
            echo "Getting APIM login credentails from -----------------------------------> 'APIM_ADMIN_USERNAME_PASSWORD_${targetEnvironment}_ENV' secret"
            echo "Setting APIM_ADMIN_USER to --------------------------------------------> ${username}"
            echo "Setting APIM_ADMIN_PASSWORD from secret 'APIM_ADMIN_USERNAME_PASSWORD_${targetEnvironment}_ENV'"
            echo "Setting EVN -----------------------------------------------------------> ${targetEnvironment.toLowerCase()}"
        }
    }

    stage('Prepare Package') {
        echo "Preparing '.zip' file using 'mvn install'"
        withEnv(["MVN_HOME=$mvnHome"]) {
            if (isUnix()) {
                sh '"$MVN_HOME/bin/mvn" clean install'
            }
        }
        echo "Prepared'.zip' file successfully"
    }

    stage("APIs Before Publish (${targetEnvironment.toLowerCase()})") {
        // Run the maven build
        withEnv(["MVN_HOME=$mvnHome"]) {
            if (isUnix()) {
                echo "Retrieving published APIs from '${targetEnvironment.toLowerCase()}' before publish"
                env.MAVEN_OPTS = '-Xms256m -Xmx512m -Dlog4j.configurationFile=src/main/resources/log4j/log4j2.xml'
                echo "Setting MAVEN_OPTS environment variable for maven build to '${env.MAVEN_OPTS}'"

                echo "------------------------------------ EXISTING APIs BEFORE PUBLISH ------------------------------------"
                sh '"$MVN_HOME/bin/mvn" exec:java@list-api'
            }
        }
    }

    stage("Publish API (${targetEnvironment.toLowerCase()})") {
        withEnv(["MVN_HOME=$mvnHome"]) {
            if (isUnix()) {

                if (nonProdSITBranches.contains(targetEnvironment))
                    targetEnvironment = 'SIT'
                else if (nonProdUATBranches.contains(targetEnvironment))
                    targetEnvironment = 'UAT'
                else
                    targetEnvironment = 'PROD'

                echo "Publishing to environment: '${targetEnvironment}'"

                env.MAVEN_OPTS = '-Xms256m -Xmx512m -Dlog4j.configurationFile=src/main/resources/log4j/log4j2.xml'

                // required in resources/api-config.json for WSDL backend in 'backendBasepath'
                env.BACKEND_PROTOCOL = backends.get(targetEnvironment).get("BACKEND_PROTOCOL")
                env.BACKEND_HOST = backends.get(targetEnvironment).get("BACKEND_HOST")
                env.BACKEND_PORT = backends.get(targetEnvironment).get("BACKEND_PORT")

                // must be defined in resources/api-config.json for 'backendBasepath'
                echo "Setting BackEnd Endpoint as: ${env.BACKEND_PROTOCOL}//${env.BACKEND_HOST}:${env.BACKEND_PORT}"

                sh '"$MVN_HOME/bin/mvn" exec:java@import-api'
            }
        }
    }

    stage("APIs After Publish (${targetEnvironment.toLowerCase()})") {
        // Run the maven build
        withEnv(["MVN_HOME=$mvnHome"]) {
            if (isUnix()) {
                echo "Retrieving APIs from '${branchName}' after publish"
                env.MAVEN_OPTS = '-Xms256m -Xmx512m -Dlog4j.configurationFile=src/main/resources/log4j/log4j2.xml'
                echo "------------------------------------ EXISTING APIs AFTER PUBLISH ------------------------------------"
                sh '"$MVN_HOME/bin/mvn" exec:java@list-api'
            }
        }
    }

    stage("Healthcheck (${targetEnvironment.toLowerCase()})") {
        echo "Checking deployed service by invoking it's healthcheck endpoint for environment: '${targetEnvironment}'"
        def healthCheck = healthCheckEndpoints.get(targetEnvironment)
        def healthCheckStatus = sh "curl ${healthCheck}"

        //https://kb.novaordis.com/index.php/Jenkins_currentBuild
        //currentBuild.result = 'FAILURE'
    }

    stage('Push Package to Nexus') {
        echo "Pushing .Zip bundle to Nexus repository"
        withEnv(["MVN_HOME=$mvnHome"]) {
            if (isUnix()) {
                // TODO: later make it deploy instead of install
                sh '"$MVN_HOME/bin/mvn" clean install'
            }
        }
    }

    stage('Git Tag') {
        withCredentials([usernamePassword(credentialsId: "gitlab.vv0053.userid.password", passwordVariable: 'password', usernameVariable: 'username')]) {
            sh("git config user.name 'Jenkins'")
            sh("git config user.email 'jenkins@bcthk.com'")
            sh("git tag -a v${env.BUILD_NUMBER}_${targetEnvironment} -m 'Tagged from Jenkins for environment ${targetEnvironment} from Build: ${env.BUILD_NUMBER}'")
            sh("git push ${gitRepoProtocol}://${username}:${password}@${gitRepo} --tags")
        }
    }
}