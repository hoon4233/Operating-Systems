# Project1 - Scheduler (MLQ, MLFQ)
## 목차
1. 디자인
   1. MLQ
   2. MLFQ
2. 구현
   1. MLQ
   2. MLFQ
   3. isyield
   4. system call(yield, getlev, setpriority)
   5. etc
3. 실행 결과
   1. FCFS Test
   2. MLFQ Test
4. 트러블슈팅

------

## 1. 디자인

### 1-1. MLQ
**Multi level queue scheduler**는 2단계의 큐를 가지고 있고 첫번째 단계의 큐는 rr(round robbin) 방식으로, 두번째 단계의 큐는 fcfs(first come first served) 방식으로 스케쥴링합니다. 이때 pid가 짝수인 프로세스들은 모두 첫번째 단계의 큐에 담기게 되고 pid가 홀수인 프로세스들은 모두 두번째 큐에 담기게 됩니다. 따라서 pid가 짝수인 프로세스들(첫번째 큐에 담겨 있는)이 먼저 rr 방식(프로세서를 돌아가면서 사용)으로 스케쥴링 되며, 짝수 프로세스가 모두 종료 된 후에 pid가 홀수인 프로세스들(두번째 큐에 담겨 있는)이 fcfs 방식(pid가 작은 프로세스가 먼저 프로세서를 사용하고 종료되면 이후 다른 프로세스가 프로세서 사용)으로 스케쥴링 되어 종료되게 됩니다.

따라서 timer interrupt가 들어와 스케쥴링을 시작 할떄 프로세스 테이블을 읽으며 pid가 짝수이며 runnable한 프로세스가 존재한다면 rr방식을 채택해야 하고 그렇지 않다면 fcfs 방식을 채택해 pid가 홀수인 프로세스들을 스케쥴링 할 것입니다. 따라서 짝수인 프로세스가 존재하는지 체크해줄 변수(rrcheck)가 필요합니다.

### 1-2. MLFQ
**MLFQ sceduler**는 L0부터 Lk-1의 큐 k (2<=k<=5,k는 정수)개를 가지며 더 낮은 수의 큐가 더 높은 우선순위를 갖습니다. 각각의 큐는 우선순위를 가지고 독립적으로 작동하며 만약 해당 레벨의 큐가 선택되었을 때 그 전에 해당 레벨에서 선택 되었던 프로세스가 아직 그 큐에 남아있다면 그 프로세스를 선택해야 하고 그렇지 않다면 같은 레벨의 큐에 들어있는 프로세스들 중 우선순위가 가장 높은 프로세스를 선택합니다. 모든 프로세스들은 처음에 L0의 큐에 담겨 있고 이후 각각의 큐에 할당된 time quantum을 모두 소진할 경우 아래 단계의 큐로 내려가게 됩니다. 각각의 큐의 time quantum은 **2*(queue의 level)+4**입니다. 또한 모든 프로세스들이 가장 마지막 큐에서 time quantum을 모두 소진한 경우거나 100 ticks가 지난 후 또는 실행 시킬 수 있는 프로세스가 없는 경우에는 모든 프로세스들을 L0큐로 이동시키며 time quantum을 0으로 만드는 priority boosting이 진행됩니다.

따라서 현재 프로세스들이 들어 있는 가장 낮은 레벨의 큐는 어디인지 확인해주는 변수(lowestlevel), 그 레벨에서 가장 높은 우선순위를 확인해줄 변수(highestpriority), 해당 레벨에서 이전에 실행했고 아직 time quantum이 남아 있는 프로세스를 기억해줄 변수(previousproc), priority boosting을 해주기 위해 ticks를 세줄 변수(boostingcount)와 priority boosting을 수행 해줄 함수(isyield), 현재 프로세스가 정해진 time quantum을 모두 소진하거나 지금 레벨의 큐보다 더 높은 우선순위를 갖는 큐에 프로세스가 있어 다음 프로세스에게 yield 해주어야 하는가?를 체크해줄 함수(isyield)가 필요합니다.


------

## 2. 구현

### 2-1. MLQ

```c
#ifdef MULTILEVEL_SCHED
    struct proc * selectedproc = 0;
    uint selectedpid = -1;
    int rrcheck = 1;

    //select scheduler
    for(p = ptable.proc;p<&ptable.proc[NPROC];p++)
    {
      if(p->pid%2==0 && p->state==RUNNABLE)
      {
        rrcheck = 1;
        break;
      }
      rrcheck = 0;
    }

    if(rrcheck == 1) //round robbin scheduler
    {   
      for(p = ptable.proc;p<&ptable.proc[NPROC];p++)
      {
        if(p->state != RUNNABLE)
          continue;
        
        if(p->pid%2==0)
        {
          c->proc = p;
          switchuvm(p);
          p->state = RUNNING;

          swtch(&(c->scheduler), p->context);
          switchkvm();
          
          c->proc = 0;
        }
      }
    }
    else if(rrcheck == 0) //fcfs scheduler
    {
      for(p = ptable.proc;p<&ptable.proc[NPROC];p++) //select process
      {
        if(p->pid%2 != 0)
        {
          if(p->state != RUNNABLE)
            continue;

          if(p->pid < selectedpid)
          {
            selectedpid = p->pid;
            selectedproc = p;
          }
        }
        
      }
      if(selectedproc != 0)
      {
        c->proc = selectedproc;
        switchuvm(selectedproc);
        selectedproc->state = RUNNING;

        swtch(&(c->scheduler), selectedproc->context);
        switchkvm();

        c->proc = 0;

      }
    }
```

make option에서 SCHED_POLICY = FCFS_SCHED인 경우 해당 부분만을 수행합니다. ptable에 있는 프로세스를 0번부터 검사하며 pid가 짝수인 프로세스가 있는지 검사합니다. 검사를 마친 후 짝수인 프로세스가 있다면 rrcheck는 1이 될 것이고 그렇지 않다면 0이 될 것입니다. 짝수인 프로세스가 있다면 rr정책을 채택해 짝수인 프로세스들을 먼저 선택해줘야 하므로 또 다시 ptable을 읽으며 짝수인 프로세스를 발견한 뒤 선택 해줍니다. 만약 rrcheck가 0이면 짝수인 프로세스가 없다는 뜻이고 ptable을 읽으며 홀수인 프로세스들 중 가장 pid가 작은 프로세스를 찾아 선택해줍니다. 이 선택된 프로세스가 NULL이 아니면 실행됩니다.


#### 2-2. MLFQ

```c
#elif MLFQ_SCHED
    struct proc * selectedproc = 0;
    uint lowestlevel = 1000;
    int highestpriority = -1;

  // select level
  for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
      if(p->state != RUNNABLE)
        continue;
      
      if(p->queuelevel < lowestlevel){
        lowestlevel = p->queuelevel;
      }
  }
    
  if(previousproc[lowestlevel] && (previousproc[lowestlevel]->state == RUNNABLE)){
    selectedproc = previousproc[lowestlevel];
  }

  else{
    for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
      if(p->state != RUNNABLE)
        continue;

      if( (p->queuelevel == lowestlevel) && (p->priority > highestpriority) ){
        highestpriority = p->priority;
        selectedproc = p;
      }
    }
  }

   if(selectedproc != 0){
    c->proc = selectedproc;
    switchuvm(selectedproc);
    selectedproc->state = RUNNING;
    
    swtch(&(c->scheduler), selectedproc->context);
    switchkvm();

    c->proc = 0;
   }
```

make option에서 SCHED_POLICY = MLFQ_SCHED인 경우 해당 부분만을 수행합니다. 지금 프로세스가 있는 가장 높은 우선순위의 큐를 찾고 만약 그 레벨의 큐에서 전에 실행되었으며 time quantum을 모두 소진 하지 못한 프로세스가 있다면 그 프로세스를 선택합니다. 만약 그러한 프로세스가 없다면 해당 레벨에 들어있는 프로세스들 중 priority가 가장 높은 프로세스를 선택합니다. 이후 선택된 프로세스가 NULL이 아니면 실행됩니다.

#### 2-3. isyield

```c
int
isyield(void){
  acquire(&ptable.lock);
  struct proc * p;
  int yieldflag = 0;
  int boostingflag = 1;

  myproc()->timequantum++;
  boostingcount++;

  //increase queue level and yield
  if(myproc()->timequantum >= (myproc()->queuelevel*2+4) ){
    previousproc[myproc()->queuelevel] = 0;

    if( myproc()->queuelevel<MLFQ_K-1 ){
      myproc()->queuelevel++;
      myproc()->timequantum = 0 ;
    }
    yieldflag = 1;
  }
  
  //there is some process in low level queue
  else{
    for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
      if(p->state != RUNNABLE)
        continue;

      if( (p->queuelevel < myproc()->queuelevel) ){
        previousproc[myproc()->queuelevel] = myproc();
        yieldflag = 1;
      }
      
    }
  }

  //priority boosting
  for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
    if(p->state != RUNNABLE){
      boostingflag = 1;
      continue;
    }
    if( (p->queuelevel != MLFQ_K-1) || (p->timequantum < 2*(MLFQ_K-1)+4) ){
      boostingflag = 0;
      break;
    }
  }

  if( (boostingcount >= 100) || (boostingflag == 1) )
  {
      boostingcount = 0;
      for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
        p->queuelevel = 0;
        p->timequantum = 0;   
      }

      for(int i=0; i<MLFQ_K; i++){
        previousproc[i] = 0;
      }
  }


  release(&ptable.lock);

  return yieldflag;
}

```

선택된 프로세스가 계속 프로세서를 사용해도 되는지 혹은 다른 프로세스에게 프로세서를 양보해야 하는지를 결정합니다. timer interrupt가 발생 했을 때마다 실행되며 해당 함수의 yieldflag 값이 1이면 다른 프로세스에게 프로세서를 양보하고 yieldflag 값이 0이라면 해당 프로세스가 계속해서 프로세서를 사용합니다. 구체적으로 프로세스가 프로세서를 양보해야 하는 경우는 해당 레벨의 time quantum을 모두 소진한 경우(방금 실행된 프로세스가 다른 레벨의 큐로 내려갈 것이니 해당 레벨의 previous process를 0으로 만들어줌),마지막 레벨에서 time quantum을 모두 소진한 경우(이 프로세스가 다시 선택되면 안되니 해당 레벨의 previous prcess를 0으로 만들어줌), 자신보다 높은 우선순위를 갖는 큐에 프로세스가 존재하는 경우(이 레벨의 큐에 다시 돌아 왔을 때 이 프로세스가 선택되어야 하니 해당레벨의 previous process에 현재 프로세스 저장)입니다. 이 경우가 아니라면 0을 리턴해 yield를 실행하지 않고 계속해서 이 프로세스가 실행됩니다.

또한 yieldflag의 값이 결정되었으면 priority boosting이 실행되어야 하는지 확인합니다.실행 수 있는 프로세스가 없을때 boostingflag는 1로 설정됩니다. 또한 100 ticks 마다 boosting을 해줘야하므로 boostingcount를 이용해 100 ticks를 측정합니다. 이후 boostingcount 가 100을 넘거나 boostingflag가 1이라면 모든 프로세스의 queuelevel, timequantum을 0으로 만들어주고 preivousproc 역시 0으로 비워주어 bossting을 실행하게 됩니다.



아래 코드는 trap.c에서 실제로 isyield가 사용되는 방식입니다.
```c
  #ifdef MLFQ_SCHED

    //change process's state
    if(myproc() && myproc()->state == RUNNING &&
      tf->trapno == T_IRQ0+IRQ_TIMER){
      
      if(isyield())
        yield();  
  }
```


#### 2-4. systemcall(yield, getlev, setpriority)

```c
void
sys_yield(void)
{
  
  struct proc * p = myproc();
  p->queuelevel = 0;
  p->timequantum = 0;
  yield();
  
}

void
yield(void)
{
  acquire(&ptable.lock);  //DOC: yieldlock
  myproc()->state = RUNNABLE;
  sched();
  release(&ptable.lock);
}
```

우리가 원하는 기능을 구현하기 위해 기존의 yield 함수를 이용하는 새로운 시스템콜 yield를 만듭니다. 시스템콜 yield가 호출되었다면 해당 프로세스는 작업을 마친 것으로 간주되어 queuelevel, timequantum 모두 0으로 초기화 됩니다.


```c
int
sys_getlev(void)
{
    return getlev();
}

int
getlev(void)
{
  uint level = myproc()->queuelevel;

  return level;
}
```

우리가 원하는 기능을 구현하기 위해 새로운 시스템콜인 getlev을 만듭니다.
본 함수는 실행되고 있는 프로세스의 queuelevel을 반환합니다.


```c
int
sys_setpriority(void)
{
    int pid;
    int priority;
    if(argint(0,&pid)!=0||argint(1,&priority)!=0)
        return -1;
    return setpriority(pid,priority);
}

int
setpriority(int pid, int priority)
{
  struct proc *p;
  int pidflag = 0;

  //if prioiry num is wrong
  if( (priority<0) || (priority>10) )
    return -2;

  acquire(&ptable.lock); 
  for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
        if( (p->pid == pid) && (p->parent->pid == myproc()->pid) )
        {
            pidflag = 1;
            p->priority = priority;
            break;
        }
    }
  release(&ptable.lock); 

  //setpriority is succeess
  if(pidflag == 1)
    return 0;
  //no process or not child
  else
    return -1;
    
}
```

우리가 원하는 기능을 구현하기 위해 새로운 시스템콜인 setpriority를 만듭니다.
본 함수는 실행되고 있는 프로세스의 priority를 변경합니다.
만약 인자로 들어온 priority 값이 올바른 priority 값이 아니라면 -2를, 해당 함수로 바꾸고자 하는 프로세스가 자식 프로세스가 아니라면 -1를, 정상적으로 실행 되었다면 0을 리턴합니다.


#### 2-5. etc

```c
static struct proc*
allocproc(void)
{
  struct proc *p;
  char *sp;

  acquire(&ptable.lock);

  for(p = ptable.proc; p < &ptable.proc[NPROC]; p++)
    if(p->state == UNUSED)
      goto found;

  release(&ptable.lock);
  return 0;

found:
  p->state = EMBRYO;
  p->pid = nextpid++;

  p->priority = 0;
  p->queuelevel = 0;
  p->timequantum = 0;

  release(&ptable.lock);
```

위의 코드는 처음 프로세스의 priority, queuelevel, timequantum을 초기화 하는 부분입니다.

아래 코드는 해당 큐 레벨에서 이전에 실행되었던 프로세스를 기억하기 위한 배열과 ticks가 100이 넘어 boosting을 해야 하는지를 카운트 해주는 변수입니다.

```c
struct proc * previousproc[MLFQ_K] ={0};
uint boostingcount = 0 ;
```


----

## 3. 실행결과

### 3.1 MLFQ Test
#### Test 1
![ml1](/uploads/69829e03c8a60cad297348979d192538/ml1.png)
![ml2](/uploads/89703b4406d911cb2a6d00c47f72fd2d/ml2.png)
아무런 조건 없이 MLQ 자체의 조건만으로 실행됩니다. 짝수인 프로세스(6,8)이 rr에 따라 번갈아 가며 실행되고 짝수인 프로세스들이 모두 종료된 뒤에 홀수인 프로세스(5,7)이 fcfs에 따라 실행되어 pid가 작은 5가 먼저 실행되고 종료된 후에 pid가 7인 프로세스가 실행되고 종료됩니다.

#### Test 2
![ml3](/uploads/aeefbf9339dc3d1d82e3ceac096ff233/ml3.png)
각 프로세스가 서로 계속 yield를 하지만 짝수인 프로세스(6,8)가 존재하면 홀수인 프로세스가 실행되지 않으므로 짝수인 프로세스들이 먼저 실행되고 끝납니다. 이때 스케쥴링 정책은 rr이므로 두 프로세스는 거의 동시에 끝나게 됩니다. 짝수인 두 프로세스가 끝나고 나서 홀수인 프로세스(5,7)가 실행되는데 스케쥴링 정책은 fcfs 이므로 pid가 5인 프로세스가 먼저 끝나고 이후에 pid가 7인 프로세스가 끝나게 됩니다.

#### Test 3
![ml4](/uploads/15a277e44aa7ad0759b8b127452a596a/ml4.png)
각 프로세스들이 일정시간동안 sleep 하며 자신을 출력합니다. 이 역시도 짝수인 프로세스(6,8)들이 모두 실행되었다가 sleep 상태에 들어가야지 홀수인 프로세스(5,7)가 실행 될 수 있습니다. 홀수인 프로세스가 실행 될 때 fcfs 정책을 따르므로 pid가 5인 프로세스가 먼저 실행되고 그 후 pid가 7인 프로세스가 실행됩니다. 


### 3.2 MLFQ Test (첫번째 사진은 K=2, 두번째 사진은 K=5 입니다.)
#### Test 1
![mlfq2-1](/uploads/c052e9834f9598c3e74b4142cf9867ca/mlfq2-1.png)
![mlfq5-1](/uploads/1326e5d1191b9c692e876ea3c24ecfe4/mlfq5-1.png)
프로세스들은 주어진 time quantum을 모두 사용하고 비슷한 시기에 낮은 레벨로 내려가기 때문에 서로 비슷한 시기에 끝나게 되고 실제로 각 프로세스들의 결과가 출력되는 시간도 거의 동일합니다.

#### Test 2
![mlfq2-2](/uploads/9135f6dbcb9447e242e2335090459501/mlfq2-2.png)
![mlfq5-2](/uploads/22d0d2e642468f6d5561f8291f4249a1/mlfq5-2.png)
pid가 높은 프로세스에게 더 높은 우선순위를 주었기 때문에 pid가 높은 프로세스가 더 일찍 끝날 가능성이 큽니다. 실제로 test프로그램을 여러번 돌려 보았을 때 pid가 높은 프로세스가 대체적으로 일찍 끝나고 출력이 되는 것을 볼 수 있습니다.

#### Test 3
![mlfq2-3](/uploads/31a8f13bd61df89bb576675346006ba8/mlfq2-3.png)
![mlfq5-3](/uploads/e5af6295dde2b1541a72adf7d5763a6c/mlfq5-3.png)
pid가 큰 프로세스에게 높은 우선순위를 부여하고 각 프로세스는 계속 시스템콜 yield를 실행해 queuelevel과 timequantum을 초기화 시키므로 pid가 큰 프로세스 순으로 먼저 끝나 먼저 출력되는 결과를 확인 할 수 있습니다.

#### Test 4
![mlfq2-4](/uploads/ebe98a4fe3090ca551bc0e670799e765/mlfq2-4.png)
![mlfq5-4](/uploads/d06e5762cf6a72e36c10e772711bee4b/mlfq5-4.png)
pid가 더 큰 프로세스에게 높은 우선순위를 부여하고 각 프로세스는 계속 시스템콜 sleep을 실행해 queuelevel과 timequantum을 초기화 합니다. 하지만 Test 3와는 다른데 sleep 되어 있는 동안 다른 프로세스가 실행되기 때문에 순차적으로 끝나 순차적으로 출력되는 것이 아닌 비슷한 시기에 끝나 거의 동시에 결과가 출력됩니다.

#### Test 5
![mlfq2-5](/uploads/09e4630bfd63dfc59590beea2c57e44a/mlfq2-5.png)
![mlfq5-5](/uploads/7ba37def9b95089347a5ff51fa15be82/mlfq5-5.png)
각 프로세스들은 자신이 있을 수 있는 가장 낮은 레벨(큐의 숫자가 큰)의 큐가 정해져 있고 그 큐 안에서만 작업을 하게 됩니다. 따라서 더 높은 레벨에만 머무는 프로세스들이 먼저 작업을 마치게 되는데 pid가 작을수록 있을 수 있는 가장 낮은 레벨이 작아지기 때문에 pid가 작은 프로세스들이 먼저 일을 마치고 출력됩니다.

#### Test 6
![mlfq2-6](/uploads/b2d46215c9b45ecbae4ead802d8b7136/mlfq2-6.png)
![mlfq5-6](/uploads/352dec4b0a01edb66abd846bbf80a0ed/mlfq5-6.png)
구현한 시스템콜 setpriority가 정상적으로 작동하는지 확인하는 부분입니다.
정상적으로 구현했다면 wrong 메세지가 아닌 done 메세지 하나를 출력하고 종료합니다.

----

## 4. 트러블슈팅


### 4-1. systemcall
기존 실습 수업에서 배웠던 대로 필요하다면 함수를 만들고 이렇게 만든 함수들과 기존의 함수들을 wrapper function으로 감싸 새로운 시스템콜을 만들었습니다. 연습을 많이 했던 부분이라 어려운 부분은 없었지만 setpriority 함수가 제대로 작동하는지 확인하기 위해 getpriority라는 추가적인 시스템콜을 만들고(코드에서는 삭제되었지만) test 프로그램의 코드를 수정하여 시스템 콜이 제대로 동작하는지 확인해보았습니다. 


### 4-2. MLQ
기존 실습에서 만져보았던 코드부분이 아니라 처음 구현 할 땐 어느 코드를 어떻게 만져야 할지 막막했습니다. 하지만 실습수업에서 알려주신 실행 흐름을 따라가며 함수들을 이해했고 구현하였습니다.
평소 c언어를 잘 사용하지 않아 자료형이나 함수들에 대한 새로운 이해가 필요했지만 좋은 공부가 된 것 같습니다.


### 4-3. MLFQ
MLFQ는 MLQ와는 달리 요구되는 조건과 기능이 복잡했기에 차근차근 구현을 진행했어야 했습니다.
그래서 scheduler에 

```c
 for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
      if(p->state != RUNNABLE)
        continue;

      if( (p->queuelevel == lowestlevel) && (p->priority > highestpriority)  ){
        highestpriority = p->priority;
        selectedproc = p;
      }
    }
```

이와 같이 이전에 실행되었던 프로세스를 생각하지 않고 단순히 가장 높은 우선순위의 큐에서 가장 높은 우선순위를 갖는 프로세스를 찾아 스케쥴링 하는 방법을 구현해놓고 나머지 기능을 완성 시킨 후 이전에 실행되었던 프로세스까지 생각하는 방식으로 접근하였습니다. 

이후 이전 MLQ와는 다르게 time quantum이라는 개념이 도입되었기 때문에 구체적으로 yield가 일어나는 시점과 yield를 할지 말지에 대해 알맞게 처리해줘야 한다는 것을 깨달았습니다. 따라서 trap.c에서 무조건 yield 하는 부분을 조건이 맞았을 때에만 yield 하도록 구조를 바꾸었고 이 조건을 체크해주기 위해 isyield라는 함수를 만들었습니다. yield 하는 경우는 나의 time quantum을 모두 사용 하였을 때, 내가 속한 큐의 레벨보다 더 높은 우선순위를 가지는 큐에 프로세스가 존재 할 때, 내가 가장 낮은 우선순위의 큐에 있고 time quantum을 모두 사용했을 때 입니다.

 이후 priorityboosting을 위한 함수를 작성했는데 기존에는 isyield함수와 별개의 함수로서 작성했습니다. 그랬더니 boosting을 체크하는 시점이 애매해졌고 하나의 코드에 합쳐서 작성하는 것이 코드를 이해하는 입장에서도 덜 복잡할 것 같아 하나의 함수로 합쳤습니다. boosting 함수는 100 ticks일때마다, 모든 프로세스가 가장 낮은 우선순위의 큐에서 time quantum을 모두 사용 되었을 때마다 실행되어야 하는데 여기서 실수 하기 쉬운 점이 실행될 수 있는 프로세스가 없는 경우에도 실행이 되어야 한다는 것입니다. 따라서 프로세스 테이블을 읽으며 RUNNABLE한 프로세스가 없을 경우에도 boosting을 하게끔 만들어 주었습니다.

```c
 for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
    if(p->state != RUNNABLE){
      boostingflag = 1;
      continue;
    }
    if( (p->queuelevel != MLFQ_K-1) || (p->timequantum < 2*(MLFQ_K-1)+4) ){
      boostingflag = 0;
      break;
    }
  }
  ```

이후 필요한 기능들을 구현한 후 앞서 scheduler함수에서 구현하지 못했던 previous process를 고려하는 방법을 구현하였습니다. 코드를 구현하고 xv6를 실행시켰더니 init과 shell이 계속 번갈아 가며 실행되었고 test 프로그램을 돌릴 수 없었습니다. 원인을 분석했고 

```c
if(previousproc[lowestlevel] && (previousproc[lowestlevel]->state == RUNNABLE)){
    selectedproc = previousproc[lowestlevel];
  }
```
에서 if문 안에 previousproc의 상태값이 RUNNABLE인지 확인을 해주어야 한다는 것을 깨달았습니다.


이후 구현을 마쳤고 K= 2,3,4,5에 대해 테스트 프로그램을 돌려보았더니 K=3,4,5에 대해선 문제 없이 돌아가지만 K가 2일때 종종 trap 14 에러를 뱉으며 xv6가 멈추는 것을 확인하였습니다. 하지만 항상 그러는 것이 아니라 테스트 프로그램을 연속적으로 빠르게 실행 시켰을 때 등에만 그러는 것을 확인하였습니다. 