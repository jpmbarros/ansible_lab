pipeline {
  agent { label 'ansible-node' }

  options {
    skipDefaultCheckout(true)
    ansiColor('xterm')
    timestamps()
  }

  parameters {
    string(name: 'BRANCH',        defaultValue: 'main',                description: 'Git branch to build')
    string(name: 'INVENTORY',     defaultValue: 'inventory/hosts.ini', description: 'Path to inventory file')
    string(name: 'PLAYBOOK',      defaultValue: 'playbooks/site.yml',  description: 'Playbook to run (path or just filename)')
    booleanParam(name: 'DRY_RUN', defaultValue: true,                  description: 'Run Ansible in --check mode first')
  }

  environment {
    INVENTORY = "${params.INVENTORY}"
    PLAYBOOK  = "${params.PLAYBOOK}"
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

    stage('Syntax check') {
      steps {
        sh '''#!/bin/bash
          PB="$PLAYBOOK"
          if [ ! -f "$PB" ]; then
            base="$(basename "$PB")"
            [ -f "playbooks/$base" ] && PB="playbooks/$base"
          fi
          if [ ! -f "$PB" ]; then
            found="$(find . -maxdepth 3 -type f -name "$(basename "$PLAYBOOK")" -print -quit)"
            [ -n "$found" ] && PB="$found"
          fi
          echo "Playbook resolved to: $PB"
          test -f "$PB"        || { echo "Playbook not found."; exit 2; }
          test -f "$INVENTORY" || { echo "Inventory $INVENTORY not found"; exit 2; }
          ansible-playbook -i "$INVENTORY" "$PB" --syntax-check
          echo "$PB" > .resolved_playbook
        '''
      }
    }

    stage('Check mode (dry run)') {
      when { expression { return params.DRY_RUN } }
      steps {
        sshagent(credentials: [ "ansible-targets" ]) {
          sh '''#!/bin/bash
            PB="$(cat .resolved_playbook 2>/dev/null || echo "$PLAYBOOK")"
            echo "== Dry run =="

            # Pre-accept host keys from the actual inventory list
            ansible -i "$INVENTORY" all --list-hosts | tail -n +2 | \
              xargs -r -n1 -I{} ssh-keyscan -T 5 {} >> ~/.ssh/known_hosts || true

            ansible-playbook -i "$INVENTORY" "$PB" --check --diff | tee ansible_dryrun.log
          '''
        }
      }
    }

    stage('Deploy (apply changes)') {
      when { expression { return !params.DRY_RUN } }
      steps {
        sshagent(credentials: [ "ansible-targets" ]) {
          sh '''#!/bin/bash
            PB="$(cat .resolved_playbook 2>/dev/null || echo "$PLAYBOOK")"
            echo "== Apply changes =="
            ansible-playbook -i "$INVENTORY" "$PB" | tee ansible_apply.log
          '''
        }
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

