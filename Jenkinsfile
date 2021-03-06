// Constants
//Where to upload the artifacts afterwards
RELEASE_URL = ""
//Small python script that increments the build number.
MY_SQNZ_URL = "http://:8081/"
// This is to ensure that a developer does not push to the wrong ready branch namespace.
TARGET_BRANCH_NAME = "master"
MERGE_BRANCH_PREFIX = "ready"
GIT_REPO= "git@github.com:sofusalbertsen/jenkins_pipeline_workspace.git"
SSH_AGENT_ID="sofusalbertsen"
// State
inputSHA = "" // Why can't Jenkins tell me this?!
authorEmail = "none"
authorName = "none"
authorFull = "none <none>"
releaseVersion = ""

// Utils
//Does the branch namespace contain "ready", and should therefore be regarded as an integration branch.
def shouldMerge() {
    return "${BRANCH_NAME}" ==~ /^${MERGE_BRANCH_PREFIX}\/.*/
}

def pipelineSuccess() {
    node("master") {
        emailext (
          subject: "Success: ${BRANCH_NAME} (${releaseVersion})",
          body: """
Status: Succeeded
Branch: ${BRANCH_NAME}
Version: ${releaseVersion}
Jenkins URL: ${env.BUILD_URL}
Release URL: ${RELEASE_URL}/${releaseVersion}
""",
          to: "${authorEmail}"
        )
    }
}

def pipelineFailure(stage, exception) {
    node("master") {
        emailext (
          subject: "Failure: ${BRANCH_NAME} (${releaseVersion}): ${stage}",
          body: """
Status: Failed at stage "${stage}" with "${exception.toString()}"
Branch: ${BRANCH_NAME}
Version: ${releaseVersion}
Jenkins URL: ${env.BUILD_URL}
""",
          to: "${authorEmail}"
        )
    }
}
// Workspace must be cleaned from last run in order to make stash and git clone work.
def cleanNode(name, closure) {
    node(name) {
        deleteDir()
        try {
            closure.call()
        } finally {
            deleteDir()
        }
    }
}
//If we want to send an email when things go wrong, we need to wrap it in a try/catch

def stageWithGuard(name, closure) {
    try {
        stage(name, closure)
    } catch(e) {
        pipelineFailure(name, e)
        throw e
    }
}
//Locking resources in order to for example make sure that only one branch is tried integrated at a time.
def conditionalLock(condition, name, closure) {
    if (condition) {
        println "Running under lock: ${name}"
        lock(name, closure)
    } else {
        println "Running without lock"
        closure.call()
    }
}



// Stages

println "Parameters: branch: ${BRANCH_NAME}, target: ${TARGET_BRANCH_NAME}, merge: ${shouldMerge()}, release URL: ${RELEASE_URL}"

conditionalLock(shouldMerge(), "pipeline-merge-lock") {
    // The following stages are locked to ensure that only one pipeline runs at
    // at a time. The stages in here must contain everything between and
    // including the original git checkout and the final git push of the merged
    // branch.

    stageWithGuard("Checkout") {
        cleanNode("master") {
            buildNumber = currentBuild.number;
            //buildNumber = sh(script: "curl -sd '${TARGET_BRANCH_NAME}' ${MY_SQNZ_URL}", returnStdout: true).trim()
            currentBuild.displayName = "${currentBuild.displayName} (${buildNumber})"
            sshagent (credentials: ["${SSH_AGENT_ID}"]) {
                timeout(1) {
                    sh "git clone --no-checkout ${GIT_REPO} ."
                }

                inputSHA = sh(script: "git rev-parse origin/${BRANCH_NAME}", returnStdout: true).trim()
                authorName = sh(script: "git log -1 --format='%an' ${inputSHA}", returnStdout: true).trim()
                authorEmail = sh(script: "git log -1 --format='%ae' ${inputSHA}", returnStdout: true).trim()
                authorFull = "${authorName} <${authorEmail}>"

                timeout(2) {
                    if (shouldMerge()) {
                        sh """
git config user.email "${authorEmail}"
git config user.name "${authorName}"
git checkout "${TARGET_BRANCH_NAME}"

if [ "\$(git branch --contains ${inputSHA} | wc -l)" -gt "0" ]
then
    echo "MERGE ERROR: origin/${BRANCH_NAME} already present in origin/${TARGET_BRANCH_NAME}"
    exit 1
fi

COMMITS="\$(git log --oneline ${TARGET_BRANCH_NAME}..${inputSHA} | wc -l)"
if [ "\${COMMITS}" -gt "1" ] || ! git merge --ff-only ${inputSHA}
then
    git reset --hard origin/${TARGET_BRANCH_NAME}
    git merge --no-ff --no-commit ${inputSHA}
    git commit --author "${authorFull}" --message "Merge branch '${BRANCH_NAME}' into '${TARGET_BRANCH_NAME}'"
fi

git submodule update --init --recursive
"""
                    } else {
                        sh "git checkout ${BRANCH_NAME} && git submodule update --init --recursive"
                    }
                }

                sha = sh(script: "git rev-parse HEAD", returnStdout: true).trim()
                shaish = sh(script: "git rev-parse --short=5 HEAD", returnStdout: true).trim()


            }
            //Stash the whole repository
            stash name: "repo", includes: "**", useDefaultExcludes:false
        }
    }

    stageWithGuard("Build") {
        def builders = [
            "linux": {
                cleanNode("linux") {
                    unstash "repo"
                    sh "ls"
                    sh "./build.sh"
                    stash name: "output", includes: "linux/**"
                }
            }
            // ,
            // "windows x86": {
            //     cleanNode("windows") {
            //         unstash "repo"
            //         bat "build.bat"
            //         stash name: "windows-suite-32", includes: "windows/**"
            //     }
            // }
        ]

        parallel builders
    }


    stageWithGuard("Push") {
        if (shouldMerge()) {
            cleanNode("master") {
                unstash "repo"

                sshagent (credentials: ["${SSH_AGENT_ID}"]) {
                    timeout(1) {
                        sh """
git push origin ${TARGET_BRANCH_NAME}
git fetch
if [ "\$(git rev-parse origin/${BRANCH_NAME})" = "${inputSHA}" ]
then
	git push origin :${BRANCH_NAME}
fi
"""
                    }
                }
            }
        }
    }
}

stageWithGuard("Document") {
    cleanNode("linux") {
        unstash "repo"
        unstash "output"
        sh "./build_documentation.sh"
        stash name: "documentation", includes: "documentation/**"
    }
}



stageWithGuard("Package") {
    def builders = [
        "linux": {
            cleanNode("linux") {
                unstash "repo"
                unstash "documentation"
                sh "./package_driver_unix.sh"
                //stash name: "linux", includes: "nt*.src.rpm"
            }
        }
        // ,
        // "windows": {
        //     cleanNode("windows") {
        //         unstash "repo"
        //         unstash "windows-suite-32"
        //         unstash "documentation"
        //         bat "jenkins/package_suite_windows.bat"
        //         stash name: "packages-windows", includes: "nt_suite_3gd_windows_${VERSION_NUMBER_FULL}.exe"
        //     }
        // }
    ]

    parallel builders
}


def verifyLinux(_node, _script) {
    return {
        cleanNode(_node) {
            unstash "repo"
            unstash "packages-linux-64"
            wrap([$class: "AnsiColorBuildWrapper", "colorMapName": "XTerm"]) {
                sh _script
            }
        }
    }
}

stageWithGuard("Verify") {
    def builders = [
        //"smoketest": verifyLinux("smoketest&&linux", "sudo -n jenkins/smoketest.sh"),
        //"centos5":   verifyLinux("centos5", "jenkins/verify_dyld.sh"),
        //"centos6":   verifyLinux("centos6", "jenkins/verify_dyld.sh"),
        "linux":   verifyLinux("linux", "smoketest.sh")
    ]

    parallel builders
}



stageWithGuard("Upload") {
    cleanNode("master") {
        unstash "repo"
        unstash "output"
        dir("upload-workspace") {
          //  unstash "linux"

        }
        // sshagent (credentials: ["4058c10a-772b-4d0a-9f8b-cb611ca66ac6"]) {
        //     sh "jenkins/upload_release.sh"
        // }
      //  println "Artifact uploaded to: ${RELEASE_URL}/${VERSION_NUMBER_FULL}"
    }
}



stageWithGuard("Dance") {
    pipelineSuccess()
    echo '''
└─(･◡･)─┐

└─(･◡･)─┘

┌─(･◡･)─┘

┌─(･◡･)─┐

└─(･◡･)─┐

└─(･◡･)─┘
'''
}
