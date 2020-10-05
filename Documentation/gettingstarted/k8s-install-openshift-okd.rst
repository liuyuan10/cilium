.. only:: not (epub or latex or html)

    WARNING: You are looking at unreleased Cilium documentation.
    Please use the official rendered version released here:
    https://docs.cilium.io

.. _k8s_install_openshift_okd:

*****************************
Installation on OpenShift OKD
*****************************

OpenShift Requirements
======================

1. Choose preferred cloud provider. This guide was tested in AWS, Azure & GCP.

2. Read `OpenShift documentation <https://docs.okd.io/latest/welcome/index.html>`_ to find out about provider-specific prerequisites

3. `Get OpenShift Installer <https://github.com/openshift/okd#getting-started>`_

.. note:: It's highly recommended to read the docs, unless you have installed
   OpenShift in the past. Here are a few notes that you may find useful.
   
   - with the AWS provider ``openshift-install`` will not work properly
     when MFA credentials are stored in ``~/.aws/credentials``, traditional credentials are required
   - with the Azure provider ``openshift-install`` will prompt for
     credentials and store them in ``~/.azure/osServicePrincipal.json``, it
     doesn't simply pickup ``az login`` credentials. It's recommended to
     setup a dedicated service principal and use it
   - with the GCP provider ``openshift-install`` will only work with a service
     account key, which has to be set using ``GOOGLE_APPLICATION_CREDENTIALS``
     environment variable (e.g. ``GOOGLE_APPLICATION_CREDENTIALS=service-account.json``)

Create an OpenShift OKD Cluster
===============================

First, set cluster name:

.. code:: bash

   CLUSTER_NAME="cluster-1"

Now, create configuration files:

   NOTE: the sample output below is showing the AWS provider, but
   it should work the same way with other providers

.. code:: shell-session

   $ openshift-install create install-config --dir "${CLUSTER_NAME}"
   ? SSH Public Key ~/.ssh/id_rsa.pub
   ? Platform aws
   INFO Credentials loaded from default AWS environment variables 
   ? Region eu-west-1
   ? Base Domain openshift-test-1.cilium.rocks
   ? Cluster Name cluster-1
   ? Pull Secret [? for help] ******************************************************************************************************************************************************************************************

And set ``networkType: Cilium``:

.. code:: bash

   sed -i 's/networkType:\ OVNKubernetes/networkType:\ Cilium/' "${CLUSTER_NAME}/install-config.yaml"

Next, generate OpenShift manifests:

.. code:: bash

   openshift-install create manifests --dir "${CLUSTER_NAME}"

Now, define ``cilium`` namespace:

.. code:: bash

   cat << EOF > "${CLUSTER_NAME}/manifests/cluster-network-03-cilium-namespace.yaml"
   apiVersion: v1
   kind: Namespace
   metadata:
     name: cilium
     annotations:
       # node selector is required to make cilium-operator run on control plane nodes
       openshift.io/node-selector: ""
     labels:
       name: cilium
       # run level sets priority for Cilium to be deployed prior to other components
       openshift.io/run-level: "0"
   EOF

.. include:: k8s-install-download-release.rst

Next, render Cilium manifest:

.. parsed-literal::

   helm template |CHART_RELEASE|  \\
      --namespace cilium \\
      --set config.ipam=cluster-pool \\
      --set global.cni.binPath=/var/lib/cni/bin \\
      --set global.cni.confPath=/var/run/multus/cni/net.d \\
      --set global.ipam.operator.clusterPoolIPv4PodCIDR=10.128.0.0/14 \\
      --set global.ipam.operator.clusterPoolIPv4MaskSize=23 \\
      --set global.nativeRoutingCIDR=10.128.0.0/14 \\
      --set config.bpfMasquerade=false \\
      --set global.endpointRoutes.enabled=true \\
      --output-dir "${OLDPWD}"
   cd "${OLDPWD}"

.. note:: If you make any changes to CIDRs in ``"${CLUSTER_NAME}/install-config.yaml"`` you need to make
   sure that ``clusterPoolIPv4PodCIDR`` and ``clusterPoolIPv4MaskSize`` reflect those changes.

Copy Cilium manifest to ``${CLUSTER_NAME}/manifests``:

.. code:: bash

   for component in config agent operator
      do for resource in cilium/charts/${component}/templates/*
          do cp "${resource}" "${CLUSTER_NAME}/manifests/cluster-network-04-cilium-${component}-$(basename ${resource})"
      done
   done

Create the cluster:

.. note:: The sample output below is showing the AWS provider, but
   it should work the same way with other providers.

.. code:: bash

   $ openshift-install create cluster --dir "${CLUSTER_NAME}"
   WARNING   Discarding the Bootstrap Ignition Config that was provided in the target directory because its dependencies are dirty and it needs to be regenerated
   INFO Consuming OpenShift Install (Manifests) from target directory
   INFO Consuming Master Machines from target directory
   INFO Consuming Worker Machines from target directory
   INFO Consuming Bootstrap Ignition Config from target directory
   INFO Consuming Common Manifests from target directory
   INFO Consuming Openshift Manifests from target directory
   INFO Credentials loaded from default AWS environment variables
   INFO Creating infrastructure resources...
   INFO Waiting up to 20m0s for the Kubernetes API at https://api.cluster-1.openshift-test-1.cilium.rocks:6443...
   INFO API v1.18.3 up
   INFO Waiting up to 40m0s for bootstrapping to complete...

Next, firewall configuration must be updated to allow `Cilium
ports <https://docs.cilium.io/en/v1.8/install/system_requirements/#firewall-rules>`_.
Please note that ``openshift-install`` doesn't support custom firewall
rules, so you will need to use one of the following scripts if you are
using AWS or GCP. Azure does not need additional configuration.

.. note:: This has to be done just after ``INFO Waiting up to 40m0s for
   bootstrapping to complete...`` appears in the logs. It is safe to apply
   these changes once, OpenShift will not override these.

.. tabs::

   .. group-tab:: AWS: enable Cilium ports 

      .. code:: bash

         infraID="$(jq -r < "${CLUSTER_NAME}/metadata.json" '.infraID')"
         aws_region="$(jq -r < "${CLUSTER_NAME}/metadata.json" '.aws.region')"
         cluster_tag="$(jq -r < "${CLUSTER_NAME}/metadata.json" '.aws.identifier[0] | to_entries | "Name=tag:\(.[0].key),Values=\(.[0].value)"')"

         worker_sg="$(aws ec2 describe-security-groups --region "${aws_region}" --filters "${cluster_tag}" "Name=tag:Name,Values=${infraID}-worker-sg" | jq -r .SecurityGroups[0].GroupId)"
         master_sg="$(aws ec2 describe-security-groups --region "${aws_region}" --filters "${cluster_tag}" "Name=tag:Name,Values=${infraID}-master-sg" | jq -r .SecurityGroups[0].GroupId)"

         aws ec2 authorize-security-group-ingress --region "${aws_region}" \
            --ip-permissions \
               "IpProtocol=udp,FromPort=8472,ToPort=8472,UserIdGroupPairs=[{GroupId=${worker_sg}},{GroupId=${master_sg}}]" \
               "IpProtocol=tcp,FromPort=4240,ToPort=4240,UserIdGroupPairs=[{GroupId=${worker_sg}},{GroupId=${master_sg}}]" \
            --group-id "${worker_sg}"

         aws ec2 authorize-security-group-ingress --region "${aws_region}" \
            --ip-permissions \
               "IpProtocol=udp,FromPort=8472,ToPort=8472,UserIdGroupPairs=[{GroupId=${worker_sg}},{GroupId=${master_sg}}]" \
               "IpProtocol=tcp,FromPort=4240,ToPort=4240,UserIdGroupPairs=[{GroupId=${worker_sg}},{GroupId=${master_sg}}]" \
            --group-id "${master_sg}"

   .. group-tab:: GCP: enable Cilium ports

      .. code:: bash       

         infraID="$(jq -r < "${CLUSTER_NAME}/metadata.json" '.infraID')"
         gcp_projectID="$(jq -r < "${CLUSTER_NAME}/metadata.json" '.gcp.projectID')"

         gcloud compute firewall-rules create \
            --project="${gcp_projectID}" \
            --network="${infraID}-network" \
            --allow=tcp:4240,udp:8472,icmp \
            --source-tags="${infraID}-worker,${infraID}-master" \
            --target-tags="${infraID}-worker,${infraID}-master" \
              "${infraID}-cilium"

Accessing the Cluster
=====================

To access the cluster you will need to use ``kubeconfig`` file from the ``$CLUSTER_NAME/auth`` directory:

.. code:: bash

   export KUBECONFIG=cluster-1/auth/kubeconfig

.. include:: k8s-install-connectivity-test.rst
