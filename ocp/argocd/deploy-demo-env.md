# Deploy OpenShift and the required Operators

1. Deploy an OpenShift or OKD Cluster. For this demo we used OpenShift Container Platform v4.7 and enable additional operatorhub sources

    ~~~sh
    oc patch operatorhubs.config.openshift.io cluster --type=json -p='[{"op": "replace", "path": "/spec/disableAllDefaultSources", "value":false}]'
    ~~~

2. Deploy Argo CD using the OperatorG

    1. Deploy the Operator

    ~~~sh
    oc create namespace argocd
    cat <<EOF | oc -n argocd create -f -
    apiVersion: operators.coreos.com/v1
    kind: OperatorGroup
    metadata:
      name: argocd-operatorgroup
    spec:
      targetNamespaces:
      - argocd
    EOF
    cat <<EOF | oc -n argocd create -f -
    apiVersion: operators.coreos.com/v1alpha1
    kind: Subscription
    metadata:
      name: argocd-operator
    spec:
      channel: alpha
      name: argocd-operator
      source: community-operators
      sourceNamespace: openshift-marketplace
    EOF
    ~~~

    2. Once the operator is running, create the operand

    ~~~sh
    cat <<EOF | oc -n argocd apply -f -
    apiVersion: argoproj.io/v1alpha1
    kind: ArgoCD
    metadata:
      name: argocd
      namespace: argocd
    spec:
      server:
        route:
          enabled: true
      dex:
        openShiftOAuth: true
        image: quay.io/redhat-cop/dex
        version: v2.22.0-openshift
      resourceCustomizations: |
        route.openshift.io/Route:
          ignoreDifferences: |
            jsonPointers:
            - /spec/host
        extensions/Ingress:
          health.lua: |
            hs = {}
            hs.status = "Healthy"
            return hs
      rbac:
        defaultPolicy: ''
        policy: |
          g, system:cluster-admins, role:admin
          g, admins, role:admin
          g, developer, role:developer
          g, marketing, role:marketing
        scopes: '[groups]'
    EOF
    ~~~

    3. Give argocd service account permissions

    ~~~sh
    oc adm policy add-cluster-role-to-user cluster-admin -z argocd-application-controller -n argocd
    ~~~

    4. We need to configure our `Secret Token` on Argo CD

    ~~~sh
    WEBHOOK_SECRET="v3r1s3cur3"
    oc -n argocd patch secret argocd-secret -p "{\"data\":{\"webhook.github.secret\":\"$(echo -n $WEBHOOK_SECRET | base64)\"}}" --type=merge
    ~~~

3. Deploy OpenShift Pipelines Operator

    ~~~sh
    cat <<EOF | oc -n openshift-operators create -f -
    apiVersion: operators.coreos.com/v1alpha1
    kind: Subscription
    metadata:
      name: openshift-pipelines-operator-rh
    spec:
      channel: stable
      installPlanApproval: Automatic
      name: openshift-pipelines-operator-rh
      source: redhat-operators
      sourceNamespace: openshift-marketplace
    EOF
    ~~~

# Create the required Tekton manifests

1. Clone the Git repositories (you will need the ssh keys already in place)

    > **NOTE**: You need to fork these repositories and use your fork (so you have full-access)

    ~~~sh
    mkdir -p /var/tmp/code-to-prod-demo/
    git clone git@github.com:llegolas/reverse-words.git /var/tmp/code-to-prod-demo/reverse-words
    git clone git@github.com:llegolas/reverse-words-cicd.git /var/tmp/code-to-prod-demo/reverse-words-cicd
    ~~~

2. Go to the reverse-words-cicd repo and checkout the CI branch which contains our Tekton manifests

    ~~~sh
    cd /var/tmp/code-to-prod-demo/reverse-words-cicd
    git checkout ci
    ~~~

3. Create a namespace for storing the configuration for our reversewords app pipeline

    ~~~sh
    oc create namespace reversewords-ci
    ~~~

4. Create a Secret containing the credentials to access our Git repository

    > **NOTE**: You need to provide a token with push access to the cicd repository

    ~~~sh
    read -s GIT_AUTH_TOKEN
    oc -n reversewords-ci create secret generic image-updater-secret --from-literal=token=${GIT_AUTH_TOKEN}
    ~~~

5. Create the all the tasks needed down the road

    ~~~sh
    oc -n reversewords-ci create -f tasks/
    ~~~

6. Edit some parameters from our Build Pipeline definition

    > **NOTE**: You need to use your forks address in the substitutions below

    ~~~sh
    sed -i "s|<reversewords_git_repo>|https://github.com/llegolas/reverse-words|" pipelines/build-pipeline/build-pipeline.yaml
    sed -i "s|<reversewords_quay_repo>|quay.io/mavazque/tekton-reversewords|" pipelines/build-pipeline/build-pipeline.yaml
    sed -i "s|<golang_package>|github.com/llegolas/reverse-words|" pipelines/build-pipeline/build-pipeline.yaml
    sed -i "s|<imageBuilder_sourcerepo>|llegolas/reverse-words-cicd|" pipelines/build-pipeline/build-pipeline.yaml
    sed -i "s|<stageAppUrl>|reversewords-dev.apps.am-week-test.okd.endava-test-domain.be|" pipelines/build-pipeline/build-pipeline.yaml
    ~~~

7. Edit some parameters from our Promoter Pipeline definition

    > **NOTE**: You need to use your forks address/quay account in the substitutions below

    ~~~sh
    sed -i "s|<reversewords_cicd_git_repo>|https://github.com/llegolas/reverse-words-cicd|" pipelines/promote-to-prod-pipeline.yaml
    sed -i "s|<reversewords_quay_repo>|quay.io/mavazque/tekton-reversewords|" pipelines/promote-to-prod-pipeline.yaml
    sed -i "s|<imageBuilder_sourcerepo>|llegolas/reverse-words-cicd|" pipelines/promote-to-prod-pipeline.yaml
    sed -i "s|<stage_deployment_file_path>|./deployment.yaml|" pipelines/promote-to-prod-pipeline.yaml
    ~~~

8. Create the Build and Promoter pipelines' definitions and resources which will be used to execute the previous tasks in an specific order with specific parameters

    ~~~sh
    oc -n reversewords-ci create -f pipelines/
    ~~~

9. Create the TriggerBinding for reading data received by a webhook and pass it to the Pipeline

    ~~~sh
    oc -n reversewords-ci create -f github-triggerbinding.yaml
    ~~~

10. Edit the route definition in ```webhook.yaml``` to reflect your clusters wildcard name

    ~~~sh
    CLUSTER_WILDCARD=$(oc -n openshift-console get route console -o jsonpath='{.spec.host}' | cut -d. -f 2-)
    sed -i "s/host: .*/reversewords-webhook-reversewords-ci\.${CLUSTER_WILDCARD}/"
    ~~~

11. Create the TriggerTemplate, Event Listener and Route to run the Pipeline when new commits hit the main branch of our app repository

    ~~~sh
    WEBHOOK_SECRET="v3r1s3cur3"
    oc -n reversewords-ci create secret generic webhook-secret --from-literal=secret=${WEBHOOK_SECRET}
    oc -n reversewords-ci create -f webhook.yaml
    ~~~

# Configure Argo CD

1. Install the Argo CD Cli to make things easier

    ~~~sh
    # Get the Argo CD Cli and place it in /usr/bin/
    sudo curl -L https://github.com/argoproj/argo-cd/releases/download/v1.7.7/argocd-linux-amd64 -o /usr/local/bin/argocd
    sudo chmod +x /usr/local/bin/argocd
    ~~~

2. Login into Argo CD from the Cli
  
    ~~~sh  
    ARGOCD_PASSWORD=$(oc -n argocd get secret argocd-cluster -o jsonpath='{.data.admin\.password}' | base64 -d)
    ARGOCD_ROUTE=$(oc -n argocd get route argocd-server -o jsonpath='{.spec.host}')
    argocd login $ARGOCD_ROUTE --insecure --username admin --password $ARGOCD_PASSWORD
    ~~~
