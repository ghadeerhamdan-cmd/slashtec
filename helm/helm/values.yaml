def branchName     = params.BranchName ?: "main"
def gitUrl         = "git@github.com:ghadeerhamdan-cmd/slashtec.git"
def gitUrlCode     = "git@github.com:ghadeerhamdan-cmd/slashtec.git"
def serviceName    = "ghadeerecr"
def EnvName        = "preprod"
def registryId     = "727245885999"
def awsRegion      = "ap-south-1"
def ecrUrl         = "727245885999.dkr.ecr.ap-south-1.amazonaws.com"
def dockerfile     = "dockerfile/Dockerfile"
def imageTag       = "${EnvName}-${BUILD_NUMBER}"
def ARGOCD_URL     = "https://127.0.0.1:8080/applications"

// AppConfig Params
def applicationName = "airport-countries"
def envName = "preprod"
def configName = "preprod"
// Fix: Use string concatenation, not arithmetic
def clientId = "${applicationName}-${envName}"
def latestTagValue = params.Tag
def namespace = "preprod"
def helmDir = "helm/helm"
def slashtecDir = "helm"

// Define the notifyBuild function
def notifyBuild(String buildStatus, String branchName) {
  // Build status of null means success
  buildStatus = buildStatus ?: 'SUCCESS'

  // Default values
  def subject = "${buildStatus}: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
  def summary = "${subject} (${env.BUILD_URL}) (Branch: ${branchName})"
  def details = """<p>STARTED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]':</p>
    <p>Check console output at &QUOT;<a href='${env.BUILD_URL}'>${env.JOB_NAME} [${env.BUILD_NUMBER}]</a>&QUOT;</p>
    <p>Branch: ${branchName}</p>"""
  def color
  def colorCode

  // Define color variables
  color = 'YELLOW'
  colorCode = '#FFFF00'

  // Override default values based on build status
  if (buildStatus == 'STARTED') {
    color = 'YELLOW'
    colorCode = '#FFFF00'
  } else if (buildStatus == 'SUCCESS') {
    color = 'GREEN'
    colorCode = '#00FF00'
  } else {
    color = 'RED'
    colorCode = '#FF0000'
  }
  
}

node {
  try {
    notifyBuild('STARTED', branchName)
      stage('cleanup') {
        cleanWs()
      }
      stage ("Get the app code") {
        checkout([$class: 'GitSCM', branches: [[name: "${branchName}"]] , extensions: [], userRemoteConfigs: [[ url: "${gitUrlCode}"]]])
        sh "rm -rf ${WORKSPACE}/slashtec"
        sh "mkdir -p ${WORKSPACE}/slashtec && cd ${WORKSPACE}/slashtec && git clone -b main ${gitUrl} ."
        sh("cp dockerfile/* . || echo 'No files in dockerfile directory'")
        sh("ls -la")
      }
      stage("Get the env variables from App") {
        sh """
          echo "=== Environment Configuration ==="
          echo "Application: ${applicationName}"
          echo "Environment: ${envName}"
          echo "Region: ${awsRegion}"
          
          if command -v aws &> /dev/null; then
            echo "AWS CLI found, attempting to fetch configuration..."
            aws appconfig get-configuration --application ${applicationName} --environment ${envName} --configuration ${configName} --client-id ${clientId} .env --region ${awsRegion} || {
              echo "AWS AppConfig fetch failed, creating fallback environment..."
              echo "# Fallback environment configuration" > .env
              echo "ENVIRONMENT=${envName}" >> .env
              echo "SERVICE_PORT=8080" >> .env
              echo "JAVA_OPTS=-Xmx512m -Xms256m" >> .env
            }
          else
            echo "AWS CLI not found on Jenkins server"
            echo "Creating comprehensive fallback .env file..."
            echo "# Fallback environment configuration" > .env
            echo "ENVIRONMENT=${envName}" >> .env
            echo "SERVICE_PORT=8080" >> .env
            echo "JAVA_OPTS=-Xmx512m -Xms256m" >> .env
            echo "AWS_REGION=${awsRegion}" >> .env
            echo "APPLICATION_NAME=${applicationName}" >> .env
            echo "Log level set to INFO for ${envName} environment" >> .env
          fi
          
          echo "Environment file contents:"
          cat .env || echo "No .env file created"
        """
      }
      stage('login to ecr') {
        sh("aws ecr get-login-password --region ${awsRegion}  | docker login --username AWS --password-stdin ${ecrUrl}")
      }
      stage('Build Docker Image') {
        sh """
          echo "=== Docker Build Debug Info ==="
          echo "Service Name: ${serviceName}"
          echo "ECR URL: ${ecrUrl}"
          echo "Image Tag: ${imageTag}"
          echo "Dockerfile: ${dockerfile}"
          echo "Available JAR files in helm/:"
          ls -la helm/*.jar || echo "No JAR files found"
          
          echo "Preparing Docker build context..."
          cp helm/*.jar dockerfile/ || echo "Warning: No JAR files copied"
          
          echo "Files in dockerfile directory:"
          ls -la dockerfile/
          
          echo "Building Docker image..."
          docker build -t ${ecrUrl}/${serviceName}:${imageTag} -f ${dockerfile} dockerfile/
          
          echo "=== Build Complete ==="
        """
      }
      stage('Push Docker Image To ECR') {
        sh("docker push ${ecrUrl}/${serviceName}:${imageTag}")
      }
      stage('Clean docker images') {
        sh("docker rmi -f ${ecrUrl}/${serviceName}:${imageTag} || :")
      }
      stage ("Deploy ${serviceName} to ${EnvName} Environment") {
        sh ("cd slashtec/${helmDir}; pathEnv=\".applications.airports.versions.image.tag\" valueEnv=\"${imageTag}\" yq 'eval(strenv(pathEnv)) = strenv(valueEnv)' -i values.yaml ; cat values.yaml")
        sh ("cd slashtec/${helmDir}; git pull ; git add values.yaml; git commit -m 'update image tag' ;git push ${gitUrl}")
      }
  
    } catch (Exception e) {
      currentBuild.result = 'FAILURE'
      throw e
    } finally {
      notifyBuild(currentBuild.result ?: 'SUCCESS', branchName)
    }
}

