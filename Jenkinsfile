pipeline {
    agent any

    environment {
        PKG_NAME    = 'bitchx'
        PKG_VERSION = '1.3'
        PKG_ARCH    = sh(script: 'dpkg --print-architecture', returnStdout: true).trim()
        DEPLOY_HOST = '192.168.122.101'
        DEPLOY_PATH = '/var/www/repository'
        DEPLOY_USER = 'jenkins'
        DEB_FILE    = "${PKG_NAME}_${PKG_VERSION}_${PKG_ARCH}.deb"
        PKG_DIR     = "${PKG_NAME}_${PKG_VERSION}_${PKG_ARCH}"
    }

    stages {

        stage('Checkout') {
            steps {
                git url: 'git@github.com:blueSherii/BitchX.git',
                    branch: 'master',
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
                sh '''
                    chmod +x autogen.sh install-sh mkinstalldirs
                    NOCONFIGURE=1 ./autogen.sh
                    ./configure \
                        --prefix=/usr \
                        --libdir=/usr/lib/bx \
                        --disable-ipv6 \
                        --disable-plugins
                '''
            }
        }

        stage('Build') {
            steps {
                sh 'make -j$(nproc)'
            }
        }

        stage('Package .deb') {
            steps {
                sh '''
                    # Create debian package directory structure
                    rm -rf "${PKG_DIR}"
                    mkdir -p "${PKG_DIR}/DEBIAN"
                    mkdir -p "${PKG_DIR}/usr/bin"
                    mkdir -p "${PKG_DIR}/usr/lib/bx/help"
                    mkdir -p "${PKG_DIR}/usr/lib/bx/script"
                    mkdir -p "${PKG_DIR}/usr/share/doc/bitchx"
                    mkdir -p "${PKG_DIR}/usr/share/man/man1"

                    # Binary
                    install -m 755 source/BitchX "${PKG_DIR}/usr/bin/BitchX"

                    # Support files
                    install -m 644 BitchX.help     "${PKG_DIR}/usr/lib/bx/"
                    install -m 644 BitchX.ircnames  "${PKG_DIR}/usr/lib/bx/"
                    install -m 644 BitchX.quit      "${PKG_DIR}/usr/lib/bx/"
                    install -m 644 BitchX.reasons   "${PKG_DIR}/usr/lib/bx/"
                    cp -r script/*                  "${PKG_DIR}/usr/lib/bx/script/" 2>/dev/null || true
                    cp -r bitchx-docs/*             "${PKG_DIR}/usr/lib/bx/help/"   2>/dev/null || true

                    # Docs
                    install -m 644 README.md  "${PKG_DIR}/usr/share/doc/bitchx/"
                    install -m 644 COPYRIGHT  "${PKG_DIR}/usr/share/doc/bitchx/"
                    install -m 644 Changelog  "${PKG_DIR}/usr/share/doc/bitchx/"

                    # Calculate installed size (in KB)
                    INSTALLED_SIZE=$(du -sk "${PKG_DIR}" | cut -f1)

                    # DEBIAN/control
                    cat > "${PKG_DIR}/DEBIAN/control" <<EOF
Package: ${PKG_NAME}
Version: ${PKG_VERSION}-${BUILD_NUMBER}
Architecture: ${PKG_ARCH}
Maintainer: blueSherii <bluesherii@github.com>
Installed-Size: ${INSTALLED_SIZE}
Depends: libncurses6, libssl3, libcrypt1
Section: net
Priority: optional
Homepage: https://github.com/blueSherii/BitchX
Description: BitchX IRC client
 BitchX is a feature-rich IRC client based on ircII.
 Built from source with GCC 14 compatibility fixes.
EOF

                    # DEBIAN/conffiles (mark support files as config)
                    cat > "${PKG_DIR}/DEBIAN/conffiles" <<EOF
/usr/lib/bx/BitchX.help
/usr/lib/bx/BitchX.ircnames
/usr/lib/bx/BitchX.quit
/usr/lib/bx/BitchX.reasons
EOF

                    # postinst: create symlink /usr/bin/bitchx -> BitchX
                    cat > "${PKG_DIR}/DEBIAN/postinst" <<'EOF'
#!/bin/sh
set -e
if [ ! -L /usr/bin/bitchx ]; then
    ln -s /usr/bin/BitchX /usr/bin/bitchx
fi
EOF
                    chmod 755 "${PKG_DIR}/DEBIAN/postinst"

                    # prerm: remove symlink
                    cat > "${PKG_DIR}/DEBIAN/prerm" <<'EOF'
#!/bin/sh
set -e
rm -f /usr/bin/bitchx
EOF
                    chmod 755 "${PKG_DIR}/DEBIAN/prerm"

                    # Build the .deb
                    dpkg-deb --build --root-owner-group "${PKG_DIR}" "${DEB_FILE}"
                    dpkg-deb --info  "${DEB_FILE}"
                    echo "Package built: ${DEB_FILE}"
                '''
            }
        }

        stage('Deploy via SFTP') {
            steps {
                sshagent(credentials: ['deploy-ssh-key']) {
                    sh '''
                        # Ensure destination directory exists
                        ssh -o StrictHostKeyChecking=no \
                            -o ConnectTimeout=10 \
                            ${DEPLOY_USER}@${DEPLOY_HOST} \
                            "mkdir -p ${DEPLOY_PATH}"

                        # Transfer the .deb
                        scp -o StrictHostKeyChecking=no \
                            "${DEB_FILE}" \
                            ${DEPLOY_USER}@${DEPLOY_HOST}:${DEPLOY_PATH}/

                        # Update the apt repository index if reprepro or dpkg-scan is available
                        ssh -o StrictHostKeyChecking=no \
                            ${DEPLOY_USER}@${DEPLOY_HOST} \
                            "cd ${DEPLOY_PATH} && \
                             if command -v dpkg-scanpackages >/dev/null 2>&1; then \
                                 dpkg-scanpackages . /dev/null | gzip -9c > Packages.gz; \
                                 echo 'Repository index updated'; \
                             fi"

                        echo "Deployed ${DEB_FILE} to ${DEPLOY_HOST}:${DEPLOY_PATH}"
                    '''
                }
            }
        }
    }

    post {
        success {
            archiveArtifacts artifacts: '*.deb', fingerprint: true
            echo "Build #${BUILD_NUMBER} succeeded. Package: ${DEB_FILE}"
        }
        failure {
            echo "Build #${BUILD_NUMBER} failed."
        }
        always {
            cleanWs()
        }
    }
}
