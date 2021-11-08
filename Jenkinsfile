def remote = [:]
remote.name = params.UNDERCLOUD_HOST
remote.host = params.UNDERCLOUD_HOST
remote.user = 'stack'
remote.password = <stack_password>
remote.allowAnyHosts = true
remote.keepAliveSec = 21600

pipeline() {
  agent any

  stages {
    stage('setup jetpack') {
        when {
            expression { TRIGGER_DEPLOYMENT == 'true' || TRIGGER_UNDERCLOUD_DEPLOYMENT == 'true' }
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
    
    stage('deploy undercloud using jetpack') {
        when {
            expression { TRIGGER_UNDERCLOUD_DEPLOYMENT == 'true' }
        }
        steps {
            script {
                try {
                    sh "cd jetpack && ansible-playbook -vvv main.yml -t undercloud"
                } catch(err) {
                    echo "Undercloud deployment failed on first try. Retrying upto 3 times."
                    retry(3) {
                        sh "cd jetpack && ansible-playbook -vvv main.yml -t undercloud"
                    }
                }
            }
        }
    }
    
    stage('introspect nodes') {
        when {
            expression { TRIGGER_INTROSPECTION == 'true' }
        }
        steps {
            script {
                try {
                    sh "cd jetpack && ansible-playbook -vvv main.yml -t introspect"
                } catch(err) {
                    echo "Introspection failed on first try. Retrying upto 3 times."
                    retry(3) {
                        sh "cd jetpack && ansible-playbook -vvv main.yml -t introspect"
                    }
                }
            }
        }
    }
    
    stage('tag nodes') {
        when {
            expression { TRIGGER_TAGGING == 'true' }
        }
        steps {
            script {
                try {
                    sh "cd jetpack && ansible-playbook -vvv main.yml -t tag"
                } catch(err) {
                    echo "Tagging failed on first try. Retrying upto 3 times."
                    retry(3) {
                        sh "cd jetpack && ansible-playbook -vvv main.yml -t tag"
                    }
                }
            }
        }
    }
    
    stage('deploy overcloud using jetpack') {
        when {
            expression { TRIGGER_OVERCLOUD_DEPLOYMENT == 'true' }
        }
        steps {
            script {
                try {
                    sh "cd jetpack && ansible-playbook -vvv main.yml -t overcloud"
                } catch(err) {
                    echo "Overcloud deployment failed on first try. Retrying upto 3 times."
                    retry(3) {
                        sh "cd jetpack && ansible-playbook -vvv main.yml -t overcloud"
                    }
                }
            }
        }
    }
    
    stage('delete existing overcloud deployment') {
        when {
            expression { TRIGGER_OVERCLOUD_REDEPLOYMENT == 'true' }
        }
        steps {
            script {
                try {
                    sshCommand remote: remote, command: "source ~/stackrc && openstack overcloud delete overcloud -y"
                } catch(err) {
                    echo err.getMessage()
                }
            }
        }
    }
    
    stage('redeploy overcloud') {
        when {
            expression { TRIGGER_OVERCLOUD_REDEPLOYMENT == 'true' }
        }
        steps {
            script {
                try {
                    sshCommand remote: remote, command: "source ~/stackrc && ./overcloud_deploy.sh"
                } catch(err) {
                    echo "Overcloud redeployment failed on first try. Retrying up to 3 times."
                    retry(3) {
                        try {
                            sshCommand remote: remote, command: "source ~/stackrc && openstack overcloud delete overcloud -y"
                        } catch(error) {
                            echo error.getMessage()
                        }
                        sshCommand remote: remote, command: "source ~/stackrc && ./overcloud_deploy.sh"
                    }
                }
            }
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
            sshCommand remote: remote, command: "cd browbeat/ansible && (nohup ansible-playbook -vv -i hosts.yml install/collectd.yml > collectd_install.out 2>&1 &)"
        }
    }
    
    stage('run API testing workloads') {
        when {
            expression { RUN_API_WORKLOADS == 'true' }
        }
        steps {
            sshCommand remote: remote, command: "rm -rf osp-api"
            sshCommand remote: remote, command: "git clone https://github.com/masco/osp-api.git"
            sshCommand remote: remote, command: "cp osp-api/browbeat_files/api_testing_config.yaml browbeat/browbeat-config.yaml"
            sshCommand remote: remote, command: "cd browbeat && source ~/stackrc && source .browbeat-venv/bin/activate && (nohup ./browbeat.py rally > api_workloads.out 2>&1 &) && (tail -f 202*.log | grep -q output.json)"
        }
    }
    
    stage('run netcreate-boot workloads') {
        when {
            expression { RUN_NETCREATE_BOOT_WORKLOADS == 'true' }
        }
        steps {
            sshCommand remote: remote, command: "rm -rf osp-api"
            sshCommand remote: remote, command: "git clone https://github.com/masco/osp-api.git"
            sshCommand remote: remote, command: "cp osp-api/browbeat_files/netcreate_boot_config.yaml browbeat/browbeat-config.yaml"
            sshCommand remote: remote, command: "ansible-playbook -vvv osp-api/browbeat_files/ansible/fill_ext_net_id.yml"
            sshCommand remote: remote, command: "cd browbeat && source ~/stackrc && source .browbeat-venv/bin/activate && ./browbeat.py rally"
        }
    }
    
    stage('run dynamic workloads') {
        when {
            expression { RUN_DYNAMIC_WORKLOADS == 'true' }
        }
        steps {
            sshCommand remote: remote, command: "rm -rf osp-api"
            sshCommand remote: remote, command: "git clone https://github.com/masco/osp-api.git"
            sshCommand remote: remote, command: "cp osp-api/browbeat_files/dynamic_workloads_config.yaml browbeat/browbeat-config.yaml"
            sshCommand remote: remote, command: "ansible-playbook -vvv osp-api/browbeat_files/ansible/fill_ext_net_id.yml"
            sshCommand remote: remote, command: "cd browbeat && source ~/stackrc && source .browbeat-venv/bin/activate && ./browbeat.py rally"
        }
    }
  }
}
