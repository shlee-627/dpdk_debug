# Current Implementation Follow Up

+ `컨테이너는 반 가상화된 드라이버인 virtio와 vhost를 통해 DPDK를 사용할 수 있다.` 언급
  + 앞선 VM에서의 여러 DPDK 적용 옵션에 대한 언급  O.
    + virtio & vhost를 기반으로한 구현체가 가장 현실성, 설득력 있다고 언급.
    + reference - **Performance of OVS-DPDK in container Networks** 
  + **OVS-DPDK**로,
    1. OVS, vhost에 packet을 뽑아오고,
    2. virtio application이 각 container 내에 돌면서 vhost로부터 packet을 가져오는 방식을 취함.
       - vhost에서 virtio application에 switching 되도록 rule을 설정하고 packet을 전달하는 방식일 듯함.
  + (Jan. 17th) 생긴 의문들:
    + 해당 방식이 과연 옳은지 검증이 필요한가 (OVS-DPDK 기반의 구현이 과연 <u>가장 optimal</u>인가. **다른 효율적인 방식** 없나?)





### virtio-vhost w/ OVS-DPDK implmentation breakdown

+ **shared memory** 기반의 packet 공유 (IPC, Inter Process Communication)
  + virtIO가 쓸 수 있게 vhost가 NIC으로부터 polling
+ **DPDK** application (e.g. pktgen etc.)이 sharedMem. 으로부터 packet을 **polling**
  + 대신 **복사는 없이**, 그 packet을 그대로 processing 하나봄.
  + 왜 이래야하지? - pps를 내고, shared memory에대한 utilization을 높이기 위해서?
    

+ Main problems:
  + 메모리 복사 (from A to **shared memory**?) : what's A?
    + NIC에서 OVS-DPDK 쪽으로 전달, 이후 **shared-mem.으로 copy**가 일어나나? (**TODO**)
  + Memory **alloc. & de-alloc.** parts
    + Shared Memory 내에서 **Available - Used field 관리하는 기법**들인듯?



+ (Jan. 17th) 궁금한 부분들:
  + virtIO-vhost is **per-container** manner?
  + Copy가 한 번 더 일어나는 이유에대해 이해할 수가 없음.
    + ~~test-pmd를 써서 일어나는 것처럼 읽히는데..~~

