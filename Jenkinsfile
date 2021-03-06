echo "Running Build ID: ${env.BUILD_ID}"

String commit_id
String build_args
String deployLogin
String docker_img_name
String portApplication
def docker_img

node {

    deleteDir()

    stage("parameters") {
        //
        // Parameters passed through from the Jenkins Pipeline configuration
        //
        string(name: 'githubUrl',
               description: 'GitHub URL for checking out project',
               defaultValue: 'https://github.com/robe16/jarvis.config_server.git')
        string(name: 'appName',
               description: 'Name of application for Docker image and container',
               defaultValue: 'jarvis.config_server')
        string(name: 'deploymentServer',
               description: 'Server to deploy the Docker container',
               defaultValue: '*')
        string(name: 'deploymentUsername',
               description: 'Username for the server the Docker container will be deployed to (used for ssh/scp)',
               defaultValue: '*')
        string(name: 'portMapped',
               description: 'Port number to map internal docker port used by application to',
               defaultValue: '*')
        string(name: 'folderConfig',
               description: 'Location of config directory on host device',
               defaultValue: '~/config/jarvis/config_server/')
        string(name: 'folderLog',
               description: 'Location of log directory on host device',
               defaultValue: '~/logs/jarvis/config_server/')
        //
        //
        portApplication = "1600"
        //
        //
        build_args = ["--build-arg portApplication=${portApplication}"].join(" ")
        //
        //
        docker_volumes = ["-v ${params.folderConfig}:/jarvis.config_server/config/config_files/",
                          "-v ${params.folderLog}:/jarvis.config_server/log/logfiles/"].join(" ")
        //
        //
        deployLogin = "${params.deploymentUsername}@${params.deploymentServer}"
        //
    }

    if (params["deploymentServer"]!="*" && params["deploymentUsername"]!="*" && params["portMapped"]!="*") {

        stage("checkout") {
            git url: "${params.githubUrl}"
            sh "git rev-parse HEAD > .git/commit-id"
            commit_id = readFile('.git/commit-id').trim()
            echo "Git commit ID: ${commit_id}"
        }

        docker_img_name_commit = "${params.appName}:${commit_id}"
        docker_img_name_latest = "${params.appName}:latest"

        stage("build") {
            try {sh "docker image rm ${docker_img_name_latest}"} catch (error) {}
            sh "docker build -t ${docker_img_name_commit} ${build_args} ."
            sh "docker tag ${docker_img_name_commit} ${docker_img_name_latest}"
        }

        stage("deploy"){
            //
            String docker_img_tar = "docker_img.tar"
            //
            try {
                sh "rm ~/${docker_img_tar}"                                                                 // remove any old tar files from cicd server
            } catch(error) {
                echo "No ${docker_img_tar} file to remove."
            }
            sh "docker save -o ~/${docker_img_tar} ${docker_img_name_commit}"                               // create tar file of image
            sh "scp -v -o StrictHostKeyChecking=no ~/${docker_img_tar} ${deployLogin}:~"                    // xfer tar to deploy server
            sh "ssh -o StrictHostKeyChecking=no ${deployLogin} \"docker load -i ~/${docker_img_tar}\""      // load tar into deploy server registry
            sh "ssh -o StrictHostKeyChecking=no ${deployLogin} \"rm ~/${docker_img_tar}\""                  // remove the tar file from deploy server
            sh "rm ~/${docker_img_tar}"                                                                     // remove the tar file from cicd server
            // Set 'latest' tag to most recently created docker image
            sh "ssh -o StrictHostKeyChecking=no ${deployLogin} \"docker tag ${docker_img_name_commit} ${docker_img_name_latest}\""
            //
        }

        stage("start container"){
            // Stop existing container if running
            sh "ssh ${deployLogin} \"docker rm -f ${params.appName} && echo \"container ${params.appName} removed\" || echo \"container ${params.appName} does not exist\"\""
            // Start new container
            sh "ssh ${deployLogin} \"docker run --restart unless-stopped -d ${docker_volumes} -p ${params.portMapped}:${portApplication} --name ${params.appName} ${docker_img_name_latest}\""
        }

    } else {
        error("Build cancelled as required parameter values not provided by pipeline configuration")
    }

}