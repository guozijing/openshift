## Permission Denied
    1 oc adm policy add-scc-to-user anyuid -z default
    2 oc edit scc anyuid
        users:
        - system:serviceaccount:default:ci
        - system:serviceaccount:ci:default
        - system:serviceaccount:innovation-2016:default

## SCTP
    1 labels
        vim /data/install/worker-ric.yaml
        ```
        apiVersion: machineconfiguration.openshift.io/v1
        kind: MachineConfigPool
        metadata:
        name: worker-ric
        labels:
            machineconfiguration.openshift.io/role: worker-ric
        spec:
        machineConfigSelector:
            matchExpressions:
            - {key: machineconfiguration.openshift.io/role, operator: In, values: [worker,worker-ric]}
        nodeSelector:
            matchLabels:
            node-role.kubernetes.io/worker-ric: ""
        ```
        oc create -f /data/install/worker-ric.yaml
        oc label node worker-0 node-role.kubernetes.io/worker-ric=""
    2 SCTP
        cat << EOF > /data/install/sctp-module.yaml
        apiVersion: machineconfiguration.openshift.io/v1
        kind: MachineConfig
        metadata:
        name: 1-worker-ric-load-sctp-module
        labels:
            machineconfiguration.openshift.io/role: worker-ric
        spec:
        config:
            ignition:
            version: 3.1.0
            storage:
            files:
                - path: /etc/modprobe.d/sctp-blacklist.conf
                mode: 0644
                overwrite: true
                contents:
                    source: data:,
                - path: /etc/modules-load.d/sctp-load.conf
                mode: 0644
                overwrite: true
                contents:
                    source: data:,sctp
        EOF
        oc create -f /data/install/sctp-module.yaml
    3 Verify
        ssh core@worker-0
        lsmod | grep sctp
    4 nodeSelector
         nodeSelector:
           node-role.kubernetes.io/worker-ric: ""

## nfs storage
    1 oc get pod -n nfs-storage-provisioner
      bash /data/ocp4/ocp4-upi-helpernode-master/files/nfs-provisioner-setup.sh

## ip
    1 virsh attach-interface ocp4-worker0 --type bridge --source baremetal --current --model virtio
    2 vim net.yaml
        apiVersion: "k8s.cni.cncf.io/v1"
        kind: NetworkAttachmentDefinition
        metadata:
        name: host-device-ric
        spec:
        config: '{
            "cniVersion": "0.3.0",
            "type": "host-device",
            "device": "enp1s0",
            "ipam": {
            "type": "host-local",
            "subnet": "192.168.7.0/24",
            "rangeStart": "192.168.7.105",
            "rangeEnd": "192.168.7.105",
            "routes": [{
                "dst": "0.0.0.0/0"
            }],
            "gateway": "192.168.7.1"
            }
        }'
    3 oc creat -f net.yaml   oc get net-attach-def
    4 template:
        metadata:
        annotations:
            k8s.v1.cni.cncf.io/networks: '[
            { "name": "host-device-ric",
                "interface": "veth11" }
            ]'

## registry
    1 vim /etc/containers/registries.conf
            [registries.insecure]
            registries = ['registry.ocp4.redhat.ren:5443']
    2 vim /etc/docker/daemon.json
            {
                "insecure-registry": ["registry.ocp4.redhat.ren:5443"]
            }

## host nic
    1 nmcli con show
    2 nmcli connection add ifname baremetal type bridge con-name baremetal ipv4.method 'manual' \
    ipv4.address "192.168.7.1/24"
    3 nmcli con add type bridge-slave ifname enp0s20f0u1u2 master baremetal
    4 nmcli con up baremetal
