# Implantar um cluster Kubernetes pronto para produção

![Kubernetes Logo](https://raw.githubusercontent.com/kubernetes-sigs/kubespray/master/docs/img/kubernetes-logo.png)

## **Começo rápido**

Para implantar o cluster, você pode usar:

### Ansible

#### **Uso**

```ShellSession
# Install dependencies from ``requirements.txt``
sudo pip3 install -r requirements.txt

# Edit inventory file
vim inventory/cluster/inventory.yaml

# Review and change parameters under ``inventory/mycluster/group_vars``
cat inventory/cluster/group_vars/all/all.yml
cat inventory/cluster/group_vars/k8s_cluster/k8s_cluster.yml

# Deploy Kubespray with Ansible Playbook - run the playbook as root
# The option `--become` is required, as for example writing SSL keys in /etc/,
# installing packages and interacting with various systemd daemons.
# Without --become the playbook will fail to run!
ansible-playbook -i inventory/cluster/inventory.yaml  --become cluster.yml
```

Nota: Quando o Ansible já está instalado por meio de pacotes de sistema na máquina de controle, outros pacotes python instalados por sudo pip install -r requirements.txt vão para uma árvore de diretório diferente (por exemplo, /usr/local/lib/python2.7/dist-packages, no Ubuntu) do Ansible (por exemplo, /usr/lib/python2.7/dist-packages/ansible, ainda no Ubuntu). Como consequência, o ansible-playbook comando falhará com:

```raw
ERROR! no action detected in task. This often indicates a misspelled module name, or incorrect module path.
```

Provavelmente apontando para uma tarefa, dependendo de um módulo presente em requirements.txt.

Uma maneira de resolver isso seria desinstalar o pacote Ansible e, em seguida, instalá-lo via pip, mas nem sempre é possível. Uma solução consiste em configuração `ANSIBLE_LIBRARY` e `ANSIBLE_MODULE_UTILS` variáveis de ambiente, respectivamente, para os `ansible/modules` e `ansible/module_utils` subdiretórios de pacotes pip local de instalação, que podem ser encontrados no campo Localização da saída do`pip show [package` antes de executar `ansible-playbook`.

Uma maneira simples de garantir que você obtenha a versão correta do Ansible é usar a pre-built docker image from Quay. Em seguida, você precisará usar bind mounts para obter o inventário e a chave ssh no contêiner, como este:

```ShellSession
docker pull quay.io/kubespray/kubespray:v2.15.1
docker run --rm -it --mount type=bind,source="$(pwd)"/inventory/sample,dst=/inventory \
  --mount type=bind,source="${HOME}"/.ssh/id_rsa,dst=/root/.ssh/id_rsa \
  quay.io/kubespray/kubespray:v2.15.1 bash
# Inside the container you may now run the kubespray playbooks:
ansible-playbook -i inventory/cluster/inventory.yaml --become --private-key /root/.ssh/id_rsa cluster.yml
```

## **Documentos**

- [Requisitos](#requisitos)
- [Getting started](docs/getting-started.md)
- [Setting up your first cluster](docs/setting-up-your-first-cluster.md)
- [Ansible inventory and tags](docs/ansible.md)
- [Deployment data variables](docs/vars.md)
- [DNS stack](docs/dns-stack.md)
- [HA mode](docs/ha-mode.md)
- [Plugins de Rede](#plugins-de-Rede)
- [Large deployments](docs/large-deployments.md)
- [Adding/replacing a node](docs/nodes.md)
- [Upgrades basics](docs/upgrades.md)

## Supported Linux Distributions

- **Flatcar Container Linux by Kinvolk**
- **Debian** Buster, Jessie, Stretch, Wheezy
- **Ubuntu** 16.04, 18.04, 20.04
- **CentOS/RHEL** 7, [8](docs/centos8.md)
- **Fedora** 32, 33
- **Fedora CoreOS** (experimental: ver [**nota fcos**](docs/fcos.md))
- **openSUSE** Leap 15.x/Tumbleweed
- **Oracle Linux** 7, [8](docs/centos8.md)
- **Alma Linux** [8](docs/centos8.md)
- **Amazon Linux 2** (experimental: veja [amazon linux notes](docs/amazonlinux.md)

Nota: Tipos de SO baseados em Upstart / SysV init não são suportados.

## **Componentes Suportados**

- Essencial
  - [kubernetes](https://github.com/kubernetes/kubernetes) v1.20.7
  - [etcd](https://github.com/coreos/etcd) v3.4.13
  - [docker](https://www.docker.com/) v19.03 (see note)
  - [containerd](https://containerd.io/) v1.4.4
  - [cri-o](http://cri-o.io/) v1.20 (experimental: veja [CRI-O Note](docs/cri-o.md). Somente em sistemas operacionais baseados em fedora, ubuntu e centos)
 
- Plugin de rede
  - [cni-plugins](https://github.com/containernetworking/plugins) v0.9.1
  - [calico](https://github.com/projectcalico/calico) v3.17.4
  - [canal](https://github.com/projectcalico/canal) (dadas as versões de calico/flannel)
  - [cilium](https://github.com/cilium/cilium) v1.8.9
  - [flanneld](https://github.com/coreos/flannel) v0.13.0
  - [kube-ovn](https://github.com/alauda/kube-ovn) v1.6.2
  - [kube-router](https://github.com/cloudnativelabs/kube-router) v1.2.2
  - [multus](https://github.com/intel/multus-cni) v3.7.0
  - [ovn4nfv](https://github.com/opnfv/ovn4nfv-k8s-plugin) v1.1.0
  - [weave](https://github.com/weaveworks/weave) v2.8.1
- Aplicativo
  - [ambassador](https://github.com/datawire/ambassador): v1.5
  - [cephfs-provisioner](https://github.com/kubernetes-incubator/external-storage) v2.1.0-k8s1.11
  - [rbd-provisioner](https://github.com/kubernetes-incubator/external-storage) v2.1.1-k8s1.11
  - [cert-manager](https://github.com/jetstack/cert-manager) v0.16.1
  - [coredns](https://github.com/coredns/coredns) v1.7.0
  - [ingress-nginx](https://github.com/kubernetes/ingress-nginx) v0.43.0

## **Notas de tempo de execução do contêiner**

- A lista de versões docker disponíveis é 18.09, 19.03 e 20.10. A versão do docker recomendada é 19.03. O kubelet pode falhar na numeração de versão não padrão do docker (ele não usa mais o controle de versão semântico). Para garantir que as atualizações automáticas não quebrem seu cluster, procure, por exemplo, o plugin yum versionlock ou apt pin).
- A versão cri-o deve ser alinhada com a respectiva versão do kubernetes (ou seja, kube_version = 1.20.x, crio_version = 1.20)

## **Requisitos**

-  **A versão mínima exigida do Kubernetes é v1.19**
- **Ansible v2.9.x, Jinja 2.11+ e python-netaddr estão instalados na máquina que executará os comandos Ansible, Ansible 2.10.x é experimentalmente compatível por enquanto**
- Os servidores de destino devem ter **acesso à Internet** para obter imagens do docker. Caso contrário, é necessária configuração adicional (consulte Ambiente offline) 
- Os servidores de destino são configurados para permitir **IPv4 forwarding**.
--   Se estiver usando IPv6 para pods e serviços, os servidores de destino serão configurados para permitir o **IPv6 forwarding**.
- Os **firewalls não são gerenciados**, você precisará implementar suas próprias regras da maneira que costumava fazer. Para evitar qualquer problema durante a implantação, você deve desabilitar o firewall.
- Se o kubespray for executado a partir de uma conta de usuário não raiz, o método correto de escalonamento de privilégios deve ser configurado nos servidores de destino. 
Em seguida, o `ansible_become` sinalizador ou os parâmetros de comando `--become or -b` devem ser especificados.

Hardware:
Esses limites são protegidos pelo Kubespray. Os requisitos reais para sua carga de trabalho podem ser diferentes. Para obter um guia de dimensionamento, vá para o guia Building Large Clusters.

- Master
  - Memory: 1500 MB
- Node
  - Memory: 1024 MB

## Plugins de Rede

Você pode escolher entre 10 plug-ins de rede. (padrão `calico`, exceto para usos do Vagrant `flannel`)

- [flannel](docs/flannel.md): rede gre / vxlan (camada 2).

- [Calico](https://docs.projectcalico.org/latest/introduction/) é um provedor de rede e política de rede. 
Calico oferece suporte a um conjunto flexível de opções de rede projetadas para fornecer a você a rede mais eficiente em uma variedade de situações, incluindo redes não sobrepostas e sobrepostas, com ou sem BGP. 
Calico usa o mesmo mecanismo para aplicar a política de rede para hosts, pods e (se estiver usando Istio e Envoy) aplicativos na camada de service mesh.

- [canal](https://github.com/projectcalico/canal): uma composição de plugins de calico and flannel.

- [cilium](http://docs.cilium.io/en/latest/): rede da camada 3/4 (bem como da camada 7 para proteger e proteger os protocolos de aplicativos), oferece suporte à inserção dinâmica de bytecode BPF no kernel do Linux para implementar serviços de segurança, rede e lógica de visibilidade.

- [ovn4nfv](docs/ovn4nfv.md): [ovn4nfv-k8s-plugins](https://github.com/opnfv/ovn4nfv-k8s-plugin) é o controlador de rede, agente OVS e servidor CNI para oferecer rede de sobreposição SFC e OVN básica.

- [weave](docs/weave.md): Weave é uma rede de sobreposição de contêiner leve que não requer um cluster de banco de dados K/V externo.
    ((Consulte a `weave` [troubleshooting documentation](https://www.weave.works/docs/net/latest/troubleshooting/)).

- [kube-ovn](docs/kube-ovn.md): Kube-OVN integra a virtualização de rede baseada em OVN com Kubernetes. Ele oferece um avançado Container Network Fabric para empresas.

- [kube-router](docs/kube-router.md): Kube-router é um L3 CNI para rede Kubernetes com o objetivo de fornecer simplicidade operacional e alto desempenho: ele usa IPVS para fornecer Kube Services Proxy (se configurado para substituir kube-proxy), iptables para políticas de rede e BGP para ods Rede L3 (com peering BGP opcional com peers BGP fora do cluster). Ele também pode anunciar rotas para CIDRs, ClusterIPs, ExternalIPs e LoadBalancerIPs de Pods de cluster do Kubernetes.

- [macvlan](docs/macvlan.md): Macvlan é um driver de rede Linux. Os pods têm seus próprios endereços Mac e IP exclusivos, conectados diretamente à rede física (camada 2).

- [multus](docs/multus.md): Multus é um plug-in meta CNI que fornece suporte a várias interfaces de rede para pods. Para cada interface, Multus delega chamadas CNI para plug-ins CNI secundários, como Calico, macvlan, etc.

A escolha é definida com a variável `kube_network_plugin`. Também há uma opção de aproveitar a rede de provedor de nuvem integrada. Veja também [Verificador de rede.](docs/netcheck.md).

## Ingress Plugins

- [nginx](https://kubernetes.github.io/ingress-nginx): o controlador de entrada NGINX.

- [metallb](docs/metallb.md): o provedor LoadBalancer de serviço bare-metal da MetalLB.

## **Testes de CI**

[![Build graphs](https://gitlab.com/kargo-ci/kubernetes-sigs-kubespray/badges/master/pipeline.svg)](https://gitlab.com/kargo-ci/kubernetes-sigs-kubespray/pipelines)

Testes de CI / ponta a ponta patrocinados por: [CNCF](https://cncf.io), [Packet](https://www.packet.com/), [OVHcloud](https://www.ovhcloud.com/), [ELASTX](https://elastx.se/).

Consulte a [matriz de teste](docs/test_cases.md) para obter detalhes.
