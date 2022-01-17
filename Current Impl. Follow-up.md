# Current Implementation Follow Up

+ `컨테이너는 반 가상화된 드라이버인 virtio와 vhost를 통해 DPDK를 사용할 수 있다.` 언급
  + 앞선 VM에서의 여러 DPDK 적용 옵션에 대한 언급  O.
    + virtio & vhost를 기반으로한 구현체가 가장 현실성, 설득력 있다고 언급.
    
    + reference - **Performance of OVS-DPDK in container Networks** 
    
      
  + **OVS-DPDK**로,
    1. ~~OVS, vhost에 packet을 뽑아오고,~~
    2. ~~virtio application이 각 container 내에 돌면서 vhost로부터 packet을 가져오는 방식을 취함.~~
       - ~~vhost에서 virtio application에 switching 되도록 rule을 설정하고 packet을 전달하는 방식일 듯함.~~

    1. pmd를 기반으로 ovs가 NIC으로부터 packet을 polling

    2. forwarding rule을 기반으로 알맞은 vhost로 패킷을 이동

       - *아마도 이 과정에서 NIC -> vhost shared memory로의 copy가 1차적으로 있을 듯.*

    3. virtio_pmd 드라이버가 shared mem.를 계속 polling하고 있다가, shared mem.에 패킷 생기면 **copying**

    4. Application이 복사해온 packet을 사용.

       

  + (Jan. 17th) 생긴 의문들:
    + 해당 방식이 과연 옳은지 검증이 필요한가 (OVS-DPDK 기반의 구현이 과연 <u>가장 optimal</u>인가. **다른 효율적인 방식** 없나?)

      + **XDP** feasibility

    + forwarding rule을 따른 후, 알맞은 vhost의 메모리 영역으로 packet을 올리는 것도 결국은 copying이 있지 않나?

      + ""그냥 이전에도 있었으니, 배제하고 생각하겠다." 라는 뉘앙스 

      - 확인이 필요... (**TODO**)



### virtio-vhost w/ OVS-DPDK implmentation breakdown

+ **shared memory** 기반의 packet 공유 (IPC, Inter Process Communication)
  + ~~virtIO가 쓸 수 있게 vhost가 NIC으로부터 polling~~
  + Shared Mem.을 선언하고, 거기다 패킷을 써놓음 -> virtio-pmd가 polling & copying을 함으로써 consuming
+ **Available** field와 **Used** field를 만들어놓고... 가용한 메모리 영역을 관리. (**정확한 구현체 확인이 필요함**)

  

+ Main problems:
  + 메모리 복사 (<u>from vhost shared memory to virtio-pmd</u>) 

    
  + Memory **alloc. & de-alloc.** parts
    + Shared Memory 내에서 **Available - Used field 관리하는 기법**들인듯?



+ (Jan. 17th) 궁금한 부분들:
  + virtIO-vhost is **per-container** manner?

    + OVS switchd, vhost 별개의 process인가?

  + Container App. 구성이 virtio-pmd + $\alpha$인가?

    + In other words, virtio-pmd library로 패킷을 받아오는 부분 구현하고, 그 이외의 processing 부분을 구현하는 형태인가?
    + 또는, 별개의 process가 돌면서 또 다른 shared memory 영역을 구성하는건가..?

  + **Core Affinity**

    + 0 - OVS, DPDK, vhost
    + 1 - container 0 (virtio-pmd + pktgen)
    + 2 - container 1 (virtio-pmd + statistical application)
    + ...

    처럼, OVS가 별도의 core를 쓰고 다른 코어에 있는 컨테이너들에게 패킷을 전달하는게 가능한지. 
    (*서로 다른 core를 쓰는* 어플리케이션 간 shared memory 형성이 가능한가.)

  + QUOTA, CPU bandwidth의 영역을 벗어난 듯한 느낌.

    + Full Line Rate를 처리할 수 있는 NF(Network Functions)들을 사용하려면.. 결국은 Core 하나를 full로 먹고 처리하는 app이 필요한거 아닌가.. 
    + *chaining이 된다치면 OVS도 결국 한 번만 거쳐가는 느낌?* (추측)
      
    + 만약 OVS를 공유한다 치면, OVS 또는 OVS - container 사이에 **QDisc (Queue Discpline)**과 같은 bw mgmt. 툴이 필요한게 아닌가.
      + 더 알려면 TX에 대한 정확한 로직이 필요함.
      + More precisely, 모든 NF들이 TX시 OVS vhost를 거쳐야하는지. (즉, 하나 떠있는 ovs-switchd를 거쳐야 하는지. **병목이 일어나는지**.)

