pipeline() {
  agent any

  stages {
    stage('setup jetpack') {
        when {
            expression { TRIGGER_UNDERCLOUD_DEPLOYMENT == 'true' }
        }
        steps {
            sh 'rm -rf osp-api'
            sh 'git clone https://github.com/masco/osp-api.git'
            sh 'rm -rf jetpack'
            sh 'git clone https://github.com/redhat-performance/jetpack.git'
            sh 'ansible-playbook -vvv osp-api/jetpack_files/ansible/create_jetpack_dir_symlink.yml'
        }
    }
    
    stage('copy alias group vars file from osp-api to jetpack') {
        when {
            expression { ALIAS_LAB == 'true' }
        }
        steps {
            sh 'cp osp-api/jetpack_files/alias_group_vars.yml jetpack/group_vars/all.yml'
        }
    }
    
    stage('copy scale group vars file from osp-api to jetpack') {
        when {
            expression { SCALE_LAB == 'true' }
        }
        steps {
            sh 'cp osp-api/jetpack_files/scale_group_vars.yml jetpack/group_vars/all.yml'
        }
    }
    
    stage('create jetpack hosts file for undercloud') {
        steps {
            sh "cd osp-api/jetpack_files/ansible && ansible-playbook -vvv create_hosts_file.yml"
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
                    echo "Overcloud post deployment tasks failed on first try. Retrying upto 3 times."
                    retry(3) {
                        sh "cd jetpack && ansible-playbook -vvv main.yml -t overcloud"
                    }
                }
            }
        }
    }
    
    stage('install browbeat') {
        when {
            expression { TRIGGER_OVERCLOUD_DEPLOYMENT == 'true' }
        }
        steps {
            script {
                try {
                    sh "cd jetpack && ansible-playbook -i hosts -vvv main.yml -t browbeat"
                } catch(err) {
                    echo "Browbeat installation tasks failed on first try. Retrying upto 3 times."
                    retry(3) {
                        sh "cd jetpack && ansible-playbook -i hosts -vvv main.yml -t browbeat"
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
            sh "cd osp-api/browbeat_files/ansible && ansible-playbook -i ../../../jetpack/hosts -vvv setup_monitoring.yml"
        }
    }
    
    stage('run API testing workloads') {
        when {
            expression { RUN_API_WORKLOADS == 'true' }
        }
        steps {
            sh "cd osp-api/browbeat_files/ansible && ansible-playbook -i ../../../jetpack/hosts -t api_workloads -vvv run_workloads.yml"
        }
    }
    
    stage('run netcreate-boot workloads') {
        when {
            expression { RUN_NETCREATE_BOOT_WORKLOADS == 'true' }
        }
        steps {
            sh "cd osp-api/browbeat_files/ansible && ansible-playbook -i ../../../jetpack/hosts -t netcreate_boot_workloads -vvv run_workloads.yml"
        }
    }
    
    stage('run dynamic workloads') {
        when {
            expression { RUN_DYNAMIC_WORKLOADS == 'true' }
        }
        steps {
            sh "cd osp-api/browbeat_files/ansible && ansible-playbook -i ../../../jetpack/hosts -t dynamic_workloads -vvv run_workloads.yml"
        }
    }
    
    stage('stop collectd containers') {
        when {
            expression { STOP_COLLECTD == 'true' }
        }
        steps {
            sh "cd osp-api/browbeat_files/ansible && ansible-playbook -i ../../../jetpack/hosts -vvv stop_collectd.yml"
        }
    }
  }
}
