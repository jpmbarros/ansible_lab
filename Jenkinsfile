pipeline {
  agent { label 'ansible-node' }

  options {
    skipDefaultCheckout(true)   // we do a single explicit checkout
    ansiColor('xterm')
    timestamps()
  }

  parameters {
    string(name: 'BRANCH',    defaultValue: 'main', description: 'Git branch to build')
    string(name: 'INVENTORY', defaultValue: 'inventory/hosts.ini', description: 'Path to inventory file')
    // your repo has playbooks/site.yml
    string(name: 'PLAYBOOK',  defaultValue: 'playbooks/site.yml', description: 'Playbook to run')
    booleanParam(name: 'DRY_RUN', defaultValue: true, description: 'Run Ansible in --check mode first')
  }

  environment {
    // export params into env vars so shell sees them
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

