node ('linuxbuildagent')
{
    //checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/hdharia/app-service-html-webapp.git']]])
    checkout scm
    
    stage('Build & Publish')
    {
        docker.withRegistry('https://techsummithd.azurecr.io', 'docker-repo-cred-id') {
    
            def customImage = docker.build("app-service-html-webapp:${env.BUILD_ID}")
    
            /* Push the container to the custom Registry */
            customImage.push()
            customImage.push('latest')
        }
    }
}
    
node('master')
{
    stage('Dev Deploy to Dev VM')
    {
        sshagent(['ssh-credential']) {
            try
            {
                withCredentials([usernamePassword(credentialsId: 'acr_id', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                    sh "ssh -o StrictHostKeyChecking=no -l hadharia dockerdevhd.usgovvirginia.cloudapp.usgovcloudapi.net \"sudo docker login techsummithd.azurecr.io -u ${USERNAME} -p ${PASSWORD}\""
                }
                sh 'ssh -o StrictHostKeyChecking=no -l hadharia dockerdevhd.usgovvirginia.cloudapp.usgovcloudapi.net \"sudo docker stop mywebapp && sudo docker rm mywebapp\"'
            }
            catch(error)
            {}
            finally
            {
                sh 'ssh -o StrictHostKeyChecking=no -l hadharia dockerdevhd.usgovvirginia.cloudapp.usgovcloudapi.net \"sudo docker run -d -p 80:80 --name mywebapp techsummithd.azurecr.io/app-service-html-webapp:latest\"'
            }
        }
    }
}

input 'Do you approve the build for production'

node ('master')
{
    stage('Build Packer VM')
    {

    }
}

input 'Do you approve the build for production deployment'

node ('master')
{
    stage('Update Production VM Scale Set')
    {

    }
}
