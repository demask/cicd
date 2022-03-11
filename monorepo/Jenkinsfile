import jenkins.*;
import jenkins.model.*;

AWS_ACCOUNT_ID="246005639140"
AWS_DEFAULT_REGION="eu-central-1" 

ENVIRONMENT_DEV = "dev"
ENVIRONMENT_DEMO = "demo"
ENVIRONMENT_QA = "qa"
ENVIRONMENT_PROD = "prod"
ENVIRONMENT_MASTER = "master"

REPOSITORY_MONOREPO = "aws_codebuild_codedeploy_nodeJs_demo"
REPOSITORY_MULTIREPO = "repo-free"

REPOSITORY_MONOREPO_URL = "https://github.com/demask/aws_codebuild_codedeploy_nodeJs_demo.git"
REPOSITORY_MULTIREPO_URL = "https://github.com/demask/repo-free.git"

TERRAFORM_REPO_URL = "https://github.com/demask/aws_codebuild_codedeploy_nodeJs_demo_terraform.git"
TERRAFORM_REPO_BRANCH = "master"

GIT_CREDENTIALS_ID = ""

def runMonorepoJobs = [
  "job1" : true,
  "job2_1" : true,
  "job2_2" : true,
  "job3" : true
]

def runMultirepoJob = [
  "other-repo-job" : true
]

// List all project paths for which git commit is watched
def jobsByProjectPath = [
    "job1/": "job1",
    "job2/": "job2_1",
    "job3/": "job3"
]

// Find all jobs for project paths in which git commit happend
def jobsToRun(build, jobsByProjectPath) {
  // Listing changes files since last build
  def changeLogSets = build.changeSets
  def changedFiles = []
  for (int i = 0; i < changeLogSets.size(); i++) {
    def entries = changeLogSets[i].items
    for (int j = 0; j < entries.length; j++) {
      def entry = entries[j]
      def files = new ArrayList(entry.affectedFiles)
      for (int k = 0; k < files.size(); k++) {
        def file = files[k]
        changedFiles.add(file.path)
      }
    }
  }

  // Listing jobs to run
  jobsToRun = [:]
  for (entry in jobsByProjectPath) {
    def pattern = entry.key
    for (int i = 0; i < changedFiles.size(); i++) {
      def file = changedFiles[i]
      if (file.contains(pattern)) {
        def jobName = entry.value
        jobsToRun[jobName] = true
        break
      }
    }
  }

  jobsToRun
}

def cloneGitRepo(branchToClone, currentRepo) {    
  def repoUrl
  if(currentRepo == REPOSITORY_MONOREPO) {
    repoUrl = REPOSITORY_MONOREPO_URL
  } else if(currentRepo == REPOSITORY_MULTIREPO) {
    repoUrl = REPOSITORY_MULTIREPO_URL
  }
  
  checkout([
    $class: 'GitSCM', 
    branches: [[name: "$branchToClone"]], 
    doGenerateSubmoduleConfigurations: false, 
    extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'code_directory']], 
    submoduleCfg: [], 
    userRemoteConfigs: [[credentialsId: "$GIT_CREDENTIALS_ID", url: "$repoUrl"]]])            
  
  checkout([
    $class: 'GitSCM', 
    branches: [[name: "*/$TERRAFORM_REPO_BRANCH"]], 
    doGenerateSubmoduleConfigurations: false, 
    extensions: [[$class: 'RelativeTargetDirectory', relativeTargetDir: 'terraform_directory']], 
    submoduleCfg: [], 
    userRemoteConfigs: [[credentialsId: "$GIT_CREDENTIALS_ID", url: "$TERRAFORM_REPO_URL"]]])
}

def runTerraform(dockerImageTag) {   
  sh 'terraform init -no-color'
                  
  sh ("""terraform apply \
    -var=\"docker_image_tag=${dockerImageTag}\" \
    -auto-approve -no-color""")
}

def createEcsServices(newDockerImageMap, environment) {
  def jobAlreadyCreated = false
  newDockerImageMap.each{
    if(!jobAlreadyCreated && (it.key == 'job2_1' || it.key == 'job2_2')) {
      dir("${env.WORKSPACE}/terraform_directory/${environment}/ecs-service/job2"){
        runTerraform(it.value)     
      } 

      jobAlreadyCreated = true
    } else if(it.key != 'job2_1' && it.key != 'job2_2') {
      dir("${env.WORKSPACE}/terraform_directory/${environment}/ecs-service/${it.key}"){
        runTerraform(it.value)     
      } 
    }
  }
}

def buildAndPushDockerImage(IMAGE_TAG, IMAGE_REPO_NAME, DOCKERFILE) {
  def REPOSITORY_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}"

  dockerImage = docker.build("${IMAGE_REPO_NAME}:${IMAGE_TAG}", "-f ${DOCKERFILE} .")   

  sh "docker tag ${IMAGE_REPO_NAME}:${IMAGE_TAG} ${REPOSITORY_URI}:${IMAGE_TAG}"
  sh "docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${IMAGE_REPO_NAME}:${IMAGE_TAG}"     
}

def createDockerImages(currentBuild, jobsByProjectPath, environment, runMonorepoJobs, runMultirepoJob, currentRepo) {
  def newDockerImageMap = [:]
  dir("${env.WORKSPACE}/code_directory"){  
    def jobs

    if(currentRepo == REPOSITORY_MULTIREPO) {
      jobs = runMultirepoJob
    } else if(environment == ENVIRONMENT_DEV || environment == ENVIRONMENT_MASTER) { 
      jobs = jobsToRun(currentBuild, jobsByProjectPath)
      addPhpIfNginxPresent(jobs)
    } else {
      jobs = runMonorepoJobs
    }
    
    def IMAGE_TAG = sh (script: 'git rev-parse --short HEAD',returnStdout: true).trim()
    if(environment == ENVIRONMENT_DEV) {
      IMAGE_TAG = "dev_" + IMAGE_TAG
    } else if(environment == ENVIRONMENT_DEMO) {
      IMAGE_TAG = "demo_" + IMAGE_TAG
    } else if(environment == ENVIRONMENT_QA) {
      IMAGE_TAG = "qa_" + IMAGE_TAG
    } else if(environment == ENVIRONMENT_PROD || environment == ENVIRONMENT_MASTER) {
      IMAGE_TAG = "prod_" + IMAGE_TAG
    }
    
    jobs.each{
      def DOCKERFILE
      def IMAGE_REPO_NAME
      if(it.key) {
        IMAGE_REPO_NAME = it.key
      }

      if(it.key == 'job1' && it.value) {
        DOCKERFILE = 'Dockerfile'
      } else if(it.key == 'job3' && it.value) {
        DOCKERFILE = 'DockerfileJob3'
      } else if(it.key == 'job2_1' && it.value) {
        DOCKERFILE = 'Dockerfile'
      } else if(it.key == 'job2_2' && it.value) {
        DOCKERFILE = 'Dockerfile2'
      } else if(it.key == 'other-repo-job' && it.value) {
        DOCKERFILE = 'Dockerfile'
      }

      if(it.key == 'job3' && it.value && DOCKERFILE && IMAGE_REPO_NAME) {
        dir("${env.WORKSPACE}/code_directory/job3"){
          buildAndPushDockerImage(IMAGE_TAG, IMAGE_REPO_NAME, DOCKERFILE)
          newDockerImageMap[IMAGE_REPO_NAME] = IMAGE_TAG
        }
      } else if(DOCKERFILE && IMAGE_REPO_NAME) {
        buildAndPushDockerImage(IMAGE_TAG, IMAGE_REPO_NAME, DOCKERFILE)
        newDockerImageMap[IMAGE_REPO_NAME] = IMAGE_TAG
      } 
    }  
  }

  return newDockerImageMap
}

def addPhpIfNginxPresent(jobsToRun) {
  def nginxPresent = false
  jobsToRun.each{ 
    if(it.key == 'job2_1' && it.value) {
      nginxPresent = true
    }
  }
  if(nginxPresent) {
    jobsToRun["job2_2"] = true
  }
}

node {
  parameters: [
    string(name: 'branch', defaultValue: '', description: 'Branch on which was merged', trim: true),
    string(name: 'merged', defaultValue: '', description: 'Did merge occured', trim: true),
    string(name: 'ref_type', defaultValue: '', description: 'Ref type', trim: true),
    string(name: 'tag', defaultValue: '', description: 'Tag', trim: true),
    string(name: 'repository', defaultValue: '', description: 'Repository', trim: true),
    string(name: 'action', defaultValue: '', description: 'Action', trim: true)
  ]

  def tagRegex = /.*DEMO.*|.*QA.*|.*PROD.*/
  def currentRepo = "$repository"
  def mergedIntoDevelop = "$action" == "closed" && "$merged" == "true" && "$branch" == "develop"
  def mergedIntoMasterMain = "$action" == "closed" && "$merged" == "true" && ("$branch" == "master" || "$branch" == "main")
  def tagPushed = "$ref_type" == "tag" && "$tag" ==~ tagRegex

  if(mergedIntoMasterMain || mergedIntoDevelop || tagPushed) {
    def branchToClone
    
    if(mergedIntoMasterMain || mergedIntoDevelop) {
      branchToClone = "$branch"
    } else {
      branchToClone = "refs/tags/$tag"
    }

    def environment
    if(mergedIntoMasterMain) {
      environment = ENVIRONMENT_MASTER
    } else if(mergedIntoDevelop) {
      environment = ENVIRONMENT_DEV
    } else if(branchToClone.contains("DEMO")) {
      environment = ENVIRONMENT_DEMO
    } else if(branchToClone.contains("QA")) {
      environment = ENVIRONMENT_QA
    } else if(branchToClone.contains("PROD")) {
      environment = ENVIRONMENT_PROD
    }

    if(environment) {
      stage('Clean workspace') {      
        cleanWs()           
      }

      stage('Logging into AWS ECR') {      
        sh "aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"           
      }
        
      stage('Cloning Git') {
        cloneGitRepo(branchToClone, currentRepo)
      }

      def newDockerImageMap
      stage('Build and push images') {
        newDockerImageMap = createDockerImages(currentBuild, jobsByProjectPath, environment, runMonorepoJobs, runMultirepoJob, currentRepo)
      }
      
      if(environment != ENVIRONMENT_MASTER && environment != ENVIRONMENT_PROD) {
        stage('Run Terraform') { 
          createEcsServices(newDockerImageMap, environment)
        }
      }
    }
  }
}