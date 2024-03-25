pipeline {
    agent {
       kubernetes {
           yaml """
apiVersion: v1 
kind: Pod 
metadata: 
    name: dind
    annotations:
      container.apparmor.security.beta.kubernetes.io/dind: unconfined
      container.seccomp.security.alpha.kubernetes.io/dind: unconfined
spec: 
    containers: 
      - name: dind
        image: docker:dind
        securityContext:
          privileged: true
        tty: true
        volumeMounts:
        - name: var-run
          mountPath: /var/run
      - name: jnlp
        securityContext:
          runAsUser: 0
          fsGroup: 0
        volumeMounts:
        - name: var-run
          mountPath: /var/run
        
    volumes:
    - emptyDir: {}
      name: var-run
"""
       }
   }
    
    parameters {  // Defines Parameters as Code
        string(name: "git_repository", defaultValue: "https://github.com/pabloopez/sysdig-jenkins-ci-training.git", trim: true, description: "Git Repo to build Dockerfile from")
        string(name: "git_branch", defaultValue: "main", trim: true, description: "Git branch to build Dockerfile from")
        string(name: "docker_tag", defaultValue: "myapp:v1.0.1", trim: true, description: "Docker Image Tag")
        string(name: "registry_url", defaultValue: "ghcr.io", trim: true, description: "Container Registry URL")
        string(name: "registry_repo", defaultValue: "pabloopez", trim: true, description: "Container Registry URL")
        string(name: "sysdig_url", defaultValue: "https://secure.sysdig.com", trim: true, description: "Sysdig URL based on Sysdig SaaS region")
        string(name: "plugin_policies_to_apply", defaultValue: "", description: "Space separated list of policies to apply (Plugin execution only)")
        booleanParam(name: 'bail_on_fail', defaultValue: true, description: 'Want to stop the Pipeline execution if the Scan returns a failed policy evaluation? (Plugin execution only)')
        booleanParam(name: 'bail_on_plugin_fail', defaultValue: true, description: 'Want to stop the pipeline if the Jenkins Plugin Fails? (Plugin execution only)')
        string(name: "sysdig_cli_args", defaultValue: "", trim: true, description: "Optional inline arguments (Sysdig CLI Scanner Execution only)")
    }
    stages {
        stage('Clean Workspace') {  // Cleans the workspace to avoid old files conflicts
            steps {
                cleanWs()
                sh 'rm -rf .git'
            }
        }
        stage('Clone repo'){  // Clones repo into the working directory
            steps{
                sh "git clone ${git_repository} ."
                sh "git checkout ${git_branch}"
            }
        }
        stage('Build image'){  // Builds the image from a Dockerfile
            steps{
                sh "docker image build --tag ${registry_url}/${registry_repo}/${docker_tag}  --label 'stage=TRAINING' ."
            }
        }
        stage('Plugin Scan'){  // Scans the built image using the Sysdig Jenkins Plugin
            steps{
                sysdig engineCredentialsId: "${sysdig-secure-api-credentials}", imageName: "${registry_url}/${registry_repo}/${docker_tag}", engineURL: "${params.sysdig_url}", policiesToApply: "${params.plugin_policies_to_apply}", bailOnFail: "${params.bail_on_fail}", bailOnPluginFail: "${params.bail_on_plugin_fail}"
            }    
        }
        stage('Push Docker Image'){  // Pushes the images to the Container Registry
            steps{
                withCredentials([usernamePassword(credentialsId: 'gcr_rw_token', usernameVariable: 'username', passwordVariable: 'password')]) {
                    sh 'echo ${password} | docker login ${registry_url} -u ${username} --password-stdin'
                    sh 'docker push ${registry_url}/${registry_repo}/${docker_tag}'
                }
            }
        }
    }
}
