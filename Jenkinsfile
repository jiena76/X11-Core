// name = sh(returnStdout:true, script: 'curl https://api.github.com/users/\$\{PULLMAKER\} | grep "name" | awk \'{print $2, $3 }\'')
// name = sh 'curl https://api.github.com/users/${PULLMAKER} | egrep "name" | awk \'{print $2, $3 }\''
// returns "First Last",
// remove " and ",
// name = name.substring(1, name.length() - 2)

def WindDown(errorname){
//         def ver_script = $/eval """curl https://api.github.com/users/${PULLMAKER} | grep 'name' | awk '{print $2, $3 }'"""/$
        name = sh(returnStdout:true, script: """
        "curl https://api.github.com/users/${PULLMAKER}""" | """grep 'name'""" | """awk '{print \$2, \$3 }'" """)
        // returns "First Last",
        // remove " and ",
        name = name.substring(1, name.length() - 2)

        SendToPi("docker stop rov")
        SendToPi("docker rm rov")
        msg = """
@${name} Pull Request #${PULLNUM}, on branch ${PULLBRANCH} Failed!
Find the logs here: http://aberdeen.purdueieee.org:1944/
        """
        slackSend(color: "#FF0000",message: msg)

        sendStatus("failure","http://aberdeen.purdueieee.org:1944/",errorname,"continuous-integration/aberdeen")
        error(errorname)
}

def SendToPi(cmd){
        try{
                sh """ssh pi@128.46.156.193 \'${cmd}\'"""
        }catch(error){
                WindDown("Sending command to pi did not work, asshole")
        }
}
def SaveLog(filename){
        try{
                sh "mv ${filename} ${env.logsite}/PR#${PULLNUM}"
        }catch(error){
                WindDown("Could Not find log")
        }
}

def sendStatus(state,target_url,description,context){
        sh """
curl --header "Content-Type: application/json" \
--request POST \
--data '{"state":"\'${state}\'","target_url":"\'${target_url}\'","description":"\'${description}\'","context": "\'${context}\'"}' \
https://api.github.com/repos/purduerov/X11-Core/statuses/\'${COMITSHA}\'?access_token=\'${env.gittoken}\'

        """
}

node {
        // name = sh(returnStdout:true, script: 'curl https://api.github.com/users/\$\{PULLMAKER\} | grep "name" | awk \'{print $2, $3 }\'')
        // name = sh 'curl https://api.github.com/users/${PULLMAKER} | egrep "name" | awk \'{print $2, $3 }\''
        // returns "First Last",
        // remove " and ",
        // name = name.substring(1, name.length() - 2)
        def app
        stage ('setupenv'){
                sh "mkdir -p ${env.logsite}/PR#${PULLNUM}"
                sh "date > meta.log"
                SaveLog("meta.log")
                withPythonEnv('/usr/bin/python'){
                        pysh 'pip install pylint'
                        sh 'rm -rf socketIO-client'
                        sh 'git clone https://github.com/AnotherOctopus/socketIO-client'
                        pysh 'pip install ./socketIO-client/'
                }
        }
        stage ('build') {
                try{
                        checkout scm
                        sh 'echo ${PULLBRANCH}'
                        sh 'git checkout ${PULLBRANCH}'
                        sh 'git pull'
                }catch(error){
                        msg = "Hum, we failed checking out the repo. Idk man"
                        // slackSend(color: "#FF0000",message: msg)
                        WindDown("SOURCE FAILED")
                }
                try{
                        app = docker.build("anotheroctopus/rovimage")
                }catch(error){
                        msg = "So the docker image didn't build, so its either Scotty's fault or the Dockerfile"
                        // slackSend(color: "#FF0000",message: msg)
                        WindDown("BUILD FAILED")
                }
                try{
                        sh "cd surface/ && npm install"
                }catch(error){
                        msg = "Hum, we failed building frontend. IAAAAAAAANNANAN"
                        // slackSend(color: "#FF0000",message: msg)
                        WindDown("Frontend  FAILED")
                }
        }
        stage ('launchROV'){
                try{
                        sh """docker login -u ${env.dhublogin} -p ${env.dhubpass}"""
                        tag = "${PULLBRANCH}"
                        app.push(tag)
                        SendToPi("docker  run -d --name=\"rov\" anotheroctopus/rovimage:'${PULLBRANCH}'")
                }catch(error){
                        msg = "Launching the ROV failed. Probably some networking nonesense"
                        // slackSend(color: "#FF0000",message: msg)
                        WindDown("ROV FAILED TO LAUNCH")
                }
        }
        stage('lint'){
                linterrmsg = ""

                // Lint Python
                withPythonEnv('/usr/bin/python'){
                        try{
                                pysh(returnStdout:true, script: 'pylint --rcfile=pylintrc.conf surface/pakfront/CV/ > pylint.log').trim()
                                pysh(returnStdout:true, script: 'pylint --rcfile=pylintrc.conf rov/ >> pylint.log').trim()
                        }catch(error){
                                linterrmsg +="@${name} Linting Python Files on PR#${PULLNUM} Failed!\n"
                        }
                }

                // Lint Esx
                try{
                        sh(returnStdout:true, script: 'cd surface && eslint -c "../eslintrc.js" . > ../eslint.log').trim()
                }catch(error){
                        linterrmsg +="@${name} Linting JSX Files on PR#${PULLNUM} Failed!\n"
                }

                //Lint Go
                try{
                        golint = sh(returnStdout:true, script: 'find . -iname "*.go" | xargs gofmt -d > golint.log').trim()
                }catch(error){
                        linterrmsg +="@${name} Linting go Files on PR#${PULLNUM} Failed!\n"
                }

                SaveLog("pylint.log")
                SaveLog("eslint.log")
                SaveLog("golint.log")

                if(linterrmsg != ""){
                        slackSend(color: "#FF0000",message: linterrmsg)
                        WindDown("LINTERROR")
                }
        }
        stage ('test'){
                withPythonEnv('/usr/bin/python'){
                        pysh 'python tests/testem.py > runlog.log 2>&1'
                }
                SaveLog("runlog.log")
        }
        stage ('post'){
                msg = """
@${name} Pull Request #${PULLNUM}, on branch ${PULLBRANCH} Passed all Tests!\n
Check out the logs here http://aberdeen.purdueieee.org:1944/
Hey ${PULLMAKER}, you should bug ${REVIEWERS} to approve your pull
"""
                // slackSend(color: "#00FF00",message:  msg)

                sendStatus("success","http://aberdeen.purdueieee.org:1944/","Everything Passed!","continuous-integration/aberdeen")

                SendToPi("docker stop rov")
                SendToPi("docker rm rov")
        }
}
