String[] pathArray = ["src/"]

node('docker') {
    ansiColor('xterm') {

        stage('Checkout') {
            checkout scm
        }

        stage('Pulling') {
            sh 'docker pull jakzal/phpqa'
        }

        stage('Prepare directory') {
            phpQa('chmod -R 777 build')
            sh 'rm -rf build/logs'
            sh 'mkdir -p build/logs'
        }

        stage('Install dependencies') {
            phpQa('composer install --ignore-platform-reqs --no-scripts --no-progress --no-suggest')
        }

        stage("Testing") {
            parallel (
                "PHPCodeSniffer": {
                    pathArray.each { path ->
                        phpQa('phpcs --report=checkstyle --report-file=build/logs/checkstyle.xml --standard=PSR2 --encoding=UTF-8 --ignore="*.js" ' + path + ' || exit 0')
                    }
                    replaceFilePath('build/logs/checkstyle.xml')
                    checkstyle pattern: 'build/logs/checkstyle.xml'
                },

                "PHPStan": {
                    pathArray.each { path ->
                        phpQa('phpstan analyse ' + path + ' || exit 0')
                    }
                },

                "PhpMetrics": {
                    pathArray.each { path ->
                        phpQa('phpmetrics --report-html=build/logs/phpmetrics.html ' + path + ' || exit 0')
                    }
                    publishHTMLReport('build/logs', 'phpmetrics.html', 'PHPMetrics')
                },

                "PHPMessDetector": {
                    pathArray.each { path ->
                        phpQa('phpmd ' + path + ' xml cleancode,codesize,unusedcode --reportfile build/logs/pmd.xml || exit 0')
                    }
                    replaceFilePath('build/logs/pmd.xml')
                    pmd canRunOnFailed: true, pattern: 'build/logs/pmd.xml'
                },

                "PHPMagicNumberDetector": {
                    pathArray.each { path ->
                        phpQa('phpmnd ' + path + ' --exclude=tests --progress --non-zero-exit-on-violation --ignore-strings=return,condition,switch_case,default_parameter,operation || exit 0')
                    }
                },

                "PHPCopyPasteDetector": {
                    pathArray.each { path ->
                        phpQa('phpcpd --log-pmd build/logs/pmd-cpd.xml ' + path + ' || exit 0')
                    }
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

def phpQa(attribute) {
    sh 'docker run --rm -v $(pwd):/project -w /project jakzal/phpqa ' + attribute
}

