# Research Summary (1)

+ **IRON Interrupt**









+ **NVMe/TCP IO-worker runtime**

  + NVMe/TCP는 WorkQueue 구현을 기반으로 request와 response를 주고받는 형식을 취하고 있음.

    + 이 IO worker의 경우, request가 enqueue 되었을 때, 또는 주기적으로 queue_work_on 함수를 통해 스케쥴 되며, 스케줄이 되어서는 NVMe/TCP queue를 체크하며 request 전송을 수행.

  + worker의 경우 가장 높은 우선순위를 갖고 스케줄링 되며, 때문에 타 프로세스의 runtime을 상대적으로 갉아먹으며 동작하게 됨.

    + 뿐만 아니라, wq의 경우 직접 target node로 request packet을 전송하는 주체로써, 많은 양의 **interrupt를 발생**시키게 됌. 

      (Network related interrupts -> IRON)

    + 그러나 이러한  interrupt들도 결국은 **wq_worker들의 interrupt**일 뿐 properly imposed 되지 않으며, 이는 결국 IRON에서 언급한대로 많은 양의 네트워크 트래픽 처리를 위한 CPU 사용량이 올바르게 부과되지 않은 채 흘러가는 문제가 발생한다.

      

    + worker가 heavy한 IO를 마주했을 때, 50% 이상의 CPU를 소모하게 되며, 이는 IO를 사용하지 않는 타 process의 동작까지도 영향을 줄 수 있음. (따지고 보면 cfs에 의해 fair한 양을 먹는 것이겠지만, IO를 사용하지 않는 process의 입장에서는 worker가 없다면, 더 많은 양을 소모할 수 있었을 것.)

    + 반대로 말하면, IO를 사용하는 process들이 worker의 사용량까지 고려하였을 때, 너무 많은 양의 CPU를 사용하게 됨.

      + 즉, CPU 사용량에 맞춰서 IO양을 restrict 하던지, 반대로 worker의 사용량을 IO-used process에 imposing 하는 scheme이 필요할 것.

        

  + *이러한 경우가 발생하는 real-world example 실험 필요.*

    

  + 구현을 위해서는 다음과 같은 기능이 필요로 한다.

    + cfs_rq context에서의 **IO 사용량 모니터링** - 각 process들의 IO 사용량을 기반으로 wq의 CPU usage (slice)를 나눠 갖는 로직 구현 필요
    + wq의 CPU time 사용량
      + 단순한 CPU 사용량은 이미 retrieve 가능함.
      + 그러나 interrupt에 의한 사용량은 고려가 안되고 있음. (iron impl.)
      + wq의 지난 runtime history에 대한 구현.
        + period 별 IO 사용량에 대한 measure 필요.
        + 지난 period의 IO 사용량 rate & io worker runtime 기반의 impose 수행 필요. 
        + <u>**Task_Group** 개념 포함되면 좀 어려울 수 있음.</u>