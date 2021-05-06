# Project2 - Advanced Process Management
## 목차
1. 디자인
   1. Administrator Mode
   2. Custom Stack Size
   3. Per-process Memory Limit
   4. Shared Memory
   5. Process Manager
2. 구현
   1. Administrator Mode
   2. Custom Stack Size
   3. Per-process Memory Limit
   4. Shared Memory
   5. Process Manager
3. 실행 결과
   1. Administrator Mode
   2. Custom Stack Size
   3. Per-process Memory Limit
   4. Shared Memory
   5. Process Manager
4. 트러블슈팅

------

## 1. 디자인

### 1-1. Administrator Mode
**Administrator Mode**는 해당 프로세스의 mode를 나타냅니다. mode는 int 값이며 mode가 3일 경우 평범한 user mode라는 의미이고 mode가 0일 경우 administrator mode라는 의미입니다.

이후 getadmin 시스템 콜을 통해 해당 process는 일반 user mode에서 administrator mode로 mode를 변경 할 수 있습니다.

### 1-2. Custom Stack Size
**Custom Stack Size**는 프로세스의 스택용 페이지를 1개 할당받는 방식이 아닌 프로세스의 스택용 페이지를 원하는 개수만큼 할당받습니다.

이후 기존의 exec 시스템 콜이 아닌 exec2 시스템 콜을 통해 3번째 인자로 원하는 개수를 넘겨주어 원하는 개수 만큼의 스택용 페이지를 가지고 있는 프로세스를 만듭니다.
exec과 exec2 또다른 차이점으로는 exec은 어느 프로세스에서나 실행 할 수 있지만 exec2는 administrator mode에서만 실행 할 수 있다는 점이 있습니다.

### 1-3. Per-process Memory Limit
**Per-process Memory Limit**는 프로세스가 할당받을 수 있는 메모리의 최대치를 지정합니다. 즉 limit을 걸어주지 않았다면 프로세스에는 제한이 없지만 이후 limit을 걸어주어 그 limit을 넘는 메모리를 추가로 할당받지 못하도록 합니다.

이후 setmemorylimit 시스템 콜을 통해 인자로 받은 pid에 해당하는 함수에 memory limit을 걸어주게됩니다.

### 1-4. Shared Memory
**Shared Memory**는 프로세스 당 공유할 수 있는 메모리 페이지를 하나씩 만듭니다. 이 shared memory는 어떤 프로세스든 read는 가능하지만 write는 해당 프로세스만 가능합니다. 또한 이 shared memory는 구조체의 sz 크기에는 반영되지 않습니다.

이후 getshmem 함수를 통해 인자로 넘겨 받은 pid의 shared memory에 접근 할 수 있습니다.


------

## 2. 구현

### 2-1. Administrator Mode

```c
int pmode;  //in proc.h, struct proc

int         //in proc.c  
getadmin(char*password){
  if(!strcmp(password,"2016025105")){
    myproc()->pmode = 0;
    return 0;
  }
  else{
    return -1;
  }
}

int        //in sysproc.c
sys_getadmin(void)
{
    char * password;
    if(argstr(0,&password)== -1)
        return -1;
    return getadmin(password);
}
```

getadmin 시스템 콜을 통해 해당 process는 administrator mode를 획득합니다. getadmin의 인자로 들어온 값이 **2016025105**와 같을 경우에만 mode를 획득하게 되고 이와 다른 password가 들어온다면 mode를 획득하지 못합니다.


#### 2-2. Custom Stack Size

```c
int
exec2(char *path, char **argv, int stacksize)
{
  char *s, *last;
  int i, off;
  uint argc, sz, sp, ustack[3+MAXARG+1];
  struct elfhdr elf;
  struct inode *ip;
  struct proghdr ph;
  pde_t *pgdir, *oldpgdir;
  struct proc *curproc = myproc();


  begin_op();

  if( myproc()->pmode != 0 || stacksize<1 || stacksize>100 ){
    end_op();
    return -1;
  }
  curproc->pmode = 3;
  curproc->mlimit = 0;
  curproc->s_size = stacksize;

  if(curproc->shmem != 0){
    kfree(curproc->shmem);
  }

  curproc->shmem = kalloc();
  memset(curproc->shmem, 0, PGSIZE);

  if((ip = namei(path)) == 0){
    end_op();
    cprintf("exec: fail\n");
    return -1;
  }
  ilock(ip);
  pgdir = 0;

  // Check ELF header
  if(readi(ip, (char*)&elf, 0, sizeof(elf)) != sizeof(elf))
    goto bad;
  if(elf.magic != ELF_MAGIC)
    goto bad;

  if((pgdir = setupkvm()) == 0)
    goto bad;

  // Load program into memory.
  sz = 0;
  for(i=0, off=elf.phoff; i<elf.phnum; i++, off+=sizeof(ph)){
    if(readi(ip, (char*)&ph, off, sizeof(ph)) != sizeof(ph))
      goto bad;
    if(ph.type != ELF_PROG_LOAD)
      continue;
    if(ph.memsz < ph.filesz)
      goto bad;
    if(ph.vaddr + ph.memsz < ph.vaddr)
      goto bad;
    if((sz = allocuvm(pgdir, sz, ph.vaddr + ph.memsz)) == 0)
      goto bad;
    if(ph.vaddr % PGSIZE != 0)
      goto bad;
    if(loaduvm(pgdir, (char*)ph.vaddr, ip, ph.off, ph.filesz) < 0)
      goto bad;
  }
  iunlockput(ip);
  end_op();
  ip = 0;

  // Allocate pages (gard + customstack) at the next page boundary.
  sz = PGROUNDUP(sz);
  if((sz = allocuvm(pgdir, sz, sz + (1)*PGSIZE)) == 0)
    goto bad;
  sp = sz;

  sz = PGROUNDUP(sz);
  if((sz = allocuvm(pgdir, sz, sz + (stacksize)*PGSIZE)) == 0)
    goto bad;
  sp = sz;
  clearpteu(pgdir, (char*)(sz - (1+stacksize)*PGSIZE));


  // Push argument strings, prepare rest of stack in ustack.
  for(argc = 0; argv[argc]; argc++) {
    if(argc >= MAXARG)
      goto bad;
    sp = (sp - (strlen(argv[argc]) + 1)) & ~3;
    if(copyout(pgdir, sp, argv[argc], strlen(argv[argc]) + 1) < 0)
      goto bad;
    ustack[3+argc] = sp;
  }
  ustack[3+argc] = 0;

  ustack[0] = 0xffffffff;  // fake return PC
  ustack[1] = argc;
  ustack[2] = sp - (argc+1)*4;  // argv pointer

  sp -= (3+argc+1) * 4;
  if(copyout(pgdir, sp, ustack, (3+argc+1)*4) < 0)
    goto bad;

  // Save program name for debugging.
  for(last=s=path; *s; s++)
    if(*s == '/')
      last = s+1;
  safestrcpy(curproc->name, last, sizeof(curproc->name));

  // Commit to the user image.
  oldpgdir = curproc->pgdir;
  curproc->pgdir = pgdir;
  curproc->sz = sz;
  curproc->tf->eip = elf.entry;  // main
  curproc->tf->esp = sp;
  switchuvm(curproc);
  freevm(oldpgdir);
  return 0;

 bad:
  if(pgdir)
    freevm(pgdir);
  if(ip){
    iunlockput(ip);
    end_op();
  }
  return -1;
}

```

custom stack size를 구현하기 위해 만든 exec2 함수입니다. 기존의 exec과 동작은 거의 같지만

```c
sz = PGROUNDUP(sz);
  if((sz = allocuvm(pgdir, sz, sz + (1)*PGSIZE)) == 0)
    goto bad;
  sp = sz;

  sz = PGROUNDUP(sz);
  if((sz = allocuvm(pgdir, sz, sz + (stacksize)*PGSIZE)) == 0)
    goto bad;
  sp = sz;
  clearpteu(pgdir, (char*)(sz - (1+stacksize)*PGSIZE));
  ```

  이 부분에서 세번째 인자로 들어온 수만큼 스택용 페이지를 할당하는 것을 볼 수 있습니다.

#### 2-3. Per-process Memory Limit

```c
int mlimit;  //in proc.h, struct proc

int          //in proc.c
setmemorylimit(int pid, int limit){
  struct proc * p;
  int pidflag = 1;
  int r = -1; //return value
  
  
  
  acquire(&ptable.lock);
  for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
    if(p->pid == pid){
      pidflag = 0;
      break;
    }
  }

  if(pidflag || limit < 0 || myproc()->pmode != 0){
    r = -1;
  }

  else{
    if( p->sz > limit ){ 
      if( limit == 0 ){
        p->mlimit = 0;
        r = 0;
      }
      else{
        r = -1;
      }
    }

    else{
      p->mlimit = limit;
      r = 0;
    }
    
  }
  release(&ptable.lock);

  return r;

}

int          //in sysproc.c
sys_setmemorylimit(void)
{
    int pid;
    int limit;
    if(argint(0,&pid)!=0||argint(1,&limit)!=0)
        return -1;
    return setmemorylimit(pid,limit);
}
```

setmemorylimit 시스템 콜을 통해 해당 프로세스가 할당 받을 수 있는 memory의 limit을 정해줍니다. ptable을 돌며 인자로 받은 pid에 해당하는 프로세스를 찾고 그 프로세스가 존재하는지, 인자로 들어온 limit 값이 음수가 아닌지, setmemorylimit을 실행하는 process의 mode가 administrator mode가 맞는지 확인한 후에 해당 pid의 프로세스의 memory limit을 정해줍니다.

이후 아래의 코드에서 볼 수 있듯이 growproc 함수 도중 현재 사용중인 memory(sz)와 새로 할당받을 메모리(n)의 합이 limit 보다 크다면 더이상 메모리를 할당하지 않고 return 합니다.

```c
int
growproc(int n)
{
  uint sz;
  struct proc *curproc = myproc();
  sz = curproc->sz;

  if( curproc->mlimit != 0 ){
    if( curproc->mlimit < (sz+n) ){
      return -1;
    }
  }

  if(n > 0){
    if((sz = allocuvm(curproc->pgdir, sz, sz + n)) == 0)
      return -1;
  } else if(n < 0){
    if((sz = deallocuvm(curproc->pgdir, sz, sz + n)) == 0)
      return -1;
  }
  curproc->sz = sz;
  switchuvm(curproc);
  return 0;
}
```

#### 2-4. Shared Memory

```c
char * shmem;    //in proc.h, struct proc

np->shmem = kalloc();  //in proc.c, fork
memset(np->shmem, 0, PGSIZE);


if(curproc->shmem != 0){    //in exec.c, exec, exec2
    kfree(curproc->shmem);
  }

curproc->shmem = kalloc();
memset(curproc->shmem, 0, PGSIZE);


extern pte_t * walkpgdir(pde_t *pgdir, const void *va, int alloc); //in proc.c

char * getshmem(int pid){   //in proc.c
  struct proc * p;
  int tpid = pid;
  char * add = 0;
  uint * pa = 0;

  acquire(&ptable.lock);
  for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
    if(p->pid == tpid){
      add = p->shmem;
      break;
    }      
  }
  release(&ptable.lock);


  if(tpid == myproc()->pid){
    pa = walkpgdir(myproc()->pgdir, (char*)PGROUNDDOWN((uint)add), 1);
    *pa = V2P(add) | PTE_P | PTE_U | PTE_W ;
    
    return add;
  }

  else{
    
    pa = walkpgdir(myproc()->pgdir, (char*)PGROUNDDOWN((uint)add), 1);
    *pa = V2P(add) | PTE_P | PTE_U ;
    return add;
  }


  return add;
}


char *       //in sysproc.c
sys_getshmem(void){
  int pid;
  
  if(argint(0,&pid)!=0)
      return 0;
  else
  {
    return getshmem(pid);
  }
  
}
```

getshmem 시스템 콜을 통해 인자로 들어온 pid에 해당하는 프로세스의 shared memory 주소 값을 받습니다. fork와 exec, exec2 호출로 새로운 프로세스를 만들 때 프로세스의 shmem에 kalloc으로 페이지를 할당합니다. 이후 getshmem 시스템 콜을 통해 shared memory에 접근하려고 하면 walkpgdir와 V2P, PTE....  등으로 적절한 권한을 부여 한 뒤 해당 주소를 return 합니다.

walkpgdir 함수는 vm.c에 있는 함수로 static 키워드를 제거하고 proc.c에서 extern 키워드를 이용해 proc.c에서 사용하였습니다.

#### 2-5. Process Manager

```c
int
main(void)
{
  static char buf[100];
  int fd;

  int  f;
  while( (f=getadmin("2016025105"))  == -1){
  }

  // Ensure that three file descriptors are open.
  while((fd = open("console", O_RDWR)) >= 0){
    if(fd >= 3){
      close(fd);
      break;
    }
  }
  
  while( 1 ){
    while(getcmd(buf, sizeof(buf))!=0);
    runcmd(parsecmd(buf));
  }
  
  exit();
}
```

process manager라는 유저 프로그램을 만들기 위해 pmanager.c 파일을 작성합니다. pmanager.c는 기존 쉘의 c파일인 sh.c의 내용을 많이 가져왔습니다. pmanager는 실행시 getadmin으로 mode를 administrator mode로 변경해주고 이후 계속해서 getcmd로 명령어를 받아오고 parsecmd로 명령어를 인자 단위로 나눠준 뒤 runcmd로 명령어를 수행합니다.

```c
int
getcmd(char *buf, int nbuf)
{
  printf(2, ">> ");
  memset(buf, 0, nbuf);
  gets(buf, nbuf);
  if(buf[0] == 0) // EOF
    return -1;
  return 0;
}
```

```c
#define LIST  1
#define KILL  2
#define EXEC  3
#define MEML  4
#define EXIT  5

struct cmd {
  int type;
  char * argv1;
  char * argv2;
};

struct cmd*
parsecmd(char *s)
{
  struct cmd* tmp;
  tmp = malloc(sizeof(*tmp));
  memset(tmp,0,sizeof(*tmp));
  char cmd[100];
  char arg1[100];
  char arg2[100];
  memset(cmd,0,sizeof(cmd));
  memset(arg1,0,sizeof(arg1));
  memset(arg2,0,sizeof(arg2));
  int i = 0;
  int argf = 0;
  char * p = cmd;

  while( 1 ){
      if(s[i] == '\n'){
          *p = s[++i];
          break;
      }
      if(s[i] == ' '){
          if(argf == 0 ){
              p = arg1;
              argf = 1 ;
          }
          else{
              p = arg2;
          }
          i++;
          continue;
      }
      *p = s[i];
      p++;
      i++;
  }

  if( !strcmp(cmd, "list") ){
      tmp->type = 1;
  }

  if( !strcmp(cmd, "kill") ){
      tmp->type = 2;
      tmp->argv1 = arg1;
  }

  if( !strcmp(cmd, "execute") ){
      tmp->type = 3;
      tmp->argv1 = arg1;
      tmp->argv2 = arg2;
  }

  if( !strcmp(cmd, "memlim") ){
      tmp->type = 4;
      tmp->argv1 = arg1;
      tmp->argv2 = arg2;
  }

  if( !strcmp(cmd, "exit") ){
      tmp->type = 5;
  }

  

  return tmp;
}
```

parsecmd는 입력받은 명령어를 인자 단위로 쪼개주어 구조체 cmd 변수 안에 넣어준 뒤 return 합니다.

```c
void
runcmd(struct cmd *cmd)
{
  
  if(cmd == 0)
    exit();

  switch(cmd->type){
  default:
    free(cmd);
    panic("runcmd");

  case LIST:
    free(cmd);
    showlist();
    break;

  case KILL:
    if(kill(atoi(cmd->argv1)) == 0){
        printf(2, "kill success!\n");
        wait();
    }
    free(cmd);
    break;

  case EXEC:
    if( fork1() == 0){
      if(exec_(cmd->argv1,&(cmd->argv1),atoi(cmd->argv2)) == 0){
        free(cmd);
      }
      else{
          printf(2, "EXEC fail!\n");
          free(cmd);
      }
    }
    break;
    
  case MEML:
    if( setmemorylimit(atoi(cmd->argv1),atoi(cmd->argv2)) == 0 ){
        printf(2, "setmemorylimit success!\n");
        free(cmd);
    }
    else{
        printf(2,"setmemorylimit fail!\n");
        free(cmd);
    }
    break;

  case EXIT:
    printf(2, "Bye!\n");
    free(cmd);
    exit();
    break;
  }
}
```
runcmd는 명령어를 쪼개어 저장한 변수를 인자로 받습니다. 이후 명령어를 실행합니다. 올바른 명령어가 들어오지 않았다면 pmanager는 panic을 실행하며 종료됩니다.

**list**

```c
int s_time;                //in proc.h, struct proc
int s_size;

void showlist(void){    //in proc.c
  struct proc * p;
      cprintf("Name       |  PID  |  TIME (ms)  |  MEMORY(bytes)  |  MEMLIM (bytes)  \n");
      
      acquire(&ptable.lock);
      for(p = ptable.proc; p < &ptable.proc[NPROC]; p++){
          if(p->state == UNUSED){
              continue;
          }
          cprintf("%s       |  %d  |  %d  |  %d  |  %d  \n",p->name,p->pid,(ticks-p->s_time),p->sz,p->mlimit);
      }
      release(&ptable.lock);

    cprintf("\n\n");
}

int      //in sysproc.c
sys_showlist(void){
  showlist();
  return 0;
}
```

명령어가 list였을 경우 showlist 시스템 콜을 호출합니다. showlist 시스템 콜은 ptable을 돌며 프로세스의 상태가 UNUSED라면 이름, pid, 실행시간, 사용중인 memory, 해당 프로세스의 memory limit을 출력합니다.

프로세스 실행시간은 allocproc 함수 안에서 프로세스가 만들어지는 시간을 s_time에 저장해두었다가 ticks에서 s_time을 빼서 구합니다.

**kill**
kill 시스템 콜을 호출하여 해당 pid의 프로세스를 죽입니다.

**execute**
path로 받은 프로그램을 exec2시스템 콜로 이용해서 실행시킵니다. fork후 자식프로세스에서만 exec2를 실행합니다.

**memlim**
setmemorylimit 시스템 콜을 통해 해당 pid에 해당하는 프로세스의 memory limit을 설정해줍니다.

**exit**
exit 시스템 콜을 통해 pmanager를 종료 시킵니다.

----

## 3. 실행결과

### 3.1 Administrator Mode ( p2_admin_test.c )
![admintest1](/uploads/a784a74f2124caf58baa4e99c469338b/admintest1.png)
![admintest2](/uploads/bd09f570e0d4b77d59e2290ec0742086/admintest2.png)
첫번째 사진은 getadmin 함수를 통해 Administrator mode로 mode 변경을 성공한 예이고 두번째 사진은 mode 변경을 실패한 예입니다.

### 3.2 Custom Stack Size ( p2_stack_test.c )
![stacktest1](/uploads/7ceff14752d79a8ecce5488d4a7a4c93/stacktest1.png)
![stacktest2](/uploads/3773dbd48f967774d38e2fd9628e05f6/stacktest2.png)
똑같은 프로그램을 stacksize를 1로, 3으로 실행시킵니다.
첫번째 사진이 1로 실행시킨 결과, 두번째 사진이 3으로 실행시킨 결과입니다.
stacksize가 많은 쪽이 얼마 더 많이 재귀를 실행 할 수 있기에 stacksize가 적은 쪽이 먼저 끝납니다.

### 3.3 Per-process Memory Limit ( p2_memory_tes.c )
![memlim](/uploads/480dda54a5cbbcefe755506413eea534/memlim.png)
![memlim1](/uploads/5068b1762d54f03863b29a2259703a96/memlim1)
![memlim2](/uploads/4ab764cef298b7658c6946f66535c154/memlim2)
프로세스에 memory limit을 겁니다.
memory limit에 성공한 경우입니다.
출력되는 memory 사용량은 동적 페이지 할당만 세기 때문에 출력되는 memory 사용량이 limit보다 조금 적어도 allocation이 fail 합니다.

![memlimfail](/uploads/4d04983412f5004d2ab3daea3a4b4faa/memlimfail)
프로세스에 memory limit을 걸었지만 현재 프로세스가 사용하고 있는 memory보다 limit 값이 적기 때문에 limit을 거는데 실패하고 프로세스는 계속해서 메모리를 할당 받습니다. 이후 list로 출력해보아도 memory limit이 여전히 0(무한) 인 것을 확인 할 수 있습니다.


### 3.4 Shared Memory ( p2_shmem_test.c )
![shmem](/uploads/5fe7900407c28dcf2f7635d5f9f5f303/shmem)
부모 프로세스가 자신의 shared memory에 내용을 작성하고 자식프로세스가 그 내용을 읽은 후 부모의 shared memory에 내용을 작성하려고 하지만 shared memory에 내용을 쓰는 행위는 자신의 shared memory에 대해서만 할 수 있기 때문에 자식 프로세스는 page fault를 내고 종료됩니다. 이후 부모 process는 내용을 한번 더 쓰고 종료됩니다.


### 3.5 Process Manager
#### list
![list1](/uploads/43bf73b5c20e3cd300b00958fdb8079f/list1)
list 명령어 수행 결과입니다.

#### execute
![execute](/uploads/a26e7bb992f6e39a3b0f44cc40309c0f/execute)
execute 명령어 수행 결과입니다.
p2_memory_test를 이용하였습니다.

#### kill
![kill](/uploads/4daf0d554dc6e992d928651a89561550/kill)
kill 명령어 수행 결과입니다.
이전에 사용했던 p2_memory_test를 이용하였습니다.

#### memlim
![memlim](/uploads/480dda54a5cbbcefe755506413eea534/memlim.png)
![memlim1](/uploads/5068b1762d54f03863b29a2259703a96/memlim1)
![memlim2](/uploads/4ab764cef298b7658c6946f66535c154/memlim2)
memlim 명령어 수행 결과입니다.
p2_memory_test를 이용하였습니다.

#### exit
![exit0](/uploads/982dd918805b43ce8a613036cea93076/exit0)
exit 명령어 수행 결과입니다.

#### Project02 Test Example
동영상 파일에서 설명해주신 예를 직접 해보았습니다.
![admintest1](/uploads/a784a74f2124caf58baa4e99c469338b/admintest1.png)
admintest 성공
![admintest2](/uploads/bd09f570e0d4b77d59e2290ec0742086/admintest2.png)
admintest 실패
![list1](/uploads/2af4b6eac727e54f7d9f869014d07fca/list1)
list 명령어 실행
![pmanagermemlim](/uploads/2bb1720a0e3423ec1898ec649e48a03e/pmanagermemlim)
pmanager의 memory limit 변경
![stacktest1](/uploads/a7647e031b3092ed40a8ce313218bb30/stacktest1)
stack size를 1로 주고 p2_stack_test 실행
![stacktest2](/uploads/175dd934ff477947ebe6c76293114f4d/stacktest2)
stack size를 3로 주고 p2_stack_test 실행
![memlimfail](/uploads/bf07d0752eeef6c5a67db44bc0ecbc44/memlimfail)
pmanager로 프로그램 실행 시키고 pmanager가 독립적으로 행동 할 수 있는지
p2_memory_test 실행시키고 list 명령어 실행
![memlim](/uploads/480dda54a5cbbcefe755506413eea534/memlim.png)
![memlim1](/uploads/5068b1762d54f03863b29a2259703a96/memlim1)
![memlim2](/uploads/4ab764cef298b7658c6946f66535c154/memlim2)
성공하는 memlim 명령어 실행 후 메모리 할당 실패했다는 메시지 출력
![memlimfail](/uploads/bf07d0752eeef6c5a67db44bc0ecbc44/memlimfail)
사용중인 메모리보다 더 작은 limit을 주어서 실패한 경우
![exit2](/uploads/4b55fc89c56950096954b5f2f5429820/exit2)
![exit3](/uploads/8f5ca07349d7cf4d573b0b19bb822b12/exit3)
exit후 다시 pmanager를 실행 시켰을때 이전 pmanager에서 종료된 프로그램이 남아있는지 확인
![shmem](/uploads/440c5e510ebe5d5e67753d322c23d22f/shmem)
p2_shmem_test가 정상적으로 작동하는지 확인

----

## 4. 트러블슈팅


### 4-1. Administrator Mode
처음 과제 명세를 보았을때 administrator mode 라는 것이 커널 모드로 설정해주라는 의미인줄 알고 커널 모드로 들어가는 코드를 살펴보았습니다. 하지만 문제를 찬찬히 이해해보니 그럴 필요가 없다는 것을 느끼고 시스템 콜로 구현하였습니다.


### 4-2. Per-process Memory Limit
프로세스의 limit 값을 설정해주는 것까진 쉬웠으나 이 limit 값을 어디서 어떻게 이용해야 할지 몰랐습니다. 그래서 코드들을 찬찬히 따라가 보았고 결국 growproc 부분에서 이 limit 값을 이용해 프로세스가 메모리를 할당 받을 수 있는지 체크해주면 된다는 것을 알았습니다.


### 4-3. Shared Memory
shared memory를 어떤식으로 구현하는 하는지는 어림짐작 하고 있었지만 원리에 대해서는 자세히 알지 못하고 있었습니다. kalloc으로 페이지를 할당 후 V2P와 P2V만으로 페이지에 권한을 주고 그 주소를 사용하려고도 해봤고 mappage 함수를 이용해야 하는지도 고민이 되었습니다. 하지만 찬찬히 코드들을 따라가 보았고 mappage안에서 walkpgdir 함수를 통해 권한을 주고 해당 주소를 사용하면 된다는 것을 깨달았고 그대로 구현을 해보니 정상적으로 작동하였습니다.


```c
static int
mappages(pde_t *pgdir, void *va, uint size, uint pa, int perm)
{
  char *a, *last;
  pte_t *pte;

  a = (char*)PGROUNDDOWN((uint)va);
  last = (char*)PGROUNDDOWN(((uint)va) + size - 1);
  for(;;){
    if((pte = walkpgdir(pgdir, a, 1)) == 0)
      return -1;
    if(*pte & PTE_P)
      panic("remap");
    *pte = pa | perm | PTE_P;
    if(a == last)
      break;
    a += PGSIZE;
    pa += PGSIZE;
  }
  return 0;
}

pte_t *
walkpgdir(pde_t *pgdir, const void *va, int alloc)
{
  pde_t *pde;
  pte_t *pgtab;

  pde = &pgdir[PDX(va)];
  if(*pde & PTE_P){
    pgtab = (pte_t*)P2V(PTE_ADDR(*pde));
  } else {
    if(!alloc || (pgtab = (pte_t*)kalloc()) == 0)
      return 0;
    // Make sure all those PTE_P bits are zero.
    memset(pgtab, 0, PGSIZE);
    // The permissions here are overly generous, but they can
    // be further restricted by the permissions in the page table
    // entries, if necessary.
    *pde = V2P(pgtab) | PTE_P | PTE_W | PTE_U;
  }
  return &pgtab[PTX(va)];
}
```


### 4-4. Process Manager
구현한 시스템콜들 중 일부를 확인하기 위해선 process manager 개발이 선행되어야 했습니다. exec2를 개발 후 pmanger의 execute 명령어를 실행하는 부분에 기존에 exec 시스템콜로 명령어가 제대로 실행되는지 확인 후 exec2로 교체 한 다음 exec2가 제대로 실행되는지 확인하였습니다. 이후 다시 기존의 exec으로 변경 한 후 memlim 명령어가 제대로 실행되는지 확인하였고 이후 exec2로 변경 후 memlim 명령어가 제대로 실행되는지 확인하여 process manager 개발을 완료화였습니다.

