# DPDK Structure

### Basic Operations

- PMD (Poll Mode Driver) 기반 - NIC에 도달한 패킷을 pre-allocated된 memory space에 전달
- Run-to-Completion , Pipe-line 방식으로 각 코어, 각 어플리케이션에 전달
  - 어플리케이션이 Library들이나 직접 구현을 기반으로 stack 처리 (packet processing)
- *이와중에 EAL layer가 수시로 memory 관리해주고, affinity 체크해주고 하는건가..?*

#### EAL(Environment Abstraction Layer)

Q. EAL이 거의 모든 일을 담당하는가?

-  자원 접근, 스레드/코어 최적화, 인터럽트 핸들링, 메모리/버퍼/큐 관리등이 보장
- 개발자는 관련 라이브러리를 기반으로 어플리케이션을 작성할 수 있음..?
- DPDK loading & launching
- multi-process, multi-threads execution allow.
- affinity & assignment setup
- memory (de-)allocation
- Atomic ops. (locking)
- memory mgmt.



#### [DPDK in VMs]

+ [1]에 따르면, 두 level의 커널을 거치는 VM 환경을 고려하면, DPDK를 어디에 어떻게 적용할지는 implementation choice라는데..
+ 대강의 흐름에 따르면 VM에서는...
  + **Physical NIC -- OVS (Open vSwitch) -- virtual NIC -- VM 내 application** 
  + 순서의 접근이 필요한 듯.

+ Choice 1) Application accesses to vNIC
  + VM 내에서 guestOS stack bypass
  + OVS, hypervisor가 packet을  vNIC으로 올려줌. 
    + 즉, 이 과정동안 발생하는 processing overhead는 still happens
+ Choice 2) applications in VM accesses vNIC directly?
  + 1과 유사하나 이번에는 **vNIC - physical NIC 사이에 direct한 path** 형성? (by hypervisor?)
  + SR-IOV 기반으로 물리적 NIC을 다수의 NIC으로 보이게 설정.
  + In summary, SR-IOV로 가상화된 NIC에 app.이 직접 접근하여  packet을 가져가는..?
    + hypervisor, ovs overhead를 아예 날려서 그냥 바로 VM이 physical NIC space를 보게끔 하는 방식인듯.
+ Choice 3) Adapt DPDK on ovs



+ Choice 4) **NetVM** [5]
  + 하이퍼 바이저와 가상 머신이 **메모리를 공유**하여, 메모리 간의 데이터 복사없이 패킷을 교환하는 방식
  + 공유 메모리와 물리적 NIC 사이의 데이터 교환은 DPDK를 사용
  + DPDK는 물리 NIC에서 공유 메모리에 패킷 데이터가 로드 된 후, 여러 가상 머신이 협력하여, 패킷에 필요한 처리를 해갑니다.
    

#### [Networking in Containerized env.]

우선적으로, docker와 같은 **containerized platform이 사용하는 networking 방식**에 대한 이해가 필요함 [6].

+ In docker, 많은 방식의 driver를 기반으로 networking을 지원하는데, for example:
  + **bridge** : 가장 기본이 되는 방식인듯? 패킷 받아서 브릿징 해주는.. 
    - A bridge network is a Link Layer device which **forwards traffic between network segments**. 
    - A bridge can be a hardware device or a software device running within a host machine’s kernel.
    - Use software bridge which allows containers connected to the same bridge network to communicate, while providing isolation from containers which are not connected to that bridge network. 
    - The term of "**same docker daemon host**" used.
    - 처음 생성하면 **default bridge network가 생성**됨: 모든 생성된 컨테이너가 해당 bridge 네트워크를 기반으로 구성되고, 패킷을 전달받는 듯.
    - `docker network create <bridge network name>`을 기반으로 user-define bridge를 생성할 수 있다.
      - 이후, container 생성시, `--network` option을 통해 bridge 네트워크를 setup 할 수 있다.
        

  + **host option** : 혼자만 network을 쓸 때 사용할 수 있는..? host network stack을 오롯이 혼자 사용하는 방식. 
    + **network isolation (x)**
  + **overlay** : 여러 container들을 묶어서, 그들끼리 swarm 할 수 있게끔 구성하는 방식인듯.
    + This strategy removes the need to do OS-level routing between these containers.
    + 우리가 target으로 하는 환경과는 거리가 멀어보이는...?
    + *또는, DPDK를 쓰는 애들끼리 overlay를 구축하고 뭔가 해볼 수 있나?*
  + **ipvlan** :  IPvlan networks give **users total control over both IPv4 and IPv6 addressing**. 
    + The VLAN driver builds on top of that in giving operators complete control of layer 2 VLAN tagging and even IPvlan L3 routing for users interested in underlay network integration. 
  + **macvlan**
  + **none** : network (x)
  + **Third-party network plugins**

+ 가장 기본이 되는, **bridge 기법을 사용**한다고 하였을 때, 구성할 수 있는 방식을 고려할 필요가 있지 않을까.



And still, bridging이 된 후에 동작방식은 잘 모르겠음. **(이후의 network stack은 또 별도로 guest OS에서 실행되는가.**)

#### [DPDK in Containerized env.]

+ 결국 그럼 VM situation과 유사한 상황이 되나?
+ bridge - physical NIC 사이에 DPDK setup 해도 guest OS의 stack은 존재하는건지.



#### [Current Implementations]

**TODO**





### Reference

[1] http://manseok.blogspot.com/2014/11/pmd-dpdk.html

[2] https://frontjang.info/entry/DPDKData-Plane-Development-Kit%EC%97%90-%EB%8C%80%ED%95%98%EC%97%AC-1-%EC%86%8C%EA%B0%9C

[3] https://jhkim3624.tistory.com/85

[4] https://doc.dpdk.org/guides/prog_guide/overview.html

[5] NetVM: High Performance and Flexible Networking Using Virtualization on Commodity Platforms

[6] https://docs.docker.com/network/