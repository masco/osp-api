def remote = [:]
remote.name = params.UNDERCLOUD_HOST
remote.host = params.UNDERCLOUD_HOST
remote.user = 'stack'
remote.password = <stack_password>
remote.allowAnyHosts = true
remote.keepAliveSec = 1800

pipeline() {
  agent any

  stages {
    stage('setup jetpack') {
        when {
            expression { TRIGGER_DEPLOYMENT == 'true' }
        }
        steps {
            sh 'rm -rf osp-api'
            sh 'git clone https://github.com/masco/osp-api.git'
            sh 'rm -rf jetpack'
            sh 'git clone https://github.com/redhat-performance/jetpack.git'
            sh 'cd jetpack && git pull origin pull/412/head'
            sh 'cp osp-api/jetpack_group_vars.yml jetpack/group_vars/all.yml'
        }
    }

    stage('deploy osp using jetpack') {
        when {
            expression { TRIGGER_DEPLOYMENT == 'true' }
        }
        steps {
            sh 'cd jetpack && ansible-playbook -vvv main.yml'
        }
    }
 
    stage('setup monitoring(collectd, graphite, grafana) using browbeat') {
        when {
            expression { SETUP_MONITORING == 'true' }
        }
        steps {
            sshCommand remote: remote, command: "rm -rf osp-api"
            sshCommand remote: remote, command: "git clone https://github.com/masco/osp-api.git"
            sshCommand remote: remote, command: "cp osp-api/browbeat_files/all.yml browbeat/ansible/install/group_vars/all.yml"
            sshCommand remote: remote, command: "sed -i 's/grafana_apikey_value/${params.GRAFANA_API_KEY}/' browbeat/ansible/install/group_vars/all.yml"
            sshCommand remote: remote, command: "cd browbeat/ansible && ansible-playbook -vv -i hosts.yml install/collectd.yml"
            sshCommand remote: remote, command: "cd browbeat/ansible && ansible-playbook -vv -i hosts.yml install/grafana-dashboards.yml"
        }
    }
    
    stage('run browbeat workloads') {
        when {
            expression { RUN_WORKLOADS == 'true' }
        }
        steps {
            sshCommand remote: remote, command: "rm -rf osp-api"
            sshCommand remote: remote, command: "git clone https://github.com/masco/osp-api.git"
            sshCommand remote: remote, command: "cp osp-api/browbeat_files/browbeat-config.yaml browbeat/browbeat-config.yaml"
            sshCommand remote: remote, command: "cd browbeat && source ~/stackrc && source .browbeat-venv/bin/activate && ./browbeat.py rally"
        }
    }
  }
}
