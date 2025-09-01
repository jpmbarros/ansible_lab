pipeline {
  agent { label 'ansible-node' }

  // We’ll do a single, explicit checkout with HTTPS+PAT
  options {
    skipDefaultCheckout(true)
    ansiColor('xterm')
    timestamps()
  }

  parameters {
    string(name: 'BRANCH',    defaultValue: 'main', description: 'Git branch to build')
    string(name: 'INVENTORY', defaultValue: 'inventory/hosts.ini', description: 'Path to inventory file')
    string(name: 'PLAYBOOK',  defaultValue: 'site.yml', description: 'Playbook to run (e.g., site.yml or playbooks/site.yml)')
    booleanParam(name: 'DRY_RUN', defaultValue: true, description: 'Run Ansible in --check mode first')
  }

  environment {
    // Safer for CI
    ANSIBLE_NOCOWS = '1'
    ANSIBLE_HOST_KEY_CHECKING = 'False'
    PY_COLORS = '1'
    ANSIBLE_FORCE_COLOR = '1'
  }

  stages {
    stage('Checkout') {
      steps {
        git branch: params.BRANCH,
            url: 'https://github.com/jpmbarros/ansible_lab.git',
            credentialsId: 'github-https'
      }
    }

    stage('Sanity: versions') {
      steps {
        sh '''
          echo "== Versions =="
          git --version
          ansible --version || true
          ansible-playbook --version || true
        '''
      }
    }

    stage('Syntax check') {
      steps {
        sh '''
          echo "== Syntax check =="
          test -f "$PLAYBOOK" || { echo "Playbook $PLAYBOOK not found"; exit 2; }
          ansible-playbook -i "$INVENTORY" "$PLAYBOOK" --syntax-check
        '''
      }
    }

    stage('Lint (optional)') {
      steps {
        sh '''
          if command -v ansible-lint >/dev/null 2>&1; then
            echo "== ansible-lint =="
            ansible-lint "$PLAYBOOK" || true
          else
            echo "ansible-lint not installed; skipping"
          fi
        '''
      }
    }

    stage('Check mode (dry run)') {
      when { expression { params.DRY_RUN } }
      steps {
        // If your playbook connects to remote hosts over SSH with a key managed by Jenkins,
        // wrap with sshagent(credentials: ['ansible-agent']) { ... }
        sh '''
          echo "== Dry run =="
          ansible-playbook -i "$INVENTORY" "$PLAYBOOK" --check --diff | tee ansible_dryrun.log
        '''
      }
    }

    stage('Deploy (apply changes)') {
      when { expression { !params.DRY_RUN } }
      steps {
        // For SSH auth to targets via Jenkins credential, uncomment and set your credential ID:
        // sshagent(credentials: ['ansible-agent']) {
        //   sh 'ansible-playbook -i "$INVENTORY" "$PLAYBOOK" | tee ansible_apply.log'
        // }
        sh '''
          echo "== Apply changes =="
          ansible-playbook -i "$INVENTORY" "$PLAYBOOK" | tee ansible_apply.log
        '''
      }
    }
  }

  post {
    always {
      archiveArtifacts artifacts: 'ansible_*.log', allowEmptyArchive: true
      junit testResults: '**/junit-*.xml', allowEmptyResults: true
    }
  }
}

