pipeline {
    agent any

    parameters {
        string(
            name: 'BRANCH',
            defaultValue: 'master',
            description: 'Git branch to build'
        )
        string(
            name: 'PKG_VERSION',
            defaultValue: '1.3',
            description: 'Package version (e.g. 1.3, 1.3.1)'
        )
        string(
            name: 'DEPLOY_HOST',
            defaultValue: '192.168.122.101',
            description: 'Target server IP or hostname for deployment'
        )
        string(
            name: 'DEPLOY_PATH',
            defaultValue: '/var/www/repository',
            description: 'Remote path to place the .deb file'
        )
        string(
            name: 'DEPLOY_USER',
            defaultValue: 'jenkins',
            description: 'SSH user on the deployment server'
        )
        string(
            name: 'CONFIGURE_OPTS',
            defaultValue: '--disable-ipv6 --disable-plugins',
            description: 'Extra options passed to ./configure'
        )
        booleanParam(
            name: 'SKIP_DEPLOY',
            defaultValue: false,
            description: 'Build and package only — skip SFTP deployment'
        )
        booleanParam(
            name: 'UPDATE_REPO_INDEX',
            defaultValue: true,
            description: 'Run dpkg-scanpackages on the remote host after upload'
        )
    }

    environment {
        PKG_NAME = 'bitchx'
        PKG_ARCH = sh(script: 'dpkg --print-architecture', returnStdout: true).trim()
        DEB_FILE = "${PKG_NAME}_${params.PKG_VERSION}-${BUILD_NUMBER}_${PKG_ARCH}.deb"
        PKG_DIR  = "${PKG_NAME}_${params.PKG_VERSION}_${PKG_ARCH}"
    }

    stages {

        stage('Checkout') {
            steps {
                echo "Checking out branch: ${params.BRANCH}"
                git url: 'git@github.com:blueSherii/BitchX.git',
                    branch: "${params.BRANCH}",
                    credentialsId: 'github-ssh-key'
            }
        }

        stage('Install Build Dependencies') {
            steps {
                sh '''
                    apt-get update -qq
                    apt-get install -y --no-install-recommends \
                        gcc make autoconf \
                        libncurses-dev \
                        libssl-dev \
                        libxcrypt-dev \
                        dpkg-dev
                '''
            }
        }

        stage('Configure') {
            steps {
                sh """
                    chmod +x autogen.sh install-sh mkinstalldirs
                    NOCONFIGURE=1 ./autogen.sh
                    ./configure \
                        --prefix=/usr \
                        --libdir=/usr/lib/bx \
                        ${params.CONFIGURE_OPTS}
                """
            }
        }

        stage('Build') {
            steps {
                sh 'make -j$(nproc)'
            }
        }

        stage('Package .deb') {
            steps {
                sh """
                    rm -rf "\${PKG_DIR}"
                    mkdir -p "\${PKG_DIR}/DEBIAN"
                    mkdir -p "\${PKG_DIR}/usr/bin"
                    mkdir -p "\${PKG_DIR}/usr/lib/bx/help"
                    mkdir -p "\${PKG_DIR}/usr/lib/bx/script"
                    mkdir -p "\${PKG_DIR}/usr/share/doc/bitchx"

                    # Binary
                    install -m 755 source/BitchX "\${PKG_DIR}/usr/bin/BitchX"

                    # Support files
                    install -m 644 BitchX.help    "\${PKG_DIR}/usr/lib/bx/"
                    install -m 644 BitchX.ircnames "\${PKG_DIR}/usr/lib/bx/"
                    install -m 644 BitchX.quit     "\${PKG_DIR}/usr/lib/bx/"
                    install -m 644 BitchX.reasons  "\${PKG_DIR}/usr/lib/bx/"
                    cp -r script/*      "\${PKG_DIR}/usr/lib/bx/script/" 2>/dev/null || true
                    cp -r bitchx-docs/* "\${PKG_DIR}/usr/lib/bx/help/"   2>/dev/null || true

                    # Docs
                    install -m 644 README.md "\${PKG_DIR}/usr/share/doc/bitchx/"
                    install -m 644 COPYRIGHT "\${PKG_DIR}/usr/share/doc/bitchx/"
                    install -m 644 Changelog "\${PKG_DIR}/usr/share/doc/bitchx/"

                    INSTALLED_SIZE=\$(du -sk "\${PKG_DIR}" | cut -f1)

                    cat > "\${PKG_DIR}/DEBIAN/control" <<EOF
Package: ${PKG_NAME}
Version: ${params.PKG_VERSION}-${BUILD_NUMBER}
Architecture: \${PKG_ARCH}
Maintainer: blueSherii <bluesherii@github.com>
Installed-Size: \${INSTALLED_SIZE}
Depends: libncurses6, libssl3, libcrypt1
Section: net
Priority: optional
Homepage: https://github.com/blueSherii/BitchX
Description: BitchX IRC client (branch: ${params.BRANCH})
 BitchX is a feature-rich IRC client based on ircII.
 Built from source with GCC 14 compatibility fixes.
EOF

                    cat > "\${PKG_DIR}/DEBIAN/conffiles" <<EOF
/usr/lib/bx/BitchX.help
/usr/lib/bx/BitchX.ircnames
/usr/lib/bx/BitchX.quit
/usr/lib/bx/BitchX.reasons
EOF

                    cat > "\${PKG_DIR}/DEBIAN/postinst" <<'SCRIPT'
#!/bin/sh
set -e
if [ ! -L /usr/bin/bitchx ]; then
    ln -s /usr/bin/BitchX /usr/bin/bitchx
fi
SCRIPT
                    chmod 755 "\${PKG_DIR}/DEBIAN/postinst"

                    cat > "\${PKG_DIR}/DEBIAN/prerm" <<'SCRIPT'
#!/bin/sh
set -e
rm -f /usr/bin/bitchx
SCRIPT
                    chmod 755 "\${PKG_DIR}/DEBIAN/prerm"

                    dpkg-deb --build --root-owner-group "\${PKG_DIR}" "\${DEB_FILE}"
                    dpkg-deb --info "\${DEB_FILE}"
                    echo "Package built: \${DEB_FILE}"
                """
            }
        }

        stage('Deploy via SFTP') {
            when {
                expression { return !params.SKIP_DEPLOY }
            }
            steps {
                echo "Deploying ${DEB_FILE} to ${params.DEPLOY_USER}@${params.DEPLOY_HOST}:${params.DEPLOY_PATH}"
                sshagent(credentials: ['deploy-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no \
                            -o ConnectTimeout=10 \
                            ${params.DEPLOY_USER}@${params.DEPLOY_HOST} \
                            "mkdir -p ${params.DEPLOY_PATH}"

                        scp -o StrictHostKeyChecking=no \
                            "\${DEB_FILE}" \
                            ${params.DEPLOY_USER}@${params.DEPLOY_HOST}:${params.DEPLOY_PATH}/

                        if [ "${params.UPDATE_REPO_INDEX}" = "true" ]; then
                            ssh -o StrictHostKeyChecking=no \
                                ${params.DEPLOY_USER}@${params.DEPLOY_HOST} \
                                "cd ${params.DEPLOY_PATH} && \
                                 dpkg-scanpackages . /dev/null | gzip -9c > Packages.gz && \
                                 echo 'Repository index updated'"
                        fi

                        echo "Deployed \${DEB_FILE} to ${params.DEPLOY_HOST}:${params.DEPLOY_PATH}"
                    """
                }
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: '*.deb', fingerprint: true
            echo "Build #${BUILD_NUMBER} succeeded — ${DEB_FILE}"
        }
        failure {
            echo "Build #${BUILD_NUMBER} failed."
        }
        always {
            cleanWs()
        }
    }
}
