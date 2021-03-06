@Library('eigi-jenkins-library') _
def gitCmd = new common.v1.GitCmd(this)
def runWith = new common.v1.RunWith(this)
def when = new common.v1.When(this)
def awsTools = new ctct.v1.AwsTools(this)

def appBaseName = 'sock-shop-front-end'
def tagVersion

node('p2-team-jenkins-slave-14.ctct.net') {
    def ecrRepo = "428791060841.dkr.ecr.us-east-1.amazonaws.com/${appBaseName}"
    def fullTagName

    dir('app-repo') {
        gitCmd.checkout()

        runWith.nodeJS('10.13.0') {
            sh('npm install')
        }

        def repo = 'front-end'

        withAWS(credentials: 'jesnkins-bfa-user',
                role: 'ctct-deploy-qa-ecr-access',
                roleAccount: '428791060841',
                region: 'us-east-1') {

            def login = ecrLogin()
            sh login

            when.buildingPR {
                tagVersion = env.BRANCH_NAME
            }

            when.buildingMaster {
                tagVersion = "latest"
            }

            fullTagName = "${ecrRepo}:${tagVersion}"
            docker.build(fullTagName, '. --network=host').push()
        }

    }

    dir('app-registry') {
        def appRegistry = 'https://github.com/citizenken/argocd-project-registry.git'
        gitCmd.checkoutRemoteWithBranch(appRegistry, 'master', 'jenkins-ssh')

        def application = readYaml file: 'apps/sock-shop/application.yaml'
        def namespace = readYaml file: 'apps/sock-shop/namespace.yaml'
        def argoManifestLocation

        when.buildingPR {
            argoManifestLocation = "pr-apps/sock-shop-${env.BRANCH_NAME}".toLowerCase()

            def appPRName = "${application.metadata.name}-${env.BRANCH_NAME}".toLowerCase()
            application.metadata.labels.release = 'pr'
            application.metadata.labels.buildNumber = env.BUILD_NUMBER
            application.metadata.name = appPRName
            application.spec.destination.namespace = appPRName
            namespace.metadata.name = appPRName
        }

        when.buildingMaster {
            application.metadata.labels.buildNumber = env.BUILD_NUMBER
            argoManifestLocation = "apps/sock-shop"
        }

        application.spec.source.helm = [
            parameters : [
                [
                    name : 'image.tag',
                    value : tagVersion
                    ],
                [
                    name : 'image.repository',
                    value : ecrRepo
                    ]
            ]
        ]

        sh """
        rm -rf ${argoManifestLocation}/application.yaml
        rm -rf ${argoManifestLocation}/namespace.yaml
        """
        writeYaml file: "${argoManifestLocation}/application.yaml", data: application
        writeYaml file: "${argoManifestLocation}/namespace.yaml", data: namespace

        sh """
        git add .
        git commit -m "updating ${appBaseName} manifests"
        git push
        """
        try {
            echo "i'm running integration tests here"
            input 'click when you want integration tests to be done'
        } catch(Exception){

        } finally {
            when.buildingPR {
                sh """
                git pull origin master
                rm -rf ${argoManifestLocation}
                git add --all
                git commit -m "removing ${appBaseName} PR manifest"
                git push
                """
            }
        }



    }
}
