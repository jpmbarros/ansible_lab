pipeline {
  agent { label 'ansible' }

  options {
    timestamps()
    ansiColor('xterm')
  }

  environment {
    // Adjust if your repo uses a different inventory path/name
    INVENTORY   = 'inventory/hosts.ini'
    PLAYBOOK    = 'site.yml'
    // Uncomment if your ansible.cfg lives in repo root and you want to force it:
    // ANSIBLE_CONFIG = "${WORKSPACE}/ansible.cfg"
  }

  stages {
    stage('Checkout') {
      steps {
        checkout([
          $class: 'GitSCM',
          branches: [[name: '*/main']],
          userRemoteConfigs: [[
            // Use SSH form if you added github-ssh:
            url: 'git@github.com:jpmbarros/ansible_lab.git',
            credentialsId: 'github-ssh'
          ]]
        ])
      }
    }

    stage('Sanity: versions') {
      steps {
        sh '''
          whoami
          ansible --version
          ansible-playbook --version
          git log -1 --oneline || true
        '''
      }
    }

    stage('Syntax check') {
      steps {
        sh '''
          ansible-playbook -i "${INVENTORY}" "${PLAYBOOK}" --syntax-check
        '''
      }
    }

    stage('Lint (optional)') {
      when { expression { return fileExists('ansible-lint.yaml') || fileExists('.ansible-lint') } }
      steps {
        sh 'ansible-lint -v || true'
      }
    }

    stage('Check mode (dry run)') {
      steps {
        // If you rely on Jenkins-provided SSH key for remote hosts, wrap with sshagent:
        // withCredentials([sshUserPrivateKey(credentialsId: 'ansible-ssh', keyFileVariable: 'SSH_KEY', usernameVariable: 'SSH_USER')]) {
        //   sh 'ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i "${INVENTORY}" "${PLAYBOOK}" --check -u "$SSH_USER" --private-key "$SSH_KEY"'
        // }

        // If the agent already has working SSH config/keys, just run:
        sh '''
          ANSIBLE_HOST_KEY_CHECKING=False \
          ansible-playbook -i "${INVENTORY}" "${PLAYBOOK}" --check
        '''
      }
    }

    stage('Deploy (apply changes)') {
      steps {
        // --- Enable this block if you use Vault from Jenkins credentials ---
        // withCredentials([string(credentialsId: 'ansible-vault-pass', variable: 'VAULT_PASS')]) {
        //   writeFile file: 'vault_pass.txt', text: VAULT_PASS
        //   sh '''
        //     ANSIBLE_HOST_KEY_CHECKING=False \
        //     ansible-playbook -i "${INVENTORY}" "${PLAYBOOK}" --vault-password-file vault_pass.txt
        //   '''
        // }

        // No vault (or the node already has your vault config/file):
        sh '''
          ANSIBLE_HOST_KEY_CHECKING=False \
          ansible-playbook -i "${INVENTORY}" "${PLAYBOOK}"
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'logs/**/*, reports/**/*', allowEmptyArchive: true
      junit testResults: 'reports/junit/*.xml', allowEmptyResults: true
    }
  }
}

