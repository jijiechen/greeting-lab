def git_repo = 'https://github.com/jijiechen/greeting-lab.git'
def public_route_prefix = ''  // put your hostname prefix here
def max_replica_count = 1

def git_branch = 'master'

// make sure that the port on which app status is hosted is not exported via the route.
def health_probe_port = '9090'
def health_probe_path = '/actuator/health'

def nexus_base_url = 'http://nexus.apps.jijiechen.com'
def nexus_deps_repo = "$nexus_base_url/repository/maven-all-public/"
def nexus_deploy_repo = "$nexus_base_url/repository/csg-apps/"
def ocp_project = 'csg-apps'

def release = 'v1.0.1'

def appName
def appFullVersion
def gitCommitId

node ('maven'){
    stage('Checkout') {
        git url: "${git_repo}", branch: "${git_branch}" 
    }

    stage('Prepare'){
         withCredentials([[$class: 'UsernamePasswordMultiBinding',
            credentialsId: 'bfsIntNexusJenkinsCredential',
            usernameVariable: 'nexus_username', passwordVariable: 'nexus_password']]) {
            prepareSettingsXml(nexus_deps_repo, nexus_username, nexus_password)
            addDistributionToPom(nexus_deploy_repo)
        }

        appName = getFromPom('name')
        if(appName == null || appName.trim() == ""){
          appName = getFromPom('artifactId')
        }
        sh "mvn -s ./cicd-template/maven/settings.xml build-helper:parse-version versions:set -DnewVersion=\\\${parsedVersion.majorVersion}.\\\${parsedVersion.minorVersion}.${BUILD_NUMBER} versions:commit"
        appFullVersion = getFromPom('version')
        gitCommitId = sh(returnStdout: true, script: 'git rev-parse HEAD').trim()
        echo "appName: '${appName}', appFullVersion:'${appFullVersion}', gitCommitId:'${gitCommitId}'"
    }
    stage('Build') {
        sh 'mvn clean package -D skipTests -s ./cicd-template/maven/settings.xml'
    }

    stage ('Test'){
        sh 'mvn test -s ./cicd-template/maven/settings.xml'
    }

    stage('Sonarqube Code Quality Scan'){
        sh 'mvn sonar:sonar -Dsonar.host.url=http://sonarqube.apps.jijiechen.com -s ./cicd-template/maven/settings.xml' 
    }

    stage ('Archive'){
        sh """
            mvn deploy  -DskipTests -s ./cicd-template/maven/settings.xml
            git checkout .
        """
    }

    stage ('OCP Build Preparation'){
        jarFile = sh(returnStdout: true, script: 'find ./target -maxdepth 1 -regextype posix-extended -regex ".+\\.(jar|war)\$" | head -n 1').trim()
        if(jarFile == null || jarFile == ""){
          error 'Can not find the generated jar/war file from "./target" directory'
        }
        jarFile = jarFile.substring('./target/'.length());

        sh """
            set -x
            set -e

            mkdir -p ./target/publish/.s2i
            cp ./target/$jarFile ./target/publish/
            echo 'JAVA_APP_JAR=/deployments/${jarFile}' > ./target/publish/.s2i/environment
        """
    }
    
    stage ('OCP Build'){
        appMajorVersion = appFullVersion.substring(0, appFullVersion.indexOf('.'))
        sh """
                set -x
                set -e

                oc project ${ocp_project}
                oc process -f ./cicd-template/openshift/build-config-template.yaml -n ${ocp_project} \
                -p S2I_BUILD_IMAGE='openjdk18-openshift:latest' \
                -p APP_NAME='${appName}' -p APP_FULL_VERSION='${appFullVersion}' -p APP_MAJOR_VERSION='${appMajorVersion}' \
                -p GIT_COMMIT_ID=${gitCommitId} -p JENKINS_BUILD_NUMBER=${BUILD_NUMBER}  \
                | oc apply -n ${ocp_project} -f -
            
                oc start-build ${appName}-v${appMajorVersion} -n ${ocp_project} --from-dir='./target/publish' --follow
           """
           // oc import-image redhat-openjdk-18/openjdk18-openshift --from=registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift --confirm
    }

    stage ('OCP ConfigMap'){
        sh """
                set -x
                set -e
                if [ -f './src/config/dev/application.properties' ]; then
                    export APP_CONFIG_DATA=\$(cat src/config/dev/application.properties)
                else 
                    echo 'Please prepare your application configuration at "src/config/dev/application.properties"'
                    exit 1
                fi

                oc project ${ocp_project}
                oc process -f ./cicd-template/openshift/configmap-template.yaml -n ${ocp_project} \
                  -p APP_NAME='${appName}' -p APP_FULL_VERSION='${appFullVersion}' -p APP_MAJOR_VERSION='${appMajorVersion}' \
                  -p GIT_COMMIT_ID=${gitCommitId} -p JENKINS_BUILD_NUMBER=${BUILD_NUMBER} -p CONFIG_DATA="\$APP_CONFIG_DATA" \
                  | oc apply -n ${ocp_project} -f -
           """
    }

    stage ('OCP Deployment'){
        sh """
            set -x
            set -e

            oc project ${ocp_project}
            oc process -f ./cicd-template/openshift/deployment-config-template.yaml -n ${ocp_project} \
                -p APP_NAME=${appName} -p APP_FULL_VERSION=${appFullVersion} -p APP_MAJOR_VERSION=${appMajorVersion}  \
                -p GIT_COMMIT_ID=${gitCommitId} -p JENKINS_BUILD_NUMBER=${BUILD_NUMBER} \
                -p HEALTH_PROBE_PORT='${health_probe_port}' -p HEALTH_PROBE_PATH='${health_probe_path}' \
                -p CPU_LIMITATION='1' -p MEM_LIMITATION='1024Mi' \
                | oc apply -n ${ocp_project} --force=true -f -
            
            """

        if (public_route_prefix != null && public_route_prefix != ''){
            sh """
                set -x
                set -e

                oc project ${ocp_project}
                oc process -f ./cicd-template/openshift/route-template.yaml -n ${ocp_project} \
                    -p APP_NAME=${appName} -p APP_FULL_VERSION=${appFullVersion} -p APP_MAJOR_VERSION=${appMajorVersion}  \
                    -p GIT_COMMIT_ID=${gitCommitId} -p PUBLIC_ROUTE_PREFIX=${public_route_prefix} -p JENKINS_BUILD_NUMBER=${BUILD_NUMBER} \
                    | oc apply -n ${ocp_project} --force=true -f -
                
                """
        }

        if (max_replica_count != null && max_replica_count > 1){
            sh """
                set -x
                set -e

                oc project ${ocp_project}                
                oc autoscale dc/${appName}-v${appMajorVersion} --min 1 --max ${max_replica_count} --cpu-percent=60                               
                oc rollout status dc/${appName}-v${appMajorVersion}
                """
        }        
    }
}


def getFromPom(key) {
    sh(returnStdout: true, script: "mvn -s ./cicd-template/maven/settings.xml -q -Dexec.executable=echo -Dexec.args='\${project.${key}}' --non-recursive exec:exec").trim()
}

/**
* Add a 'distribution-management' section into pom.xml
* So that the jar/war artifact could be packed and uploaded to repository
*/
def addDistributionToPom(nexus_deploy_repo) {
    pom = 'pom.xml'
    distMngtSection = readFile('./cicd-template/maven/pom-distribution-management.xml') 
    distMngtSection = distMngtSection.replaceAll('\\$nexus_deploy_repo', nexus_deploy_repo)

    content = readFile(pom)
    newContent = content.substring(0, content.lastIndexOf('</project>')) + distMngtSection + '</project>'
    writeFile file: pom, text: newContent
}


/**
* Create a 'settings.xml' file for maven to use
* So that maven can use internal nexus repo as a mirror server when installing dependencies
*/
def prepareSettingsXml(nexus_deps_repo, nexus_username, nexus_password) {
    settingsXML = readFile('./cicd-template/maven/settings.xml') 
    settingsXML = settingsXML.replaceAll('\\$nexus_deps_repo', nexus_deps_repo)
    settingsXML = settingsXML.replaceAll('\\$nexus_username', nexus_username)
    settingsXML = settingsXML.replaceAll('\\$nexus_password', nexus_password)

    writeFile file: './cicd-template/maven/settings.xml', text: settingsXML
}
