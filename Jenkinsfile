#!/usr/bin/env groovy
@Library('cookbookLibrary@master') _

def build_node = 'docker'
def version_info = [:]
def notifyJob = false
def slackChannel = 'build'

def upload_file(String target, String dest, String creds) {
    repo_host = env.REPO_HOST ?: 'repo.ijetonboard.com'
    dest_base = '/opt/yum/repo'
    withCredentials([file(credentialsId: creds, variable: 'KEY_FILE')]) {
        sh "scp -o 'UserKnownHostsFile /dev/null' -o 'StrictHostKeyChecking no' -i ${env.KEY_FILE} ${target} yum@${repo_host}:${dest_base}/${dest}"
    }
}

def determine_version(String branch) {
    version_base = readFile('VERSION').trim()
    version_map = [:]
    switch(branch) {
        case 'ijet-integration':
            version_map.version = "${version_base}.${env.BUILD_NUMBER}"
            version_map.dest = 'dep'
            break
        default:
            branch_name = branch.replaceAll(/[^a-zA-Z0-9\.]/, '')
            version_map.version = "${version_base}.${env.BUILD_NUMBER}.dev0+${branch_name}"
            version_map.dest = 'dev'
            break
    }
    println("Building version ${version_map.version}")
    return version_map
}

def build_env(Map version_map, String project) {
    build_url = env.BUILD_URL ?: 'Null'
    writeFile(file: "${project}_build_url.info", text: build_url)
    git_commit = sh(returnStdout: true, script: 'git rev-parse HEAD')
    writeFile(file: "${project}_git_commit.info", text: git_commit)
    writeFile(file: 'VERSION', text: version_map['version'])
    writeFile(file: "${project}_git_url.info", text: "git@github.com:ijettech-dev/${project}.git")

    docker.withRegistry('https://docker.ijetonboard.com/v2', 'dockerhub-password') {
        docker.image('centos7-py36-build').inside() {
            sh "python3.6 -m virtualenv buildenv"
            sh """source buildenv/bin/activate
                python setup.py bdist_wheel"""
        }
    }
}

node(build_node) {
    try {
        stage('Preperation') {
            step([$class: 'WsCleanup'])
            wrap([$class: 'BuildUser']) {
                builder = env.BUILD_USER == null ? 'Jenkins' : env.BUILD_USER
                notifySlack('warning', "${env.JOB_NAME} #${currentBuild.id} started by ${builder}", notifyJob, slackChannel)
            }

            checkout scm
            // Determine destination repo based on branch?
            version_info = determine_version(env.BRANCH_NAME)
        }
        stage('Build') {
            build_env(version_info, 'pyorient')
        }
        stage('Upload') {
            upload_file('dist/*.whl', "${version_info.dest}/python/pyorient", 'yumrepo')
            notifySlack('good', "${env.JOB_NAME} #${currentBuild.id} succeeded", notifyJob, slackChannel)
        }
    }
    catch (err) {
        notifySlack('danger', "${env.JOB_NAME} #${currentBuild.id} failed", notifyJob, slackChannel)
        throw err
    }
    finally {
        stage('Cleanup') {
            step([$class: 'WsCleanup'])
        }
    }
}
