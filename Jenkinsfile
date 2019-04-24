pipeline {
    agent any

    options {
        // set a timeout of 5 minutes for this pipeline
        timeout(time: 5, unit: 'MINUTES')
    } //options

    environment {
        APP_NAME    = "podcicd"
        GIT_REPO    = "https://github.com/williamcaban/podcicd.git"
        GIT_BRANCH  = "master"
        CONTEXT_DIR = "myapp"
        CURR_BUILD  = "${currentBuild.number}"
        PREV_BUILD  = "${currentBuild.previousBuild.getNumber()}"
        BUILD_CAUSE = "${currentBuild.rawBuild.getCauses()}"

        CICD_PRJ    = "cicd"
        CICD_DEV    = "${CICD_PRJ}"+"-dev"
        CICD_PROD   = "${CICD_PRJ}"+"-prod"
        CICD_STAGE  = "${CICD_PRJ}"+"-staging"
        SVC_PORT    = 8080
    }

    stages {
            stage('Build') {
                steps {
                    echo "Sample Build stage using project ${CICD_DEV}"
                    script {
                        openshift.withCluster() {
                            openshift.withProject("${CICD_DEV}")
                            {

                                if (openshift.selector("bc",APP_NAME).exists()) {
                                    echo "Using existing BuildConfig. Running new Build"
                                    def bc = openshift.startBuild(APP_NAME)
                                    openshift.set("env dc/${APP_NAME} BUILD_NUMBER=${CURR_BUILD}")
                                    // output build logs to the Jenkins conosole
                                    echo "Logs from build"
                                    def result = bc.logs('-f')
                                    // actions that took place
                                    echo "The logs operation require ${result.actions.size()} 'oc' interactions"
                                    // see exactly what oc command was executed.
                                    echo "Logs executed: ${result.actions[0].cmd}"
                                } else {
                                    echo "No proevious BuildConfig. Creating new BuildConfig."
                                    def myNewApp = openshift.newApp (
                                        "${GIT_REPO}#${GIT_BRANCH}", 
                                        "--name=${APP_NAME}", 
                                        "--context-dir=${CONTEXT_DIR}", 
                                        "-e BUILD_NUMBER=${CURR_BUILD}", 
                                        "-e BUILD_ENV=${openshift.project()}"
                                        )
                                    echo "new-app myNewApp ${myNewApp.count()} objects named: ${myNewApp.names()}"
                                    myNewApp.describe()
                                    // selects the build config 
                                    def bc = myNewApp.narrow('bc')
                                    // output build logs to the Jenkins conosole
                                    echo "Logs from build"
                                    def result = bc.logs('-f')
                                    // actions that took place
                                    echo "The logs operation require ${result.actions.size()} 'oc' interactions"
                                    // see exactly what oc command was executed.
                                    echo "Logs executed: ${result.actions[0].cmd}"
                                } //else

                                echo "Tag Container image with 'build number' as version"
                                openshift.tag("${APP_NAME}:latest", "${APP_NAME}:v${BUILD_NUMBER}")

                                echo "Validating Route for Service exist, if Not create Route"
                                if (!openshift.selector("route",APP_NAME).exists()) {
                                    openshift.selector("svc",APP_NAME).expose()
                                }

                            } // project
                        } // cluster
                    } // script
                } // steps
            } //stage-build

            stage('Test') {

                steps {
                    echo "Testing if 'Service' resource is operational and responding"
                    script {
                        openshift.withCluster() {
                                openshift.withProject() {
                                    echo sh (script: "curl -I ${APP_NAME}.${CICD_DEV}.svc:${SVC_PORT}/healthz", returnStdout: true)
                                } // withProject
                        } // withCluster
                    } // script
                } // steps
            } //stage 
            
            stage('Promote to Staging') {
                steps {
                    echo "Setup for Staging"
                    script {
                        openshift.withCluster() {
                            openshift.withProject("${CICD_STAGE}") {
                                echo "Tag new image for staging"
                                openshift.tag("${CICD_DEV}/${APP_NAME}:v${BUILD_NUMBER}", "${CICD_STAGE}/${APP_NAME}:v${BUILD_NUMBER}")
                                openshift.tag("${CICD_STAGE}/${APP_NAME}:v${BUILD_NUMBER}", "${CICD_STAGE}/${APP_NAME}:latest")
                                echo "Deploying to project: ${openshift.project()}"
                                def myStagingApp = openshift.newApp(
                                    "${APP_NAME}:v${BUILD_NUMBER}",
                                    "--name=${APP_NAME}-${BUILD_NUMBER}", 
                                    "-e BUILD_NUMBER=${CURR_BUILD}", 
                                    "-e BUILD_ENV=${openshift.project()}"
                                )
                                myStagingApp.narrow("svc").expose()
                            }
                        }
                    } // script
                } //steps 
            } //stage

            stage('Promote to Prod'){
                steps {
                    echo "Promote to production"
                    script {
                        openshift.withCluster() {
                            openshift.withProject("${CICD_PROD}") {
                                echo "Deploying to project: ${openshift.project()}"
                            }
                        }
                    } // script
                } // steps
            } //stage

    } // stages
} // pipeline
