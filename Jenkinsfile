node {

    git url:'git@github.com:ipcrm/flask_puppet.git', branch: 'master'

    def hostaddress = 'jenkins.demo.lan'
    puppet.credentials 'pe-access-token'

    stage 'Install Dev Tools'
    sh '''
        PATH=$WORKSPACE/venv_python:bin:/usr/local/bin:$PATH
        test -d "venv_python" || virtualenv venv_python
        . venv_python/bin/activate
        pip install -r requirements.txt --download-cache=/tmp/$JOB_NAME
        pip install discover
    '''

    stage 'Unit Testing'
    sh '''
        . venv_python/bin/activate
        python ./test.py -v
    '''

    stage 'Build sdist archive'
    sh '''
        . venv_python/bin/activate
        python ./setup.py sdist
    '''

    $pkgversion = sh(returnStdout: true, script: '''
      python ./setup.py --version
    ''')

    archive "dist/flask_puppet-${pkgversion}.tar.gz"

    step([$class: 'CopyArtifact', filter: "flask_puppet-${version}.tar.gz", fingerprintArtifacts: true, projectName: env.JOB_NAME, selector: [$class: 'SpecificBuildSelector', buildNumber: env.BUILD_ID], target: '/var/www/html/builds/flask_puppet'])

    stage 'Deploy to Dev'
    puppet.hiera scope: 'flask_puppet_dev', key: 'dist_file', value: "http://" + hostaddress + "/builds/flask_puppet/flask_puppet-${pkgversion}.tar.gz"
    puppet.job 'production', query: 'nodes { facts { name = "role" and value = "flask_puppet" } and facts { name = "appenv" and value = "dev"}}'

    stage 'Dev Acceptance Test(s)'
    devnodes = puppet.query 'nodes { facts { name = "role" and value = "flask_puppet" } and facts { name = "appenv" and value = "dev"}}'
    for (Map node : devnodes) {
      sh "curl -o /dev/null --silent --head --write-out '%{http_code}\n' http://${devnode.certname}/|grep 200 &> /dev/null"
    }

    stage 'Simulate Deploy to Prod'
    puppet.hiera scope: 'flask_puppet_prod', key: 'dist_file', value: "http://" + hostaddress + "/builds/flask_puppet/flask_puppet-${pkgversion}.tar.gz"
    puppet.job 'production', noop: true, query: 'nodes { facts { name = "role" and value = "flask_puppet" } and facts { name = "appenv" and value = "prod"}}'


    stage 'Deploy to Prod'
    input "Ready to Deploy to production?"
    puppet.hiera scope: 'flask_puppet_prod', key: 'dist_file', value: "http://" + hostaddress + "/builds/flask_puppet/flask_puppet-${pkgversion}.tar.gz"
    puppet.job 'production', query: 'nodes { facts { name = "role" and value = "flask_puppet" } and facts { name = "appenv" and value = "prod"}}'

    stage 'Prod Acceptance Test(s)'
    prodnodes = puppet.query 'nodes { facts { name = "role" and value = "flask_puppet" } and facts { name = "appenv" and value = "prod"}}'
    for (Map node : prodnodes) {
      sh "curl -o /dev/null --silent --head --write-out '%{http_code}\n' http://${devnode.certname}/|grep 200 &> /dev/null"
    }

}
