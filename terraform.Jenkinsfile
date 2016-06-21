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
    def gitCreds = gitCredsId(env)
    git credentialsId: gitCreds, url: gitUrl(env)

    withCredentials([[$class: 'StringBinding', credentialsId: getAwsSecretKeyId(env), variable: 'awsSecretKey']]) {

        stage 'Remote Config'
        tfRemoteConfig env

        stage 'Get'
        // Make sure we have the latest version of any modules
        sh "rm -R ${workingDirectory}/.terraform/modules"
        terraform "get"

        stage 'Plan'
        terraform "plan"
        input 'Apply the plan?'

        stage 'Apply'
        terraform "apply"
    }
}

def gitUrl(Map env = null) {
    "${env.GIT_URL}"
}

def gitCredsId(Map env = null) {
    "${env.GIT_CREDS_ID}"
}

def tfRemoteConfig(Map env) {
    def run = getTerraformCmd env
    // Check if we're already working with a remote state. If not, pull the remote state.
    def remoteArgs = getTfRemoteArgs env

    sh "(head -n20 ${getWorkingDirectory env}/.terraform/terraform.tfstate 2>/dev/null | grep -q remote) || ${run} remote config ${remoteArgs}"
}

def terraform(String tfArgs) {
    terraform(null, tfArgs)
}

def terraform(Map params, String tfArgs) {
    sh "${getTerraformCmd(params)} ${tfArgs} ${getTfVars params}"
}

String getTerraformCmd(Map params = null) {
    "docker run --rm -v ${getWorkingDirectory params}:${getTempDirectory params} -w=${getTempDirectory params} -e AWS_ACCESS_KEY_ID=${getAwsAccessKey params} -e AWS_SECRET_ACCESS_KEY=${getAwsSecretKey(params)} hashicorp/terraform:${getTfVersion params}"
}

String getId(Map env = null) {
    "${env.JOB_NAME}-${env.BUILD_ID}"
}

String getAwsAccessKey(Map env = null) {
    "${env.AWS_ACCESS_KEY}"
}

String getAwsSecretKeyId(Map env = null) {
    "${env.AWS_SECRET_KEY_ID}"
}

String getAwsSecretKey(Map env = null) {
    "${env.AWS_SECRET_KEY}"
}

String getTempDirectory(Map env = null) {
    "/tmp/${getId env}"
}

String getWorkingDirectory(Map env = null) {
    "${pwd()}/${env.GIT_SUBDIR}"
}

String getTfVersion(Map env = null) {
    env.TF_VERSION ?: "latest"
}

String getTfRemoteArgs(Map env = null) {
    if (env.TF_REMOTE_BACKEND == "s3") {
        "-backend=${env.TF_REMOTE_BACKEND} -backend-config='bucket=${env.TF_REMOTE_S3_BUCKET}' -backend-config='key=${env.TF_REMOTE_S3_KEY}' -backend-config='${env.TF_REMOTE_S3_REGION}"
    } else {
        "${env.TF_REMOTE_ARGS}"
    }
}

String getTfVars(Map env = null) {
    // Add values from JSON in 'TF_VARS'
    Map<String, Object> varMap = getTfVarsMap(env)

    // Pull in all environment variables prefixed with 'TF_VAR_'
    env?.each { String key, String value ->
        if (key.startsWith("TF_VAR_")) {
            varMap.put key.substring("TF_VAR_".length()), value
            vars += " -var ${key.substring("TF_VAR_".length())}=${value}"
        }
    }

    // Look up any credentials where the variable name ends with '*'
    varMap.each { String key, value ->
        if (key.endsWith('*')) {
            withCredentials([[$class: 'StringBinding', credentialsId: value, variable: 'cred']]) {
                varMap.put key.substring(0, key.length()-1), env.cred
            }
        }
    }

    StringBuilder vars = new StringBuilder()
    varMap.each { key, value ->
        vars << " -var ${key}=${value}"
    }
    return vars.toString()
}

Map<String, Object> getTfVarsMap(Map env = null) {
    // Default to include the AWS Access Key and Secret Key
    def result = [
            aws_access_key: getAwsAccessKey(env),
            aws_secret_key: getAwsSecretKey(env)
    ]
    // Slurp the JSON from TF_VARS
    def vars = env?.tfVars ?: env?.TF_VARS
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

