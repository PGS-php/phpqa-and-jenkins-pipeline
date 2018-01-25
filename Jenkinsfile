String path = 'src/'

node('docker') {
    ansiColor('xterm') {

        stage('Checkout') {
            checkout scm
        }

        stage('Pulling') {
            sh 'docker pull jakzal/phpqa'
        }

        stage('Prepare directory') {
            sh 'rm -rf build/logs'
            sh 'mkdir -p build/logs'
        }

        stage('Install dependencies') {
            dockerRunTest('composer install --ignore-platform-reqs --no-scripts --no-progress --no-suggest')
        }

        stage("Testing") {
            parallel (
                "PHPCodeSniffer": {
                    dockerRunTest('phpcs --report=checkstyle --report-file=build/logs/checkstyle.xml --standard=PSR2 --encoding=UTF-8 --ignore="*.js" ' + path + ' || exit 0')
                    replaceFilePath('build/logs/checkstyle.xml')
                    checkstyle pattern: 'build/logs/checkstyle.xml'
                },

                "PHPStan": {
                    dockerRunTest('phpstan analyse ' + path + ' || exit 0')
                },

                "PhpMetrics": {
                    dockerRunTest('phpmetrics --report-html=build/logs/phpmetrics.html ' + path + ' || exit 0')
                    publishHTMLReport('build/logs', 'phpmetrics.html', 'PHPMetrics')
                },

                "PHPMessDetector": {
                    dockerRunTest('phpmd ' + path + ' xml cleancode,codesize,unusedcode --reportfile build/logs/pmd.xml || exit 0')
                    replaceFilePath('build/logs/pmd.xml')
                    pmd canRunOnFailed: true, pattern: 'build/logs/pmd.xml'
                },

                "PHPMagicNumberDetector": {
                    dockerRunTest('phpmnd ' + path + ' --exclude=tests --progress --non-zero-exit-on-violation --ignore-strings=return,condition,switch_case,default_parameter,operation || exit 0')
                },

                "PHPCopyPasteDetector": {
                    dockerRunTest('phpcpd --log-pmd build/logs/pmd-cpd.xml ' + path + ' || exit 0')
                    replaceFilePath('build/logs/pmd-cpd.xml')
                    dry canRunOnFailed: true, pattern: 'build/logs/pmd-cpd.xml'
                }
            )
        }
    }
}

def replaceFilePath(filePath) {
    sh "sed -i 's#/project/#${workspace}/#g' ${filePath}"
}

def publishHTMLReport(reportDir, file, reportName) {
    if (fileExists("${reportDir}/${file}")) {
        publishHTML(target: [
            allowMissing         : true,
            alwaysLinkToLastBuild: true,
            keepAll              : true,
            reportDir            : reportDir,
            reportFiles          : file,
            reportName           : reportName
        ])
    }
}

def dockerRunTest(attribute) {
    sh 'docker run --rm -v $(pwd):/project -w /project jakzal/phpqa ' + attribute
}

