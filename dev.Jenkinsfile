def appName = "validasipibk"
def projectName = "testpibk"
def repoUrl = "github.com/Adhitia09/adhit.git"
def branchName = "master"

def pullImageDC = "quayuser-djbc-dc-pull-secret"
def pullImageDRC = "quayuser-djbc-drc-pull-secret"

def intRegistryDev = "default-route-openshift-image-registry.apps.dev.customs.go.id"
def extRegistryQuayDC = "quay-registry.apps.proddc.customs.go.id"
def extRegistryQuayDRC = "quay-registry.apps.proddrc.customs.go.id"

pipeline {
    agent any

    stages {
        stage('Git Clone') {
            steps {
                sh "git config --global http.sslVerify false"
                sh "git clone https://${repoUrl} source"
            }
        }

        stage('App Build') {
            steps {
                dir("source") {
                    sh "git fetch"
                    sh "git switch ${branchName}"
                }
            }
        }

        stage('App Push') {
            steps {
                dir("source") {
                    sh """
                        mkdir -p build-folder/target/ build-folder/apps/
                        cp ocp.Dockerfile build-folder/Dockerfile
                        cp app.py build-folder/app.py
                        cp requirements.txt build-folder/requirements.txt
                    """

                    def tag = sh(returnStdout: true, script: "git rev-parse --short=8 HEAD").trim()
                    def tokenLocal = sh(script: 'oc whoami -t', returnStdout: true).trim()

                    sh "oc delete bc ${appName} || true"
                    sh "cat build-folder/Dockerfile | oc new-build -D - --name ${appName} || true"
                    sh "oc start-build ${appName} --from-dir=build-folder/. --follow --wait"
                    sh "oc tag cicd4/${appName}:latest ${projectName}/${appName}:${tag}"

                    withCredentials([usernamePassword(credentialsId: 'quay-dc-credential', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        sh """
                            skopeo copy --remove-signatures --src-creds=jenkins:${tokenLocal} --src-tls-verify=false \
                                docker://${intRegistryDev}/${projectName}/${appName}:${tag} \
                                docker://${extRegistryQuayDC}/djbc/${projectName}_${appName}-dev:${tag} --dest-creds \${USERNAME}:\${PASSWORD} --dest-tls-verify=false
                        """
                    }

                    withCredentials([usernamePassword(credentialsId: 'quay-drc-credential', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                        sh """
                            skopeo copy --remove-signatures --src-creds=jenkins:${tokenLocal} --src-tls-verify=false \
                                docker://${intRegistryDev}/${projectName}/${appName}:${tag} \
                                docker://${extRegistryQuayDRC}/djbc/${projectName}_${appName}-dev:${tag} --dest-creds \${USERNAME}:\${PASSWORD} --dest-tls-verify=false
                        """
                    }
                }
            }
        }

        stage('Deploy to Dev') {
            parallel {
                stage('App Deploy to DEV DC OCP') {
                    steps {
                        dir("source") {
                            sh """
                                cp kubernetes-dev.yaml kubernetes-dc.yaml
                                oc apply -f quayuser-djbc-dc-pull-secret.yaml -n ${projectName}
                                sed 's,\\\$REGISTRY/\\\$HARBOR_NAMESPACE/\\\$APP_NAME:\\\$BUILD_NUMBER,${extRegistryQuayDC}/djbc/${projectName}_${appName}-dev:latest,g' \
                                    kubernetes-dc.yaml > kubernetes-dc-quay.yaml
                                sed 's,\\\$IMAGEPULLSECRET,${pullImageDC},g' kubernetes-dc-quay.yaml > kubernetes-ocp-quay-dc.yaml
                                oc apply -f kubernetes-ocp-quay-dc.yaml -n ${projectName}
                                oc set triggers deployment/${appName} -c ${appName} -n ${projectName} || true
                                oc rollout restart deployment/${appName} -n ${projectName}
                            """
                        }
                    }
                }

                stage('App Deploy to DEV DRC OCP') {
                    steps {
                        dir("source") {
                            withCredentials([file(credentialsId: 'drc-dev-ocp', variable: 'KUBE_CONFIG_DEV_DRC')]) {
                                sh """
                                    cp kubernetes-dev.yaml kubernetes-drc.yaml
                                    oc --kubeconfig=\$KUBE_CONFIG_DEV_DRC apply -f quayuser-djbc-drc-pull-secret.yaml -n ${projectName}
                                    sed 's,\\\$REGISTRY/\\\$HARBOR_NAMESPACE/\\\$APP_NAME:\\\$BUILD_NUMBER,${extRegistryQuayDRC}/djbc/${projectName}_${appName}-dev:latest,g' \
                                        kubernetes-drc.yaml > kubernetes-drc-quay.yaml
                                    sed 's,\\\$IMAGEPULLSECRET,${pullImageDRC},g' kubernetes-drc-quay.yaml > kubernetes-ocp-quay-drc.yaml
                                    oc --kubeconfig=\$KUBE_CONFIG_DEV_DRC apply -f kubernetes-ocp-quay-drc.yaml -n ${projectName}
                                    oc --kubeconfig=\$KUBE_CONFIG_DEV_DRC set triggers deployment/${appName} -c ${appName} -n ${projectName} || true
                                    oc --kubeconfig=\$KUBE_CONFIG_DEV_DRC rollout restart deployment/${appName} -n ${projectName}
                                """
                            }
                        }
                    }
                }
            }
        }
    }
}
