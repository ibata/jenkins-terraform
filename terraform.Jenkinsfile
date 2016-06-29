// Reusable Jenkinsfile for Terraform projects. An Atlas replacement.
// 
// Git:
// * GIT_URL:             The URL for the Git repository.
// * GIT_CREDS_ID:        A User/Password Credential for accessing the Git repository.
// * GIT_SUBDIR:          The subdirectory within the Git repository containing the Terraform config to execute.

// Terraform:
// * TF_VERSION:          The version number using tags in dockerhub for hashicorp/terraform. Uses 'latest' by default.
// * TF_AWS_CREDS:        A User/Password Credential for managing AWS via Terraform, where the username is the
//                        AWS Access Key and the password is the Secret Key.

// Remote Config:
// * TF_REMOTE_ARGS:      The full set of arguments to use when configuring remote state for Terraform.
// Alternately...
// * TF_REMOTE_BACKEND:   The backend to use. (e.g. 's3')
// Terraform S3 Remote Config:
// * TF_REMOTE_S3_BUCKET: The S3 bucket name.
// * TF_REMOTE_S3_KEY:    The path in the bucket where the state is stored. (e.g. 'instance/foo/terraform.tfstate')
// * TF_REMOTE_S3_REGION: The AWS Region where is is stored. (e.g. 'us-east-1')

// Plan/Apply:
// * TF_APPLY_ARGS:      The full set of arguments to use when planning or applying Terraform.

import groovy.json.JsonSlurper

node {
    stage "Setup"
    // Check out the project
    git credentialsId: GIT_CREDS_ID, url: GIT_URL

    // Reset the .terraform directory
    dir(path: "${workingDirectory}/${instanceSubDir}/.terraform") {
        deleteDir()
    }

    withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: TF_AWS_CREDS, usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY']]) {
        // Pull the remote config
        terraform "remote config ${tfRemoteArgs}"

        // Update any modules
        terraform "get -update=true"

        stage 'Plan'
        terraform "plan -input=false ${TF_APPLY_ARGS}"
        input 'Apply the plan?'

        stage 'Apply'
        terraform "apply -input=false ${TF_APPLY_ARGS}"
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
