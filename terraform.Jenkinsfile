// Reusable Jenkinsfile for Terraform projects. An Atlas replacement.
// 
// Vars:
// AWS: AWS_ACCESS_KEY, AWS_SECRET_KEY_ID,
// Git: GIT_CREDS_ID, GIT_URL, GIT_SUBDIR,

// Terraform:
// * TF_VERSION: The version number using tags in dockerhub for hashicorp/terraform. Uses 'latest' by default
// * TF_REMOTE_ARGS: The full set of arguments to use when configuring remote state for Terraform. Alternately...
// * TF_REMOTE_BACKEND: The backend to use, eg 's3'
// Terraform S3 Remote Config:
// * TF_REMOTE_S3_BUCKET: The S3 bucket name
// * TF_REMOTE_S3_KEY: The path in the bucket where the state is stored. (e.g. 'instance/cdn-rocket/terraform.tfstate')
// * TF_REMOTE_S3_REGION: The AWS Region where is is stored (e.g. 'us-east-1')
// Variables:
// * TF_VARS: JSON String with variables named with their value and key. E.g. "{ 'foo': 'bar' }"
// * TF_VAR_xxx: Defines an individual variable named 'xxx' (substitute with the desired variable name, case-sensitive).
//               May be more than one with different names. E.g. "TF_VAR_foo = 'bar'" will be passed in as '-var foo=bar'
import groovy.json.JsonSlurper

node {
    println "working directory: ${workingDirectory}"
    println "temp directory: ${tempDirectory}"

    stage "Check Out Project"

    git credentialsId: gitCredsId(), url: gitUrl()

    withCredentials([[$class: 'StringBinding', credentialsId: getAwsSecretKeyId(), variable: 'awsSecretKey']]) {

        stage 'Remote Config'
        tfRemoteConfig awsSecretKey: env.awsSecretKey

        stage 'Get Modules'
        // Make sure we have the latest version of any modules
        println "Removing any existing modules"
        dir(path: "${workingDirectory}/.terraform/modules") {
            deleteDir()
        }
        println "Getting the latest version of any requird modules"
        terraform "get"

        stage 'Plan Infrastructure'
        terraform "plan", awsSecretKey: env.awsSecretKey
        input 'Apply the plan?'

        stage 'Apply Infrastructure'
        terraform "apply", awsSecretKey: env.awsSecretKey
    }
}

def gitUrl(Map params = null) {
    params?.GIT_URL ?: "${GIT_URL}"
}

def gitCredsId(Map params = null) {
    params?.GIT_CREDS_ID ?: "${GIT_CREDS_ID}"
}

def tfRemoteConfig(Map params) {
    def run = getTerraformCmd params
    // Check if we're already working with a remote state. If not, pull the remote state.
    def remoteArgs = getTfRemoteArgs params

    withEnv(["AWS_ACCESS_KEY_ID=${getAwsSecretKey(params)}", "AWS_SECRET_ACCESS_KEY=${getAwsSecretKey(params)}"]) {
        sh "(head -n20 ${workingDirectory}/.terraform/terraform.tfstate 2>/dev/null | grep -q remote) || ${run} remote config ${remoteArgs}"
    }
}

def terraform(String tfArgs) {
    terraform(null, tfArgs)
}

def terraform(Map params, String tfArgs) {
    withEnv(["AWS_ACCESS_KEY_ID=${getAwsSecretKey(params)}", "AWS_SECRET_ACCESS_KEY=${getAwsSecretKey(params)}"]) {
        sh "${getTerraformCmd params} ${tfArgs} ${getTfVars params}"
    }
}

String getTerraformCmd(Map params = null) {
    "docker run --rm -v ${workingDirectory}:${tempDirectory} -w=${tempDirectory} hashicorp/terraform:${getTfVersion params}"
}

String getId() {
    "${env.JOB_NAME}-${env.BUILD_ID}"
}

String getAwsAccessKey(Map params = null) {
    params?.awsAccessKey ?: "${AWS_ACCESS_KEY}"
}

String getAwsSecretKeyId(Map params = null) {
    params?.awsSecretKeyId ?: "${AWS_SECRET_KEY_ID}"
}

String getAwsSecretKey(Map params = null) {
    params?.awsSecretKey ?: ""
}

String getTempDirectory() {
    "/tmp/${id}"
}

String getWorkingDirectory() {
    "${pwd()}/${GIT_SUBDIR}"
}

String getTfVersion(Map params = null) {
    params.TF_VERSION ?: TF_VERSION ?: "latest"
}

String getTfRemoteArgs(Map params = null) {
    def backend = params?.TF_REMOTE_BACKEND ?: TF_REMOTE_BACKEND
    if (backend == "s3") {
        "-backend=${TF_REMOTE_BACKEND} -backend-config='bucket=${TF_REMOTE_S3_BUCKET}' -backend-config='key=${TF_REMOTE_S3_KEY}' -backend-config='region=${TF_REMOTE_S3_REGION}'"
    } else {
        "${TF_REMOTE_ARGS ?: ''}"
    }
}

String getTfVars(Map params = null) {
    // Add values from JSON in 'TF_VARS'
    Map<String, Object> varMap = getTfVarsMap(params)

    // Pull in all environment variables prefixed with 'TF_VAR_'
    params?.each { String key, String value ->
        if (key.startsWith("TF_VAR_")) {
            varMap.put key.substring("TF_VAR_".length()), value
            vars += " -var ${key.substring("TF_VAR_".length())}=${value}"
        }
    }

    // Look up any credentials where the variable name ends with '*'
    varMap.each { String key, value ->
        if (key.endsWith('*')) {
            withCredentials([[$class: 'StringBinding', credentialsId: value, variable: 'cred']]) {
                varMap.put key.substring(0, key.length() - 1), params.cred
            }
        }
    }

    StringBuilder vars = new StringBuilder()
    varMap.each { key, value ->
        vars.append " -var ${key}=${value}"
    }
    return vars.toString()
}

Map<String, Object> getTfVarsMap(Map params = null) {
    // Default to include the AWS Access Key and Secret Key
    def result = [
            aws_access_key: getAwsAccessKey(params),
            aws_secret_key: getAwsSecretKey(params)
    ]
    // Slurp the JSON from TF_VARS
    def vars = params?.tfVars ?: TF_VARS
    if (vars instanceof String) {
        vars = new JsonSlurper().parseText(vars as String)
    }

    if (vars instanceof Map<String, Object>) {
        result.putAll(vars)
    } else if (vars != null) {
        throw new IllegalArgumentException("The TF_VARS environment variable or 'tfVars' parameter must be a String or a Map")
    }
    return result
}

