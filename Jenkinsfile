pipeline {
  agent any

  environment {
    TF_IN_AUTOMATION = 'true'
  }

  stages {
    stage('Firstly Terraform Init & Apply') {
      environment {
        DO_CLIENT_ID       = credentials('azure-client-id')
        DO_CLIENT_SECRET   = credentials('azure-client-secret')
        DO_TENANT_ID       = credentials('azure-tenant-id')
        DO_SUBSCRIPTION_ID = credentials('azure-subscription-id')
      }
      steps {
        dir('terraform') {
          sh '''
            terraform init
            terraform apply -auto-approve \
              -var="client_id=$DO_CLIENT_ID" \
              -var="client_secret=$DO_CLIENT_SECRET" \
              -var="tenant_id=$DO_TENANT_ID" \
              -var="subscription_id=$DO_SUBSCRIPTION_ID"
          '''
        }
      }
    }

    stage('getting a public ip address from Terraform') {
      steps {
        script {
          env.VM_PUBLIC_IP = sh(
            script: 'terraform -chdir=terraform output -raw vm_ip',
            returnStdout: true
          ).trim()
          echo "Azure VM IP: ${env.VM_PUBLIC_IP}"
        }
      }
    }

    stage('exceute Ansible Playbook') {
      steps {
        withCredentials([sshUserPrivateKey(credentialsId: 'ansible_ssh_key', keyFileVariable: 'SSH_KEY')]) {
          sh '''
            # Create dynamic inventory
            export ANSIBLE_HOST_KEY_CHECKING=False
            echo "[web]" > inventory.ini
            echo "$VM_PUBLIC_IP ansible_user=devops ansible_ssh_private_key_file=$SSH_KEY" >> inventory.ini

            # Run Ansible playbook
            ansible-playbook ansible/install_web.yml -i inventory.ini

            # Clean up
            rm -f inventory.ini
          '''
        }
      }
    }

    stage(' Now Verify the  Deployment') {
      steps {
        sh 'curl -s http://$VM_PUBLIC_IP | grep -i "devops project completed"'
      }
    }
  }

  
}
