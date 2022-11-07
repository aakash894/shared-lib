def skipRemainingStages = false
pipeline {
    agent any
    parameters {
        choice choices: ['Apply', 'Destroy'], name: 'Infrastruture'
        choice choices: ['us-west-2a', 'us-west-2b', 'us-west-2c', 'us-west-2d'], description: 'Select the availability zone in which you want to create your subnet', name: 'AZ1'
        choice choices: ['us-west-2a', 'us-west-2b', 'us-west-2c', 'us-west-2d'], description: 'Select the availability zone for higher availibity of your resources', name: 'AZ2'
        string defaultValue: 'ami-017fecd1353bcc96e', description: 'Input amazon image id', name: 'AMI'
        string defaultValue: 't2.medium', description: 'Input instance type', name: 'instance_type'
        string defaultValue: 'oregon', description: 'Input key pair name which you want to provide to your machine & ensure it will be pre-generated', name: 'key_name'
        string defaultValue: '3', description: 'Input node count for your database cluster', name: 'node_count'
        string defaultValue: 'ubuntu', description: 'Specify the user of your ec2 instances [ubuntu for ubuntu and ec2-user for redhat]', name: 'VM_USER', trim: true
        choice choices: ['apt', 'yum'], description: 'Select package manager i.e. in case of ubuntu -> apt and in redhat -> yum', name: 'Package_manager'
        string defaultValue: 'oregon.pem', description: 'Specify the key pair name of your VM in .pem ext format [for e.g abc.pem]', name: 'Key_pair_name', trim: true
        choice choices: ['4.0.7', '3.11.14', '3.0.28'], description: 'Select your cassandra database version', name: 'version'
    }
    stages {
        stage('terraform format check') {
            steps{
                sh 'terraform fmt'
            }
        }
        stage('terraform Init') {
            steps{
                sh 'terraform init'
            }
        }
        stage('terraform destroy'){
            when {
                expression { params.Infrastruture == 'Destroy' }
            }
            steps {
                script {
                    skipRemainingStages = true
                    input('Do you really want to destroy Infrastructure?')
                    sh 'terraform destroy --auto-approve'
                }
            }    
        }
        stage('terraform apply'){
            when {
                expression { params.Infrastruture == 'Apply' }
            }
            steps {
                script{
                    input('Do you want to create Infrastructure?')
                    sh 'terraform apply --auto-approve -var="AZ1=${AZ1}" -var="AZ2=${AZ2}" -var="ami=${AMI}" -var="instance_type=${instance_type}" -var="key_name=${key_name}" -var="node_count=${node_count}"'
                }
            }
        }
        stage('terraform output'){
            when {
                expression {
                    !skipRemainingStages
                }
            }
            steps {
                sh'''
                chmod +x file.sh
                ./file.sh $VM_USER /home/$VM_USER/$Key_pair_name
                '''
            }
        }
        stage('Copy data'){
            when {
                expression {
                    !skipRemainingStages
                }
            }
            steps {
                sh'''
                IP=$(terraform output -json Bastion-publicIP | jq -r)
                echo $IP
                ssh -i "~/$Key_pair_name" -o StrictHostKeyChecking=no -tt $VM_USER@$IP "rm -rf Invnetory $Key_pair_name"
                scp -i "~/$Key_pair_name" -o StrictHostKeyChecking=no -r Invnetory $VM_USER@$IP:~
                scp -i "~/$Key_pair_name" -o StrictHostKeyChecking=no -r ~/$Key_pair_name $VM_USER@$IP:~
                '''
            }
        }
        stage('Configure ansible'){
            when {
                expression {
                    !skipRemainingStages
                }
            }
            steps {
                sh'''
                IP=$(terraform output -json Bastion-publicIP | jq -r)
                echo $IP
                ssh -i "~/$Key_pair_name" -o StrictHostKeyChecking=no -tt $VM_USER@$IP "sudo $Package_manager update -y && sudo $Package_manager install git -y && sudo $Package_manager install ansible -y"
                '''
            }
        }
        stage('Git clone'){
            when {
                expression {
                    !skipRemainingStages
                }
            }
            steps {
                sh'''
                IP=$(terraform output -json Bastion-publicIP | jq -r)
                echo $IP
                ssh -i "~/$Key_pair_name" -o StrictHostKeyChecking=no -tt $VM_USER@$IP "rm -rf cassandra"
                ssh -i "~/$Key_pair_name" -o StrictHostKeyChecking=no -tt $VM_USER@$IP "git clone https://github.com/aakash894/cassandra.git"
                '''
            }
        }
        stage('Cluster setup'){
            when {
                expression {
                    !skipRemainingStages
                }
            }
            steps {
                sh'''
                IP=$(terraform output -json Bastion-publicIP | jq -r)
                echo $IP
                ssh -i "~/$Key_pair_name" -o StrictHostKeyChecking=no -tt $VM_USER@$IP "ansible-playbook -e version=${version} -i Invnetory ~/cassandra/Cassandra_review/test/cassandra.yml"
                '''
            }
        }
    }    
}
