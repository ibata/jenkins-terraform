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
    stage "Check Out Project"

    git credentialsId: gitCredsId, url: gitUrl

    stage 'Remote Config'
    terraform "version"

    tfRemoteConfig()

    stage 'Get Modules'
    // Make sure we have the latest version of any modules
    println "Removing any existing modules"
    dir(path: "${workingDirectory}/${instanceSubDir}.terraform/modules") {
        deleteDir()
    }
    println "Getting the latest version of any required modules"
    println "PWD: ${pwd()}"
    terraform "get -update=true"

    stage 'Plan Infrastructure'
    println "PWD: ${pwd()}"
    terraform "plan -input=false ${tfVarsDirect}"
    input 'Apply the plan?'

    stage 'Apply Infrastructure'
    println "PWD: ${pwd()}"
    terraform "apply -input=false ${tfVarsDirect}"
}

def getGitUrl() {
    "${GIT_URL}"
}

def getGitCredsId() {
    "${GIT_CREDS_ID}"
}

def tfRemoteConfig() {
    withEnv(["AWS_ACCESS_KEY_ID=${awsAccessKey}", "AWS_SECRET_ACCESS_KEY=${awsSecretKey}"]) {
        sh "(head -n20 ${workingDirectory}/${instanceSubDir}/.terraform/terraform.tfstate 2>/dev/null | grep -q remote) || ${terraformCmd} remote config ${tfRemoteArgs}"
    }
}

def terraform(String tfArgs) {
    withEnv(["AWS_ACCESS_KEY_ID=${awsAccessKey}", "AWS_SECRET_ACCESS_KEY=${awsSecretKey}"]) {
        sh "${terraformCmd} ${tfArgs}"
    }
}

String getTerraformCmd() {
    "docker run --rm -v ${workingDirectory}:${tempDirectory} -w=${tempDirectory}/${instanceSubDir} -e AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY hashicorp/terraform:${tfVersion}"
}

String getId() {
    "${env.JOB_NAME}-${env.BUILD_ID}"
}

String getAwsAccessKey() {
    "${AWS_ACCESS_KEY}"
}

String getAwsSecretKey(Map params = null) {
    def value = ""
    if (params?.awsSecretKey) {
        value = params.awsSecretKey
    } else {
        if (AWS_SECRET_KEY_ID) {
            withCredentials([[$class: 'StringBinding', credentialsId: AWS_SECRET_KEY_ID, variable: 'awsSecretKey']]) {
                value = "${env.awsSecretKey}"
            }
        }
    }
    value
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

String getTfVarsDirect() {
    "${TF_VARS_DIRECT ?: ''}"
}

String getTfVars() {
    // Add values from JSON in 'TF_VARS'
    Map<String, Object> varsMap = getTfVarsMap()
    Map<String, Object> resolved = [:]

    // Look up any credentials where the variable name ends with '*'
    varsMap.each { String key, value ->
        if (value.startsWith('$')) {
            // variable values matching '$...' are property lookups
            def propertyName = value.substring(1)
            if (propertyName.startsWith('*')) {
                // property lookups starting with '*' are credential lookups
                propertyName = propertyName.substring(1)
                println "Looking up '${propertyName}' credential for -var '${key}'"
                withCredentials([[$class: 'StringBinding', credentialsId: propertyName, variable: 'cred']]) {
                    // Only update it if the credential can be found.
                    if (env.cred != null)
                        value = env.cred
                }
            } else {
                println "Looking up '${propertyName}' property for -var '${key}'"
                def property = getProperty(propertyName)
                // Only update it if the property exists.
                if (property != null)
                    value = property
            }
        }
        resolved.put key, value
    }

    StringBuilder varString = new StringBuilder()
    resolved.each { key, value ->
        varString.append " -var '${key}=${value}'"
    }
    return varString.toString()
}

Map<String, Object> getTfVarsMap() {
    // Slurp the JSON from TF_VARS
    def vars = TF_VARS as String
    if (vars && vars.trim().length() > 0) {
        vars = new JsonSlurper().parseText(vars as String)
    }
    vars
}

