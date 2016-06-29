// Reusable Jenkinsfile for Terraform projects. An Atlas replacement.
// 
// Git:
// * GIT_URL:             The URL for the Git repository.
// * GIT_CREDS_ID:        A User/Password Credential for accessing the Git repository.
// * GIT_SUBDIR:          The subdirectory within the Git repository containing the Terraform config to execute.

// Terraform:
// * TF_VERSION:          The version number using tags in dockerhub for hashicorp/terraform. Uses 'latest' by default.

// Remote Config:
// * TF_REMOTE_AWS_CREDS: A User/Password Credential for accessing remote config, where the username is the AWS Access Key
//                        and the password is the Secret Key.
// * TF_REMOTE_ARGS:      The full set of arguments to use when configuring remote state for Terraform.
// Alternately...
// * TF_REMOTE_BACKEND:   The backend to use. (e.g. 's3')
// Terraform S3 Remote Config:
// * TF_REMOTE_S3_BUCKET: The S3 bucket name.
// * TF_REMOTE_S3_KEY:    The path in the bucket where the state is stored. (e.g. 'instance/foo/terraform.tfstate')
// * TF_REMOTE_S3_REGION: The AWS Region where is is stored. (e.g. 'us-east-1')

// Plan/Apply:
// * TF_APPLY_AWS_CREDS: A User/Password Credential for planning/applying, where the username is the AWS Access Key
//                       and the password is the Secret Key.
// * TF_APPLY_ARGS:      The full set of arguments to use when planning or applying Terraform.

import groovy.json.JsonSlurper

node {
    stage "Setup"
    // Check out the project
    git credentialsId: GIT_CREDS_ID, url: GIT_URL
    // Pull the remote config
    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: TF_REMOTE_AWS_CREDS, usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
        withEnv(["AWS_ACCESS_KEY_ID=${env.AWS_ACCESS_KEY_ID}", "AWS_SECRET_ACCESS_KEY=${env.AWS_SECRET_ACCESS_KEY}"]) {
            sh 'echo DEBUG: AWS_ACCESS_KEY_ID is $AWS_ACCESS_KEY_ID'
            tfRemoteConfig()
        }
    }
    // Update any modules
    terraform "get -update=true"

    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: TF_APPLY_AWS_CREDS, usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
        withEnv(["AWS_ACCESS_KEY_ID=${env.AWS_ACCESS_KEY_ID}", "AWS_SECRET_ACCESS_KEY=${env.AWS_SECRET_ACCESS_KEY}"]) {
            stage 'Plan'
            terraform "plan -input=false ${TF_APPLY_ARGS}"
            input 'Apply the plan?'

            stage 'Apply'
            terraform "apply -input=false ${TF_APPLY_ARGS}"
        }
    }
}

boolean hasRemoteConfig() {
    def terraformState = "${workingDirectory}/${instanceSubDir}/.terraform/terraform.tfstate"
    if (fileExists(terraformState)) {
        println "Found the terraform.state"
        def config = new JsonSlurper().parseText(readFile(terraformState))
        println "config.remote: ${config?.remote}"
        return config && config.remote
    }
    return false
}

def tfRemoteConfig() {
    if (!hasRemoteConfig()) {
        terraform "remote config ${tfRemoteArgs}"
    } else {
        terraform "remote pull"
    }
}

def terraform(String tfArgs) {
    sh "${terraformCmd} ${tfArgs}"
}

def getTerraformCmd() {
    "docker run --rm -v ${workingDirectory}:${tempDirectory} -w=${tempDirectory}/${instanceSubDir} -e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY hashicorp/terraform:${tfVersion}"
}

def getId() {
    "${env.JOB_NAME}-${env.BUILD_ID}"
}

String getTempDirectory() {
    "/tmp/${id}"
}

String getWorkingDirectory() {
    "${pwd()}"
}

String getInstanceSubDir() {
    "${GIT_SUBDIR}"
}

String getTfVersion() {
    TF_VERSION ?: "latest"
}

String getTfRemoteArgs() {
    if (TF_REMOTE_BACKEND == "s3") {
        "-backend=${TF_REMOTE_BACKEND} -backend-config='bucket=${TF_REMOTE_S3_BUCKET}' -backend-config='key=${TF_REMOTE_S3_KEY}' -backend-config='region=${TF_REMOTE_S3_REGION}'"
    } else {
        "${TF_REMOTE_ARGS ?: ''}"
    }
}
