node ('linuxbuildagent')
{
    //checkout([$class: 'GitSCM', branches: [[name: '*/master']], doGenerateSubmoduleConfigurations: false, extensions: [], submoduleCfg: [], userRemoteConfigs: [[url: 'https://github.com/hdharia/app-service-html-webapp.git']]])
    checkout scm
    
    stage('Build & Publish')
    {
        docker.withRegistry('https://hdacrmag.azurecr.us', 'docker-repo-cred-id') {
    
            def customImage = docker.build("app-service-html-webapp:${env.BUILD_ID}")
    
            /* Push the container to the custom Registry */
            customImage.push()
            customImage.push('latest')
        }
    }
}

node('master')
{
    stage('Start Dev VM')
    {
        withCredentials([azureServicePrincipal('mag-svp')]) {
            sh 'az cloud set --name AzureUSGovernment'   
            sh 'az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET -t $AZURE_TENANT_ID'
            
            sh 'az vm start -g docker_dev -n docker_devvm'
        }
    }
}
    
node('master')
{
    checkout scm
    
    stage('Dev Deploy to Dev VM')
    {
        sshagent(['ssh-credential']) {
            try
            {
                withCredentials([usernamePassword(credentialsId: 'acr_id', usernameVariable: 'USERNAME', passwordVariable: 'PASSWORD')]) {
                    sh "ssh -o StrictHostKeyChecking=no -l hadharia dockerdevhd.usgovvirginia.cloudapp.usgovcloudapi.net \"sudo docker login hdacrmag.azurecr.us -u ${USERNAME} -p ${PASSWORD}\""
                }
                sh 'ssh -o StrictHostKeyChecking=no -l hadharia dockerdevhd.usgovvirginia.cloudapp.usgovcloudapi.net \"sudo docker stop mywebapp && sudo docker rm mywebapp\"'
            }
            catch(error)
            {}
            finally
            {
                sh 'ssh -o StrictHostKeyChecking=no -l hadharia dockerdevhd.usgovvirginia.cloudapp.usgovcloudapi.net \"sudo docker run -d -p 80:80 --name mywebapp hdacrmag.azurecr.us/app-service-html-webapp:latest\"'
            }
        }
    }
}

input 'Do you approve the build for production'

node('master')
{
    stage('Deallocate Dev VM')
    {
        withCredentials([azureServicePrincipal('mag-svp')]) {
            sh 'az cloud set --name AzureUSGovernment'   
            sh 'az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET -t $AZURE_TENANT_ID'
            
            sh 'az vm deallocate -g docker_dev -n docker_devvm'
        }
    }
}

input 'Do you approve the build for baking'

node ('master')
{
    stage('Build Packer VM for Prod')
    {
        withCredentials([azureServicePrincipal('mag-svp'), usernamePassword(credentialsId: 'acr_id', passwordVariable: 'ACR_USER_PWD', usernameVariable: 'ACR_USER_ID'),
                sshUserPrivateKey(credentialsId: 'windows10-spro-key', keyFileVariable: 'SSH_PASS', passphraseVariable: '', usernameVariable: 'SSH_USER')]) {
                    sh "cd packer \
                        && \
                        packer build -var 'client_id=${AZURE_CLIENT_ID}' \
                                    -var 'client_secret=${AZURE_CLIENT_SECRET}' \
                                    -var 'tenant_id=${AZURE_TENANT_ID}' \
                                    -var 'subscription_id=${AZURE_SUBSCRIPTION_ID}' \
                                    -var 'acr_user_id=${ACR_USER_ID}' \
                                    -var 'password_acr=${ACR_USER_PWD}' \
                                    -var 'ssh_user=${SSH_USER}' \
                                    -var 'ssh_pass=${SSH_PASS}' \
                                    -var 'BUILD_ID=${BUILD_NUMBER}' \
                                    dockervm.json"
        }
    }
}

input 'Do you approve the build for production deployment'

node ('master')
{
    stage('Update Production VM Scale Set')
    {
        withCredentials([azureServicePrincipal('mag-svp')]) {
            sh 'az cloud set --name AzureUSGovernment'   
            sh 'az login --service-principal -u $AZURE_CLIENT_ID -p $AZURE_CLIENT_SECRET -t $AZURE_TENANT_ID'
            
            //get instance count for vmms
            
            def instances=sh (script: 'az vmss list-instances -g ubuntuvmms -n myScaleSet --output table --query "[].[instanceId]" | tail -c 2', returnStdout: true)
            echo "Total Instance Count: ${instances}"
            
            //Get managed disk id
            def RESOURCE_ID = sh (script: 'az image show --name dockervmdisk_${BUILD_NUMBER} -g myPackerRG --output tsv --query [id]', returnStdout: true)
            echo "Resource id ${RESOURCE_ID}"
            
            //Update vmss data disk
            azureVMSSUpdate azureCredentialsId: 'mag-svp', imageReference: [id: "${RESOURCE_ID}", offer: '', publisher: '', sku: '', version: ''], name: 'myScaleSet', resourceGroup: 'ubuntuvmms'
            
            //update instance
            def INSTANCECOUNT = new Integer(instances.trim()).intValue()
            
            for (int i = 0; i <= INSTANCECOUNT; i++) {
                echo "Updating Instance ${i}" 
                azureVMSSUpdateInstances azureCredentialsId: 'mag-svp', instanceIds: "${i}", name: 'myScaleSet', resourceGroup: 'ubuntuvmms'
            }
        }
    }
}
