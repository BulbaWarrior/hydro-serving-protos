def checkoutSource(gitCredentialId, organization, repository) {
    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: gitCredentialId, usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD']]) {
        git url: "https://github.com/${organization}/${repository}.git", branch: env.BRANCH_NAME, credentialsId: gitCredentialId
        sh """
            git config --global push.default simple
            git config --global user.name '${GIT_USERNAME}'
            git config --global user.email '${GIT_USERNAME}'
        """
    }
}

def pushSource(gitCredentialId, organization, repository, pushCommand) {
    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: gitCredentialId, usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD']]) {
        sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/${organization}/${repository}.git ${pushCommand}"
    }
}

def isReleaseJob() {
    return "true".equalsIgnoreCase(env.IS_RELEASE_JOB)
}

def generateTagComment(releaseVersion) {
    commitsList = sh(returnStdout: true, script: "git log --pretty=\"%B\n\r (%an)\" -1").trim()
    return "${commitsList}"
}

def createReleaseInGithub(gitCredentialId, organization, repository, releaseVersion, message) {
    bodyMessage = message.replaceAll("\n", "<br />").replace("\r", "")
    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: gitCredentialId, usernameVariable: 'GIT_USERNAME', passwordVariable: 'GIT_PASSWORD']]) {
        def request = """
            {
                "tag_name": "${releaseVersion}",
                "name": "${releaseVersion}",
                "body": "${bodyMessage}",
                "draft": false,
                "prerelease": false
            }
        """
        echo request
        def response = httpRequest consoleLogResponseBody: true, acceptType: 'APPLICATION_JSON', contentType: 'APPLICATION_JSON', httpMode: 'POST', requestBody: request, url: "https://api.github.com/repos/${organization}/${repository}/releases?access_token=${GIT_PASSWORD}"
        return response.content
    }
}

def currentVersion() {
    return readFile("version").trim()
}

def changeVersion(version) {
    writeFile file: 'version', text: version
}

def calculateReleaseVersion(currentVersion) {
    def index = currentVersion.lastIndexOf("-SNAPSHOT")
    def releaseVersion
    if (index > -1) {
        releaseVersion = currentVersion.substring(0, index)
    } else {
        releaseVersion = currentVersion
    }
    return releaseVersion
}

def calculateNextDevVersion(releaseVersion) {
    int index = releaseVersion.lastIndexOf('.')
    String minor = releaseVersion.substring(index + 1)
    int m = minor.toInteger() + 1
    return releaseVersion.substring(0, index + 1) + m + "-SNAPSHOT"
}

node("JenkinsOnDemand") {
    def repository = 'hydro-serving-protos'
    def organization = 'Hydrospheredata'
    def gitCredentialId = 'HydrospheredataGithubAccessKey'

    stage('Checkout') {
        deleteDir()
        checkoutSource(gitCredentialId, organization, repository)
        echo currentVersion()
    }

    if (isReleaseJob()) {
        stage('Set release version') {
            def curVersion = currentVersion()
            def releaseVersion = calculateReleaseVersion(curVersion)
            sh "git checkout -b release_temp"
            changeVersion(releaseVersion)
        }
    }


    stage('Build') {
        sh "./build.sh all"
    }

    if (isReleaseJob()) {
        if (currentBuild.result == 'UNSTABLE') {
            currentBuild.result = 'FAILURE'
            error("Errors in tests")
        }

        stage('Push to PYPI') {
            sh 'pip install --upgrade pip'
            sh 'sudo pip install twine'
            configFileProvider([configFile(fileId: 'PYPIDeployConfiguration', targetLocation: 'target/.pypirc', variable: 'PYPI_SETTINGS')]) {
                sh "twine upload --config-file ${env.WORKSPACE}/target/.pypirc -r pypi ${env.WORKSPACE}/target/python/dist/*.tar.gz"
            }
        }
        stage('Push to Maven') {
            sh "${env.WORKSPACE}/sbt/sbt 'set pgpPassphrase := Some(Array())' publishSigned"
            sh "${env.WORKSPACE}/sbt/sbt 'sonatypeRelease'"
        }

        stage("Create tag"){
            def curVersion = currentVersion()
            tagComment=generateTagComment(curVersion)
            sh "git commit -a -m 'Releasing ${curVersion}'"
            sh "git tag -a ${curVersion} -m '${tagComment}'"

            sh "git checkout ${env.BRANCH_NAME}"

            def nextVersion=calculateNextDevVersion(curVersion)
            changeVersion(nextVersion)

            sh "git commit -a -m 'Development version increased: ${nextVersion}'"

            pushSource(gitCredentialId, organization, repository, "")
            pushSource(gitCredentialId, organization, repository, "refs/tags/${curVersion}")
            createReleaseInGithub(gitCredentialId, organization, repository, curVersion, tagComment)
        }
    } else {
        echo "Do nothing"
    }

}