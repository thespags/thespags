https://jfrog.com/blog/secure-access-development-jfrog-github-oidc/
```
name: "Setup Artifactory"
description: |
  This action combines getting an artifactory token with setting up artifactory docker hub, cli, netrc and go proxy.
  If you need additional access, the token is available in the output.
  The action exports `GOPROXY` and `GONOSUMDB` to work with our internal repositories.
outputs:
  access_token:
    description: "The access token for Artifactory authentication."
    value: ${{ steps.auth_artifactory.outputs.access_token }}
  artifactory_successful:
    description: "If we were able to authenticate with Artifactory."
    value: ${{ steps.check_artifactory.outputs.value }}
runs:
  using: "composite"
  steps:
    - shell: bash
      id: check_artifactory
      run: |
        echo "Setup env"
        echo "JF_HOST=artifactory.ops.lacework.engineering" >> $GITHUB_ENV
        echo "JF_USER=service-account-releng" >> $GITHUB_ENV
        ${GITHUB_ACTION_PATH}/check_artifactory.sh

    - name: Auth Artifactory
      if: ${{ steps.check_artifactory.outputs.value == 'true' }}
      id: dynamic_auth
      uses: lacework-dev/actions/dynamic-uses-v1@main
      env:
        action_ref: ${{ github.action_ref || env.action_ref || 'main' }}
      with:
        uses: lacework-dev/actions/auth-artifactory-v1@${{ github.repository == 'lacework-dev/actions' && github.sha || env.action_ref }}
        with: |
          {
            "jf_host": "${{ env.JF_HOST }}",
            "jf_user": "${{ env.JF_USER }}"
          }

    - name: Login to Artifactory
      if: ${{ steps.check_artifactory.outputs.value == 'true' }}
      id: auth_artifactory
      shell: bash
      run: |
          echo "access_token=${{ fromJSON(steps.dynamic_auth.outputs.outputs).artifactory_access_token }}" >> $GITHUB_OUTPUT

    - name: Login to Artifactory
      if: ${{ steps.check_artifactory.outputs.value == 'true' }}
      uses: docker/login-action@v3
      with:
        registry: ${{ env.JF_HOST }}
        username: ${{ env.JF_USER }}
        password: ${{ steps.auth_artifactory.outputs.access_token }}

    - name: Setup JFrog CLI
      if: ${{ steps.check_artifactory.outputs.value == 'true' }}
      uses: jfrog/setup-jfrog-cli@v4
      env:
        JF_ACCESS_TOKEN: ${{ steps.auth_artifactory.outputs.access_token }}
        JF_URL: https://artifactory.ops.lacework.engineering

    - shell: bash
      if: ${{ steps.check_artifactory.outputs.value == 'true' }}
      run: |
        echo "Setup netrc"
        set -x
        {
          echo "machine"
          echo ${JF_HOST}
          echo "login"
          echo ${JF_USER}
          echo "password"
          echo "${{ steps.auth_artifactory.outputs.access_token }}"
        } > ~/.netrc
        cat ~/.netrc

    - shell: bash
      if: ${{ steps.check_artifactory.outputs.value == 'true' }}
      run: |
        echo "Setup go"
        set -x
        GOPROXY="https://service-account-releng:${{ steps.auth_artifactory.outputs.access_token }}@artifactory.ops.lacework.engineering/artifactory/api/go/go-virtual"
        GONOSUMDB="github.com/lacework/*,github.com/lacework-dev/*"
        echo "GOPROXY=${GOPROXY}" >> $GITHUB_ENV
        echo "GONOSUMDB=${GONOSUMDB}" >> $GITHUB_ENV

    - name: Auth GitHub no Artifactory
      if: ${{ steps.check_artifactory.outputs.value == 'false' }}
      shell: bash
      run: |
        echo "Setup netrc"
        set -x
        {
          echo "machine"
          echo github.com
          echo "login"
          echo lacework-releng
          echo "password"
          echo "${GITHUB_RELENG_TOKEN}"
        } > ~/.netrc
        cat ~/.netrc
        git config --global --add url."https://lacework-releng:${GITHUB_RELENG_TOKEN}@github.com/".insteadOf "https://github.com/"
```