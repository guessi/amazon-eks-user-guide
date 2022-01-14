# Managing the Amazon EBS CSI self\-managed add\-on<a name="managing-ebs-csi-self-managed-add-on"></a>

This topic shows you how to deploy the Amazon EBS CSI Driver as a self\-managed add\-on to your Amazon EKS cluster and verify that it works\.

**Note**  
The driver isn't supported on Fargate\. Alpha features of the Amazon EBS CSI Driver aren't supported on Amazon EKS clusters\.

For detailed descriptions of all the available parameters and complete examples that demonstrate the driver's features, see the [Amazon EBS Container Storage Interface \(CSI\) driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver) project on GitHub\.

**Prerequisites**
+ An existing 1\.17 or later cluster\. If you don't have one, see [Getting started with Amazon EKS](getting-started.md) to create one\.
+ An existing IAM OpenID Connect \(OIDC\) provider for your cluster\. To determine whether you already have one, or to create one, see [Create an IAM OIDC provider for your cluster](enable-iam-roles-for-service-accounts.md)\.
+ AWS CLI version 1\.22\.30 or later or 2\.4\.9 installed on your computer\. To install or upgrade the AWS CLI, see [Installing, updating, and uninstalling the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)\.
+ `kubectl` version 1\.17 or later installed on your computer\. To install or upgrade `kubectl`, see [Installing `kubectl`](install-kubectl.md)\.

**To deploy the Amazon EBS CSI driver to an Amazon EKS cluster**

1. Create an IAM policy that allows the CSI driver's service account to make calls to AWS APIs on your behalf\. You can view the policy document [on GitHub](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/docs/example-iam-policy.json)\.

   1. Download the IAM policy document from GitHub\.

      ```
      curl -o example-iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/docs/example-iam-policy.json
      ```

   1. Create the policy\. You can change `AmazonEKS_EBS_CSI_Driver_Policy` to a different name\. If you change it, make sure to change it in later steps\.

      ```
      aws iam create-policy \
          --policy-name AmazonEKS_EBS_CSI_Driver_Policy \
          --policy-document file://example-iam-policy.json
      ```

1. Create an IAM role and attach the IAM policy to it\. You can use either `eksctl` or the AWS CLI\.

------
#### [ eksctl ]

   Replace `my-cluster` with the name of your cluster and *111122223333* with your account ID\.

   ```
   eksctl create iamserviceaccount \
       --name ebs-csi-controller-sa \
       --namespace kube-system \
       --cluster my-cluster \
       --attach-policy-arn arn:aws:iam::111122223333:policy/AmazonEKS_EBS_CSI_Driver_Policy \
       --approve \
       --override-existing-serviceaccounts
   ```

   Retrieve the ARN of the created role and note the returned value for use in a later step\.

   ```
   aws cloudformation describe-stacks \
       --stack-name eksctl-my-cluster-addon-iamserviceaccount-kube-system-ebs-csi-controller-sa \
       --query='Stacks[].Outputs[?OutputKey==`Role1`].OutputValue' \
       --output text
   ```

   <a name="ebs-csi-ekscl-role-output"></a>The output is as follows\.

   ```
   arn:aws:iam::111122223333:role/eksctl-my-cluster-addon-iamserviceaccount-kube-sy-Role1-1J7XB63IN3L6T
   ```

------
#### [ AWS CLI ]

   1. View your cluster's OIDC provider URL\. Replace `my-cluster` with your cluster name\. If the output from the command is `None`, review the **Prerequisites**\.

      ```
      aws eks describe-cluster \
          --name my-cluster \
          --query "cluster.identity.oidc.issuer" \
          --output text
      ```

      The output is as follows\.

      ```
      https://oidc.eks.region-code.amazonaws.com/id/oidc-id
      ```

   1. Create the IAM role\.

      1. Copy the following contents to a file that's named `trust-policy.json`\. Replace `111122223333` with your account ID, `region-code` with your Region, and `oidc-id` with the value returned in the previous step\.

         ```
         {
           "Version": "2012-10-17",
           "Statement": [
             {
               "Effect": "Allow",
               "Principal": {
                 "Federated": "arn:aws:iam::111122223333:oidc-provider/oidc.eks.region-code.amazonaws.com/id/oidc-id"
               },
               "Action": "sts:AssumeRoleWithWebIdentity",
               "Condition": {
                 "StringEquals": {
                   "oidc.eks.region-code.amazonaws.com/id/oidc-id:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
                 }
               }
             }
           ]
         }
         ```

      1. Create the role\. You can change `AmazonEKS_EBS_CSI_DriverRole` to a different name\. If you change it, make sure to change it in later steps\.

         ```
         aws iam create-role \
             --role-name AmazonEKS_EBS_CSI_DriverRole \
             --assume-role-policy-document file://"trust-policy.json"
         ```

   1. Attach the IAM policy to the role\. Replace `111122223333` with your account ID\.

      ```
      aws iam attach-role-policy \
        --policy-arn arn:aws:iam::111122223333:policy/AmazonEKS_EBS_CSI_Driver_Policy \
        --role-name AmazonEKS_EBS_CSI_DriverRole
      ```

------

1. You can deploy the driver by using Helm or a manifest\. For driver compatibility using Helm, see the [Kubernetes Version Compatibility Matrix](https://github.com/kubernetes-sigs/aws-ebs-csi-driver#kubernetes-version-compatibility-matrix) on GitHub\. If you want to install an older version of the Amazon EBS CSI driver, use manifest\.

------
#### [ Helm ]

   Install the Amazon EBS CSI driver by using Helm V3 or later\. To install or update Helm, see [Using Helm with Amazon EKS](helm.md)\.

   1. Add the `aws-ebs-csi-driver` Helm repository:

      ```
      helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
      helm repo update
      ```

   1. Install a release of the driver using the Helm chart\. Replace the repository address with the [container image address](add-ons-images.md) of the cluster\. Use the command that matches the tool that you used to create the role in a previous step\.
      + If you used `eksctl` to create the role, use the following command\.

        ```
        helm upgrade -install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
            --namespace kube-system \
            --set image.repository=123456789012.dkr.ecr.region-code.amazonaws.com/eks/aws-ebs-csi-driver \
            --set controller.serviceAccount.create=false \
            --set controller.serviceAccount.name=ebs-csi-controller-sa
        ```
      + If you used AWS CLI to create the role, use the following command with the correct role ARN that you created\.

        ```
        helm upgrade -install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
            --namespace kube-system \
            --set image.repository=123456789012.dkr.ecr.region-code.amazonaws.com/eks/aws-ebs-csi-driver \
            --set controller.serviceAccount.create=true \
            --set controller.serviceAccount.name=ebs-csi-controller-sa \
            --set controller.serviceAccount.annotations."eks\.amazonaws\.com/role-arn"="arn:aws:iam::111122223333:role/AmazonEKS_EBS_CSI_DriverRole"
        ```

------
#### [ Manifest ]

   You can deploy the driver to create volumes with or without tags\.
   + **With tags** – Deploy the driver so that it tags all Amazon EBS volumes that it creates with tags that you specify\.

     1. Clone the [Amazon EBS Container Storage Interface \(CSI\) driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver) GitHub repository to your computer\.

        ```
        git clone https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git
        ```

     1. Navigate to the `base` example folder\.

        ```
        cd aws-ebs-csi-driver/deploy/kubernetes/base/
        ```

     1. Edit the `controller.yaml` file\. Find the section of the file with the following text and add `--extra-tags` to it\. The following text shows the section of the file with the existing and added text\. This example causes the controller to add `department` and `environment` tags to all volumes that it creates\.

        ```
        ...
        containers:
                - name: ebs-plugin
                  image: amazon/aws-ebs-csi-driver:latest
                  imagePullPolicy: IfNotPresent
                  args:
                    # - {all,controller,node} # specify the driver mode
                    - --endpoint=$(CSI_ENDPOINT)
                    - --logtostderr
                    - --v=5
                    - --extra-tags=department=accounting,environment=dev
        ...
        ```

     1. Navigate to the `ecr` folder\.

        ```
        cd ../overlays/stable/ecr
        ```
**Note**  
If your cluster isn't in the `us-west-2` Region, change `602401143452.dkr.ecr.us-west-2.amazonaws.com` to the [container image address](add-ons-images.md) of your cluster in the `kustomization.yaml` file\.

     1. Apply the modified manifest to your cluster\.

        ```
        kubectl apply -k ../ecr
        ```

     1. Annotate the `ebs-csi-controller-sa` Kubernetes service account with the ARN of the IAM role that you created previously\. Use the command that matches the tool that you used to create the role in a previous step\. Replace `111122223333` with your account ID\.
        + If you used `eksctl` to create the role, use this command\.

          ```
          kubectl annotate serviceaccount ebs-csi-controller-sa \
              -n kube-system \
              eks.amazonaws.com/role-arn=arn:aws:iam::111122223333:role/eksctl-my-cluster-addon-iamserviceaccount-kube-sy-Role1-1J7XB63IN3L6T
          ```
        + If you used AWS CLI to create the role, use this command\.

          ```
          kubectl annotate serviceaccount ebs-csi-controller-sa \
              -n kube-system \
              eks.amazonaws.com/role-arn=arn:aws:iam::111122223333:role/AmazonEKS_EBS_CSI_DriverRole
          ```

     1. Restart the `ebs-csi-controller` deployment\.

        ```
        kubectl rollout restart \
            deployment ebs-csi-controller \
            -n kube-system
        ```
   + **Without tags** – Deploy the driver so that it doesn't tag the Amazon EBS volumes that it creates\. To see or download the `kustomization.yaml` file, see the [file](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/deploy/kubernetes/overlays/stable/ecr) on GitHub\.
**Note**  
If your cluster isn't in the `us-west-2` Region, change `602401143452.dkr.ecr.us-west-2.amazonaws.com` to the [container image address](add-ons-images.md) of your cluster in the `kustomization.yaml` file\. Then, apply the manifest locally\.

     1. Apply the manifest\.

        ```
        kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/ecr/?ref=master"
        ```

     1. Annotate the `ebs-csi-controller-sa` Kubernetes service account with the ARN of the IAM role that you created previously\. Use the command that matches the tool that you used to create the role in a previous step\. Replace `111122223333` with your account ID\.
        + If you used `eksctl` to create the role, use this command\.

          ```
          kubectl annotate serviceaccount ebs-csi-controller-sa \
              -n kube-system \
              eks.amazonaws.com/role-arn=arn:aws:iam::111122223333:role/eksctl-my-cluster-addon-iamserviceaccount-kube-sy-Role1-1J7XB63IN3L6T
          ```
        + If you used AWS CLI to create the role, use this command\.

          ```
          kubectl annotate serviceaccount ebs-csi-controller-sa \
              -n kube-system \
              eks.amazonaws.com/role-arn=arn:aws:iam::111122223333:role/AmazonEKS_EBS_CSI_DriverRole
          ```

     1. Delete the driver pods\. They're automatically redeployed with the IAM permissions from the IAM policy assigned to the role\.

        ```
        kubectl delete pods \
            -n kube-system \
            -l=app=ebs-csi-controller
        ```

------

**To deploy a sample application and verify that the CSI driver is working**

This procedure uses the [Dynamic volume provisioning](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/examples/kubernetes/dynamic-provisioning) example from the [Amazon EBS Container Storage Interface \(CSI\) driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver) GitHub repository to consume a dynamically provisioned Amazon EBS volume\. You can deploy sample applications that use [volume snapshots](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/tree/master/examples/kubernetes/snapshot) or [volume resizing](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/blob/master/examples/kubernetes/resizing/README.md) by following the instructions on GitHub\.

1. Clone the [Amazon EBS Container Storage Interface \(CSI\) driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver) GitHub repository to your local system\.

   ```
   git clone https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git
   ```

1. Navigate to the `dynamic-provisioning` example directory\.

   ```
   cd aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/
   ```

1. Deploy the `ebs-sc` storage class, `ebs-claim` persistent volume claim, and `app` sample application from the `specs` directory\.

   ```
   kubectl apply -f specs/
   ```

1. Describe the `ebs-sc` storage class\.

   ```
   kubectl describe storageclass ebs-sc
   ```

   The output is as follows\.

   ```
   Name:            ebs-sc
   IsDefaultClass:  No
   Annotations:     kubectl.kubernetes.io/last-applied-configuration={"apiVersion":"storage.k8s.io/v1","kind":"StorageClass","metadata":{"annotations":{},"name":"ebs-sc"},"provisioner":"ebs.csi.aws.com","volumeBindingMode":"WaitForFirstConsumer"}
   
   Provisioner:           ebs.csi.aws.com
   Parameters:            <none>
   AllowVolumeExpansion:  <unset>
   MountOptions:          <none>
   ReclaimPolicy:         Delete
   VolumeBindingMode:     WaitForFirstConsumer
   Events:                <none>
   ```
**Note**  
The storage class uses the `WaitForFirstConsumer` volume binding mode\. This means that volumes aren't dynamically provisioned until a pod makes a persistent volume claim\. For more information, see [Volume Binding Mode](https://kubernetes.io/docs/concepts/storage/storage-classes/#volume-binding-mode) in the Kubernetes documentation\.

1. Watch the pods in the default namespace\. After a few minutes, the `app` pod's status changes to `Running`\.

   ```
   kubectl get pods --watch
   ```

    Enter `Ctrl`\+`C` to return to a shell prompt\.

1. List the persistent volumes in the default namespace\. Look for a persistent volume with the `default/ebs-claim` claim\.

   ```
   kubectl get pv
   ```

   The output is as follows\.

   ```
   NAME                                         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM               STORAGECLASS   REASON   AGE
   pvc-37717cd6-d0dc-11e9-b17f-06fad4858a5a   4Gi        RWO            Delete           Bound    default/ebs-claim   ebs-sc                  30s
   ```

1. Describe the persistent volume\. Replace `pvc-37717cd6-d0dc-11e9-b17f-06fad4858a5a` with the value from the output in the previous step\.

   ```
   kubectl describe pv pvc-37717cd6-d0dc-11e9-b17f-06fad4858a5a
   ```

   The output is as follows\.

   ```
   Name:              pvc-37717cd6-d0dc-11e9-b17f-06fad4858a5a
   Labels:            <none>
   Annotations:       pv.kubernetes.io/provisioned-by: ebs.csi.aws.com
   Finalizers:        [kubernetes.io/pv-protection external-attacher/ebs-csi-aws-com]
   StorageClass:      ebs-sc
   Status:            Bound
   Claim:             default/ebs-claim
   Reclaim Policy:    Delete
   Access Modes:      RWO
   VolumeMode:        Filesystem
   Capacity:          4Gi
   Node Affinity:
     Required Terms:
       Term 0:        topology.ebs.csi.aws.com/zone in [region-code]
   Message:
   Source:
       Type:              CSI (a Container Storage Interface (CSI) volume source)
       Driver:            ebs.csi.aws.com
       VolumeHandle:      vol-0d651e157c6d93445
       ReadOnly:          false
       VolumeAttributes:      storage.kubernetes.io/csiProvisionerIdentity=1567792483192-8081-ebs.csi.aws.com
   Events:                <none>
   ```

   The Amazon EBS volume ID is the value for `VolumeHandle` in the previous output\.

1. Verify that the pod is writing data to the volume\.

   ```
   kubectl exec -it app -- cat /data/out.txt
   ```

   The output is as follows\.

   ```
   Wed May 5 16:17:03 UTC 2021
   Wed May 5 16:17:08 UTC 2021
   Wed May 5 16:17:13 UTC 2021
   Wed May 5 16:17:18 UTC 2021
   ...
   ```

1. After you're done, delete the resources for this sample application\.

   ```
   kubectl delete -f specs/
   ```