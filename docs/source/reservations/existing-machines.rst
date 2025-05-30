.. _existing-machines:

Deploy SkyPilot on existing machines
====================================

This guide will help you deploy SkyPilot on your existing machines — whether they are on-premises or reserved instances on a cloud provider.


.. tip::

    To run SkyPilot on your local machine, use ``sky local up`` to :ref:`deploy a kubernetes cluster <kubernetes-setup-kind>` with `kind <https://kind.sigs.k8s.io/>`_.

**Given a list of IP addresses and SSH credentials,**
SkyPilot will install necessary dependencies on the remote machines and configure itself to run jobs and services on the cluster.

..
   Figure v1 (for deploy.sh): https://docs.google.com/drawings/d/1Jp1tTu1kxF-bIrS6LRMqoJ1dnxlFvn-iobVsXElXfAg/edit?usp=sharing
   Figure v2: https://docs.google.com/drawings/d/1hMvOe1HX0ESoUbCvUowla2zO5YBacsdruo0dFqML9vo/edit?usp=sharing
   Figure v2 Dark: https://docs.google.com/drawings/d/1AEdf9i3SO6MVnD7d-hwRumIfVndzNDqQmrFvRwwVEiU/edit

.. figure:: ../images/sky-existing-infra-workflow-light.png
   :width: 85%
   :align: center
   :alt: Deploying SkyPilot on existing machines
   :class: no-scaled-link, only-light

   Given a list of IP addresses and SSH keys, ``sky local up`` will install necessary dependencies on the remote machines and configure SkyPilot to run jobs and services on the cluster.

.. figure:: ../images/sky-existing-infra-workflow-dark.png
   :width: 85%
   :align: center
   :alt: Deploying SkyPilot on existing machines
   :class: no-scaled-link, only-dark

   Given a list of IP addresses and SSH keys, ``sky local up`` will install necessary dependencies on the remote machines and configure SkyPilot to run jobs and services on the cluster.


.. note::

    Behind the scenes, SkyPilot deploys a lightweight Kubernetes cluster on the remote machines using `k3s <https://k3s.io/>`_.

    **Note that no Kubernetes knowledge is required for running this guide.** SkyPilot abstracts away the complexity of Kubernetes and provides a simple interface to run your jobs and services.

Prerequisites
-------------

**Local machine (typically your laptop):**

* `kubectl <https://kubernetes.io/docs/tasks/tools/install-kubectl/>`_
* `SkyPilot <https://docs.skypilot.co/en/latest/getting-started/installation.html>`_

**Remote machines (your cluster, optionally with GPUs):**

* Debian-based OS (tested on Debian 11)
* SSH access from local machine to all remote machines with key-based authentication
* It's recommended to use passwordless sudo for all remote machines. If passwordless sudo cannot be used, all machines must use the same password for the SSH username to use sudo.
* All machines must use the same SSH key and username
* All machines must have network access to each other
* Port 6443 must be accessible on at least one node from your local machine

Deploying SkyPilot
------------------

1. Create a file ``ips.txt`` with the IP addresses of your machines with one IP per line.
   The first node will be used as the head node — this node must have port 6443 accessible from your local machine.

   Here is an example ``ips.txt`` file:

   .. code-block:: text

      192.168.1.1
      192.168.1.2
      192.168.1.3

   In this example, the first node (``192.168.1.1``) has port 6443 open and will be used as the head node.

2. Run ``sky local up`` and pass the ``ips.txt`` file, SSH username, and SSH key as arguments:

   .. code-block:: bash

      IP_FILE=ips.txt
      SSH_USER=username
      SSH_KEY=path/to/ssh/key
      CONTEXT_NAME=mycluster  # Optional, sets the context name in the kubeconfig. Defaults to "default".
      sky local up --ips $IP_FILE --ssh-user $SSH_USER --ssh-key-path $SSH_KEY --context-name $CONTEXT_NAME

   .. tip::
      If your cluster does not have passwordless sudo, specify the sudo password with the ``--password`` option:

      .. code-block:: bash

         PASSWORD=password
         sky local up --ips $IP_FILE --ssh-user $SSH_USER --ssh-key-path $SSH_KEY --context-name $CONTEXT_NAME --password $PASSWORD

   SkyPilot will deploy a Kubernetes cluster on the remote machines, set up GPU support, configure Kubernetes credentials on your local machine, and set up SkyPilot to operate with the new cluster.

   Example output of ``sky local up``:

   .. code-block:: console

      $ sky local up --ips ips.txt --ssh-user gcpuser --ssh-key-path ~/.ssh/id_rsa --context-name mycluster
      To view detailed progress: tail -n100 -f ~/sky_logs/sky-2024-09-23-18-53-14-165534/local_up.log
      ✔ K3s successfully deployed on head node.
      ✔ K3s successfully deployed on worker node.
      ✔ kubectl configured for the remote cluster.
      ✔ Remote k3s is running.
      ✔ Nvidia GPU Operator installed successfully.
      Cluster deployment done. You can now run tasks on this cluster.
      E.g., run a task with: sky launch --infra kubernetes -- echo hello world.
      🎉 Remote cluster deployed successfully.


4. To verify that the cluster is running, run:

   .. code-block:: bash

      sky check kubernetes

   You can now use SkyPilot to launch your :ref:`development clusters <dev-cluster>` and :ref:`training jobs <ai-training>` on your own infrastructure.

   .. code-block:: console

      $ sky show-gpus --infra k8s
      Kubernetes GPUs
      GPU   REQUESTABLE_QTY_PER_NODE  UTILIZATION
      L4    1, 2, 4                   12 of 12
      H100  1, 2, 4, 8                16 of 16

      Kubernetes per node GPU availability
      NODE                       GPU       UTILIZATION
      my-cluster-0               L4        4 of 4
      my-cluster-1               L4        4 of 4
      my-cluster-2               L4        2 of 2
      my-cluster-3               L4        2 of 2
      my-cluster-4               H100      8 of 8
      my-cluster-5               H100      8 of 8

      $ sky launch --infra k8s --gpus H100:1 -- nvidia-smi

   .. tip::

      To enable shared access to a Kubernetes cluster, you can deploy a :ref:`SkyPilot API server <sky-api-server>`.

What happens behind the scenes?
-------------------------------

When you run ``sky local up``, SkyPilot runs the following operations:

1. Install and run `k3s <https://k3s.io/>`_ Kubernetes distribution as a systemd service on the remote machines.
2. [If GPUs are present] Install `Nvidia GPU Operator <https://docs.nvidia.com/datacenter/cloud-native/gpu-operator/latest/index.html>`_ on the newly provisioned k3s cluster. Note that this step does not modify your local nvidia driver/cuda installation, and only runs inside the cluster.
3. Expose the Kubernetes API server on the head node over port 6443. API calls are on this port are secured with a key pair generated by the cluster.
4. Configure ``kubectl`` on your local machine to connect to the remote cluster.


Cleanup
-------

To clean up all state created by SkyPilot on your machines, use the ``--cleanup`` flag:

.. code-block:: bash

    IP_FILE=ips.txt
    SSH_USER=username
    SSH_KEY=path/to/ssh/key
    sky local up --ips $IP_FILE --ssh-user $SSH_USER --ssh-key-path $SSH_KEY --cleanup

.. tip::
   If your cluster does not have passwordless sudo, specify the sudo password with the ``--password`` option:

   .. code-block:: bash

      PASSWORD=password
      sky local up --ips $IP_FILE --ssh-user $SSH_USER --ssh-key-path $SSH_KEY --password $PASSWORD --cleanup

This will stop all Kubernetes services on the remote machines.


Setting up multiple clusters
----------------------------

You can set up multiple Kubernetes clusters with SkyPilot by using different ``context-name`` values for each cluster:

.. code-block:: bash

    # Set up first cluster and save the kubeconfig
    sky local up --ips cluster1-ips.txt --ssh-user user1 --ssh-key-path key1.pem --context-name cluster1
    # Set up second cluster
    sky local up --ips cluster2-ips.txt --ssh-user user2 --ssh-key-path key2.pem --context-name cluster2


You can then configure SkyPilot to use :ref:`multiple Kubernetes clusters <multi-kubernetes>` by adding them to ``allowed_contexts`` in your ``~/.sky/config.yaml`` file:

.. code-block:: yaml

   # ~/.sky/config.yaml
    allowed_contexts:
      - cluster1-ctx
      - cluster2-ctx


.. code-block:: bash

    # Run on cluster1
    sky launch --infra k8s/cluster1-ctx -- echo "Running on cluster 1"

    # Run on cluster2
    sky launch --infra k8s/cluster2-ctx -- echo "Running on cluster 2"

    # Let SkyPilot automatically select the cluster with available resources
    sky launch --infra k8s -- echo "Running on SkyPilot selected cluster"

You can view the available clusters and GPUs using:

.. code-block:: bash

    # List GPUs on cluster1
    sky show-gpus --infra k8s/cluster1-ctx

    # List GPUs on cluster2
    sky show-gpus --infra k8s/cluster2-ctx
