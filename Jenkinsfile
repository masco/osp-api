node('osp_api') {
  checkout scm

  stage('setup jetpack') {
    sh 'rm -rf jetpack'
    sh 'git clone https://github.com/redhat-performance/jetpack.git'
    sh 'cp jetpack/ci/osp_api.yml jetpack/group_vars/all.yml'
  }

  stage('deploy osp using jetpack') {
    sh 'cd jetpack && ansible-playbook -vvv main.yml'
  }
}
